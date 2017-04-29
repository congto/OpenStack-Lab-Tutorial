# Multiple Cinder Storage Backends
Configuring multiple-storage backends, allows to create several backend storage solutions that serve the same OpenStack configuration and one cinder-volume service is launched for each backend storage. Multiple backends can reside on the same Storage node or they can be distributed on different Storage nodes. For example, a Storage node can have both SSD and SATA disks. The Cinder configuration can define two different backends, one for SSD disks and the other one for SATA disks.

In a multiple-storage configuration, each backend has a name. The name of the backend is declared as an extra-specification of a volume type. When a volume is created, the scheduler chooses an appropriate backend to handle the request, according to the volume type specified by the user.

To enable a multiple-storage backends, the ``enabled_backends`` option in the cinder configuration file need to be used. This option defines the names of the configuration groups for the different backends. Each name is associated to one configuration group for a backend. The configuration group name is not related to the name of the backend.

As example, we define two LVM backends named **LOLLO** and **LOLLA** respctively. In the cinder configuration file, we are going to declare two configuration groups, one for each backend, called **lvm1** and **lvm2** respectively. We set the multiple backend option as  ``enabled_backends=lvm1,lvm2`` in the configuration file. On the Controller node (in this example acting as Storage node too), it will look like:

```
[DEFAULT]
debug=True
verbose=True
log_dir=/var/log/cinder
use_syslog=False
auth_strategy = keystone
enabled_backends=lvm1,lvm2

rabbit_userid = guest
rabbit_password = guest
rabbit_host = 10.10.10.30 #controller
rabbit_use_ssl = false
rabbit_port = 5672

[database]
sql_connection = mysql://cinder:******@controller/cinder
idle_timeout=3600
min_pool_size=1
max_retries=10
retry_interval=10

[keystone_authtoken]
admin_tenant_name = services
admin_user = cinder
admin_password = ******
auth_host = controller
auth_port = 35357
auth_protocol = http

[lvm1]
iscsi_helper=lioadm
iscsi_ip_address=192.168.2.30 #controller on the Storage network
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name=LOLLA
volume_group=storage1

[lvm2]
iscsi_helper=lioadm
iscsi_ip_address=192.168.2.30 #controller on the Storage network
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name=LOLLO
volume_group=storage2

```
To enable the new configuration, restart the cinder service
```
# openstack-service restart cinder
```
Create two new volume types and associate them with the two backends
```
# cinder type-create lvm_gold
# cinder type-create lvm_silver
# cinder type-list
+--------------------------------------+------------+
|                  ID                  |    Name    |
+--------------------------------------+------------+
| 60ee47a6-ebaa-4d7d-8586-b722fb00677f |  lvm_gold  |
| e67d309c-a3e7-42a0-b8ba-f34485582734 | lvm_silver |
+--------------------------------------+------------+
# cinder type-key lvm_silver set volume_backend_name=LOLLO
# cinder type-key lvm_gold set volume_backend_name=LOLLA

```
Now the two new backends can be used to create volumes
```
# cinder create --display-name vol1 --volume-type lvm_gold 5
# cinder create --display-name vol2 --volume-type lvm_silver 5
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 0b1acd71-651f-41eb-a4e8-7f1fe8f39825 | available |     vol1     |  5   |   lvm_gold  |  false   |             |
| 9ae1f05e-ccf0-4532-96be-c1c21a293130 | available |     vol2     |  5   |  lvm_silver |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```

Note that each backend has its own cinder volume service
```
# cinder-manage service list
Binary           Host                                 Zone             Status     State Updated At
cinder-scheduler controller                            nova             enabled    :-)   2015-05-18 18:05:26
cinder-volume    controller@lvm2                       nova             enabled    :-)   2015-05-18 18:05:27
cinder-volume    controller@lvm1                       nova             enabled    :-)   2015-05-18 18:05:27
```

### Multiple Cinder Storage Nodes
In an OpenStack production setup, one or more Storage nodes are used. This section describes how to install and configure two storage nodes for the Block Storage service. The service provisions logical volumes on this device using the LVM driver and provides them to instances via iSCSI transport. In this example, each Storage node defines a different storage backend with different performances. Basing on the volume type specificed in the volume creation request, the Cinder scheduler will chose the backend where to deploy the volume.

Install the Storage nodes and connect them with Controller node and Compute nodes using an isolate Storage network. In this example, the storage network is 192.168.2.0/24, the Management network is 10.10.10.0/24 and the Tenant network is 192.168.1.0/24.

