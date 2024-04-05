# VIP in GCP

If you're running IRIS in a mirrored configuration for HA in GCP, the question of providing a [Mirror VIP](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set_config#GHA_mirror_set_virtualip) (Virtual IP) becomes relevant. Virtual IP offers a way for downstream systems to interact with IRIS using one IP address. Even when a failover happens, downstream systems can reconnect to the same IP address and continue working.

The main issue, when deploying to GCP, is that an IRIS VIP has a requirement of IRIS being essentially a network admin, per the [docs](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GHA_mirror_set).

To get HA, IRIS mirror members must be deployed to different availability zones in one subnet (which is possible in GCP as subnets always span the entire region). One of the solutions might be load balancers, but they, of course, cost extra, and you need to administrate them.

In this article, I would like to provide a way to configure a Mirror VIP without the using Load Balancers suggested in most other [GCP reference architectures](https://community.intersystems.com/post/intersystems-iris-example-reference-architectures-google-cloud-platform-gcp). 

# Architecture

![Architecture]()

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




