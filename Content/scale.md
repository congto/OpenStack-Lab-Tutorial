# Cloud Scaling
When a new OpenStack cloud is started, all servers run over the same hardware and all servers are in the same building, room, rack, even a chassis when the cloud is in the first growth paces. After a while, workloads increase and the current hardware set is not enough to process that workloads and new hardware is bought. This hardware has different storage disks, CPU, RAM and so on. Also racks, rooms and buildings are too small and the growing cloud needs redundancy between cities or regions.

OpenStack offers solutions for scaling the Cloud, called **Regions**, **Cells**, **Availability Zones** and **Host Aggregates**. Each method provides different functionality and can be best divided into two groups:

1. **Cells** and **Regions**: segregate an entire Cloud and result in running separate Compute deployments.
2. **Availability Zones** and **Host Aggregates**: divide a single Compute deployment.

**Cells** are still considered experimental.

**Regions** have a separate API endpoint per installation, allowing for a discrete separation of Compute deployments. Users wanting to run instances across sites have to explicitly select a region. Each region has a full nova installation, the only shared services are Keysone and Horizon. A different API endpoint exists for every region.

**Availability Zones** inside a region, represent a logical partition of the a single Compute deployment in the form of racks, rooms, buildings, etc. Users can run instances in the desired Availability Zone. Keystone and all nova services are shared between Availability Zones.

**Host Aggregates** represent a logical set of properties/characteristics a group of hosts owns in the form of metadata. For example, if some of servers have SSD and the other have SATA, you can map those properties to a group of hosts, when a image or flavor with the meta parameter associated is started, the Nova Scheduler will filter the available hosts with the meta parameter value and boot the instance on hosts with the desired property. Host Aggregates are managed by OpenStack admins only.


### Implementing Availability Zones and Host Aggregates
Considering an OpenStack Cloud made of a single Compute deployment. Default Availability Zone (AZ) is the _nova_ zone.
```
# nova availability-zone-list
+-----------------------+----------------------------------------+
| Name                  | Status                                 |
+-----------------------+----------------------------------------+
| internal              | available                              |
| |- controller         |                                        |
| | |- nova-conductor   | enabled :-) 2016-04-13T08:52:32.000000 |
| | |- nova-consoleauth | enabled :-) 2016-04-13T08:52:32.000000 |
| | |- nova-scheduler   | enabled :-) 2016-04-13T08:52:34.000000 |
| | |- nova-cert        | enabled :-) 2016-04-13T08:52:32.000000 |
| nova                  | available                              |
| |- compute03          |                                        |
| | |- nova-compute     | enabled :-) 2016-04-13T08:52:33.000000 |
| |- compute04          |                                        |
| | |- nova-compute     | enabled :-) 2016-04-13T08:52:34.000000 |
+-----------------------+----------------------------------------+
```

Creating an AZ, it actually creates a Host Aggregate with your wanted AZ name
```
# nova aggregate-create AZ01_SilverHosts AZ01
```
Add an host to the aggregate means add the host to the AZ too
```
# nova aggregate-add-host AZ01_SilverHosts compute03
# nova aggregate-add-host AZ01_SilverHosts compute04
# nova availability-zone-list
+-----------------------+----------------------------------------+
| Name                  | Status                                 |
+-----------------------+----------------------------------------+
| internal              | available                              |
| |- controller         |                                        |
| | |- nova-conductor   | enabled :-) 2016-04-13T09:05:02.000000 |
| | |- nova-consoleauth | enabled :-) 2016-04-13T09:05:02.000000 |
| | |- nova-scheduler   | enabled :-) 2016-04-13T09:05:04.000000 |
| | |- nova-cert        | enabled :-) 2016-04-13T09:05:02.000000 |
| AZ01                  | available                              |
| |- compute03          |                                        |
| | |- nova-compute     | enabled :-) 2016-04-13T09:05:03.000000 |
| |- compute04          |                                        |
| | |- nova-compute     | enabled :-) 2016-04-13T09:05:04.000000 |
| nova                  | not available                          |
+-----------------------+----------------------------------------+

# nova aggregate-details AZ01_SilverHosts
+----+------------------+-------------------+--------------------------+--------------------------+
| Id | Name             | Availability Zone | Hosts                    | Metadata                 |
+----+------------------+-------------------+--------------------------+--------------------------+
| 3  | AZ01_SilverHosts | AZ01              | 'compute03', 'compute04' | 'availability_zone=AZ01' |
+----+------------------+-------------------+--------------------------+--------------------------+
```

Create a second AZ and add one host to each AZ
```
# nova aggregate-create AZ02_SilverHosts AZ02
# nova aggregate-add-host AZ02_SilverHosts compute04
# nova availability-zone-list
+-----------------------+----------------------------------------+
| Name                  | Status                                 |
+-----------------------+----------------------------------------+
| internal              | available                              |
| |- controller         |                                        |
| | |- nova-conductor   | enabled :-) 2016-04-13T09:11:02.000000 |
| | |- nova-consoleauth | enabled :-) 2016-04-13T09:11:02.000000 |
| | |- nova-scheduler   | enabled :-) 2016-04-13T09:11:04.000000 |
| | |- nova-cert        | enabled :-) 2016-04-13T09:11:02.000000 |
| AZ01                  | available                              |
| |- compute03          |                                        |
| | |- nova-compute     | enabled :-) 2016-04-13T09:11:03.000000 |
| AZ02                  | available                              |
| |- compute04          |                                        |
| | |- nova-compute     | enabled :-) 2016-04-13T09:11:04.000000 |
| nova                  | not available                          |
+-----------------------+----------------------------------------+
```

Start a Virtual Machine specifying the AZ
```
# source keystonerc_tenant
# nova boot --flavor small --image cirros --nic net-id=tenant-network --availability-zone AZ01 VM-in-AZ01
# nova show VM-in-AZ01 | grep availability_zone
| OS-EXT-AZ:availability_zone          | AZ01                                                     |
```