|Management IP|Storage IP|Node role|
|-------------|----------|---------|
| 10.10.10.30 | n/a          | Controller|
| 10.10.10.36 | 192.168.2.36 | Storage 01|
| 10.10.10.37 | 192.168.2.37 | Storage 02|
| 10.10.10.32 | 192.168.2.32 | Compute 01|
| 10.10.10.34 | 192.168.2.34 | Compute 02|

On the first Storage node, install the LVM package and create the volume group
```
# yum install -y lvm2
# pvcreate /dev/sdb
# vgcreate cinder-volumes /dev/sdb
Volume group "cinder-volumes" successfully created
```

On the first Storage node, install and configure the components
```
# yum install openstack-cinder targetcli python-oslo-policy
# systemctl enable openstack-cinder-volume
# systemctl enable target
```

On the first Storage node, edit the ``/etc/cinder/cinder.conf`` file 
```
[root@osstorage]# cat /etc/cinder/cinder.conf
[DEFAULT]
glance_host = 10.10.10.30 # controller
enable_v1_api = True
enable_v2_api = True
storage_availability_zone = nova
default_availability_zone = nova
auth_strategy = keystone
enabled_backends = lvm2
osapi_volume_listen = 0.0.0.0
osapi_volume_workers = 1
nova_catalog_info = compute:Compute Service:publicURL
nova_catalog_admin_info = compute:Compute Service:adminURL
debug = True
verbose = True
notification_driver = messagingv2
rpc_backend = rabbit
control_exchange = openstack
api_paste_config=/etc/cinder/api-paste.ini
amqp_durable_queues=False

[keystone_authtoken]
auth_uri = http://10.10.10.30:5000 #controller
auth_url = http://10.10.10.30:35357 #controller
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = cinder
password = *******

[database]
connection = mysql://cinder:*******@10.10.10.30/cinder #controller
[oslo_messaging_rabbit]
rabbit_host = 10.10.10.30 #controller
rabbit_port = 5672
rabbit_hosts = 10.10.10.30:5672 #controller
rabbit_use_ssl = False
rabbit_userid = guest
rabbit_password = guest
rabbit_ha_queues = False
heartbeat_timeout_threshold = 0
heartbeat_rate = 2

[lvm1]
iscsi_helper=lioadm
volume_group=cinder-volumes
iscsi_ip_address=192.168.2.36 #local IP of the first Storage node on the Storage network
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_protocol=iscsi
volume_backend_name=silver
```

On the first Storage node, start the cinder-volume and the iscsi target services
```
# systemctl start openstack-cinder-volume
# systemctl start target
```

and check the service list
```
[root@osstorage01]# cinder-manage service list
Binary           Host                                   Zone             Status     State Updated At
cinder-scheduler oscontroller                           nova             enabled    :-)   2016-03-01 14:15:34
cinder-volume    osstorage01@lvm1                       nova             enabled    :-)   2016-03-01 14:15:34
[root@osstorage01]#
```

On the seconde Storage node install the above stuff and edit the ``/etc/cinder/cinder.conf`` file
```
...
[lvm2]
iscsi_helper=lioadm
volume_group=cinder-volumes
iscsi_ip_address=192.168.2.37 #local IP of the second Storage node on the Storage network
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_protocol=iscsi
volume_backend_name=gold
```

On the second Storage node, start the cinder-volume and the iscsi target services
```
# systemctl start openstack-cinder-volume
# systemctl start target
```

and check the service list
```
[root@osstorage02]# cinder-manage service list
Binary           Host                                   Zone             Status     State Updated At
cinder-scheduler oscontroller                           nova             enabled    :-)   2016-03-01 14:15:34
cinder-volume    osstorage01@lvm1                       nova             enabled    :-)   2016-03-01 14:15:34
cinder-volume    osstorage02@lvm2                       nova             enabled    :-)   2016-03-01 14:15:34
[root@osstorage02]#
```

Create two new Cinder backend types
```
# cinder type-create lvm_silver
# cinder type-create lvm_gold
# cinder type-key lvm_silver set volume_backend_name=silver
# cinder type-key lvm_gold set volume_backend_name=gold
# cinder type-list
+--------------------------------------+------------+-------------+-----------+
|                  ID                  |    Name    | Description | Is_Public |
+--------------------------------------+------------+-------------+-----------+
| 74055a9a-a576-49a3-92fa-dfa7a935cdba | lvm_silver |      -      |    True   |
| 9c975ae9-f16d-4a4a-a930-8ae0efb5fab8 |  lvm_gold  |      -      |    True   |
+--------------------------------------+------------+-------------+-----------+

# cinder extra-specs-list
+--------------------------------------+------------+-----------------------------------------+
|                  ID                  |    Name    |               extra_specs               |
+--------------------------------------+------------+-----------------------------------------+
| 07f2ecf9-d486-424f-a885-33a8c86ae409 | lvm_silver | {u'volume_backend_name': u'silver'}     |
| 313c5fa7-217f-4236-ba62-76cc84511a63 |  lvm_gold  |  {u'volume_backend_name': u'gold'}      |
+--------------------------------------+------------+-----------------------------------------+

```

