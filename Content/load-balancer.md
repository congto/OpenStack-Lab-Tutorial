# Configure Load Balancer LBaaS Service
A Load Balancer enables networking to distribute incoming requests evenly among designated virtual machines instances. This distribution ensures that the workload is shared predictably among instances and enables more effective use of system resources. Load Balancers use one of these balancing methods to distribute incoming requests:

|Method|Description|
|------|-----------|
|Round Robin|Rotates requests evenly between multiple instances|
|Source IP|Requests from a unique source IP address are consistently directed to the same instance|
|Least connections|Allocates requests to the instance with the least number of active connections|

In this section, we are going to configure a simple load balancer to balance incoming traffic toward a couple of virtual machines running a web server. Both Load Balancer V1 and V2 are available. Version 1 will be replaced by Version 2 but it is still available. At time of writing, the Version 2 is not yet supported via Horizon GUI.

#### LBaaS Version V1
To configure the LBaaS, edit the Load Balancer agent configuration by editing the ``/etc/neutron/lbaas_agent.ini`` initialization file on the Network node
```
[DEFAULT]
debug = True
periodic_interval = 10
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
# device_driver = neutron.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver

[haproxy]
loadbalancer_state_path = $state_path/lbaas
user_group = haproxy
send_gratuitous_arp = 3
```

Start and enable the agent service on the Network node
```
# systemctl start neutron-lbaas-agent
# systemctl enable neutron-lbaas-agent
```

On the Controller node, add the service plug-in to the configuration directive in ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
...
service_plugins = lbaas
```

On the Controller node, add the service provider to the ``/etc/neutron/neutron_lbaas.conf`` configuration file
```
[service_providers]
...
service_provider = LOADBALANCER:Haproxy:neutron_lbaas.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

and restart the Neutron server
```
# systemctl restart neutron-server
```

As admin user, check the neutron agent list
```
# neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 75968236-ff9c-44cb-a982-f93bcded63f4 | Loadbalancer agent | network   | :-)   | True           | neutron-lbaas-agent       |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

As tenant user, create a couple of small virtual machine running on the tenant network
```
# nova boot \
--image $(nova image-list | awk '/ubuntu/ {print $2}') \
--flavor $(nova flavor-list | awk '/small/ {print $2}') \
--nic net-id=$(neutron net-list | awk '/tenant/ {print $2}') \
--key-name mytenant \
--security-groups myaccess \
VM01

# nova boot \
--image $(nova image-list | awk '/ubuntu/ {print $2}') \
--flavor $(nova flavor-list | awk '/small/ {print $2}') \
--nic net-id=$(neutron net-list | awk '/tenant/ {print $2}') \
--key-name mytenant \
--security-groups myaccess \
VM02

