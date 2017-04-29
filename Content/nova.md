# Nova Computing Service
The OpenStack Compute service allows to control an Infrastructure-as-a-Service (IaaS) Cloud Computing platform. It provides control over instances and networks, and allows to manage the cloud through users and projects. The Nova Compute Service in OpenStack does not include virtualization software. Instead, it defines drivers that interact with underlying virtualization mechanisms running on host operating systems, and exposes functionality over a web-based API.

The Compute Service provides:

1. **openstack-nova-api**. Provides the OpenStack Compute API service. The Controller node hosts an instance of the service and shoud be pointed to by the Identity service endpoint definition for the Compute service.

2. **openstack-nova-compute**. Provides the OpenStack Compute service.

3. **openstack-nova-scheduler**. Provides the Compute scheduler service. The scheduler handles scheduling of requests made to the API across all the available Compute nodes.

4. **openstack-nova-conductor**. Handles database requests made by Compute nodes, ensuring that individual Compute node do not require direct database access. 

## Implementing the Nova service
The Controller node is the node that runs most of the Nova services, especially the nova-scheduler, which coordinates the activities of the various Nova services. The Compute node runs the virtualization software to launch and manage instances for OpenStack.

Install and configure the Nova components on the Controller node
```
# yum install -y https://www.rdoproject.org/repos/rdo-release.rpm
# yum install -y openstack-nova
```

Source the admin user and setup the Nova service
```
# source /root/keystonerc_admin
# openstack-db --init --service nova --password <password> --rootpw <password>
```

Create the nova user and the services. 
```
# keystone user-create --name compute --pass <password>
# keystone user-role-add --user compute --role admin --tenant services
# keystone service-create --name compute --type compute --description "Openstack Compute Service"
# keystone service-list
+----------------------------------+------------+---------------+---------------------------------+
|                id                |    name    |      type     |           description           |
+----------------------------------+------------+---------------+---------------------------------+
| 12d24a91d5a54add9fd684ed94f205dd |  keystone  |    identity   |    OpenStack Identity Service   |
| 57cc90c8c9024957bcebe13b26a65149 |    nova    |    compute    |    Openstack Compute Service    |
+----------------------------------+------------+---------------+---------------------------------+

# keystone endpoint-create \
   --service compute
   --publicurl "http://controller:8774/v2/%(tenant_id)s" \
   --adminurl "http://controller:8774/v2/%(tenant_id)s" \
   --internalurl "http://controller:8774/v2/%(tenant_id)s"
   --region 'RegionOne'
```

On the Controller node, the configuration file ``/etc/nova/nova.conf`` contains all the relevant parameters for the Nova service. Edit the configuration file
```
# vi /etc/nova/nova.conf

[DEFAULT]
novncproxy_host = 0.0.0.0
novncproxy_port = 6080
notify_api_faults = False
state_path=/var/lib/nova
report_interval = 10
enabled_apis = osapi_compute,metadata
osapi_compute_listen = 0.0.0.0
osapi_compute_listen_port = 8774
osapi_compute_workers = 1
metadata_listen = 0.0.0.0
metadata_listen_port = 8775
metadata_workers = 1
service_down_time = 60
rootwrap_config = /etc/nova/rootwrap.conf
volume_api_class = nova.volume.cinder.API
auth_strategy = keystone
use_forwarded_for = False
cpu_allocation_ratio = 16.0
ram_allocation_ratio = 1.5
network_api_class = nova.network.neutronv2.api.API
default_floating_pool = public
force_snat_range = 0.0.0.0/0
metadata_host = <controller>
dhcp_domain = novalocal
security_group_api = neutron
scheduler_driver = nova.scheduler.filter_scheduler.FilterScheduler
vif_plugging_is_fatal = True
vif_plugging_timeout = 300
firewall_driver = nova.virt.firewall.NoopFirewallDriver
debug = True
verbose = True
log_dir = /var/log/nova
use_syslog = False
syslog_log_facility = LOG_USER
use_stderr = True
notification_topics = notifications
rpc_backend = rabbit
amqp_durable_queues = False
sql_connection = mysql://nova:<nova db password>@<controller>/nova
image_service = nova.image.glance.GlanceImageService
lock_path = /var/lib/nova/tmp
osapi_volume_listen = 0.0.0.0
novncproxy_base_url = http://0.0.0.0:6080/vnc_auto.html

[cinder]
catalog_info = volumev2:cinderv2:publicURL

[glance]
api_servers = <controller>:9292

[keystone_authtoken]
auth_uri = http://<controller>:5000/v2.0
identity_uri = http://<controller>:35357
admin_user = nova
admin_password = <nova service password>
admin_tenant_name = services

[libvirt]
vif_driver = nova.virt.libvirt.vif.LibvirtGenericVIFDriver

[neutron]
service_metadata_proxy = True
metadata_proxy_shared_secret = <shared secret>
url = http://<controller>:9696
admin_username = neutron
admin_password = <neutron service password>
admin_tenant_name = services
region_name = RegionOne
admin_auth_url = http://<controller>:5000/v2.0
auth_strategy = keystone
ovs_bridge = br-int
extension_sync_interval = 600
timeout = 30
default_tenant_id = default

[oslo_messaging_rabbit]
rabbit_host = <controller>
rabbit_port = 5672
rabbit_hosts = <controller>:5672
rabbit_use_ssl = False
rabbit_userid = guest
rabbit_password = guest
rabbit_virtual_host = /
rabbit_ha_queues = False
heartbeat_timeout_threshold = 0
heartbeat_rate = 2

[osapi_v3]
enabled = False
```

