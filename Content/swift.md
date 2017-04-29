# Swift Object Storage

The object storage service provides object storage in virtual containers, allowing users to store and retrieve objects such as files without a filesystem interface. The service's distributed architecture supports horizontal scaling; object redundancy is provided through software-based data replication. By supporting eventual consistency through asynchronous replication, it is well-suited to multiple datacenter deployment.
Object storage uses the concept of:

* **Storage replicas** Used to maintain the state of objects in the case of outage. A minimum of three replicas is recommended.

* **Storage zones** Used to host replicas. Zones ensure that each replica of a given object can be stored separately. A zone might represent an individual disk drive or array, a server, all the servers in a rack, or even an entire datacenter.

* **Storage regions** Essentially a group of zones sharing a location. Regions can be groups of servers or server farms, usually located in the same geographical area. Regions have a separate API endpoint per object storage service installation, which allows for discrete separation of services.

The OpenStack object storage service is a modular service with the following components:

* _openstack-swift-proxy_: the proxy service uses the object ring to decide where to store newly uploaded objects. It updates the relevant container database to reflect the presence of a new object. If a newly uploaded object goes to a new container, the proxy service also updates the relevant account database to reflect the new container. The proxy service also directs get requests to one of the nodes where a replica of the requested object is stored, either randomly or based on response time from the node. It provides access to the public Swift API, and is responsible for handling and routing requests. The proxy server streams objects to users without spooling. Objects can also be served via HTTP.

* _openstack-swift-object_: the object service is responsible for storing data objects in partitions on disk devices. Each partition is a directory, and each object is held in a subdirectory of its partition directory. A MD5 hash of the path to the object is used to identify the object itself. The service stores, retrieves, and deletes objects.

* _openstack-swift-container_: the container service maintains databases of objects in containers. There is one database file for each container, and they are replicated across the cluster. Containers are defined when objects are put in them. Containers make finding objects faster by limiting object listings to specific container namespaces. The container service is responsible for listings of containers using the account database.

* _openstack-swift-account_: the account service maintains databases of all of the containers accessible by any given account. There is one database file for each account, and they are replicated across the cluster. Any account has access to a particular group of containers. An account maps to a tenant in the identity service. The account service handles listings of objects (what objects are in a specific container) using the container database.

## Implementing the Swift Object Storage Service
On the Controller node, install the necessary components for the Swift object storage service, including the swift client and memcached. In a production envinronment, the Swift proxy server shoud spawn on a couple of standalone nodes for redundancy reason. If a proxy fails, the other will take over. Howewer, in this example, we are going to install the proxy service on the Controller node. 

```
# yum install -y openstack-swift-proxy
# yum install -y python-swiftclient
# yum install -y memcached
```

Source the Keystone environment variables with the authentication information and create a swift user with the admin role. Then create a services tenant and add the swift user to it.

```
# source /root/keystonerc_admin
# openstack user create --domain default --project service --password servicepassword swift 
# openstack role add --project service --user swift admin
```

Create the Swift Object Storage Service
```
# keystone service-create --name swift object-store --description "Swift Object Storage Service"
```

Create the end point for the service you just declared
```
# openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(tenant_id\)s
# openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(tenant_id\)s
# openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1  
```

Check the services in Keystone
```
# keystone service-list
+----------------------------------+----------+--------------+------------------------------+
|                id                |   name   |     type     |         description          |
+----------------------------------+----------+--------------+------------------------------+
| 623343aa351e4c5fb726e0932a89bbfb | keystone |   identity   |  Keystone Identity Service   |
| 350e61c1866d4f468fa8ded69ed848d1 |  swift   | object-store | Swift Object Storage Service |
+----------------------------------+----------+--------------+------------------------------+
```

Configure the Swift service by editing the ``/etc/swift/swift.conf`` configuration file
```
# vi /etc/swift/swift.conf
[swift-hash]
swift_hash_path_suffix = swift_shared_path # it is shared among all Swift nodes, any words you like
```

Configure the Swift proxy service by editing the ``/etc/swift/proxy-server.conf`` configuration file
```
# vi /etc/swift/proxy-server.conf
...
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = <service password>
delay_auth_decision = true
```

