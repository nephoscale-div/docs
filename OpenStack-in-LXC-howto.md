# **How to deploy OpenStack under LXC**


# Intro

This howto describes how to deploy full OpenStack (OS for short) environment
under LXC isolation environment. Each container emulates real hardware box that
performs some role of OS entity (like OS controller, OS compute node, OS network
node and etc).

This setup emulates three hardware networks: management network (for OS services
interconnection and administration), tenant network (for private tenants
networks), and external network (this one emulates provider network connected to
internet; from this network tenants will get floating IPs, use this network as
tenant network router gateway and so on).

This howto assumes that all these LXC containers are co-exist on same host. It
is possible to deploy them on different hardware hosts but, configuration of
networks and interconnecting containers, in this case, is out of scope of this
document.

The host operating system assumed in this howto is *Debian 9 (Stretch) x64*.

The operating system inside of LXC containers is *Ubuntu 14.04 (Trusty) x64*
cause OpenStack Liberty version is known to work best on this Ubuntu version
only.

All commands in this howto assumes `root` user if not specifically mentioned
otherwise.


# Host setup

### Host network configuration

1. Install required packages
```
apt install bridge-utils lxc
```

*NOTE: Debian 9 provides LXC version 2 by default. Make sure to install LXCv2 in
different distro.*

2. Adjust `/etc/network/interfaces` and configure bridges that emulate physical
networks:
```
# management network
auto lxcbr0
iface lxcbr0 inet static
	address 10.0.0.1
	netmask 255.255.255.0
	bridge_ports none

# external network
auto lxcbr1
iface lxcbr1 inet static
	address 172.22.22.1
	netmask 255.255.255.0
	bridge_ports none

# tenants network
auto lxcbr2
iface lxcbr2 inet manual
	bridge_ports none
```
Here:

- `lxcbr0` is an emulation of 'management' physical network; inside of LXC
container `eth0` interfaces will be linked with this bridge. `10.0.0.0/24` is
management network, all OS services are interconnected through this network.

- `lxcbr1` is an emulation of 'external' network; inside of LXC containers
`eth1` interfaces will be linked with this bride.

- `lxcbr2` is an emulation of 'tenant' physical network; inside of LXC container
`eth2` interfaces will be linked with this bridge. This network caries packages
of tenant private networks incapsulated into VLAN, VXLAN, GRE and etc (see
Neutron documentation for more info). This howto assumes VXLANs as tenant
network isolation method.

**Description**:

Bridge `lxcbr1` mentions IP from network `172.22.22.0/24`. This IP/network must
be treated as an configuration of **real** provider IP range. It is just a
simulation: think about it like it is a real IP range, not like a 'gray' subnet.

As a result, package forwarding must be enabled on host to allow packages from
LXC containers reach internet (see option `net.ipv4.ip_forward=1` in
`/etc/sysctl.conf`).

Additionally, network address translation must be configured at some point for
both networks: 'management' and 'external'. This could be done either on host
with commands like:

```
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.22.22.0/24 -j MASQUERADE
```

or on LAN gateway connected to internet (refer to router manual for this).

*Note: Any dedicated 'gray' network might be used as wide area network emulation
or even just real IP/network could be used if it is available. Actually, linking
virtual LXC 'external' network with internet is totally depends on local
circumstances and there is no 'all-in-one' solution.*


### Setup LXC

*NOTE: It is not mandatory to adjust default LXC config (it could be a problem
in case if you are planning to have another LXC containers on your host). It is
possible to adjust configs of specific containers to reflect configuration below
(see LXC docs). In this howto default config modified just for conveniece.*

1. Make sure LXC is not using own preconfigured bridge (adjust
`/etc/default/lxc`):
```
...
USE_LXC_BRIDGE="false"
...
```
2. Adjust LXC default config (`/etc/lxc/default.conf`) to contain configuration
and linking with emulated networks configured above:
```
...

lxc.network.type = veth
lxc.network.link = lxcbr0
lxc.network.flags = up
lxc.network.hwaddr = 00:AB:CD:EF:00:xx
lxc.network.name = eth0

lxc.network.type = veth
lxc.network.link = lxcbr1
lxc.network.flags = up
lxc.network.hwaddr = 00:AB:CD:EF:01:xx
lxc.network.name = eth1

lxc.network.type = veth
lxc.network.link = lxcbr2
lxc.network.flags = up
lxc.network.hwaddr = 00:AB:CD:EF:02:xx
lxc.network.name = eth2
```

