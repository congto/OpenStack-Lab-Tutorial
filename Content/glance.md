# The Glance Image Service
The Glance image server requires Keystone to be in place for identity management and authorization. It uses a MySQL database to store the metadata information for the images, such as their types, location, or size. Glance supports a variety of disk formats, such as:

|Format|Description|
|------|-----------|
|raw	|An unstructured disk image format.|
|vhd	|A common disk format used by virtual machine monitors from VMware, Xen, Microsoft, VirtualBox, and others.|
|vmdk	|Another common disk format supported by many common virtual machine monitors.|
|vdi	|A disk format supported by VirtualBox virtual machine monitor and the QEMU emulator.|
|iso	|An archive format for the data contents of an optical disc (e.g., CD-ROM).|
|qcow2	|A disk format supported by the QEMU emulator that can expand dynamically and supports the Copy on Write feature.|
|aki	|This indicates what is stored in Glance is an Amazon kernel image.|
|ari	|This indicates what is stored in Glance is an Amazon ramdisk image.|
|ami	|This indicates what is stored in Glance is an Amazon machine image.|

Glance also supports a variety of container formats:

|Format|Description|
|------|-----------|
|bare	|This indicates there is no container or metadata envelope for the image.|
|ovf	|This is the OVF container format.|
|ova	|This indicates what is stored in Glance is an OVA tar archive file.|
|aki	|This indicates what is stored in Glance is an Amazon kernel image.|
|ari	|This indicates what is stored in Glance is an Amazon ramdisk image.|
|ami	|This indicates what is stored in Glance is an Amazon machine image.|

### Implementing the Glance Image Service

On the Controller node, install and init the Image service

```
# yum install -y openstack-glance
# openstack-db --init --service glance --password <password>
```

Integrate Glance with Keystone. Make sure the services tenant exists before proceede
```
# source keystonerc_admin
# openstack user create --domain default --project service --password <service_password> glance
# openstack role add --project service --user glance admin
# openstack service create --name glance --description "OpenStack Image service" image
# openstack endpoint create --region RegionOne image public http://$controller:9292
# openstack endpoint create --region RegionOne image internal http://$controller:9292
# openstack endpoint create --region RegionOne image admin http://$controller:9292 
```

Configure the Glance api configuration to use Keystone as the identity service
```
# vi /etc/glance/glance-api.conf
[database]
connection=mysql://glance:<password>@controller/glance
[keystone_authtoken]
auth_uri=http://controller:5000
auth_url=http://controller:35357
auth_plugin=password
project_domain_id=default
user_domain_id=default
project_name=service
username=glance
password=<service_password>
flavor=keystone
```

Update the Glance registry configuration to use Keystone as the identity service
```
# vi /etc/glance/glance-registry.conf
[database]
connection=mysql://glance:<password>@controller/glance
[keystone_authtoken]
auth_uri=http://controller:5000
auth_url=http://controller:35357
auth_plugin=password
project_domain_id=default
user_domain_id=default
project_name=service
username=glance
password=<service_password>
flavor=keystone
```
Start and enable the services
```
# systemctl start openstack-glance-registry
# systemctl enable openstack-glance-registry
# systemctl start openstack-glance-api
# systemctl enable openstack-glance-api
```

### Configure Glance to use the Swift Storage Backend
In this section, we are going to re-configure the Glance image service to use backend store as Swift object storage. By default, Image service will use the local filesystem to store the images when uploading it. The default local filesystem store directory is ``/var/lib/glance/images/`` on the Controller node where the Glance image service is configured. The local filesystem store will not be helpful when scaling the environment.

On the Controller node, change the ``/etc/glance/glance-api.conf`` configuration file
```
[glance_store]
stores=file,http,swift
default_store = swift
#filesystem_store_datadir=/var/lib/glance/images/
swift_store_create_container_on_put = True
```

Restart the Glance services
```
# systemctl restart openstack-glance-api
# systemctl restart openstack-glance-registry
```
