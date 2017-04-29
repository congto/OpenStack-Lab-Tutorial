# Keystone Authentication Service
The Keystone identity service is a project providing identity, token, catalog, and policy services for use with OpenStack. Keystone provides token and password-based authentication and high-level authorization, and a central directory of users mapped to the services they can access. The following commands should be available on the command-line path:

* **keystone** the Keystone client, used to interact with Keystone
* **keystone-manage** used to bootstrap Keystone data
* **keystone-all** used to run the Keystone services

You will find the configuration file in /etc/keystone/keystone.conf

## Deploying the Keystone Identity Service

Install the openstack-keystone package and the openstack-selinux package that will provide the SELinux policy.

``# yum install -y openstack-keystone openstack-selinux``

Install and start the MySQL server (MariaDB)
```
# yum install -y openstack-utils
# openstack-db --init --service keystone
```

Start the openstack-keystone service and make sure the service is persistent.

```
# systemctl start openstack-keystone
# systemctl enable openstack-keystone
```

Do the same for the MariaDB service:

``# systemctl enable mariadb.service``

Allow the incoming connections for the service:

```
# firewall-cmd --add-port=35357/tcp --permanent
# firewall-cmd --add-port=5000/tcp --permanent
# firewall-cmd --reload
```

Add Keystone as an end point in the registry of end points in Keystone, which is required for all the OpenStack services. Note that the ID returned from the service-create command is then used as a part of the endpoint-create command:

```
# keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |    Keystone Identity Service     |
|   enabled   |               True               |
|      id     | fc9f35131ae749f8b1577013d62f6d38 |
|     name    |             keystone             |
|     type    |             identity             |
+-------------+----------------------------------+
```

Create the end point for the service you just declared (make sure to use the ID the previous command returned):

```
# keystone endpoint-create --service-id fc9f35131ae749f8b1577013d62f6d38 --publicurl 'http://caldera:5000/v2.0' --adminurl 'http://caldera:35357/v2.0' --internalurl 'http://caldera:5000/v2.0'
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |    http://caldera:35357/v2.0     |
|      id     | 31f34a9169ba46e0be4c513b309813ff |
| internalurl |     http://caldera:5000/v2.0     |
|  publicurl  |     http://caldera:5000/v2.0     |
|    region   |            regionOne             |
|  service_id | fc9f35131ae749f8b1577013d62f6d38 |
+-------------+----------------------------------+
```

Check the output carefully for mistakes. If needed, delete the end point, then recreate it with the same command above

``# keystone endpoint-delete fc9f35131ae749f8b1577013d62f6d38 ``

## Managing Users with the keystone command
To setup of the Keystone environment, create an admin user. The admin user of the admin tenant has to be associated with an admin role. A keystonerc_admin script makes authentication as the admin user easy.

Create the admin user with a corresponding password <password>

```
# keystone user-create --name admin --pass <password>
+----------+---------------------------------------+
| Property |              Value                    |
+----------+---------------------------------------+
|  email   |                                       |
| enabled  |              True                     |
|    id    |  34567890abcdef1234567890abcdef12     |
|   name   |              admin                    |
| username |              admin                    |
+----------+---------------------------------------+
```

Create an admin role and an admin tenant

```
# keystone role-create --name admin
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|    id    | fad9876543210fad9876543210fad987 |
|   name   |              admin               |
+----------+----------------------------------+

# keystone tenant-create --name admin
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | 4567890abcdef1234567890abcdef123 |
|     name    |              admin               |
+-------------+----------------------------------+
```

Add the user from the admin tenant to the admin role

``# keystone user-role-add --user admin --role admin --tenant admin``

Test the keystonerc_admin file by running the command to list users, since only an administrator can perform this action.

```
# source /root/keystonerc_admin
[~(keystone_admin)]# keystone user-list
+----------------------------------+-------+---------+-------+
|                id                |  name | enabled | email |
+----------------------------------+-------+---------+-------+
| 34567890abcdef1234567890abcdef12 | admin |   True  |       |
+----------------------------------+-------+---------+-------+
```

If the /root/keystonerc_token script was created, source the script

``# source /root/keystonerc_token``

Add a new user called myuser with Member role as part of myopenstack tenant. Before you begin, ensure the keystonerc_admin script exists and can be used to set the admin environment.

Source the admin Keystone environment variables if not currently set. 

````
# source /root/keystonerc_admin
# [~(keystone_admin)]keystone user-create --name myuser --pass <password>
+----------+-------------------------------------------+
| Property |                   Value                   |
+----------+-------------------------------------------+
|  email   |                                           |
| enabled  |                    True                   |
|    id    |    90abcdef1234567890abcdef12345678       |
|   name   |                  myuser                   |
| username |                  myuser                   |
+----------+-------------------------------------------+

[~(keystone_admin)]# keystone role-create --name Member
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|    id    | 234567890abcdef1234567890abcdef1 |
|   name   |              Member              |
+----------+----------------------------------+

