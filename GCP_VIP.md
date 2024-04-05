# VIP in GCP

If you're running IRIS in a mirrored configuration for HA in GCP, the question of providing a [Mirror VIP](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set_config#GHA_mirror_set_virtualip) (Virtual IP) becomes relevant. Virtual IP offers a way for downstream systems to interact with IRIS using one IP address. Even when a failover happens, downstream systems can reconnect to the same IP address and continue working.

The main issue, when deploying to GCP, is that an IRIS VIP has a requirement of IRIS being essentially a network admin, per the [docs](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set).

To get HA, IRIS mirror members must be deployed to different availability zones in one subnet (which is possible in GCP as subnets always span the entire region). One of the solutions might be load balancers, but they, of course, cost extra, and you need to administrate them.

In this article, I would like to provide a way to configure a Mirror VIP without the using Load Balancers suggested in most other [GCP reference architectures](https://community.intersystems.com/post/intersystems-iris-example-reference-architectures-google-cloud-platform-gcp). 

# Architecture

![GCP VIP](https://github.com/eduard93/Articles/assets/5127457/148cac1e-b385-4bad-982f-ebf60ff0dc9b)

We have a subnet running across the region (I simplify here - of course, you'll probably have public subnets, arbiter in another az, and so on, but this is an absolute minimum enough to demonstrate this approach). Subnet's CIRD is `10.0.0.0/24`, which means it is allocated IPs `10.0.0.1` to `10.0.0.255`. As GCP [reserves](https://cloud.google.com/vpc/docs/subnets#unusable-ip-addresses-in-every-subnet) the first and last two addresses, we can use `10.0.0.2` to `10.0.0.253`.

We will implement both public and private VIPs at the same time. If you want, you can implement only the private VIP.

# Idea

Virtual Machines in GCP have [Network Interfaces](https://cloud.google.com/compute/docs/networking/network-overview). These Network Interfaces have [Alias IP Ranges](https://cloud.google.com/compute/docs/reference/rest/v1/instances/updateNetworkInterface) which are private IP addresses. Public IP Addresses can be added by specifying [Access Config](https://cloud.google.com/compute/docs/reference/rest/v1/instances/addAccessConfig)

Network Interfaces configuration is a combination of Public and/or Private IPs, and it's routed automatically to the Virtual Machine associated with the Network interface. So there is no need to update the routes. What we'll do is, during a mirror failover event, delete the VIP IP configuration from the old primary and create it for a new primary. All operations to do that take 5-20 seconds for Private VIP only, from 5 seconds and up to a minute for a Public/Private VIP IP combination.

# Implementing VIP

1. Allocate IP address to use as a public VIP. Skip this step if you want private VIP only.
2. Decide on a private VIP value. I will use `10.0.0.250`.
3. Provision your IRIS Instances with a [service account](https://cloud.google.com/iam/docs/service-account-overview)
- compute.instances.get
- compute.addresses.use
- compute.addresses.useInternal
- compute.instances.updateNetworkInterface
- compute.subnetworks.use

For External VIP you'll also need:
- compute.instances.addAccessConfig
- compute.instances.deleteAccessConfig
- compute.networks.useExternalIp
- compute.subnetworks.useExternalIp
- compute.addresses.list


4. When a current mirror member becomes primary, we'll use a [ZMIRROR](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set_config#GHA_mirror_set_tunable_params_zmirror_routine) callback to delete a VIP IP configuration on another mirror member's network interface and create a VIP IP configuration pointing at itself.

That's it.

```objectscript
ROUTINE ZMIRROR

NotifyBecomePrimary() PUBLIC {
    #include %occMessages
    set sc = ##class(%SYS.System).WriteToConsoleLog("Setting Alias IP instead of Mirror VIP"_$random(100))
    set sc = ##class(%SYS.Python).Import("set_alias_ip")
    quit sc
}
```

And here's `set_alias_ip.py` which must be placed into `mgr\python` directory:

```python
"""
This script adds Alias IP (https://cloud.google.com/vpc/docs/alias-ip) to the VM Network Interface.

You can allocate alias IP ranges from the primary subnet range, or you can add a secondary range to the subnet
and allocate alias IP ranges from the secondary range.
For simplicity, we use the primary subnet range.

Using google cli, gcloud, this action could be performed in this way:
$ gcloud compute instances network-interfaces update <instance_name> --zone=<subnet_zone> --aliases="10.0.0.250/32"

Note that the command for alias removal looks similar - just provide an empty `aliases`:
$ gcloud compute instances network-interfaces update <instance_name> --zone=<subnet_zone> --aliases=""

We leverage Google Compute Engine Metadata API to retrieve <instance_name> as well as <subnet_zone>.

Also note https://cloud.google.com/vpc/docs/subnets#unusable-ip-addresses-in-every-subnet.

Google Cloud uses the first two and last two IPv4 addresses in each subnet primary IPv4 address range to host the subnet.
Google Cloud lets you use all addresses in secondary IPv4 ranges, i.e.:
- 10.0.0.0 - Network address
- 10.0.0.1 - Default gateway address
- 10.0.0.254 - Second-to-last address. Reserved for potential future use
- 10.0.0.255 - Broadcast address

After adding Alias IP, you can check its existence using 'ip' utility:
$ ip route ls table local type local dev eth0 scope host proto 66
local 10.0.0.250
"""

import subprocess
import requests
import re
import time
from google.cloud import compute_v1

ALIAS_IP = "10.0.0.250/32"
METADATA_URL = "http://metadata.google.internal/computeMetadata/v1/"
METADATA_HEADERS = {"Metadata-Flavor": "Google"}
project_path = "project/project-id"
instance_path = "instance/name"
zone_path = "instance/zone"
network_interface = "nic0"
mirror_public_ip_name = "isc-mirror"
access_config_name = "isc-mirror"
mirror_instances = ["isc-primary-001", "isc-backup-001"]


def get_metadata(path: str) -> str:
    return requests.get(METADATA_URL + path, headers=METADATA_HEADERS).text


def get_zone() -> str:
    return get_metadata(zone_path).split('/')[3]


client = compute_v1.InstancesClient()
project = get_metadata(project_path)
availability_zone = get_zone()


def get_ip_address_by_name():
    ip_address = ""
    client = compute_v1.AddressesClient()
    request = compute_v1.ListAddressesRequest(
        project=project,
        region='-'.join(get_zone().split('-')[0:2]),
        filter="name=" + mirror_public_ip_name,
    )
    response = client.list(request=request)
    for item in response:
        ip_address = item.address
    return ip_address


def get_zone_by_instance_name(instance_name: str) -> str:
    request = compute_v1.AggregatedListInstancesRequest()
    request.project = project
    instance_zone = ""
    for zone, response in client.aggregated_list(request=request):
        if response.instances:
            if re.search(f"{availability_zone}*", zone):
                for instance in response.instances:
                    if instance.name == instance_name:
                        return zone.split('/')[1]
    return instance_zone


def update_network_interface(action: str, instance_name: str, zone: str) -> None:
    if action == "create":
        alias_ip_range = compute_v1.AliasIpRange(
            ip_cidr_range=ALIAS_IP,
        )
    nic = compute_v1.NetworkInterface(
        alias_ip_ranges=[] if action == "delete" else [alias_ip_range],
        fingerprint=client.get(
            instance=instance_name,
            project=project,
            zone=zone
        ).network_interfaces[0].fingerprint,
    )
    request = compute_v1.UpdateNetworkInterfaceInstanceRequest(
        project=project,
        zone=zone,
        instance=instance_name,
        network_interface_resource=nic,
        network_interface=network_interface,
    )
    response = client.update_network_interface(request=request)
    print(instance_name + ": " + str(response.status))


def get_remote_instance_name() -> str:
    local_instance = get_metadata(instance_path)
    mirror_instances.remove(local_instance)
    return ''.join(mirror_instances)


def delete_remote_access_config(remote_instance: str) -> None:
    request = compute_v1.DeleteAccessConfigInstanceRequest(
        access_config=access_config_name,
        instance=remote_instance,
        network_interface="nic0",
        project=project,
        zone=get_zone_by_instance_name(remote_instance),
    )
    response = client.delete_access_config(request=request)
    print(response)


def add_access_config(public_ip_address: str) -> None:
    access_config = compute_v1.AccessConfig(
        name = access_config_name,
        nat_i_p=public_ip_address,
    )
    request = compute_v1.AddAccessConfigInstanceRequest(
        access_config_resource=access_config,
        instance=get_metadata(instance_path),
        network_interface="nic0",
        project=project,
        zone=get_zone_by_instance_name(get_metadata(instance_path)),
    )
    response = client.add_access_config(request=request)
    print(response)


# Get another failover member's instance name and zone
remote_instance = get_remote_instance_name()
print(f"Alias IP is going to be deleted at [{remote_instance}]")

# Remove Alias IP from a remote failover member's Network Interface
#
# TODO: Perform the next steps when an issue https://github.com/googleapis/google-cloud-python/issues/11931 will be closed:
# - update google-cloud-compute pip package to a version containing fix (>1.15.0)
# - remove a below line calling gcloud with subprocess.run()
# - uncomment update_network_interface() function
subprocess.run([
    "gcloud",
    "compute",
    "instances",
    "network-interfaces",
    "update",
    remote_instance,
    "--zone=" + get_zone_by_instance_name(remote_instance),
    "--aliases="
])
# update_network_interface("delete",
#                          remote_instance,
#                          get_zone_by_instance_name(remote_instance)


# Add Alias IP to a local failover member's Network Interface
update_network_interface("create",
                         get_metadata(instance_path),
                         availability_zone)


# Handle public IP switching
public_ip_address = get_ip_address_by_name()
if public_ip_address:
    print(f"Public IP [{public_ip_address}] is going to be switched to [{get_metadata(instance_path)}]")
    delete_remote_access_config(remote_instance)
    time.sleep(10)
    add_access_config(public_ip_address)
```
# Demo

Now let's deploy this IRIS architecture into GCP using Terraform and Ansible. If you already running IRIS in GCP or using a different tool, the ZMIRROR script is available [here](https://github.com/eduard93/gcp-infra/blob/main/docker-compose/iris/set_alias_ip.py).

## Tools

We'll need the following tools. As Ansible is Linux only I highly recommend running it on Linux, althrough I confirmed that it works on Windows in WSL2 too.

[gcloud](https://cloud.google.com/sdk/docs/install):
```bash
$ gcloud version
Google Cloud SDK 459.0.0
...
```

[terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli):
```bash
$ terraform version
Terraform v1.6.3
```

[python](https://www.python.org/downloads/):
```bash
$ python3 --version
Python 3.10.12
```

[ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html):
```bash
$ ansible --version
ansible [core 2.12.5]
...
```

[ansible-playbook](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html):
```bash
$ ansible-playbook --version
ansible-playbook [core 2.12.5]
...
```

## WSL2

If you're running in WSL2 on Windows, you'll need to restart ssh agent by running:

```
eval `ssh-agent -s`
```

Also sometimes (when Windows goes to sleep/hibernate and back) the WSL clock is not synced, you might need to sync it explicitly:

```
sudo hwclock -s
```

## Headless servers

If you're runnning a headless server, use `gcloud auth login --no-browser` to authenticate against GCP.

## IaC
We leverage Terraform and store its state in a Cloud Storage. See details below about how this storage is created.

### Define required variables
```bash
$ export PROJECT_ID=<project_id>
$ export REGION=<region> # For instance, us-west1
$ export TF_VAR_project_id=${PROJECT_ID}
$ export TF_VAR_region=${REGION}
$ export ROLE_NAME=MyTerraformRole
$ export SA_NAME=isc-mirror
```
**Note**: If you'd like to add Public VIP which exposes IRIS Mirror ports publicly (it's not recommended) you could enable it with:
```bash
$ export TF_VAR_enable_mirror_public_ip=true

```

### Prepare Artifact Registry
It's [recommended](https://cloud.google.com/container-registry/docs/advanced-authentication) to leverage Google Artifact Registry instead of Container Registry. So let's create registry first:
```bash
$ cd <root_repo_dir>/terraform
$ cat ${SA_NAME}.json | docker login -u _json_key --password-stdin https://${REGION}-docker.pkg.dev
$ gcloud artifacts repositories create --repository-format=docker --location=${REGION} intersystems
```

### Prepare Docker images
Let's assume that VM instances don't have an access to ISC container repository. But you personally do have and at the same do not want to put your personal credentials on VMs.

In that case you can pull IRIS Docker images from ISC container registry and push them to Google container registry where VMs have an access to:
```bash
$ docker login containers.intersystems.com
$ <Put your credentials here>

$ export IRIS_VERSION=2023.2.0.221.0

$ cd docker-compose/iris
$ docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/intersystems/iris:${IRIS_VERSION} .

$ for IMAGE in webgateway arbiter; do \
    docker pull containers.intersystems.com/intersystems/${IMAGE}:${IRIS_VERSION} \
    && docker tag containers.intersystems.com/intersystems/${IMAGE}:${IRIS_VERSION} ${REGION}-docker.pkg.dev/${PROJECT_ID}/intersystems/${IMAGE}:${IRIS_VERSION} \
    && docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/intersystems/${IMAGE}:${IRIS_VERSION}; \
done

$ docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/intersystems/iris:${IRIS_VERSION}
```

### Put IRIS license
Put IRIS license key file, `iris.key` to `<root_repo_dir>/docker-compose/iris/iris.key`. Note that a license has to support Mirroring.

### Create Terraform Role
This role will be used by Terraform for managing needed GCP resources:

```bash
$ cd <root_repo_dir>/terraform/
$ gcloud iam roles create ${ROLE_NAME} --project ${PROJECT_ID} --file=terraform-permissions.yaml
```
**Note**: use `update` for later usage:
```bash
$ gcloud iam roles update ${ROLE_NAME} --project ${PROJECT_ID} --file=terraform-permissions.yaml
```

### Create Service Account with Terraform role
```bash
$ gcloud iam service-accounts create ${SA_NAME} \
    --description="Terraform Service Account for ISC Mirroring" \
    --display-name="Terraform Service Account for ISC Mirroring"

$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role=projects/${PROJECT_ID}/roles/${ROLE_NAME}
```

### Generate Service Account key
Generate Service Account key and store its value in a certain environment variable:
```bash
$ gcloud iam service-accounts keys create ${SA_NAME}.json \
    --iam-account=${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com

$ export GOOGLE_APPLICATION_CREDENTIALS=<absolute_path_to_root_repo_dir>/terraform/${SA_NAME}.json
```

### Generate SSH keypair
Store a private part locally as `.ssh/isc_mirror` and make it visible for `ssh-agent`. Put a public part to a file [isc_mirror.pub](../terraform/templates/isc_mirror.pub):
```bash
$ ssh-keygen -b 4096 -C "isc" -f ~/.ssh/isc_mirror
$ ssh-add  ~/.ssh/isc_mirror
$ ssh-add -l # Check if 'isc' key is present
$ cp ~/.ssh/isc_mirror.pub <root_repo_dir>/terraform/templates/
```

### Create Cloud Storage
Cloud Storage is used for storing [Terraform state remotely](https://developer.hashicorp.com/terraform/language/state/remote). You could take a look at [Store Terraform state in a Cloud Storage bucket](https://cloud.google.com/docs/terraform/resource-management/store-state) as an example.

**Note**: created Cloud Storage will have a name like `isc-mirror-demo-terraform-<project_id>`:
```bash
$ cd <root_repo_dir>/terraform-storage/
$ terraform init
$ terraform plan
$ terraform apply
```

### Create resources with Terraform
```bash
$ cd <root_repo_dir>/terraform/
$ terraform init -backend-config="bucket=isc-mirror-demo-terraform-${PROJECT_ID}"
$ terraform plan
$ terraform apply
```
**Note 1**: Four virtual machines will be created. Only one of them has a public IP address and plays a role of bastion host. This machine is called `isc-client-001`. You can find a public IP of `isc-client-001` instance by running the following command:
```bash
$ export ISC_CLIENT_PUBLIC_IP=$(gcloud compute instances describe isc-client-001 --zone=${REGION}-c --format=json | jq -r '.networkInterfaces[].accessConfigs[].natIP')
```

**Note 2**: Sometimes Terraform fails with errors like:
```bash
Failed to connect to the host via ssh: kex_exchange_identification: Connection closed by remote host...
```
In that case try to clean a local `~/.ssh/known_hosts` file:
```bash
$ for IP in ${ISC_CLIENT_PUBLIC_IP} 10.0.0.{3..6}; do ssh-keygen -R "[${IP}]:2180"; done
```
and then repeat `terraform apply`.


## Quick test
### Access to IRIS mirror instances with SSH
All instances, except `isc-client-001`, are created in a private network to increase a security level. But you can access them using [SSH ProxyJump](https://goteleport.com/blog/ssh-proxyjump-ssh-proxycommand/) feature. Get the `isc-client-001` public IP first:
```bash
$ export ISC_CLIENT_PUBLIC_IP=$(gcloud compute instances describe isc-client-001 --zone=${REGION}-c --format=json | jq -r '.networkInterfaces[].accessConfigs[].natIP')
```
Then connect to, for example, `isc-primary-001` with a private SSH key. Note that we use a custom SSH port, `2180`:
```bash
$ ssh -i ~/.ssh/isc_mirror -p 2180 isc@10.0.0.3 -o ProxyJump=isc@${ISC_CLIENT_PUBLIC_IP}:2180
```
After connection, let's check that Primary mirror member has Alias IP:
```bash
[isc@isc-primary-001 ~]$ ip route ls table local type local dev eth0 scope host proto 66
local 10.0.0.250

[isc@isc-primary-001 ~]$ ping -c 1 10.0.0.250
PING 10.0.0.250 (10.0.0.250) 56(84) bytes of data.
64 bytes from 10.0.0.250: icmp_seq=1 ttl=64 time=0.049 ms
```

### Access to IRIS mirror instances Management Portals
To open mirror instances Management Portals located in a private network, we leverage [SSH Socks Tunneling](https://goteleport.com/blog/ssh-tunneling-explained/).

Let's connect to `isc-primary-001` instance. Note that a tunnel will live in a background after the next command:
```bash
$ ssh -f -N  -i ~/.ssh/isc_mirror -p 2180 isc@10.0.0.3 -o ProxyJump=isc@${ISC_CLIENT_PUBLIC_IP}:2180 -L 8080:10.0.0.3:8080
```
Port 8080, instead of a familiar 52773, is used because we start IRIS with a dedicated WebGateway running on port 8080.

After successful connection, open [http://127.0.0.1:8080/csp/sys/UtilHome.csp](http://127.0.0.1:8080/csp/sys/UtilHome.csp) in a browser. You should see a Management Portal. Credentials are typical: `_system/SYS`.

The same approach works for all instances: primary (10.0.0.3), backup (10.0.0.4) and arbiter (10.0.0.5). Just make an SSH connection to them first.

### Test
Let's connect to `isc-client-001`:
```bash
$ ssh -i ~/.ssh/isc_mirror -p 2180 isc@${ISC_CLIENT_PUBLIC_IP}
```

Check Primary mirror member's Management Portal availability on Alias IP address:
```bash
$ curl -s -o /dev/null -w "%{http_code}\n" http://10.0.0.250:8080/csp/sys/UtilHome.csp
200
```
Let's connect to `isc-primary-001` on another console:
```bash
$ ssh -i ~/.ssh/isc_mirror -p 2180 isc@10.0.0.3 -o ProxyJump=isc@${ISC_CLIENT_PUBLIC_IP}:2180
```
And switch the current Primary instance off. Note that IRIS as well as its WebGateway is running in Docker:
```bash
[isc@isc-primary-001 ~]$ docker-compose -f /isc-mirror/docker-compose.yml down
```
Let's check mirror member's Management Portal availability on Alias IP address again from `isc-client-001`:
```bash
[isc@isc-client-001 ~]$ curl -s -o /dev/null -w "%{http_code}\n" http://10.0.0.250:8080/csp/sys/UtilHome.csp
200
```
It should work as Alias IP was moved to `isc-backup-001` instance:
```bash
$ ssh -i ~/.ssh/isc_mirror -p 2180 isc@10.0.0.4 -o ProxyJump=isc@${ISC_CLIENT_PUBLIC_IP}:2180
[isc@isc-backup-001 ~]$ ip route ls table local type local dev eth0 scope host proto 66
local 10.0.0.250
```
**TODO** - describe how to return the former primary instance back after failover.

## Cleanup

### Remove infrastructure
```bash
$ cd <root_repo_dir>/terraform/
$ terraform init -backend-config="bucket=isc-mirror-demo-terraform-${PROJECT_ID}"
$ terraform destroy
```

### Remove Artifact Registry
```bash
$ cd <root_repo_dir>/terraform
$ cat ${SA_NAME}.json | docker login -u _json_key --password-stdin https://${REGION}-docker.pkg.dev

$ for IMAGE in iris webgateway arbiter; do \
    gcloud artifacts docker images delete ${REGION}-docker.pkg.dev/${PROJECT_ID}/intersystems/${IMAGE}
done
$ gcloud artifacts repositories delete intersystems --location=${REGION}
```

### Remove Cloud Storage
Remove Cloud Storage where Terraform stores its state. In our case, it's a `isc-mirror-demo-terraform-<project_id>`.


### Remove Terraform Role
Remove Terraform Role created in [Create Terraform Role](#create-terraform-role).

# Conclusion

And that's it! We change networking configuration pointing to a current mirror Primary when the NotifyBecomePrimary event happens.

Author would like to thank Mikhail Khomenko, Vadim Aniskin, and Evgeny Shvarov for the Community Ideas Program which made this article possible.