# nova list
+--------------------------------------+------+--------+------------+-------------+-----------------------------+
| ID                                   | Name | Status | Task State | Power State | Networks                    |
+--------------------------------------+------+--------+------------+-------------+-----------------------------+
| 1ee95232-e527-47d9-ab7e-df1a675704e5 | VM01 | ACTIVE | -          | Running     | tenant-network=192.168.1.13 |
| f0f78f85-a6c3-4847-b497-5130bc981194 | VM02 | ACTIVE | -          | Running     | tenant-network=192.168.1.14 |
+--------------------------------------+------+--------+------------+-------------+-----------------------------+
```

Enable security group ingress access for HTTP protocol
```
# neutron security-group-rule-create \
--protocol tcp \
--port-range-min 80 \
--port-range-max 80 \
--direction ingress \
myaccess
```

On each instance, login via SSH and start a simple HTTP service
```
ubuntu@vm01:~$ MYIP=$(ifconfig eth0|grep 'inet addr'|awk -F: '{print $2}'| awk '{print $1}'
ubuntu@vm01:~$ while true; do echo -e "HTTP/1.0 200 OK\r\n\r\nWelcome to $MYIP" | sudo nc -l -p 80 ; done &

ubuntu@vm02:~$ MYIP=$(ifconfig eth0|grep 'inet addr'|awk -F: '{print $2}'| awk '{print $1}'
ubuntu@vm02:~$ while true; do echo -e "HTTP/1.0 200 OK\r\n\r\nWelcome to $MYIP" | sudo nc -l -p 80 ; done &
```

As tenant user, create a load balancer pool based on Round Robin method and HTTP protocol
```
# neutron lb-pool-create \
--lb-method ROUND_ROBIN \
--protocol HTTP \
--subnet-id tenant-subnetwork \
--name my-lb-pool
```

Add the two Virtual Machines created above as memenbers to the Load Balancer pool
```
# neutron lb-member-create \
--weight 1 \
--address 192.168.1.17 \
--protocol-port 80 \
my-lb-pool

# neutron lb-member-create \
--weight 1 \
--address 192.168.1.18 \
--protocol-port 80 \
my-lb-pool

# neutron lb-member-list
+--------------------------------------+--------------+---------------+--------+----------------+--------+
| id                                   | address      | protocol_port | weight | admin_state_up | status |
+--------------------------------------+--------------+---------------+--------+----------------+--------+
| a0828bd5-da97-45b1-b4d4-8fef63acbf1b | 192.168.1.17 |            80 |      1 | True           | ACTIVE |
| ebd9e210-df12-4e64-bf2a-00ac57449200 | 192.168.1.18 |            80 |      1 | True           | ACTIVE |
+--------------------------------------+--------------+---------------+--------+----------------+--------+
```

Create a Virtual IP and associate to the pool. The Virtual IP will be the IP address of the Load Balancer
```
neutron lb-vip-create \
--name LB-Virtual-IP \
--address 192.168.1.250 \
--protocol HTTP \
--protocol-port 80 \
--subnet-id $(neutron subnet-list | awk '/tenant/ {print $2}') \
my-lb-pool
```

At this point the Load Balancer has been successfully created and should be functional. Traffic sent to address 192.168.1.250:80 will be balanced across all active members of the pool.

By default, the Load Balancer is performed by an HA HTTP Proxy instance running inside a private namespace on the Network node
```
# ip netns
qlbaas-8d333495-8228-46d5-bd62-f0845d16fe9b
qrouter-9a45bdb6-b7ff-4329-8333-7339050ebcf9
qdhcp-9a7f354c-7a46-420d-98a5-3508e6f3caf1

# ip netns exec qlbaas-8d333495-8228-46d5-bd62-f0845d16fe9b ifconfig
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
tapd784d30b-d8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.250  netmask 255.255.255.0  broadcast 192.168.1.255
```

The HA Proxy relevant configuration is available on the Network node
```
# cat /var/lib/neutron/lbaas/<load_balacer_id>/conf
global
        daemon
        user nobody
        group haproxy
        log /dev/log local0
        log /dev/log local1 notice
        stats socket /var/lib/neutron/lbaas/8d333495-8228-46d5-bd62-f0845d16fe9b/sock mode 0666 level user
defaults
        log global
        retries 3
        option redispatch
        timeout connect 5000
        timeout client 50000
        timeout server 50000
frontend 1e140afe-ba44-419f-b4b3-4c8732856ef1
        option tcplog
        bind 192.168.1.250:80
        mode http
        default_backend 8d333495-8228-46d5-bd62-f0845d16fe9b
        option forwardfor
backend 8d333495-8228-46d5-bd62-f0845d16fe9b
        mode http
        balance roundrobin
        option forwardfor
        server d901017d-25e9-44c2-9784-2ec14eb0119d 192.168.1.17:80 weight 1
        server fafff626-4e3a-4def-b8f3-c489e3171d20 192.168.1.18:80 weight 1
```

The next step in the Load Balancer setup is to create a health monitor. The health monitor is responsible for periodically checking the health of each member, so that unresponsive servers are quickly removed from the pool
```
# neutron lb-healthmonitor-create --delay 5 --type HTTP --max-retries 3 --timeout 2
Created a new health_monitor:
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| admin_state_up | True                                 |
| delay          | 5                                    |
| expected_codes | 200                                  |
| http_method    | GET                                  |
| id             | 0947aaf6-5d24-4e83-a053-1a22517738bb |
| max_retries    | 3                                    |
| pools          |                                      |
| tenant_id      | 22bdc5a0210e4a96add0cea90a6137ed     |
| timeout        | 2                                    |
| type           | HTTP                                 |
| url_path       | /                                    |
+----------------+--------------------------------------+
```

The above health monitor will perform an HTTP GET of the root path of the members. This health check expects an HTTP status of 200 in the response, and the connection must be established within 2 seconds. This check will be retried a maximum of 3 times before a member is determined to be failed and removed from the pool. Associate the healt monitor to the load balancer pool
```
# neutron lb-healthmonitor-associate 0947aaf6-5d24-4e83-a053-1a22517738bb my-lb-pool
Associated health monitor 0947aaf6-5d24-4e83-a053-1a22517738bb
```

Now the Load Balancer setup is fully working. To make the load balancer externally accessible, create a floating IP address and associate it with the Virtual IP address.
```
# neutron floatingip-create external-flat-network
Created a new floatingip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| fixed_ip_address    |                                      |
| floating_ip_address | 172.16.1.209                         |
| floating_network_id | 0588949f-7a2a-43cc-a879-2ddbabea0d4a |
| id                  | 0f16047b-bc77-472c-b6f4-d62dc7993adb |
| status              | DOWN                                 |
| tenant_id           | 22bdc5a0210e4a96add0cea90a6137ed     |
+---------------------+--------------------------------------+

# neutron lb-vip-show LB-Virtual-IP
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| address             | 192.168.1.250                        |
| admin_state_up      | True                                 |
| id                  | 21e16c3a-cd16-4666-a851-023d0407d53e |
| name                | LB-Virtual-IP                        |
| pool_id             | 32dc27ea-ed0d-4a7e-8dde-734b210f50ca |
| port_id             | e9afb9b3-ab00-4188-921d-6c676fef6c12 |
| protocol            | HTTP                                 |
| protocol_port       | 80                                   |
| session_persistence |                                      |
| status              | ACTIVE                               |
+---------------------+--------------------------------------+

# neutron floatingip-associate 0f16047b-bc77-472c-b6f4-d62dc7993adb e9afb9b3-ab00-4188-921d-6c676fef6c12
```

#### LBaaS Version V2
To enable LBaaS Version 2, remove any configuration related to the Version 1. Since Version 1 and Version 2 cannot run at the same time, stop and disable the Neutron LBaaS Version 1 on the Network node
```
# systemctl stop neutron-lbaas-agent
# systemctl disable neutron-lbaas-agent
```

On the Network node, edit the ``/etc/neutron/lbaas_agent.ini`` initialization file
```
[DEFAULT]
debug = True
periodic_interval = 10
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
# device_driver = neutron.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver

[haproxy]
loadbalancer_state_path = $state_path/lbaas
user_group = haproxy
send_gratuitous_arp = 3
```

Start and enable the Load Balancer agent service Version 2 on the Network node
```
# systemctl start neutron-lbaasv2-agent
# systemctl enable neutron-lbaasv2-agent
```

On the Controller node, remove the Version 1 and add the Version 2 service plug-in to the configuration directive in ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
...
service_plugins = neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
```

On the Controller node, add the Version 2 service provider to the ``/etc/neutron/neutron_lbaas.conf`` configuration file 
```
[service_providers]
...
service_provider = LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

and restart the Neutron server
```
# systemctl restart neutron-server
```

On the Controller node, as admin user, update the Neutron agent list
```
# neutron agent-list
+--------------------------------------+----------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type           | host      | alive | admin_state_up | binary                    |
+--------------------------------------+----------------------+-----------+-------+----------------+---------------------------+
| 75968236-ff9c-44cb-a982-f93bcded63f4 | Loadbalancer agent   | network   | XXX   | True           | neutron-lbaas-agent       |
| 20e11acf-4d36-443e-af52-602fa2ef824a | Loadbalancerv2 agent | network   | :-)   | True           | neutron-lbaasv2-agent     |
+--------------------------------------+----------------------+-----------+-------+----------------+---------------------------+

# neutron agent-delete 75968236-ff9c-44cb-a982-f93bcded63f4
# neutron agent-list
+--------------------------------------+----------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type           | host      | alive | admin_state_up | binary                    |
+--------------------------------------+----------------------+-----------+-------+----------------+---------------------------+
| 20e11acf-4d36-443e-af52-602fa2ef824a | Loadbalancerv2 agent | network   | :-)   | True           | neutron-lbaasv2-agent     |
+--------------------------------------+----------------------+-----------+-------+----------------+---------------------------+
```

On the Controller node, run the Neutron database migration from Version 1 to Version 2
```
# neutron-db-manage --service lbaas upgrade head
```

Now we can setup a Load Balancer service. As tenant user, create a load balancer on the tenant network
```
# neutron lbaas-loadbalancer-create \
--name load-balancer \
tenant-subnetwork