[~(keystone_admin)]# keystone tenant-create --name myopenstack
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | 890abcdef1234567890abcdef1234567 |
|     name    |           myopenstack            |
+-------------+----------------------------------+

[~(keystone_admin)]# keystone user-role-add --user myuser --role Member --tenant myopenstack
[~(keystone_admin)]# keystone user-role-list --user myuser --tenant myopenstack
```

Verify the user is configured correctly by attempting to obtain a Keystone token. Since non-admin users are not able to list users, use token-get to test whether the user exists and has correctly set environment variables in the keystonerc_myuser script.

```
[~(keystone_admin)]# source /root/keystonerc_myuser
[~(keystone_myuser)]# keystone token-get --wrap 60
+-----------+----------------------------------+
|  Property |              Value               |
+-----------+----------------------------------+
| expires   |        2013-03-19T13:29:37Z      |
| id        | 04b23f424b56440fb39378b844a754fe |
...
| tenant_id | 890abcdef1234567890abcdef1234567 |
| user_id   | 90abcdef1234567890abcdef12345678 |
+-----------+----------------------------------+
```

To delete myuser and its envinronment, source the admin Keystone environment variables if not currently set and then remove user, tenant and member

```
# source /root/keystonerc_admin
[~(keystone_admin)]$ keystone user-list
+----------------------------------+--------+---------+-------+
|                id                |  name  | enabled | email |
+----------------------------------+--------+---------+-------+
| 6e4c0f935a6b4a71bdec91924c803363 | admin  |   True  |       |
| d07bf0a35ae3403594d7ce140eb7a677 | myuser |   True  |       |
+----------------------------------+--------+---------+-------+
[~(keystone_admin)]$ keystone user-delete myuser
[~(keystone_admin)]$ keystone tenant-delete myopenstack
[~(keystone_admin)]$ keystone role-delete Member
```

Delete the /root/keystonerc_myuser file

``[~(keystone_admin)]$ rm -rf /root/keystonerc_myuser``

## Hack the Keystone Identity Service

Check the status of the keystone service

```
# openstack-status
== Keystone service ==
openstack-keystone:                     active
== Support services ==
openvswitch:                            active
dbus:                                   active
rabbitmq-server:                        active
== Keystone users ==
Warning keystonerc not sourced

# systemctl status openstack-keystone
openstack-keystone.service - OpenStack Identity Service (code-named Keystone)
   Loaded: loaded (/usr/lib/systemd/system/openstack-keystone.service; enabled)
   Active: active (running) since Mon 2015-01-19 18:51:06 CET; 58min ago
 Main PID: 4267 (keystone-all)
   CGroup: /system.slice/openstack-keystone.service
           ├─4267 /usr/bin/python /usr/bin/keystone-all
           ├─4274 /usr/bin/python /usr/bin/keystone-all
           ├─4275 /usr/bin/python /usr/bin/keystone-all
           ├─4276 /usr/bin/python /usr/bin/keystone-all
           └─4277 /usr/bin/python /usr/bin/keystone-all
```
Keystone service uses a MySQL (Maria DB) database. Connect to keystone database instance

```
# mysql -h caldera -P 3306 -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 73
Server version: 5.5.40-MariaDB-wsrep MariaDB Server, wsrep_25.11.r4026
Copyright (c) 2000, 2014, Oracle, Monty Program Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
MariaDB [(none)]> use keystone
Reading table information for completion of table and column names
...
Database changed
MariaDB [keystone]> show tables;
+-----------------------+
| Tables_in_keystone    |
+-----------------------+
| assignment            |
| credential            |
| domain                |
| endpoint              |
| group                 |
| id_mapping            |
| migrate_version       |
| policy                |
| project               |
| region                |
| revocation_event      |
| role                  |
| service               |
| token                 |
| trust                 |
| trust_role            |
| user                  |
| user_group_membership |
+-----------------------+
18 rows in set (0.00 sec)

MariaDB [keystone]> select *from service;
+----------------------------------+----------+------------------------------------------------------------------+---------+
| id                               | type     | extra                                                            | enabled |
+----------------------------------+----------+------------------------------------------------------------------+---------+
| f6b78f968f1c456ca493454cefa8a3ce | identity | {"name": "keystone", "description": "Keystone Identity Service"} |       1 |
+----------------------------------+----------+------------------------------------------------------------------+---------+
1 row in set (0.00 sec)
MariaDB [keystone]> quit
Bye
```

## Remove the Keystone Identity Service
To remove the Keystone service, first stop the service, then drop the keystone database and disable it

```
# systemctl stop openstack-keystone
# openstack-db --drop --service keystone
Please enter the password for the 'root' MySQL user:
Verified connectivity to MySQL.
Dropping 'keystone' database.
Complete!
# systemctl disable openstack-keystone
rm '/etc/systemd/system/multi-user.target.wants/openstack-keystone.service'
```

Stop and disable the MariaDB service

```
# systemctl stop mariadb.service
# systemctl disable mariadb.service
rm '/etc/systemd/system/multi-user.target.wants/mariadb.service'
```