Cinder volumes will be created on different Storage nodes, depending on the backends type
```
# cinder create --display-name vol1 --volume-type lvm_silver 5
# cinder create --display-name vol2 --volume-type lvm_gold 5
# cinder list
+--------------------------------------+-----------+------+------+-------------+----------+-------------+-------------+
|                  ID                  |   Status  | Name | Size | Volume Type | Bootable | Multiattach | Attached to |
+--------------------------------------+-----------+------+------+-------------+----------+-------------+-------------+
| 579756be-6f03-4ded-8e65-dd46e6568e95 | available | vol1 |  5   |  lvm_silver |  false   |    False    |             |
| adb625f1-f212-45d6-8150-bc4687d1f0d1 | available | vol2 |  5   |   lvm_gold  |  false   |    False    |             |
+--------------------------------------+-----------+------+------+-------------+----------+-------------+-------------+
```
The ``vol1`` is created on the first Storage node while the ``vol2`` is created on the second Storage node.

In this section section, we defined two different storage backends with different storage type, each on a different Storage node. Below, we are going to assign the same backend name to two different Storage nodes. This option permits to migrate a volume of the same type from the first Storage node to the second one.

On the second Storage node, edit the ``/etc/cinder/cinder.conf`` file
```
...
[lvm1]
iscsi_helper=lioadm
volume_group=cinder-volumes
iscsi_ip_address=192.168.2.37
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_protocol=iscsi
volume_backend_name=silver

[lvm2]
iscsi_helper=lioadm
volume_group=cinder-volumes
iscsi_ip_address=192.168.2.37
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_protocol=iscsi
volume_backend_name=gold
```

Restart the Cinder service on the second Storage node
```
# systemctl restart openstack-cinder-volume
```

and check the services list running on all the Storage nodes
```
[root@osstorage02]# cinder-manage service list
Binary           Host                                   Zone             Status     State Updated At
cinder-scheduler oscontroller                           nova             enabled    :-)   2016-03-01 16:15:34
cinder-volume    osstorage01@lvm1                       nova             enabled    :-)   2016-03-01 16:15:34
cinder-volume    osstorage02@lvm1                       nova             enabled    :-)   2016-03-01 16:15:34
cinder-volume    osstorage02@lvm2                       nova             enabled    :-)   2016-03-01 16:15:34
[root@osstorage02]#
```

We see the same storage type running on different Storage nodes. The above configuration permits to migrate a volume of the same type (e.g. ``lvm_silver``) from the first Storage node to the second one.


### Default volume type
When creating volumes, Cinder set the volume type to ``none`` if not specified.
To specify the volume type
```
# cinder create --display-name vol1 --volume-type lvm_silver 1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|                   id                  | 6844d194-5d18-40a1-85fe-4fa79a32077c |
|                  size                 |                  1                   |
|                 status                |               creating               |
|                  name                 |                 vol1                 |
|              volume_type              |              lvm_silver              |
+---------------------------------------+--------------------------------------+
```

If the volume type is not specificed
```
# cinder create --display-name vol2 1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|                   id                  | cccac79e-7617-4a7f-920b-e3b5334bdf50 |
|                  size                 |                  1                   |
|                 status                |               creating               |
|                  name                 |                 vol2                 |
|              volume_type              |                 none                 |
+---------------------------------------+--------------------------------------+
```

To specify a default volume type, edit the ``cinder.conf`` configuration file on the Controller node
```
# vi /etc/cinder/cinder.conf
[DEFAULT]
...
default_volume_type = lvm_silver
...

```
and restart the Cinder services
```
# openstack-service restart cinder
```

Starting from now, the new volumes will be created with default type ``lvm_silver`` if not specified
```
# cinder create --display-name vol3 1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|                   id                  | cc9b5518-8d56-43d8-a337-db3c60703083 |
|                  size                 |                  1                   |
|                  name                 |                 vol3                 |
|                 status                |               creating               |
|              volume_type              |               lvm_silver             |
+---------------------------------------+--------------------------------------+
```

### NFS Cinder Storage Backend
Cinder Storage Service can use a Network File System storage as backend. Howewer, the Cinder service provides Block Storage devices (i.e. volumes) to the users even if the backend is NFS. This section explains how to configure OpenStack Block Storage to use NFS storage.