# neutron lbaas-loadbalancer-show load-balancer
+---------------------+------------------------------------------------+
| Field               | Value                                          |
+---------------------+------------------------------------------------+
| admin_state_up      | True                                           |
| description         |                                                |
| id                  | c111c728-3c04-498c-8e9b-349b52648579           |
| listeners           | {"id": "7012b0f4-9780-49d5-b2f1-47c9ad7963b7"} |
| name                | load-balancer                                  |
| operating_status    | ONLINE                                         |
| provider            | haproxy                                        |
| provisioning_status | ACTIVE                                         |
| tenant_id           | 22bdc5a0210e4a96add0cea90a6137ed               |
| vip_address         | 192.168.1.35                                   |
| vip_port_id         | b17fb7e6-8492-4d35-9d33-bf05943ed31b           |
| vip_subnet_id       | a1499bcd-9ce3-4cef-9a53-0c267c1d5ca7           |
+---------------------+------------------------------------------------+
```

The load balancer just created is active and ready to serve traffic on **192.168.1.35**. With the load balancer online, add a listener for HTTP traffic on port 80
```
# neutron lbaas-listener-create \
--loadbalancer load-balancer \
--protocol HTTP \
--protocol-port 80 \
--name listener

# neutron lbaas-listener-show listener
+---------------------------+------------------------------------------------+
| Field                     | Value                                          |
+---------------------------+------------------------------------------------+
| admin_state_up            | True                                           |
| connection_limit          | -1                                             |
| default_pool_id           | 4e9dd23e-faa7-4d75-ae6d-ad64f28d3261           |
| id                        | 7012b0f4-9780-49d5-b2f1-47c9ad7963b7           |
| loadbalancers             | {"id": "c111c728-3c04-498c-8e9b-349b52648579"} |
| name                      | listener                                       |
| protocol                  | HTTP                                           |
| protocol_port             | 80                                             |
| tenant_id                 | 22bdc5a0210e4a96add0cea90a6137ed               |
+---------------------------+------------------------------------------------+
```

Build a load balancer pool 
```
# neutron lbaas-pool-create \
--lb-algorithm ROUND_ROBIN \
--listener listener \
--protocol HTTP \
--name mypool

