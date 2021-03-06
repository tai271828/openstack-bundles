# Basic OpenStack Cloud

*DEV/TEST ONLY*: This unstable, development example bundle deploys a basic OpenStack Cloud (Ussuri with Ceph Octopus) on Ubuntu 20.04 LTS (Focal), providing Dashboard, Compute, Network, Block Storage, Object Storage, Identity and Image services.  See also: [Stable Bundles](https://jujucharms.com/u/openstack-charmers).

## Requirements

This example bundle is designed to run on bare metal using Juju 2.x with [MAAS][] (Metal-as-a-Service); you will need to have setup a [MAAS][] deployment with a minimum of 4 physical servers prior to using this bundle.

Certain configuration options within the bundle may need to be adjusted prior to deployment to fit your particular set of hardware. For example, network device names and block device names can vary, and passwords should be yours.

For example, a section similar to this exists in the bundle.yaml file.  The third "column" are the values to set.  Some servers may not have eno2, they may have something like eth2 or some other network device name.  This needs to be adjusted prior to deployment.  The same principle holds for osd-devices.  The third column is a whitelist of devices to use for Ceph OSDs.  Adjust accordingly by editing bundle.yaml before deployment.

```
variables:
  openstack-origin:    &openstack-origin     distro-proposed
  data-port:           &data-port            br-ex:eno2
  worker-multiplier:   &worker-multiplier    0.25
  osd-devices:         &osd-devices          /dev/sdb /dev/vdb
```

Servers should have:

 - A minimum of 8GB of physical RAM.
 - Enough CPU cores to support your capacity requirements.
 - Two disks (identified by /dev/sda and /dev/sdb); the first is used by MAAS for the OS install, the second for Ceph storage.
 - Two cabled network ports on eno1 and eno2 (see below).

Servers should have two physical network ports cabled; the first is used for general communication between services in the Cloud, the second is used for 'public' network traffic to and from instances (North/South traffic) running within the Cloud.

## Components

 - 3 Nodes for Nova Compute and Ceph, with RabbitMQ, MySQL, Keystone, Glance, Neutron, OVN, Nova Cloud Controller, Ceph RADOS Gateway, Cinder and Horizon under LXC containers.

All physical servers (not LXC containers) will also have NTP installed and configured to keep time in sync.

## Deployment

With a Juju controller bootstrapped on a MAAS cloud with no network spaces
defined, a basic non-HA cloud can be deployed with the following command:

    juju deploy bundle.yaml

When network spaces exist in the MAAS cluster, it is necessary to clarify
and define the network space(s) to which the charm applications will deploy.
This can be done with an overlay bundle.  An example overlay yaml file is
provided, which most likely needs to be edited (before deployment) to
represent the intended network spaces in the existing MAAS cluster.  Example
usage:

    juju deploy bundle.yaml --overlay openstack-base-spaces-overlay.yaml

## Scaling

Nova Compute and Ceph services are designed to be horizontally scalable.

To horizontally scale Nova Compute:

    juju add-unit nova-compute # Add one more unit
    juju add-unit -n5 nova-compute # Add 5 more units

To horizontally scale Ceph:

    juju add-unit ceph-osd # Add one more unit
    juju add-unit -n50 ceph-osd # add 50 more units

**Note:** Ceph can be scaled alongside Nova Compute by adding units using the --to option:

    juju add-unit --to <machine-id-of-compute-service> ceph-osd

**Note:** Other services in this bundle can be scaled in-conjunction with the hacluster charm to produce scalable, highly avaliable services - that will be covered in a different bundle.

## Ensuring it's working

To ensure your cloud is functioning correctly, download this bundle and then run through the following sections.

All commands are executed from within the expanded bundle.

### Install OpenStack client tools

In order to configure and use your cloud, you'll need to install the appropriate client tools:

    sudo snap install openstackclients

### Accessing the cloud

Check that you can access your cloud from the command line:

    source openrc
    openstack catalog list

You should get a full listing of all services registered in the cloud which should include identity, compute, image and network.

### Configuring an image

In order to run instances on your cloud, you'll need to upload an image to boot instances:

    curl https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img | \
        openstack image create --public --container-format=bare \
            --disk-format=qcow2 focal

Images for other architectures can be obtained from [Ubuntu Cloud Images][].  Be sure to use the appropriate image for the cpu architecture.

**Note:** for ARM 64-bit (arm64) guests, you will also need to configure the image to boot in UEFI mode:

    curl http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-arm64.img | \
        openstack image create --public --container-format=bare \
            --disk-format=qcow2 --property hw_firmware_type=uefi focal

### Configure networking

For the purposes of a quick test, we'll setup an 'external' network and shared
router ('provider-router') which will be used by all tenants for public access
to instances:

for example (for a private cloud):

    openstack network create --external --provider-network-type flat \
        --provider-physical-network physnet1 ext_net

    openstack subnet create --subnet-range 192.0.2.0/24 --no-dhcp \
        --gateway 192.0.2.1 --network ext_net \
        --allocation-pool start=192.0.2.10,end=192.0.2.254 ext

You'll need to adapt the parameters for the network configuration that eno2 on
all the servers is connected to; in a public cloud deployment these ports would
be connected to a publicly addressable part of the Internet.

We'll also need an 'internal' network for the admin user which instances are
actually connected to:

    openstack network create internal

    openstack subnet create --network internal \
        --subnet-range 198.51.100.0/24 \
        --dns-nameserver 8.8.8.8 \
        internal_subnet

    openstack router create provider-router

    openstack router set --external-gateway ext_net provider-router

    openstack router add subnet provider-router internal_subnet

Neutron provides a wide range of configuration options; see the [OpenStack Neutron][] documentation for more details.

### Configuring a flavor

Starting with the OpenStack Newton release, default flavors are no longer created at install time. You therefore need to create at least one machine type before you can boot an instance:

    openstack flavor create --ram 2048 --disk 20 --ephemeral 20 m1.small

### Booting an instance

First generate a SSH keypair so that you can access your instances once you've booted them:

    mkdir -p ~/.ssh
    touch ~/.ssh/id_rsa_cloud
    chmod 600 ~/.ssh/id_rsa_cloud
    openstack keypair create mykey > ~/.ssh/id_rsa_cloud

**Note:** you can also upload an existing public key to the cloud rather than generating a new one:

    openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

You can now boot an instance on your cloud:

    openstack server create --image focal --flavor m1.small --key-name mykey \
        --network internal focal-test

### Attaching a volume

First, create a 10G volume in cinder:

    openstack volume create --size=10 <name-of-volume>

then attach it to the instance we just booted:

    openstack server add volume focal-test <name-of-volume>

The attached volume will be accessible once you login to the instance (see below).  It will need to be formatted and mounted!

### Accessing your instance

In order to access the instance you just booted on the cloud, you'll need to
assign a floating IP address to the instance:

    FIP=$(openstack floating ip create -f value -c floating_ip_address ext_net)
    openstack server add floating ip focal-test $FIP

and then allow access via SSH (and ping) - you only need to do these steps
once:

    PROJECT_ID=$(openstack project list -f value -c ID \
	       --domain admin_domain)

    SECGRP_ID=$(openstack security group list --project $PROJECT_ID \
        | awk '/default/{print$2}')

    openstack security group rule create $SECGRP_ID \
        --protocol icmp --ingress --ethertype IPv4

    openstack security group rule create $SECGRP_ID \
        --protocol icmp --ingress --ethertype IPv6

    openstack security group rule create $SECGRP_ID \
        --protocol tcp --ingress --ethertype IPv4 --dst-port 22

    openstack security group rule create $SECGRP_ID \
        --protocol tcp --ingress --ethertype IPv6 --dst-port 22

After running these commands you should be able to access the instance:

    ssh ubuntu@$FIP

## What next?

Configuring and managing services on an OpenStack cloud is complex; take a look a the [OpenStack Admin Guide][] for a complete reference on how to configure an OpenStack cloud for your requirements.

## Useful Cloud URLs

 - OpenStack Dashboard: http://openstack-dashboard_ip/horizon

[MAAS]: http://maas.ubuntu.com/docs
[Simplestreams]: https://launchpad.net/simplestreams
[OpenStack Neutron]: http://docs.openstack.org/admin-guide-cloud/content/ch_networking.html
[OpenStack Admin Guide]: http://docs.openstack.org/user-guide-admin/content
[Ubuntu Cloud Images]: http://cloud-images.ubuntu.com/focal/current/
