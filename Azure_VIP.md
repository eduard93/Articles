# VIP in Azure

If you're running IRIS in a mirrored configuration for HA in Azure, the question of providing a [Mirror VIP](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set_config#GHA_mirror_set_virtualip) (Virtual IP) becomes relevant. Virtual IP offers a way for downstream systems to interact with IRIS using one IP address. Even when a failover happens, downstream systems can reconnect to the same IP address and continue working.

The main issue, when deploying to Azure, is that an IRIS VIP has a requirement of IRIS being essentially a network admin, per the [docs](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set).

To get HA, IRIS mirror members must be deployed to different availability zones in one subnet (which is possible in Azure as subnets can span several zones). One of the solutions might be load balancers, but they, of course, cost extra, and you need to administrate them.

In this article, I would like to provide a way to configure a Mirror VIP without the using Load Balancers suggested in most other [Azure reference architectures](https://community.intersystems.com/post/intersystems-example-reference-architecture-microsoft-azure-resource-manager-arm). 

# Architecture

![Architecture](https://github.com/eduard93/Articles/assets/5127457/68e25544-1067-412c-adc8-442afbaa8079)

We have a subnet running across two availability zones (I simplify here - of course, you'll probably have public subnets, arbiter in another az, and so on, but this is an absolute minimum enough to demonstrate this approach). Subnet's CIRD is `10.0.0.0/24`, which means it is allocated IPs `10.0.0.1` to `10.0.0.255`. As Azure [reserves](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq#are-there-any-restrictions-on-using-ip-addresses-within-these-subnets) the first four addresses and the last address, we can use `10.0.0.4` to `10.0.0.254`.

We will implement both public and private VIPs at the same time. If you want, you can implement only the private VIP.

# Idea

Virtual Machines in Azure have [Network Interfaces](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-network-interface?tabs=azure-portal). These Network Interfaces have [IP Configurations](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/virtual-network-network-interface-addresses?tabs=nic-address-portal). IP configuration is a combination of Public and/or Private IPs, and it's routed automatically to the Virtual Machine associated with the Network interface. So there is no need to update the routes. What we'll do is, during a mirror failover event, delete the VIP IP configuration from the old primary and create it for a new primary. All operations to do that take 5-20 seconds for Private VIP only, from 5 seconds and up to a minute for a Public/Private VIP IP combination.

# Implementing VIP

1. [Allocate](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/create-public-ip-cli?tabs=create-public-ip-standard%2Ccreate-public-ip-zonal%2Crouting-preference) External IP to use as a public VIP. Skip this step if you want private VIP only. If you do allocate the VIP, it must reside in the same resource group and in the same region and be in all zones with primary and backup. You'll need an External IP name.
2. Decide on a private VIP value. I will use the last available IP: `10.0.0.254`.
3. On each VM, allocate the private VIP IP address on the `eth0:1` network interface.

```
cat << EOFVIP >> /etc/sysconfig/network-scripts/ifcfg-eth0:1
          DEVICE=eth0:1
          ONPARENT=on
          IPADDR=10.0.0.254
          PREFIX=32
          EOFVIP
sudo chmod -x /etc/sysconfig/network-scripts/ifcfg-eth0:1
sudo ifconfig eth0:1 up
```

If you want just to test, run (but it won't survive system restart):

```
sudo ifconfig eth0:1 10.0.0.254
```

Depending on the os you might need to run:

```
ifconfig eth0:1
systemctl restart network
```
4. For each VM, enable [System or User assigned identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations).
5. For each identity, assign the permissions to modify Network Interfaces. To do that create a [custom role](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles), the minimum permission set in that case would be:
```json
{
  "roleName": "custom_nic_write",
  "description": "IRIS Role to assign VIP",
  "assignableScopes": [
    "/subscriptions/{subscriptionid}/resourceGroups/{resourcegroupid}/providers/Microsoft.Network/networkInterfaces/{nicid_primary}",
    "/subscriptions/{subscriptionid}/resourceGroups/{resourcegroupid}/providers/Microsoft.Network/networkInterfaces/{nicid_backup}"
  ],
  "permissions": [
    {
      "actions": [
        "Microsoft.Network/networkInterfaces/write"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ]
}
```

You might also need to grant `Microsoft.Network/networkInterfaces/read`. For non-production environments you might use a [Network Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#network-contributor) system role on the resource group, but that is not a recommended approach as Network Contributor is a very broad role.

6. Each network interface in Azure can have a set of IP configurations. When a current mirror member becomes primary, we'll use a [ZMIRROR](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set_config#GHA_mirror_set_tunable_params_zmirror_routine) callback to delete a VIP IP configuration on another mirror member's network interface and create a VIP IP configuration pointing at itself: 

Here are the Azure CLI commands for both nodes assuming `rg` resource group, `vip` IP configuration, and `my_vip_ip` External IP:

```
az login --identity
az network nic ip-config delete --resource-group rg --name vip --nic-name mirrorb280_z2
az network nic ip-config create --resource-group rg --name vip --nic-name mirrora290_z1 --private-ip-address 10.0.0.254 --public-ip-address my_vip_ip
```
and:
```
az login --identity
az network nic ip-config delete --resource-group rg --name vip --nic-name mirrora290_z1
az network nic ip-config create --resource-group rg --name vip --nic-name mirrorb280_z2 --private-ip-address 10.0.0.254 --public-ip-address my_vip_ip
```

And the same code as a `ZMIRROR` routine:

```objectscript
ROUTINE ZMIRROR

NotifyBecomePrimary() PUBLIC {
    #include %occMessages
    set rg = "rg"
    set config = "vip"
    set privateVIP = "10.0.0.254"
    set publicVIP = "my_vip_ip"

    set nic = "mirrora290_z1"
    set otherNIC = "mirrorb280_z2"
    if ##class(SYS.Mirror).DefaultSystemName() [ "MIRRORB" {
        // we are on mirrorb node, swap
        set $lb(nic, otherNIC)=$lb(otherNIC, nic)
    }

    set rc1 = $zf(-100, "/SHELL", "export", "AZURE_CONFIG_DIR=/tmp", "&&", "az", "login", "--identity")
    set rc2 = $zf(-100, "/SHELL", "export", "AZURE_CONFIG_DIR=/tmp", "&&", "az", "network", "nic", "ip-config", "delete", "--resource-group", rg, "--name", config, "--nic-name", otherNIC)
    set rc3 = $zf(-100, "/SHELL", "export", "AZURE_CONFIG_DIR=/tmp", "&&", "az", "network", "nic", "ip-config", "create", "--resource-group", rg, "--name", config, "--nic-name",      nic,  "--private-ip-address", privateVIP, "--public-ip-address", publicVIP)
    quit 1
}
```

The routine is the same for both mirror members, we just swap the NIC names based on a current mirror member name. You might not need `export AZURE_CONFIG_DIR=/tmp`, but sometimes `az` tries to write credentials into the root home dir, which might fail. Instead of `/tmp`, it's better to use the IRIS user's home subdirectory (or you might not even need that environment variable, depending on your setup). 

And if you want to use Embedded Python, here's Azure Python SDK code:

```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.network import NetworkManagementClient
from azure.mgmt.network.models import NetworkInterface, NetworkInterfaceIPConfiguration, PublicIPAddress

sub_id = "AZURE_SUBSCRIPTION_ID"
client = NetworkManagementClient(credential=DefaultAzureCredential(), subscription_id=sub_id)

resource_group_name = "rg"
nic_name = "mirrora290_z1"
other_nic_name = "mirrorb280_z2"
public_ip_address_name = "my_vip_ip"
private_ip_address = "10.0.0.254"
vip_configuration_name = "vip"


# remove old VIP configuration
nic: NetworkInterface = client.network_interfaces.get(resource_group_name, other_nic_name)
ip_configurations_old_length = len(nic.ip_configurations)
nic.ip_configurations[:] = [ip_configuration for ip_configuration in nic.ip_configurations if
                            ip_configuration.name != vip_configuration_name]

if ip_configurations_old_length != len(nic.ip_configurations):
    poller = client.network_interfaces.begin_create_or_update(
        resource_group_name,
        other_nic_name,
        nic
    )
    nic_info = poller.result()

# add new VIP configuration
nic: NetworkInterface = client.network_interfaces.get(resource_group_name, nic_name)
ip: PublicIPAddress = client.public_ip_addresses.get(resource_group_name, public_ip_address_name)
vip = NetworkInterfaceIPConfiguration(name=vip_configuration_name,
                                      private_ip_address=private_ip_address,
                                      private_ip_allocation_method="Static",
                                      public_ip_address=ip,
                                      subnet=nic.ip_configurations[0].subnet)
nic.ip_configurations.append(vip)

poller = client.network_interfaces.begin_create_or_update(
    resource_group_name,
    nic_name,
    nic
)
nic_info = poller.result()
```

# Initial start

`NotifyBecomePrimary` is also called automatically on system start (after mirror reconnection), but if you want your non-mirrored environments to acquire VIP the same way use [^%ZSTART](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GSTU_customize_startstop) routine:

```objectscript
SYSTEM() PUBLIC {
  if '$SYSTEM.Mirror.IsMember() {
    do NotifyBecomePrimary^ZMIRROR()
  }
  quit 1
}
```

# Conclusion

And that's it! We change IP configuration pointing to a current mirror Primary when the `NotifyBecomePrimary` event happens.