On the Controller node, start and enable the services
```
# systemctl start openstack-nova-api
# systemctl start openstack-nova-scheduler
# systemctl start openstack-nova-cert
# systemctl start openstack-nova-conductor
# systemctl start openstack-nova-consoleauth

# systemctl enable openstack-nova-api
# systemctl enable openstack-nova-scheduler
# systemctl enable openstack-nova-cert
# systemctl enable openstack-nova-conductor
# systemctl enable openstack-nova-consoleauth
```

Check all services are up and running
```
# nova service-list
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-consoleauth | controller | internal | enabled | up    | 2016-04-12T08:06:25.000000 | -               |
| 2  | nova-scheduler   | controller | internal | enabled | up    | 2016-04-12T08:06:30.000000 | -               |
| 3  | nova-conductor   | controller | internal | enabled | up    | 2016-04-12T08:06:28.000000 | -               |
| 4  | nova-cert        | controller | internal | enabled | up    | 2016-04-12T08:06:30.000000 | -               |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
```

## Add a Compute node
On the Compute node, install and start the KVM hypervisor
```
# yum install -y libvirt qemu-kvm
# systemctl start libvirtd
# systemctl enable libvirtd
```

Determine whether the Compute node supports hardware acceleration for virtual machines
```
# egrep -c '(vmx|svm)' /proc/cpuinfo
```

If the above command returns a zero value, the Compute node does not support hardware acceleration and you must configure the ``libvirtd`` service to use **QEMU** instead of **KVM**. To use **QEMU**, set ``virt_type=qemu`` int the ``/etc/nova/nova.conf`` configuration file as pointed below, otherwise to use **KVM**, instead set ``virt_type=kvm``.

Edit the ``/etc/sysconfig/libvirtd`` ``/etc/libvirt/libvirtd.conf`` configuration files
```
# vi /etc/sysconfig/libvirtd
...
LIBVIRTD_ARGS="--listen"
...

# vi /etc/libvirt/libvirtd.conf
...
listen_tls = 0
listen_tcp = 1
auth_tcp = “none”
...
```

Restart the service and make sure it is listening on 16509
```
# systemctl restart libvirtd
# ps aux | grep libvirtd
root      6128  0.3  0.5 936688 21892 ?        Ssl  16:53   0:01 /usr/sbin/libvirtd --listen

# netstat -natp | grep 16509
tcp        0      0 0.0.0.0:16509           0.0.0.0:*               LISTEN      6128/libvirtd
```

Then install the Compute Nova Service
```
# yum install -y https://www.rdoproject.org/repos/rdo-release.rpm
# yum install -y openstack-nova-compute
```