# neutron lbaas-pool-show mypool
+---------------------+------------------------------------------------+
| Field               | Value                                          |
+---------------------+------------------------------------------------+
| admin_state_up      | True                                           |
| id                  | 4e9dd23e-faa7-4d75-ae6d-ad64f28d3261           |
| lb_algorithm        | ROUND_ROBIN                                    |
| listeners           | {"id": "7012b0f4-9780-49d5-b2f1-47c9ad7963b7"} |
| members             |                                                |
|                     |                                                |
| name                | mypool                                         |
| protocol            | HTTP                                           |
| session_persistence |                                                |
| tenant_id           | 22bdc5a0210e4a96add0cea90a6137ed               |
+---------------------+------------------------------------------------+
```

Add members to the pool to serve HTTP content on port 80. For this example, the web servers are **192.168.1.17** and **192.168.1.18**
```
# neutron lbaas-member-create  \
--subnet tenant-subnetwork \
--address 192.168.1.17 \
--protocol-port 80 \
mypool

# neutron lbaas-member-create  \
--subnet tenant-subnetwork \
--address 192.168.1.18 \
--protocol-port 80 \
mypool
```

Add a health monitor so that unresponsive servers are removed from the pool. 
```
# neutron lbaas-healthmonitor-create \
--delay 5 \
--max-retries 2 \
--timeout 10 \
--type HTTP \
--pool mypool

