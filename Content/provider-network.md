# Provider Network Scenario
The basic Neutron configuration uses tenant networks to provide internet access to the instances. All traffic coming from the Compute nodes is routed through the Network node. In this sections we are going to change the basic scenario by introducing Provider networks where instances on Compute nodes are directly attached to the physical network.

#### Flat Provider network scenario
Provider networks are created by the OpenStack administrator and map directly to an existing physical network in the data center. Use flat provider networks to connect instances directly to the external network. 

On the Controller node, edit the ``/etc/neutron/plugin.ini`` initialization file

```
[ml2]
type_drivers = vxlan,flat
...
[ml2_type_flat]
# use flat_networks = * to allow flat networks with arbitrary names
flat_networks = *
...
```

Restart the Neutron service to apply the change
```
# systemctl restart neutron-server
```

On the Network node, configure the bridge mapping in the Open vSwitch Agent initialization file ``/etc/neutron/plugins/ml2/openvswitch_agent.ini``. The bridge mapping maps the external network bridge ``br-ex`` with a reference name for the external network. In this case, we are using ``external`` as reference name but this is simply a string of chars.
```
[ovs]
bridge_mappings = external:br-ex
...
```

and restart the service
```
# systemctl restart neutron-openvswitch-agent
```

Configure the L3 Agent by editing the ``/etc/neutron/l3_agent.ini`` initialization file
```
[DEFAULT]
external_network_bridge =
...
```

Restart neutron-l3-agent for the changes to take effect.
```
# systemctl restart neutron-l3-agent
```

On each Compute node, create an additional bridge interface to connect the nodes to the external network, and allow instances to communicate directly with the external network. For example, if we have a physical interface called ``ens36``, create an external network bridge ``br-ex`` and associate it
```
# vi /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
TYPE=OVSBridge
DEVICETYPE=ovs
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none

# vi /etc/sysconfig/network-scripts/ifcfg-ens36
DEVICE=ens36
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none
```

Restart the network service to enable the bridge
```
# systemctl restart network
```

On each Compute node, configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
bridge_mappings = external:br-ex
...
```

and restart the service
```
# systemctl restart neutron-openvswitch-agent
```

Now we are going to create an external network as a flat provider network and associate it with the configured physical network. Configuring it as a shared network will allow all tenants to create instances directly to it. 

```
# source keystonerc_admin
# neutron net-create public_net \
  --provider:network_type flat \
  --provider:physical_network external \
  --router:external=True \
  --shared

# neutron net-show public_net
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | c6d6e7cd-ba7d-4cfd-acb1-4cf37e2e4aaa |
| mtu                       | 0                                    |
| name                      | public_net                           |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 613b2bc016c5428397b3fea6dc162af1     |
+---------------------------+--------------------------------------+
```

Create the related subnet
```
# source keystonerc_admin
# neutron subnet-create \
 --name public_subnet  public_net 172.120.1.0/24 \
 --enable_dhcp=True \
 --dns-nameserver=8.8.8.8 \
 --allocation_pool start=172.120.1.200,end=172.120.1.210 \
 --gateway=172.120.1.1


Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "172.120.1.200", "end": "172.120.1.210"} |
| cidr              | 172.120.1.0/24                                     |
| dns_nameservers   | 8.8.8.8                                            |
| enable_dhcp       | True                                               |
| gateway_ip        | 172.120.1.1                                        |
| host_routes       |                                                    |
| id                | e52476da-5f0f-4606-8160-3324f58fcd5b               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | public_subnet                                      |
| network_id        | c6d6e7cd-ba7d-4cfd-acb1-4cf37e2e4aaa               |
| subnetpool_id     |                                                    |
| tenant_id         | 613b2bc016c5428397b3fea6dc162af1                   |
+-------------------+----------------------------------------------------+
```

Login as standard user and start a new VM on this network
```
# source keystonerc_demo
# nova boot myinstance \
  --flavor small \
  --image cirros  \
  --key_name demokey \
  --security-groups default \
  --nic net-id=<public_net_id>