We are assuming a NFS server is already available on the Storage Network with IP 192.168.2.20 and the server is sharing the ``/var/shared`` folder. On the Storage node 192.168.2.36 install the NFS client.

```
# yum install -y nfs-utils
# systemctl start rpcbind
# systemctl enable rpcbind
```

On the Storage node, create a shared resource catalog
```
# vi /etc/cinder/nfs_shares
192.168.2.20:/var/shared
# chown root:cinder /etc/cinder/nfs_shares
# chmod 0640 /etc/cinder/nfs_shares
```

Configure the cinder volume service to use the ``/etc/cinder/nfs_shares`` file by editing the ``/etc/cinder/cinder.conf`` configuration file
```
[defaults]
...
enabled_backends = nfs
...
[nfs]
volume_driver = cinder.volume.drivers.nfs.NfsDriver
nfs_shares_config = /etc/cinder/nfs_shares
#nfs_mount_options = <nfs mounting options>
nfs_sparsed_volumes = true
volume_backend_name=nfs
```

Optional mounting options ``nfs_mount_options`` are the usual mount options to be used when accessing NFS shares. The ``nfs_sparsed_volumes`` configuration key determines whether volumes are created as sparse files and grown as needed or fully allocated up front. The default and recommended value is ``true``, which ensures volumes are initially created as sparse files. Setting the key to ``false`` will result in volumes being fully allocated at the time of creation. This leads to increased delays in volume creation.

Restart the Cinder services on both the Storage node
```
# openstack-service restart cinder
# cinder-manage service list
Binary           Host                                Zone             Status     State Updated At
cinder-scheduler oscontroller                        nova             enabled    :-)   2016-03-01 17:09:25
cinder-volume    osstorage01@lvm1                    nova             enabled    :-)   2016-03-01 17:09:26
cinder-volume    osstorage02@lvm2                    nova             enabled    :-)   2016-03-01 17:09:18
cinder-volume    osstorage01@lvm1                    nova             enabled    :-)   2016-03-01 17:09:18
cinder-volume    osstorage01@nfs                     nova             enabled    :-)   2016-03-01 17:09:17
```

Create the new Cinder backend type
```
# cinder type-create nfs
# cinder type-key nfs set volume_backend_name=nfs
# cinder type-list
+--------------------------------------+------------+-------------+-----------+
|                  ID                  |    Name    | Description | Is_Public |
+--------------------------------------+------------+-------------+-----------+
| 2e7a05d0-c3d4-4eee-880c-5cd707efa5e3 |   iscsi    |      -      |    True   |
| 5bb9a70c-64ee-412f-bb28-d909ba22c17c |    nfs     |      -      |    True   |
| 74055a9a-a576-49a3-92fa-dfa7a935cdba | lvm_silver |      -      |    True   |
| 9c975ae9-f16d-4a4a-a930-8ae0efb5fab8 |  lvm_gold  |      -      |    True   |
+--------------------------------------+------------+-------------+-----------+
```

Create new volumes on NFS Storage backend
```
# cinder create --display-name vol3 --volume-type nfs 5
```
The above volume can be attached as Block Storage device to the VM instances.

### Configure Volumes Backup Service
By default, Cinder uses the Object Storage service to backup the volumes. To achieve volumes backups, install first and start the Swift Object Storage service. 

On each Storage node, configure the backup service by editing the ``cinder.conf`` configuration file
```
[default]
...
backup_metadata_version = 2
backup_compression_algorithm = zlib
backup_driver = cinder.backup.drivers.swift
backup_manager = cinder.backup.manager.BackupManager
backup_api_class = cinder.backup.api.API
backup_swift_url = http://controller:8080/v1/AUTH_
backup_swift_auth = per_user
backup_swift_auth_version = 1
backup_swift_container = volumes_backup
backup_name_template = backup-%s
```

Start and enable the backup service
```
# systemctl start openstack-cinder-backup
# systemctl enable openstack-cinder-backup
```

If instead of Object Storage, we want to use an NFS export as the backup repository, enable the appropriate configuration options to the ``cinder.conf`` configuration file

```
[default]
...
backup_metadata_version = 2
backup_compression_algorithm = zlib
backup_driver = cinder.backup.drivers.nfs
backup_manager = cinder.backup.manager.BackupManager
backup_api_class = cinder.backup.api.API
backup_share = <nfs_server:shared>
backup_mount_point_base = /mnt/nfs/cinder_backup
```

Make sure the backup mount point is owned by Cinder and restart the backup service
```
# chown -R cinder:cinder /mnt/nfs/cinder_backup
# systemctl restart openstack-cinder-backup
```

