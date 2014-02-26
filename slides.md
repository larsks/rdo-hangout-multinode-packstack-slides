# RDO Hangout: Multinode OpenStack with Packstack

Lars Kellogg-Stedman <lars@redhat.com>

---

## What are we going to do today?

A multinode OpenStack install using packstack

- One controller
- One network host
- Two compute hosts

---

## What is packstack?

- Single host (`--allinone`) or multinode
- Proof of Concept ("PoC") deployments


- Servers must be pre-installed
- No HA
- Little scalability
- No shared filesystems

---

## Supported platforms

- RHEL
- CentOS
- Fedora

---

## Architecture Overview

![Architecture overview](assets/overview.svg)

---

## Controller host

![Controller host services](assets/controller_host.svg)

---

## Network host

![Network host services](assets/network_host.svg)

---

## Compute host

![Compute host services](assets/compute_host.svg)

---

# Getting started

---

## Install packstack

Make the RDO repositories available:

    # yum install -y http://rdo.fedorapeople.org/rdo-release.rpm

And install `packstack`:

    # yum -y install openstack-packstack

---

## The answers file

You can set all sorts of parameters on the command line, but I like to
generate an "answers" file and edit it:

    # packstack --gen-answer-file packstack-answers.txt


For this hangout, our `packstack-answers.txt` file differs from the
default like this:

    CONFIG_CEILOMETER_INSTALL=n
    CONFIG_NOVA_COMPUTE_HOSTS=10.15.0.2,10.15.0.8
    CONFIG_NEUTRON_SERVER_HOST=10.15.0.7
    CONFIG_NEUTRON_L3_HOSTS=10.15.0.7
    CONFIG_NEUTRON_DHCP_HOSTS=10.15.0.7
    CONFIG_NEUTRON_LBAAS_HOSTS=10.15.0.7
    CONFIG_NEUTRON_METADATA_HOSTS=10.15.0.7
    CONFIG_NEUTRON_OVS_TENANT_NETWORK_TYPE=gre
    CONFIG_NEUTRON_OVS_TUNNEL_RANGES=1000:3000
    CONFIG_NEUTRON_OVS_TUNNEL_IF=eth2

---

# Run packstack

---

# Post-install Configuration

---

Source your `admin` credentials:

    # . /root/keystonrc_admin

Create a disk image:

    glance image-create \
      --copy-from http://download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img \
      --is-public true \
      --container-format bare \
      --disk-format raw \
      --name cirros

---

Create external network:

    # neutron net-create external --router:external=True
    # neutron subnet-create --disable-dhcp external 172.16.13.0/24

---

Create a flavor for testing:

    # nova flavor-create m1.nano auto 128 1 1

This flavor consumes minimal memory and disk so it is better than the
default flavors for testing in constrained environments.

---

Create a non-admin user:

    # keystone tenant-create --name demo
    # keystone user-create --name demo --tenant demo --pass demo

And store the credentials in `/root/keystonerc_demo`:

    export OS_USERNAME=demo
    export OS_TENANT_NAME=demo
    export OS_PASSWORD=demo
    export OS_AUTH_URL=http://10.15.0.7:35357/v2.0/
    export PS1='[\u@\h \W(keystone_demo)]\$ '

---

## Create tenant networks

Use the non-admin user we just created:

    # . /root/keystonerc_demo

Create a private network:

    # neutron net-create net0
    # neutron subnet-create --name net0-subnet0 net0 10.0.0.0/24


Create a router and connect it to the private network and the external
network:

    # neutron router-create extrouter
    # neutron router-gateway-set extrouter external
    # neutron router-interface-add extrouter net0-subnet0


Now we should have something like:

    # neutron net-list
    +--------------------------------------+----------+--------------------------------------------------+
    | id                                   | name     | subnets                                          |
    +--------------------------------------+----------+--------------------------------------------------+
    | 77cafb07-a793-41cb-8a96-58d04408e10d | net0     | f0beab82-0673-40eb-8934-68acc6bd635a 10.0.0.0/24 |
    | e1de0593-73d4-427d-89f6-9c7b0e7c7ef9 | external | 57c65000-0782-40c3-906e-09d9a4ad5113             |
    +--------------------------------------+----------+--------------------------------------------------+

---

## Booting an instance

We'll need the UUID for network `net0` that we created in the previous
step:

    # nova boot --flavor m1.nano --image cirros \
      --nic net-id=77cafb07-a793-41cb-8a96-58d04408e10d test0

---

## Create and assign a floating ip

Allocate a floating ip address from the *external* network:

    # nova floating-ip-create external
    +-------------+-------------+----------+----------+
    | Ip          | Instance Id | Fixed Ip | Pool     |
    +-------------+-------------+----------+----------+
    | 172.16.13.3 | None        | None     | external |
    +-------------+-------------+----------+----------+
    
Assign it to the new instance:

    # nova add-floating-ip test0 172.16.13.3

---

## External network connectivity

In the real world:

![External connectivity](assets/external_connectivity.svg)


In our demo:

    # ip addr add 172.16.13.1/24 dev br-ex
    # iptables -t nat -I POSTROUTING 1 -s 172.16.13.0/24 -j MASQUERADE

