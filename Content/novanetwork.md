# Configure nova-network FlatDHCP topology
Nova-network is a legacy alternative to network Neutron service for OpenStack platform. If you do not want to split your VMs into isolated groups (tenants), you can choose the Nova-network with FlatDHCP topology or Flat topology. In this case, you will have one big network for all tenants and their VMs.

FlatDHCP topology uses linux bridged networking, with the NAT bridge option. For each compute node, there is a single linux bridge created, the name of which is specified in the Nova configuration file using the option: ``flat_network_bridge=br100``

All the VMs spawned by OpenStack get attached to this dedicated bridge. On each compute node:

* the network bridge interface takes an address from the "flat" network
* a dnsmasq DHCP server process is spawned and listens on the network bridge interface
* the network bridge interface acts as default gateway with NAT functionality for all the VMs running on the compute node

#### Setup FlatDHCP topology
We setup 2 compute node: caldera01 and caldera03 and the controller node: caldera02. Each compute node has two NIC interfaces:

1. ``ens0 IP 10.10.10.X/24``  for Control and External network 
2. ``ens1 IP Unnumbered`` for Tennant network

The controller node caldera02 has only one NIC interface:

1. ``ens0 IP 10.10.10.98/24`` for Control network

Since we are not using Neutron, there is no need for the Network node. Each compute node, has a linux bridged NAT configuration with DHCP enabled and uses the Tenant network to broadcast the DHCP requests when booting the VMs. So, please make sure no other DHCP servers are listening on the Tennant network. In that case, the VMs can get the IP from the external DHCP server instead of nova-network DHCP server leaving the setup in an inconsistent state.

Relevant nova configuration parameters in ``/etc/nova/nova.conf`` for the Compute nodes are:

```
[DEFAULT]
network_manager=nova.network.manager.FlatDHCPManager
auto_assign_floating_ip=False
dhcpbridge_flagfile=/etc/nova/nova.conf
public_interface=ens0
dhcpbridge=/usr/bin/nova-dhcpbridge
flat_network_bridge=br100
flat_network_dns=8.8.8.8
flat_injected=False
flat_interface=ens1
force_dhcp_release=True
dhcp_domain=novalocal
multi_host=True
fixed_range=192.168.100.0/24
floating_range=10.10.10.240/28
```

On the compute node caldera03, the linux bridge ``br100`` is made of:

```
# brctl show
bridge name     bridge id               STP enabled     interfaces
br100           8000.26a6c3b347d1       no              ens1
                                                        vnet0
                                                        vnet1

# ifconfig br100
br100: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.100.3  netmask 255.255.255.0  broadcast 192.168.100.255
        inet6 fe80::fcfa:28ff:fe8e:af4f  prefixlen 64  scopeid 0x20<link>
        ether 26:a6:c3:b3:47:d1  txqueuelen 0  (Ethernet)
        RX packets 132919  bytes 23091506 (22.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 195084  bytes 213329748 (203.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

and the ``ens1`` is the Tennant network interface used by the Compute node. The ``vnet0`` and ``vnet1` are the virtual interfaces used by the VM0 and VM1 on the caldera03 node.

On the other compute node, caldera01, the linux bridge ``br100`` is made of:

```
# brctl show
bridge name     bridge id               STP enabled     interfaces
br100           8000.26a6c3b347d1       no              ens1
                                                        vnet0

# ifconfig br100
br100: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.100.5  netmask 255.255.255.0  broadcast 192.168.100.255
        inet6 fe80::684d:63ff:fed1:50d  prefixlen 64  scopeid 0x20<link>
        ether 7a:c1:c6:d4:b5:60  txqueuelen 0  (Ethernet)
        RX packets 46229  bytes 3412720 (3.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 139635  bytes 200560503 (191.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

The ``vnet0`` is the virtual interface used by the VM3 on the caldera01 node. The two bridges ``br100`` and the related VMs are able to talk each other via the Tennant network interface ``ens1``. Please note, that the ``ens1`` interface can be unnumbered interface, without IP address assigned on both the compute nodes.

The packstack installation file used for this setup is [here] (https://github.com/kalise/rdo_openstack_admin/blob/master/nova-network-flatdhcp/packstack.txt)

#### FlatDHCP variant
In case of a single compute node or isolated compute nodes that are not requested to talk each other,, the Tennant network can be avoided and the ``ens1`` interface is replaced by a ``dummy0`` interface in the ``nova.conf`` file.

```
# vi /etc/nova/nova.conf
flat_interface=dummy0
```

Add the dummy interface which is for the flat DHCP bridge
```
# vi /etc/sysconfig/network-scripts/ifcfg-dummy0
DEVICE=dummy0
BOOTPROTO=none
ONBOOT=yes
TYPE=Ethernet
NM_CONTROLLED=no

# echo "alias dummy0 dummy" > /etc/modprobe.d/dummy.conf 
# ifup dummy0
# systemctl restart openstack-nova
# brctl show
bridge name     bridge id               STP enabled     interfaces
br100           8000.26a6c3b347d1       no              dummy0
                                                        vnet0
```

#### Troubleshooting the FlatDHCP topology
OpenStack upstream documentation reports some guidelines to debug networking issues. See [here] (http://docs.openstack.org/openstack-ops/content/network_troubleshooting.html).

A common issue with FlatDHCP topology is when an instance boots successfully but is not reachable because it failed to obtain an IP address from the DHCP server instance of nova-network. In that case, kill the DHCP server instance and restart the nova-network

```
# ps aux | grep dnsmasq
# killall dnsmasq
# restart openstack-nova-network
```