```

We see the new VM getting IP address on the provider network
```
# nova list
+--------------------------------------+------------+--------+------------+-------------+--------------------------+
| ID                                   | Name       | Status | Task State | Power State | Networks                 |
+--------------------------------------+------------+--------+------------+-------------+--------------------------+
| 25705301-f197-4ace-bc82-2d537f136755 | myinstance | ACTIVE | -          | Running     | public_net=172.120.1.203 |
+--------------------------------------+------------+--------+------------+-------------+--------------------------+
```

Please note that the external network is a network outside the domain of the OpenStack setup. The IP addressing schema, the default gateway and DNS server are defined by the network administrator of the organization. Only the DHCP server is provided by Neutron. When we are going to spawn an instance it will pick the address from Network node instead of external DHCP server. With external DHCP server Neutron will not have any control over the IP address assigned to instance hence the command output ```nova list``` and actual instance IP will mismatch.  


#### Enable compute metadata in Provider networks scenario
In the Provider networks scenario, the instances are directly attached to the provider external networks, and have an external routers configured as their default gateway. No Neutron routers are used. This means that neutron routers cannot be used to proxy metadata requests from instances to the metadata server, which may result in failures while running the cloud init initial task. However, this issue can be resolved by configuring the Neutron DHCP Agent to proxy metadata requests.

To enable this functionality, edit the ``/etc/neutron/dhcp_agent.ini`` initialization file on the Network node
```
[DEFAULT]
enable_isolated_metadata = True
...
```

#### VLAN based Provider networks scenario
In this section, we are going to create multiple VLAN provider networks that can connect instances directly to the external network. This is required if you want to connect multiple VLAN tagged interfaces, on a single NIC, to multiple provider networks. This example uses a physical network VLAN tagged with a range of VLAN ID 120-121. The Network node and all Compute nodes are connected to the physical network using a physical interface on them called ``ens36``. The switch ports to which these interfaces are connected must be configured to trunk the required VLAN ranges.


On the Controller node, edit the ``/etc/neutron/plugin.ini`` initialization file

```
[ml2]
type_drivers = vxlan,flat,vlan
# use flat_networks = * to allow flat networks with arbitrary names
flat_networks = *
...

[ml2_type_vlan]
network_vlan_ranges=external:120:121
...
```

Restart the Neutron service to apply the change
```
# systemctl restart neutron-server
```

On the Network node, configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
bridge_mappings = external:br-ex
...
```

and restart the service
```
# systemctl restart neutron-openvswitch-agent
```

Configure the L3 Agent by editing the ``/etc/neutron/l3_agent.ini`` initialization file
```
[DEFAULT]
external_network_bridge =
...
```

Restart neutron-l3-agent for the changes to take effect.
```
# systemctl restart neutron-l3-agent
```

On each Compute node, create an additional bridge interface to connect the nodes to the external network, and allow instances to communicate directly with the external network. Since we have a physical interface called ``ens36``, create an external network bridge ``br-ex`` and associate it
```
# vi /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
TYPE=OVSBridge
DEVICETYPE=ovs
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none

# vi /etc/sysconfig/network-scripts/ifcfg-ens36
DEVICE=ens36
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none
```

Restart the network service to enable the bridge
```
# systemctl restart network
```

On each Compute node, configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
bridge_mappings = external:br-ex
...
```

and restart the service
```
# systemctl restart neutron-openvswitch-agent
```

Create the external networks with type VLAN, and associate them to the configured physical network. This example creates two networks: one for VLAN 120, and another for VLAN 121:

```
# source keystonerc_admin
# neutron net-create provider-vlan120 \
  --provider:network_type vlan \
  --router:external true \
  --provider:physical_network external \
  --provider:segmentation_id 120 --shared
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 9389d089-bc21-4b8d-8403-b42692f26569 |
| mtu                       | 0                                    |
| name                      | provider-vlan120                     |
| provider:network_type     | vlan                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  | 120                                  |
| router:external           | True                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 613b2bc016c5428397b3fea6dc162af1     |
+---------------------------+--------------------------------------+

