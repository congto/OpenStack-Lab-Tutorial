# Neutron Networking Service
OpenStack administrators can configure rich network topologies by creating and configuring networks and subnets. In particular, OpenStack networking supports each tenant having multiple private networks, and allows tenants to choose their own IP addressing space, even if those IP addresses overlap with those used by other tenants. This enables advanced cloud networking use cases, such as building multitiered web applications and allowing applications to be migrated to the cloud without changing IP addresses.

OpenStack networking uses the concept of a plug-in, which is a pluggable back-end implementation of the OpenStack networking API. A plug-in can use a variety of technologies. Some OpenStack networking plug-ins might use basic Linux networking, while others might use more advanced technologies, such as Open vSwitch, to provide similar benefits.

To enable OpenStack use Neutron for networking, on the Controller node, create the Keystone user and network service
```
# source /root/keystonerc_admin
# openstack-db --init --service neutron --password <password> --rootpw <password>
# keystone service-create --name neutron --type network --description 'Neutron Networking Service'
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |    Neutron Networking Service    |
|   enabled   |               True               |
|      id     | fe006cc01d234e93a308933d60c396f2 |
|     name    |             neutron              |
|     type    |             network              |
+-------------+----------------------------------+

# keystone endpoint-create \
--service network \
--publicurl http://controller:9696 \
--adminurl http://controller:9696 \
--internalurl http://controller:9696

# keystone user-create --name neutron --pass <password>
# keystone user-role-add --user neutron --role admin --tenant services
```

The basic network implementation in OpenStack is made of a self-service virtual data center infrastructure permitting regular users to manage one or more virtual networks within a project. Connectivity to the external networks such as Internet is provided via the physical network infrastructure. Following concepts are introduced:

  * **Tenant networks**: networks providing connectivity to instances whithin a project. Regular users can manage project networks with the allocation that an administrator defines for for them. Tenant networks can use VLAN, GRE, or VXLAN transport methods depending on the allocation. Tenant networks generally use private IP address ranges and lack connectivity to external networks. IP addresses on the project networks are private IP space within the project and for this reason, they can overlap between different projects. An embedded DHCP service assignes the IP addresses to the Virtual Machines within the project.

  * **External network**: networks providing connectivity to external networks such as the Internet. Only administrative users can manage external networks because they use the physical network infrastructure. External networks can use Flat or VLAN transport methods depending on the physical network infrastructure and generally use public IP address ranges. A flat network essentially uses the untagged frames. Similar to physical networks, only one flat network can exist. In most cases, production deployments should use VLAN transport for external networks instead of a single flat network.

  * **Routers**: typically connect tenant and external networks by implementing source NAT to provide outbound external connectivity for instances on tenant networks. Each router uses an IP address in the external network allocation for source NAT. Routers also use destination NAT to provide inbound external connectivity for instances on tenant networks. The IP addresses on routers that provide inbound external connectivity for instances on tenant networks are refered as floating IP addresses. Routers can also connect tenant networks that belong to the same project.

  * **Provider networks**: networks providing connectivity to instances by mapping directly to an existing physical network in the data center. Provider networks generally offer simplicity, performance, and reliability at the cost of flexibility. Unlike tenant networks, only administrators can manage provider networks because they require configuration of physical network infrastructure. Also, provider networks lack the concept of fixed and floating IP addresses because they only handle layer-2 connectivity for the instances running on Compute nodes. Network types for provider networks are flat (untagged) and VLAN (tagged). It is possible to allow provider networks to be shared among tenants as part of the network creation process.

### Setup Neutron networking
Install and configure Neutron services as follow

|Service|Configuration File(s)|Host Role
|-------|---------------------|---------------------|
|Neutron Server|neutron.conf, ml2_conf.ini|Controller Node|
|Neutron Open vSwitch Agent|openvswitch_agent.ini|Network Node, Compute Node|
|Neutron L3 Agent|l3_agent.ini|Network Node|
|Neutron DHCP Agent|dhcp_agent.ini|Network Node|
|Neutron Metadata Agent|metadata_agent.ini|Network Node|

On the Controller node, install the Neutron packages
```
# yum -y install openstack-neutron
# yum -y install openstack-neutron-ml2
```

Configure the Neutron service by editing the ``/etc/neutron/neutron.conf`` configuration file
```
# vi /etc/neutron/neutron.conf
[DEFAULT]
...
verbose = True
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
dhcp_agent_notification = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
auth_strategy = keystone
rpc_backend = rabbit
...
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = <service password>

[database]
connection = mysql://neutron:<password>@controller/neutron_ml2

[nova]
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = nova
password = <service password>

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = <rabbit password>
```

Configure the ML2 plugin by editing the ``/etc/neutron/plugin.ini`` initialization file.
```
# ll /etc/neutron/plugin.ini
lrwxrwxrwx. 1 root root 37 Mar 14 22:55 /etc/neutron/plugin.ini -> /etc/neutron/plugins/ml2/ml2_conf.ini

# ll /etc/neutron/plugins/ml2/ml2_conf.ini
-rw-r-----. 1 root neutron 5252 Jul 29 01:34 /etc/neutron/plugins/ml2/ml2_conf.ini

# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = flat, vxlan, vlan, gre
tenant_network_types = vxlan, vlan, gre
mechanism_drivers = openvswitch, l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = physnet
# use flat_networks = * to allow flat networks with arbitrary names

[ml2_type_vxlan]
vni_ranges = 1001:2000
vxlan_group =239.1.1.2

[ml2_type_vlan]
network_vlan_ranges = physnet:1001:2000

[ml2_type_gre]
tunnel_id_ranges =1:1000

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_ipset = True
```

