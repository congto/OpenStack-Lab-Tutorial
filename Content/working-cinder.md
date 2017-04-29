# Managing Block Storage Service
Volumes are block storage devices that you attach to instances to enable persistent storage. Persistent storage is not deleted when you terminate an instance. You can attach a volume to a running instance or detach a volume and attach it to another instance at any time. You can also delete a volume. Only administrative users can create volume types.

On the other side, the disks associated with the instances are ephemeral, meaning that the data is deleted when the instance is terminated. Snapshots created from a running instance will remain, but any additional data added to the ephemeral storage since last snapshot will be lost.

## Create a non bootable volume
In this example, boot an instance from an image and attach a non-bootable volume
```
# nova image-list
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| 8c28b5a7-3959-4f61-9f2f-b8ca720be3c2 | linux        | ACTIVE |        |
| 291c448c-2f1d-411e-9706-fab665ad6a33 | ubuntu-14.04 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+
```

Create a 10GB non bootable volume specifying the volume type (i.e. the backend) 
```
# cinder create --display-name my-volume 10 --volume-type lvm_gold
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Multiattach |             Attached
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
| f3af9952-8658-4cc1-ae72-2cb1574a1e28 | available |  my-volume   |  10  |   lvm_gold  |  false   |    False    |                     
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
```

Boot an instance from an image and attach the empty volume to the instance. Make sure to specify flavour, image, network interface and volume ID
```
# nova boot --flavor small --image ubuntu-14.04 --nic net-id=a284bf6b-ea4a-40f5-b573-d9393db5a6dc --block-device source=volume,id=f3af9952-8658-4cc1-ae72-2cb1574a1e28,dest=volume,shutdown=preserve myInstanceWithVolume
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ID                                   | Name                 | Status | Task State | Power State | Networks                    |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ecc04ee0-fdd0-4760-bcc7-969adf95c4cb | myInstanceWithVolume | ACTIVE | -          | Running     | tenant-network=192.168.1.15 |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
```

To detach the volume from the image, specify the instance name and the volume ID
```
# nova volume-detach myInstanceWithVolume f3af9952-8658-4cc1-ae72-2cb1574a1e28
```

## Create a bootable volume
In this example, create a bootable volume from an image and boot a persistent instance.

Make sure to specify:
* --flavor=ID
* --image=ID
* --nic net-id=ID
* --block-device source=volume,id=ID,dest=DEST,size=SIZE,shutdown={preserve|remove},bootindex=0

```
# nova list
# nova flavor-list
# neutron net-list
+--------------------------------------+-----------------------+-----------------------------------------------------+
| id                                   | name                  | subnets                                             |
+--------------------------------------+-----------------------+-----------------------------------------------------+
| a284bf6b-ea4a-40f5-b573-d9393db5a6dc | tenant-network        | 75208e5c-a3f0-4f9f-a763-dd3b256fec56 192.168.1.0/24 |
| 47f17be1-7a93-4045-85c7-12e3a3b30446 | dmz-network           | 358702ed-1d7a-4c72-a455-faff96b616ae 192.168.0.0/24 |
| 66cde443-7030-4413-816d-ee140442d89e | provider-flat-network | 4b57b58b-146e-4170-992e-40131df9e8c3 172.16.1.0/24  |
+--------------------------------------+-----------------------+-----------------------------------------------------+
# nova image-list
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| 8c28b5a7-3959-4f61-9f2f-b8ca720be3c2 | linux        | ACTIVE |        |
| 291c448c-2f1d-411e-9706-fab665ad6a33 | ubuntu-14.04 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+

# nova boot --flavor m1.small --nic net-id=a284bf6b-ea4a-40f5-b573-d9393db5a6dc --block-device source=image,id=291c448c-2f1d-411e-9706-fab665ad6a33,dest=volume,size=10,shutdown=preserve,bootindex=0 myInstanceFromVolume

# cinder list
+--------------------------------------+--------+------+------+-------------+----------+-------------+--------------------------------
|                  ID                  | Status | Name | Size | Volume Type | Bootable | Multiattach |             Attached to        
+--------------------------------------+--------+------+------+-------------+----------+-------------+--------------------------------
| 756dc2a8-444f-4f16-ab6a-1474300b4cde | in-use |      |  10  |   lvm_gold  |   true   |    False    | 5d4682a8-5791-4bc2-86fd-9aeff69f517a |
+--------------------------------------+--------+------+------+-------------+----------+-------------+--------------------------------
# nova list
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| ID                                   | Name                 | Status | Task State | Power State | Networks                    |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
| 5d4682a8-5791-4bc2-86fd-9aeff69f517a | myInstanceFromVolume | ACTIVE | -          | Running     | tenant-network=192.168.1.17 |
+--------------------------------------+----------------------+--------+------------+-------------+-----------------------------+
```