## Deploying the Swift Object Storage Service
The object storage service stores objects on the file system, usually on a number of connected physical storage devices. All of the devices that will be used for object storage must be formatted with XFS, and mounted under the ``/srv/node/`` directory of each Storage node. 

In this section, we are going to configure the Storage node of the OpenStack setup as a Swift cluster. In a production envinronment, the Swift cluster should be made of minimum 3 separate nodes, each containing one or more zones. Each piece of data is placed in three zones for redundancy. If a zone goes down, data is replicated to other zones. To keep things simple, our demo cluster will be made of only one node containing 3 separate zones. Each zone is backed by a separate logical volume of the Storage node. 

The Swift Object Storage Service Ring maps partitions to physical locations on the disk. When any other component needs to perform any operation on an object, a container, or an account, it needs to interact with the Ring to determine the location of the object or the container in the cluster. The Ring maintains this mapping. In additions, the Ring is responsible to determine which devices are used for handoff when a failure occurs.

Three ring files need to be created. The ring files are used to deduce where a particular piece of data is stored

  1. ``object.ring`` to track the objects stored by the object storage service
  2. ``container.ring`` to track the containers where the objects are placed in
  3. ``account.ring`` to track which accounts (users) can access which containers.

On the Controller node, create the Rings files with 2^12=4096 partitions, data replica of 3 and 1 hour of rebalancing time
```
# source /root/keystonerc_admin
# swift-ring-builder /etc/swift/object.builder create 12 3 1
# swift-ring-builder /etc/swift/container.builder create 12 3 1
# swift-ring-builder /etc/swift/account.builder create 12 3 1
```

Add the devices to the Ring specifiyng the zone number, the Storage server address, port, device and weight
```
# swift-ring-builder /etc/swift/object.builder add z1-<storage_address_device1>:6000/device1 100
# swift-ring-builder /etc/swift/container.builder add z1-<storage_address_device1>:6001/device1 100 
# swift-ring-builder /etc/swift/account.builder add z1-<storage_address_device1>:6002/device1 100 

# swift-ring-builder /etc/swift/object.builder add z2-<storage_address_device2>:6000/device2 100
# swift-ring-builder /etc/swift/container.builder add z2-<storage_address_device2>:6001/device2 100 
# swift-ring-builder /etc/swift/account.builder add z2-<storage_address_device2>:6002/device2 100 

# swift-ring-builder /etc/swift/object.builder add z3-<storage_address_device3>:6000/device3 100 
# swift-ring-builder /etc/swift/container.builder add z3-<storage_address_device3>:6001/device3 100 
# swift-ring-builder /etc/swift/account.builder add z3-<storage_address_device3>:6002/device3 100 
```

Distribute the partitions across the drives in the ring
```
# swift-ring-builder /etc/swift/object.builder rebalance
# swift-ring-builder /etc/swift/container.builder rebalance
# swift-ring-builder /etc/swift/account.builder rebalance

# chown swift:swift /etc/swift/*.gz 
# ls -lrt /etc/swift/*gz
-rw-r--r--. 1 swift swift 1851 Jan 22 15:47 /etc/swift/account.ring.gz
-rw-r--r--. 1 swift swift 1841 Jan 22 15:47 /etc/swift/container.ring.gz
-rw-r--r--. 1 swift swift 1834 Jan 22 15:47 /etc/swift/object.ring.gz
```

On the Controller node running the proxy service, enable the memcached and openstack-swift-proxy permanently
```
# systemctl start memcached
# systemctl enable memcached
# systemctl start openstack-swift-proxy
# systemctl enable openstack-swift-proxy
```

On each Storage node belonging to the Swift cluster, install the following packages
```
# yum install -y openstack-swift-object
# yum install -y openstack-swift-container
# yum install -y openstack-swift-account
```
On each Storage node, create the mount points and mount the logical volumes persistently to the appropriate directories and set the ownership to the swift user.
```
# lvscan | grep swift
  ACTIVE            '/dev/swift03/zone03' [16.00 GiB] inherit
  ACTIVE            '/dev/swift02/zone02' [16.00 GiB] inherit
  ACTIVE            '/dev/swift01/zone01' [16.00 GiB] inherit

# mkdir -p /srv/node/device1
# mkdir -p /srv/node/device2
# mkdir -p /srv/node/device3
# vi /etc/fstab
...
/dev/swift03/zone03     /srv/node/device3       xfs     noatime,nodiratime,nobarrier,logbufs=8  0       0
/dev/swift02/zone02     /srv/node/device2       xfs     noatime,nodiratime,nobarrier,logbufs=8  0       0
/dev/swift01/zone01     /srv/node/device1       xfs     noatime,nodiratime,nobarrier,logbufs=8  0       0

# mount -a
# chown -R swift:swift /srv/node
```