After this step, new LXC container will have three ethernet NICs inside of it,
linked as described above.


# Instances deployment

Before creating LXC containers make sure that there is enough room for storing
instances. LXC utilizes `/var/lib/lxc` directory for storing containers. There
must be at least 50Gb of free space available at this file system point (it is
possible to use LVM with LXC, refer to related docs on this). This howto assumes
only two OS nodes: controller and compute. Other service boxes (like block
storage, or object storage) might be setup in analogues way.

1. Create instances:
```
lxc-create --name os-controller -t download -- --no-validate --d ubuntu -r trusty -a amd64
lxc-create --name os-compute-01 -t download -- --no-validate --d ubuntu -r trusty -a amd64
```

2. Run instances:
```
lxc-start -n os-controller
lxc-start -n os-compute-01
```

# Configure instances and perform common setup:

1. Attach to instance console:
```
lxc-attach -n os-controller
```

2. Configure instance management network interface (adjust
`/etc/network/interfaces`):
```
auto eth0
iface  eth0 inet manual
  address 10.0.0.11
  netmask 255.255.255.0
  gateway 10.0.0.1
  dns_nameservers 8.8.8.8
```

3. Restart instance and re-attach its console.

4. Update operating system to latest packages:
```
apt-get update && apt-get upgrade
```

5. Install OpenStack common packages:
```
apt-get install software-properties-common
add-apt-repository cloud-archive:liberty
apt-get update && apt-get dist-upgrade
apt-get install python-openstackclient
```

6. Configure names resolution (adjust `/etc/hosts` file):
```
# controller
10.0.0.11       controller

# compute1
10.0.0.31       compute1

# block1 (optional, set this if Cinder will be installed)
10.0.0.41       block1
```

6. Repeat steps 1-5 on `os-compute-01` instance, using `10.0.0.31` as IP address
for network configuration.

# Setup OpenStack

1. Install & configure MySQL server (`os-controller` instance):

    https://docs.openstack.org/liberty/install-guide-ubuntu/environment-sql-database.html

2. Install & configure NoSQL database (`os-controller` instance):

    https://docs.openstack.org/liberty/install-guide-ubuntu/environment-nosql-database.html

3. Install & configure RabbitMQ server (`os-controller` instance):

    https://docs.openstack.org/liberty/install-guide-ubuntu/environment-messaging.html

4. Install & configure Keystone (`os-controller` instance):

    https://docs.openstack.org/liberty/install-guide-ubuntu/keystone.html

5. Install & configure Glance (`os-controller` instance):

    https://docs.openstack.org/liberty/install-guide-ubuntu/glance.html

    *NOTE: It is possible to use separate instance for Glance but in this guide
    Glance is deployed on `os-controller` for simplicity.*

6. Install & configure Nova (`os-controller` & `os-compute-01` instances):

    https://docs.openstack.org/liberty/install-guide-ubuntu/nova.html

7. Install & configure Neutron (`os-controller` & `os-compute-01` instances):

    https://docs.openstack.org/liberty/install-guide-ubuntu/neutron.html

8. Install & configure Horizon (dashboard) (`os-controller` instance):

    https://docs.openstack.org/liberty/install-guide-ubuntu/horizon.html

    *NOTE: At this point you should have minimal working OpenStack environment.*

9. Install & configure Cinder (create & configure separate LXC instance for this
service in same way as described above):

    https://docs.openstack.org/liberty/install-guide-ubuntu/cinder.html

10. Install & configure Ceilometer (`os-controller` instance):

    https://docs.openstack.org/liberty/install-guide-ubuntu/ceilometer.html

11. Install CloudKitty service & dashboard (`os-controller` instance):

    https://docs.openstack.org/cloudkitty/latest/install/install-source.html

12. Configure CloudKitty service & dashboard (`os-controller` instance):

    https://docs.openstack.org/cloudkitty/latest/configuration/index.html

Done.


*P.S.: Use following guide as OpenStack setup handbook:
https://docs.openstack.org/liberty/install-guide-ubuntu/
This guide contains almost everything is needed for setting up base OpenStack
environment.*