## Migrate a volume
In this section we create a volume of a given type and then migrate the volume from a backend to another one defined on a different node. The requirement for volume migration is that the destination backend must support the same volume type of the source backend.

Check the service list
```
# cinder-manage service list
Binary           Host                                 Zone             Status     State Updated At
cinder-scheduler oscontroller                         nova             enabled    :-)   2016-03-06 10:23:59
cinder-volume    osstorage01@lvm1                     nova             enabled    :-)   2016-03-06 10:24:01
cinder-volume    osstorage02@lvm2                     nova             enabled    :-)   2016-03-06 10:24:01
cinder-volume    osstorage02@nfs                      nova             enabled    :-)   2016-03-06 10:24:00
cinder-volume    osstorage02@lvm1                     nova             enabled    :-)   2016-03-06 10:23:56
```

Both the Storage nodes run the same backend type called ``lvm1``. This is achieved by the ``cinder.conf`` file.

On the first Storage node
```
[root@osstorage01 ~]# cat /etc/cinder/cinder.conf
...
[lvm1]
iscsi_helper=lioadm
volume_group=cinder-volumes
iscsi_ip_address=192.168.2.36
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_protocol=iscsi
volume_backend_name=silver
```

On the second Storage node
```
[root@osstorage02 ~]# cat /etc/cinder/cinder.conf
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

[nfs]
volume_driver = cinder.volume.drivers.nfs.NfsDriver
nfs_shares_config = /etc/cinder/nfs_shares
nfs_mount_point_base = $state_path/mnt
volume_backend_name=nfs
```

On the Controller node, check the available storage types
```
# cinder type-list
+--------------------------------------+------------+-------------+-----------+
|                  ID                  |    Name    | Description | Is_Public |
+--------------------------------------+------------+-------------+-----------+
| 5bb9a70c-64ee-412f-bb28-d909ba22c17c |    nfs     |      -      |    True   |
| 74055a9a-a576-49a3-92fa-dfa7a935cdba | lvm_silver |      -      |    True   |
| 9c975ae9-f16d-4a4a-a930-8ae0efb5fab8 |  lvm_gold  |      -      |    True   |
+--------------------------------------+------------+-------------+-----------+
```

As user, create a 5GB volume of type ``lvm_silver``
```
# cinder create --display-name vol1 --volume-type lvm_silver 5
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Multiattach |             Attached
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
| 5e45bfed-69db-46c9-9dd6-92d26315d82d | available | volToMigrate |  5   |  lvm_silver |  false   |    False    |                     
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
```

Volume migration is allowed only to the admin user. Login as admin user and check the volume status
```
# source keystonerc_admin
# cinder show 5e45bfed-69db-46c9-9dd6-92d26315d82d
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|                   id                  | 5e45bfed-69db-46c9-9dd6-92d26315d82d |
|            migration_status           |                 None                 |
|                  name                 |             volToMigrate             |
|         os-vol-host-attr:host         |        osstorage01@lvm1#silver       |
|     os-vol-mig-status-attr:migstat    |                 None                 |
|     os-vol-mig-status-attr:name_id    |                 None                 |
|                  size                 |                  5                   |
|                 status                |              available               |
|                user_id                |   67bfe7c3d56f4d8a9f243cc810d2980f   |
|              volume_type              |              lvm_silver              |
+---------------------------------------+--------------------------------------+
```

Take note of the following attributes:

* ``os-vol-host-attr:host`` =  the volume’s current storage backend
* ``os-vol-mig-status-attr:migstat`` = the status of this volume’s migration