Copy the Ring files from the Controller node to all Storage nodes of the cluster. 
```
# scp /etc/swift/*.gz storage:/etc/swift/ 
```

On each Storage node, configure the Swift service
```
# chown swift. /etc/swift/*.gz 
# vi /etc/swift/swift.conf
[swift-hash] swift_hash_path_suffix = swift_shared_path # the same value that is set on the Controller node
```

Configure the Swift account service
```
# vi /etc/swift/account-server.conf
[DEFAULT]
devices = /srv/node
bind_ip = <local address>
bind_port = 6002
mount_check = false
user = swift
workers = 1
log_name = account-server
log_facility = LOG_LOCAL2
log_level = INFO
log_address = /dev/log
[pipeline:main]
pipeline = account-server
[app:account-server]
use = egg:swift#account
set log_name = account-server
set log_facility = LOG_LOCAL2
set log_level = INFO
set log_requests = true
set log_address = /dev/log
[account-replicator]
concurrency = 1
[account-auditor]
[account-reaper]
concurrency = 1
```

Configure the Swift container service
```
# vi /etc/swift/container-server.conf
[DEFAULT]
devices = /srv/node
bind_ip = <local address>
bind_port = 6001
mount_check = false
user = swift
log_name = container-server
log_facility = LOG_LOCAL2
log_level = INFO
log_address = /dev/log
workers = 1
allowed_sync_hosts = 127.0.0.1
[pipeline:main]
pipeline = container-server
[app:container-server]
allow_versions = true
use = egg:swift#container
set log_name = container-server
set log_facility = LOG_LOCAL2
set log_level = INFO
set log_requests = true
set log_address = /dev/log
[container-replicator]
concurrency = 1
[container-updater]
concurrency = 1
[container-auditor]
[container-sync]
```

Configure the Swift object service
```
# vi /etc/swift/object-server.conf
[DEFAULT]
devices = /srv/node
mount_check = false
user = swift
log_name = object-server
log_facility = LOG_LOCAL2
log_level = INFO
log_address = /dev/log
bind_ip = <local address>
bind_port = 6000
workers = 3

[pipeline:main]
pipeline = object-server

[app:object-server]
use = egg:swift#object
set log_name = object-server
set log_facility = LOG_LOCAL2
set log_level = INFO
set log_requests = true
set log_address = /dev/log

[object-replicator]
concurrency = 1

[object-updater]
concurrency = 1
```

Configure the synchonization
```
# vi /etc/rsyncd.conf
...
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log
uid = swift
gid = swift
address = <local address>

[account]
path            = /srv/node
read only       = false
write only      = no
list            = yes
incoming chmod  = 0644
outgoing chmod  = 0644
max connections = 25
lock file =     /var/lock/account.lock

[container]
path            = /srv/node
read only       = false
write only      = no
list            = yes
incoming chmod  = 0644
outgoing chmod  = 0644
max connections = 25
lock file =     /var/lock/container.lock

[object]
path            = /srv/node
read only       = false
write only      = no
list            = yes
incoming chmod  = 0644
outgoing chmod  = 0644
max connections = 25
lock file =     /var/lock/object.lock

[swift_server]
path            = /etc/swift
read only       = true
write only      = no
list            = yes
incoming chmod  = 0644
outgoing chmod  = 0644
max connections = 5
lock file =     /var/lock/swift_server.lock
```

Start up the services and make them persistent at boot
```
# systemctl start openstack-swift-account
# systemctl enable openstack-swift-account
# systemctl start openstack-swift-container
# systemctl enable openstack-swift-container
# systemctl start openstack-swift-object
# systemctl enable openstack-swift-object
```