Created a new healthmonitor:
+----------------+------------------------------------------------+
| Field          | Value                                          |
+----------------+------------------------------------------------+
| admin_state_up | True                                           |
| delay          | 5                                              |
| expected_codes | 200                                            |
| http_method    | GET                                            |
| id             | d6f714a9-d6bf-410a-81a8-d8ad24963de9           |
| max_retries    | 2                                              |
| pools          | {"id": "4e9dd23e-faa7-4d75-ae6d-ad64f28d3261"} |
| tenant_id      | 22bdc5a0210e4a96add0cea90a6137ed               |
| timeout        | 10                                             |
| type           | HTTP                                           |
| url_path       | /                                              |
+----------------+------------------------------------------------+
```

The health monitor just created removes an unresponsive server from the pool if it fails a health check at 10 seconds intervals. When the server recovers and begins responding to health checks again, it is added to the pool once again.

To make load balancer reachable from the outside networks, associate a Floating IP with the Virtual IP of the load balancer
```
# neutron floatingip-create external-flat-network
Created a new floatingip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| floating_ip_address | 172.16.1.210                         |
| floating_network_id | 0588949f-7a2a-43cc-a879-2ddbabea0d4a |
| id                  | bc2d4d00-e5d0-48c2-9872-9a01912a402b |
| tenant_id           | 22bdc5a0210e4a96add0cea90a6137ed     |
+---------------------+--------------------------------------+

# neutron lbaas-loadbalancer-show load-balancer
+---------------------+------------------------------------------------+
| Field               | Value                                          |
+---------------------+------------------------------------------------+
| admin_state_up      | True                                           |
| description         |                                                |
| id                  | c111c728-3c04-498c-8e9b-349b52648579           |
| listeners           | {"id": "7012b0f4-9780-49d5-b2f1-47c9ad7963b7"} |
| name                | load-balancer                                  |
| operating_status    | ONLINE                                         |
| provider            | haproxy                                        |
| provisioning_status | ACTIVE                                         |
| tenant_id           | 22bdc5a0210e4a96add0cea90a6137ed               |
| vip_address         | 192.168.1.35                                   |
| vip_port_id         | b17fb7e6-8492-4d35-9d33-bf05943ed31b           |
| vip_subnet_id       | a1499bcd-9ce3-4cef-9a53-0c267c1d5ca7           |
+---------------------+------------------------------------------------+

# neutron floatingip-associate bc2d4d00-e5d0-48c2-9872-9a01912a402b b17fb7e6-8492-4d35-9d33-bf05943ed31b
Associated floating IP bc2d4d00-e5d0-48c2-9872-9a01912a402b

# neutron floatingip-show bc2d4d00-e5d0-48c2-9872-9a01912a402b
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| fixed_ip_address    | 192.168.1.35                         |
| floating_ip_address | 172.16.1.210                         |
| floating_network_id | 0588949f-7a2a-43cc-a879-2ddbabea0d4a |
| id                  | bc2d4d00-e5d0-48c2-9872-9a01912a402b |
| port_id             | b17fb7e6-8492-4d35-9d33-bf05943ed31b |
| router_id           | 9a45bdb6-b7ff-4329-8333-7339050ebcf9 |
| status              | ACTIVE                               |
| tenant_id           | 22bdc5a0210e4a96add0cea90a6137ed     |
+---------------------+--------------------------------------+
```