Migrate this volume to the second Storage node
```
# cinder migrate 5e45bfed-69db-46c9-9dd6-92d26315d82d osstorage02@lvm1#silver
Request to migrate volume <Volume: 5e45bfed-69db-46c9-9dd6-92d26315d82d> has been accepted.

# cinder show 5e45bfed-69db-46c9-9dd6-92d26315d82d
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|                   id                  | 5e45bfed-69db-46c9-9dd6-92d26315d82d |
|            migration_status           |              migrating               |
|                  name                 |             volToMigrate             |
|         os-vol-host-attr:host         |       osstorage01@lvm1#silver        |
|     os-vol-mig-status-attr:migstat    |              migrating               |
|                  size                 |                  5                   |
|                 status                |              available               |
|              volume_type              |              lvm_silver              |
+---------------------------------------+--------------------------------------+

# cinder show 5e45bfed-69db-46c9-9dd6-92d26315d82d
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|                   id                  | 5e45bfed-69db-46c9-9dd6-92d26315d82d |
|            migration_status           |               success                |
|                  name                 |             volToMigrate             |
|         os-vol-host-attr:host         |       osstorage02@lvm1#silver        |
|     os-vol-mig-status-attr:migstat    |               success                |
|                  size                 |                  5                   |
|                 status                |              available               |
|              volume_type              |              lvm_silver              |
+---------------------------------------+--------------------------------------+
```

The migration is not visible to non-admin users. However, some operations are not allowed while a migration is taking place, such as attaching or detaching a volume and deleting a volume. If a user performs such an action during a migration, an error is returned.

Please, note you can migrate only detached volumes with no snapshots.

## Extend a volume
A volume can be extended in size. To resize your volume, you must first detach it from the server. Check if the volume is not attached to any server, then extend it
```
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Multiattach |             Attached
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
| 5e45bfed-69db-46c9-9dd6-92d26315d82d | available | volToExtend  |  8   |  lvm_silver |  false   |    False    |                     
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------

# cinder extend volToExtend 12
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Multiattach |             Attached
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
| 5e45bfed-69db-46c9-9dd6-92d26315d82d | available | volToExtend  |  12  |  lvm_silver |  false   |    False    |                     
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
```

## Create a volume snapshot
Volume snapshots capture the point in time state of a volume. A volume from which snapshots have been created cannot be deleted while any of these snapshots exist. Volumes must be in an unattached state before a snapshot can be taken from them. To use a snapshot, a new volume must be created from a snapshot since volume snapshots cannot be attached or used directly.

Volume snapshots are different from volume backups. Backups are full copies of volumes stored in Object Storage repository while snapshots are like a photo in time of a volume. Since backups are full copies of the volume, they take longer to create than snapshots. Once created, backups are independent of the original volume.

Create a volume snapshot
```
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Multiattach |             Attached
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
| 5e45bfed-69db-46c9-9dd6-92d26315d82d | available |   myVolume   |  12  |  lvm_silver |  false   |    False    |                     
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+---------------------
# cinder snapshot-create myVolume --name myVolumeSnapshot
+-------------+--------------------------------------+
|   Property  |                Value                 |
+-------------+--------------------------------------+
|  created_at |      2016-03-06T12:31:04.027864      |
| description |                 None                 |
|      id     | 4195c0c2-c350-458a-8ebd-afe08a5ad3f8 |
|   metadata  |                  {}                  |
|     name    |           myVolumeSnapshot           |
|     size    |                  3                   |
|    status   |               creating               |
|  volume_id  | 5e45bfed-69db-46c9-9dd6-92d26315d82d |
+-------------+--------------------------------------+
# cinder snapshot-list
+--------------------------------------+--------------------------------------+-----------+------------------+------+
|                  ID                  |              Volume ID               |   Status  |       Name       | Size |
+--------------------------------------+--------------------------------------+-----------+------------------+------+
| 4195c0c2-c350-458a-8ebd-afe08a5ad3f8 | 5e45bfed-69db-46c9-9dd6-92d26315d82d | available | myVolumeSnapshot |  3   |
+--------------------------------------+--------------------------------------+-----------+------------------+------+
```

To use the snapshot above, create a new volume from it
```
# cinder create --snapshot-id=4195c0c2-c350-458a-8ebd-afe08a5ad3f8 --display-name myVolFromSnap --volume-type lvm_silver 3
# cinder list 
+--------------------------------------+-----------+------------------------------+------+-------------+----------+-------------+-----
|                  ID                  |   Status  |             Name             | Size | Volume Type | Bootable | Multiattach |     
+--------------------------------------+-----------+------------------------------+------+-------------+----------+-------------+-----
| 44c3ed14-73d7-453d-950f-8297a034732a | available |        myVolFromSnap         |  3   |  lvm_silver |  false   |    False    |     
+--------------------------------------+-----------+------------------------------+------+-------------+----------+-------------+-----
```

