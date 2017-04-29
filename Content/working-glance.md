# Working with Glance Service
The glance command can be used to manage images. If the image is already available in the desired format, get an image
```
# cd /data
# wget http://cloud.fedoraproject.org/fedora-19.x86_64.qcow2
```
Upload it to Glance.
```
# source /root/keystonerc_admin
# glance image-create --name "fedora-19" --disk-format qcow2 --container-format bare --file /data/fedora-19.x86_64.qcow2

# glance image-list
+--------------------------------------+-----------+-------------+------------------+-----------+--------+
| ID                                   | Name      | Disk Format | Container Format | Size      | Status |
+--------------------------------------+-----------+-------------+------------------+-----------+--------+
| 8badaf16-7fa0-44ae-97ce-807d2aa50892 | fedora-19 | qcow2       | bare             | 239534080 | active |
+--------------------------------------+-----------+-------------+------------------+-----------+--------+
```
If the system image is stored at a remote location, ``--location`` can be used to provide the URL directly without first downloading the system image. The ``--copy-from`` option is similar to the ``--location`` option, but will also populate the image in the Glance cache.

By default, images are stored in ``/var/lib/glance/images/`` of the node where glance service is running. Check out ``/etc/glance/glance-api.conf`` to see what format is being used.

```
# cat /etc/glance/glance-api.conf
...
[glance_store]
stores=file,http
default_store=file
filesystem_store_datadir=/var/lib/glance/images/
...
```