# source keystonerc_admin
# neutron net-create provider-vlan121 \
  --provider:network_type vlan \
  --router:external true \
  --provider:physical_network external \
  --provider:segmentation_id 121 --shared
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 5d3dab45-2f2d-4dae-877d-cc17c03ee901 |
| mtu                       | 0                                    |
| name                      | provider-vlan121                     |
| provider:network_type     | vlan                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  | 121                                  |
| router:external           | True                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 613b2bc016c5428397b3fea6dc162af1     |
+---------------------------+--------------------------------------+
```

Create a number of related subnets and configure them to use the external network. Pay attention to make certain that the external subnet details you have received from network administrator are correctly associated with each VLAN. In this example, VLAN 120 uses subnet 172.120.1.0/24 and VLAN 121 uses 172.121.1.0/24

```
# source keystonerc_admin
# neutron subnet-create \
      --name subnet-provider-120 provider-vlan120 172.120.1.0/24 \
      --enable-dhcp=True \
      --dns-nameserver=8.8.8.8 \
      --allocation_pool start=172.120.1.200,end=172.120.1.210 \
      --gateway=172.120.1.1
Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "172.120.1.200", "end": "172.120.1.210"} |
| cidr              | 172.120.1.0/24                                     |
| dns_nameservers   | 8.8.8.8                                            |
| enable_dhcp       | True                                               |
| gateway_ip        | 172.120.1.1                                        |
| host_routes       |                                                    |
| id                | 0cbba1b7-b364-4b23-a7f5-0a57f10d699e               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | subnet-provider-120                                |
| network_id        | 9389d089-bc21-4b8d-8403-b42692f26569               |
| subnetpool_id     |                                                    |
| tenant_id         | 613b2bc016c5428397b3fea6dc162af1                   |
+-------------------+----------------------------------------------------+

# source keystonerc_admin
# neutron subnet-create \
      --name subnet-provider-121 provider-vlan121 172.121.1.0/24 \
      --enable-dhcp=True \
      --dns-nameserver=8.8.8.8 \
      --allocation_pool start=172.121.1.200,end=172.121.1.210 \
      --gateway=172.121.1.1
Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "172.121.1.200", "end": "172.121.1.210"} |
| cidr              | 172.121.1.0/24                                     |
| dns_nameservers   | 8.8.8.8                                            |
| enable_dhcp       | True                                               |
| gateway_ip        | 172.121.1.1                                        |
| host_routes       |                                                    |
| id                | 80e90654-3d13-4b6d-bdf7-703a14b375f2               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | subnet-provider-121                                |
| network_id        | 5d3dab45-2f2d-4dae-877d-cc17c03ee901               |
| subnetpool_id     |                                                    |
| tenant_id         | 613b2bc016c5428397b3fea6dc162af1                   |
+-------------------+----------------------------------------------------+
```

Login as a standard user and start new VMs on these networks
```
# source keystonerc_demo

# nova boot instance_on_120_vlan \
 --flavor small \
 --image cirros  \
 --key_name demokey \
 --security-groups default \
 --nic net-id=<public_net_id>

# nova boot instance_on_121_vlan \
 --flavor small \
 --image cirros  \
 --key_name demokey \
 --security-groups default \
 --nic net-id=<public_net_id>
```

We see the VMs getting IP address on the providers networks
```
# nova list
+--------------------------------------+----------------------+--------+------------+-------------+--------------------------------+
| ID                                   | Name                 | Status | Task State | Power State | Networks                       |
+--------------------------------------+----------------------+--------+------------+-------------+--------------------------------+
| bc9a443b-4515-4c82-9bec-caaf001b4262 | instance_on_120_vlan | ACTIVE | -          | Running     | provider-vlan120=172.120.1.201 |
| 20462abd-41df-4bbc-8192-57305be4dfcc | instance_on_121_vlan | ACTIVE | -          | Running     | provider-vlan121=172.121.1.201 |
+--------------------------------------+----------------------+--------+------------+-------------+--------------------------------+