Snapshot backends are typically colocated with volume backends in order to minimize latency during cloning. By contrast, a backup repository is usually located in a different location to protect the backup repository from any damage that might occur to the volume backend.

## Create a volume backup
By default, the Swift Object Store is used as the backup repository. Create a first volume backup. The first backup of a volume is always a full backup. Further backups of the same volume can be incremental. Attempting to do an incremental backup without any existing backups will fail.
```
# cinder list 
+--------------------------------------+-----------+------------------------------+------+-------------+----------+-------------+-----
|                  ID                  |   Status  |             Name             | Size | Volume Type | Bootable | Multiattach |     
+--------------------------------------+-----------+------------------------------+------+-------------+----------+-------------+-----
| 8fbb95ec-f21e-45bb-8ea4-819d750c7c33 | available |        myVolToBackup         |  3   |  lvm_silver |  false   |    False    |     
+--------------------------------------+-----------+------------------------------+------+-------------+----------+-------------+-----

# cinder backup-create myVolume --name=myVolumeBackup
+-----------+--------------------------------------+
|  Property |                Value                 |
+-----------+--------------------------------------+
|     id    | f64d251a-b0f2-4bc0-9e99-b336e4c1aab9 |
|    name   |            myVolumeBackup            |
| volume_id | 8fbb95ec-f21e-45bb-8ea4-819d750c7c33 |
+-----------+--------------------------------------+

# cinder backup-show f64d251a-b0f2-4bc0-9e99-b336e4c1aab9
+-----------------------+--------------------------------------+
|        Property       |                Value                 |
+-----------------------+--------------------------------------+
|   availability_zone   |                 nova                 |
|       container       |            volumes_backup            |
|       created_at      |      2016-03-07T16:33:23.000000      |
|      description      |                 None                 |
|      fail_reason      |                 None                 |
| has_dependent_backups |                False                 |
|           id          | f64d251a-b0f2-4bc0-9e99-b336e4c1aab9 |
|     is_incremental    |                False                 |
|          name         |            myVolumeBackup            |
|      object_count     |                  42                  |
|          size         |                  2                   |
|         status        |              available               |
|       volume_id       | 8fbb95ec-f21e-45bb-8ea4-819d750c7c33 |
+-----------------------+--------------------------------------+
```

Further backups of the same volume can be incremental
```
# cinder backup-create --incremental myVolume --name=myVolumeBackup-Incremental
+-----------+--------------------------------------+
|  Property |                Value                 |
+-----------+--------------------------------------+
|     id    | 1894f30e-53e2-41aa-91dc-4bea6e848f74 |
|    name   |      myVolumeBackup-Incremental      |
| volume_id | 8fbb95ec-f21e-45bb-8ea4-819d750c7c33 |
+-----------+--------------------------------------+

# cinder backup-show 1894f30e-53e2-41aa-91dc-4bea6e848f74
+-----------------------+--------------------------------------+
|        Property       |                Value                 |
+-----------------------+--------------------------------------+
|   availability_zone   |                 nova                 |
|       container       |            volumes_backup            |
|       created_at      |      2016-03-07T17:00:38.000000      |
|      description      |                 None                 |
|      fail_reason      |                 None                 |
| has_dependent_backups |                False                 |
|           id          | 1894f30e-53e2-41aa-91dc-4bea6e848f74 |
|     is_incremental    |                 True                 |
|          name         |      myVolumeBackup-Incremental      |
|      object_count     |                  7                   |
|          size         |                  2                   |
|         status        |              available               |
|       volume_id       | 8fbb95ec-f21e-45bb-8ea4-819d750c7c33 |
+-----------------------+--------------------------------------+
```

The incremental backup is based on a parent backup which is an existing backup with the latest timestamp. The parent backup can be a full backup or an incremental backup depending on the timestamp. There is an ``is_incremental`` flag that indicates whether a backup is incremental when showing details on the backup. Another flag, ``has_dependent_backups``, indicates that the backup has dependent backups. If it is true, attempting to delete this backup will fail.

An incremental backup captures any changes to the volume since the last backup. Performing full backups of a volume can become resource intensive as the volume size grows. On the other side, incremental backups allow to capture changes to the volume while minimizing resource usage.

To restore a volume from a backup
```
# cinder backup-list
# cinder backup-restore myVolumeBackupID
```

When restoring from an incremental backup, a list of backups is built based on the IDs of the parent backups. A full restore is performed based on the full backup first, then restore is done based on the incremental backup, laying on top of it in order.