Start and enable the Neutron service
```
# systemctl start neutron-server 
# systemctl enable neutron-server 
```

On the Controller node, update Nova compute service to use the Neutron service by adding the following lines to the ``/etc/neutron/nova.conf`` configuration file

```
# vi /etc/nova/nova.conf
[DEFAULT]
...
network_api_class=nova.network.neutronv2.api.API
security_group_api=neutron

[neutron]
url=http://controller:9696
auth_strategy=keystone
admin_auth_url=http://controller:35357/v2.0
admin_tenant_name=service
admin_username=neutron
admin_password= <service password>
```

and restart the Nova API service
```
# systemctl restart openstack-nova-api
```

On the Network node, install the Neutron packages
```
# yum install -y openstack-neutron
# yum install -y openstack-neutron-openvswitch
```

And configure the Neutron service by editing the ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
allow_overlapping_ips = True
rpc_backend = rabbit

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = <service password>

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = <password>
```

Configure the Neutron Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
int_peer_patch_port = patch-tun
tun_peer_patch_port = patch-int
enable_tunneling = True
local_ip = LOCAL_TUNNEL_INTERFACE_IP_ADDRESS
bridge_mappings = physnet:br-ex

[agent]
tunnel_types = vxlan, vlan, gre
vxlan_udp_port = 4789
enable_distributed_routing = False
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

Configure the Neutron L3 Agent by editing the ``/etc/neutron/l3_agent.ini`` initialization file
```
[DEFAULT]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
external_network_bridge = br-ex
enable_metadata_proxy = True
metadata_port = 9697
agent_mode = legacy
```

Configure the Neutron DHCP Agent by editing the ``/etc/neutron/dhcp_agent.ini`` initialization file
```
[DEFAULT]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
ovs_integration_bridge = br-int
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
enable_isolated_metadata = True
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
```

Configure the Neutron Metadata Agent by editing the ``/etc/neutron/metadata_agent.ini`` initialization file
```
[DEFAULT]
verbose = True
nova_metadata_ip = controller
# Nova service on Compute nodes must use the same shared secret
metadata_proxy_shared_secret = <metadata shared secret>
```

Finally, start and enable the Neutron agents
```
# systemctl start neutron-openvswitch-agent
# systemctl start neutron-l3-agent
# systemctl start neutron-dhcp-agent
# systemctl start neutron-metadata-agent

# systemctl enable neutron-openvswitch-agent
# systemctl enable neutron-l3-agent
# systemctl enable neutron-dhcp-agent
# systemctl enable neutron-metadata-agent
```

On all the Compute nodes, install the Neutron package
```
# yum install -y openstack-neutron-openvswitch
```

and configure Neutron by editing the ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
allow_overlapping_ips = True
rpc_backend = rabbit

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = <service password>

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = <password>
```

Configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
int_peer_patch_port = patch-tun
tun_peer_patch_port = patch-int
enable_tunneling = True
local_ip = LOCAL_TUNNEL_INTERFACE_IP_ADDRESS
# uncomment when compute node is directly attached to external network
# bridge_mappings = physnet:br-ex

[agent]
tunnel_types = vxlan, vlan, gre
vxlan_udp_port = 4789
enable_distributed_routing = False
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

Finally, start and enable the Neutron agents
```
# systemctl start neutron-openvswitch-agent
# systemctl enable neutron-openvswitch-agent
```

On all the Compute nodes, update Nova compute service to use the Neutron service by adding the following lines to the ``/etc/neutron/nova.conf`` configuration file

```
[DEFAULT]
...
network_api_class=nova.network.neutronv2.api.API
security_group_api=neutron
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver=nova.virt.firewall.NoopFirewallDriver
metadata_listen=0.0.0.0
metadata_host=controller
vif_plugging_is_fatal=True
vif_plugging_timeout=300

[neutron]
service_metadata_proxy=True
#Metadata Agent on Network node must use the same shared secret
metadata_proxy_shared_secret = <metadata shared secret>
url=http://controller:9696
auth_strategy=keystone
admin_auth_url=http://controller:35357/v2.0
admin_tenant_name=service
admin_username=neutron
admin_password= <service password>
default_tenant_id=default
```

and restart the Nova service
```
# systemctl restart openstack-nova-api
# systemctl restart openstack-nova-compute
```

Check the list of Agents
```
# neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 14dbe6a2-7e8a-48ff-80b6-6dac6c6220a0 | DHCP agent         | network   | :-)   | True           | neutron-dhcp-agent        |
| 21fda60f-739e-4c2c-8235-a90847d6c347 | Open vSwitch agent | compute01 | :-)   | True           | neutron-openvswitch-agent |
| 55cc04e9-b0a9-4c26-a712-6dcab8ef2351 | Open vSwitch agent | compute02 | :-)   | True           | neutron-openvswitch-agent |
| 75968236-ff9c-44cb-a982-f93bcded63f4 | Loadbalancer agent | network   | :-)   | True           | neutron-lbaas-agent       |
| 98e68a68-1a73-47e7-938c-d3e0d0f36173 | Metadata agent     | network   | :-)   | True           | neutron-metadata-agent    |
| 9fb1d4f9-4a34-4d70-8823-a5ed17124618 | Open vSwitch agent | network   | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```