Configure the compute service by editing the ``/etc/nova/nova.conf`` configuration file
```
[DEFAULT]
internal_service_availability_zone=internal
default_availability_zone=nova
notify_api_faults=False
state_path=/var/lib/nova
report_interval=10
compute_manager=nova.compute.manager.ComputeManager
service_down_time=60
rootwrap_config=/etc/nova/rootwrap.conf
volume_api_class=nova.volume.cinder.API
auth_strategy=keystone
heal_instance_info_cache_interval=60
reserved_host_memory_mb=512
network_api_class=nova.network.neutronv2.api.API
force_snat_range =0.0.0.0/0
metadata_host=<controller>
dhcp_domain=novalocal
security_group_api=neutron
compute_driver=libvirt.LibvirtDriver
vif_plugging_is_fatal=True
vif_plugging_timeout=300
firewall_driver=nova.virt.firewall.NoopFirewallDriver
force_raw_images=True
debug=false
verbose=True
log_dir=/var/log/nova
use_syslog=False
syslog_log_facility=LOG_USER
use_stderr=True
notification_topics=notifications
rpc_backend=rabbit
amqp_durable_queues=False
vncserver_proxyclient_address=<compute node>
sql_connection=mysql://nova@<controller>/nova
vnc_enabled=True
image_service=nova.image.glance.GlanceImageService
lock_path=/var/lib/nova/tmp
vncserver_listen=0.0.0.0
novncproxy_base_url=http://<controller>:6080/vnc_auto.html

[glance]
api_servers=<controller>:9292

[libvirt]
virt_type=qemu
inject_password=False
inject_key=False
inject_partition=-1
live_migration_uri=qemu+tcp://nova@%s/system
cpu_mode=none
vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver

[neutron]
url=http://<controller>:9696
admin_username=neutron
admin_password=<neutron service password>
admin_tenant_name=services
region_name=RegionOne
admin_auth_url=http://<controller>:5000/v2.0
auth_strategy=keystone
ovs_bridge=br-int
extension_sync_interval=600
timeout=30
default_tenant_id=default

[oslo_messaging_rabbit]
rabbit_host=<controller>
rabbit_port=5672
rabbit_hosts=<controller>:5672
rabbit_use_ssl=False
rabbit_userid=guest
rabbit_password=guest
rabbit_virtual_host=/
rabbit_ha_queues=False
heartbeat_timeout_threshold=0
heartbeat_rate=2
```

The Compute node needs to be configured with Neutron networking support. Install Open vSwitch and Neutron support on the Compute node
```
# yum install -y openstack-neutron-openvswitch
# yum install -y openstack-neutron
```

Enable the Linux kernel networking by editing the ``/etc/sysctl.conf`` system configuration file
```
# vi /etc/sysctl.conf
...
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

# sysctl -p
```

Configure the Neutron service by editing the ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
state_path = /var/lib/neutron
use_syslog = False
use_stderr = True
log_dir =/var/log/neutron
bind_host = 0.0.0.0
bind_port = 9696
core_plugin =neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins =lbaas,router,firewall,vpnaas
auth_strategy = keystone
mac_generation_retries = 16
dhcp_lease_duration = 86400
dhcp_agent_notification = True
allow_bulk = True
allow_pagination = False
allow_sorting = False
allow_overlapping_ips = True
advertise_mtu = False
dhcp_agents_per_network = 1
use_ssl = False
rpc_response_timeout=60
rpc_backend=rabbit
control_exchange=neutron
lock_path=/var/lib/neutron/lock

[agent]
root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
report_interval = 30

[keystone_authtoken]
auth_uri = http://127.0.0.1:35357/v2.0/
identity_uri = http://127.0.0.1:5000
admin_tenant_name = %SERVICE_TENANT_NAME%
admin_user = %SERVICE_USER%
admin_password = %SERVICE_PASSWORD%

[oslo_messaging_rabbit]
rabbit_host = <controller>
rabbit_port = 5672
rabbit_hosts = <controller>:5672
rabbit_use_ssl = False
rabbit_userid = guest
rabbit_password = guest
rabbit_virtual_host = /
rabbit_ha_queues = False
heartbeat_rate=2
heartbeat_timeout_threshold=0
```

Configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
# vi /etc/neutron/plugins/ml2/openvswitch_agent.ini

[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
local_ip = <compute node>
enable_tunneling=True

[agent]
polling_interval = 2
tunnel_types =vxlan
vxlan_udp_port =4789
l2_population = False
arp_responder = False
prevent_arp_spoofing = True
enable_distributed_routing = False
drop_flows_on_start=False

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

Start and enable the Open vSwitch
```
# systemctl start openvswitch
# systemctl enable openvswitch
```

Start and enable both the compute and network service
```
# systemctl start neutron-openvswitch-agent
# systemctl enable neutron-openvswitch-agent

# systemctl start openstack-nova-compute
# systemctl enable openstack-nova-compute
```

On the Controller node, check if the compute service is running
```
# nova service-list
# nova service-list
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-consoleauth | controller | internal | enabled | up    | 2016-04-12T09:10:25.000000 | -               |
| 2  | nova-scheduler   | controller | internal | enabled | up    | 2016-04-12T09:10:32.000000 | -               |
| 3  | nova-conductor   | controller | internal | enabled | up    | 2016-04-12T09:10:28.000000 | -               |
| 4  | nova-cert        | controller | internal | enabled | up    | 2016-04-12T09:10:32.000000 | -               |
| 5  | nova-compute     | compute    | nova     | enabled | up    | 2016-04-12T09:10:26.000000 | -               |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
```

You should see a nova-compute service running for each configured Compute node.
