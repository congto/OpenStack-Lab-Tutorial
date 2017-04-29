# Working with Heat Templates
In addition to the AWS CloudFormation templates, the OpenbStack Heat Orchestration service uses the Heat OpenStack Templates (**HOT**) written in **YAML**. YAML stands for “Yet Another Markup Language” and is refreshingly easy to read and understand by everyone.

Some basic terminology is in order to help navigate the YAML structure:

1. **Stack**: this is what we are creating: a collection of VMs and their associated configuration. In Heat, the term “Stack” has a very specific meaning. It refers to a collection of resources. Resources could be instances, networks, security groups, and even auto-scaling rules.

2. **Template**: this is the design of the resources that will make up the stack. For example, if we want to have two instances connected on a private network, we need to define a template for each instance as well as the network.

A template is made up of three different sections:
 * **Resources**: these are the details of the specific stack. These are the objects that will be created or modified when the template runs. Resources could be Instances, Volumes, Security Groups, Floating IPs, or any number of objects in OpenStack. A resouce has properties and attributes.
 * **Parameters**: these are the values assigned to the properties of a resource and that must be passed to the template. In HOT format, they appear before the resources section and are directly mapped to properties.
 * **Outputs**: these are the attributes of a resource and are passed back to the user. It may be displayed in the dashboard, or revealed in command line ``heat stack-show`` commands.

In the following sections, we are going to work with examples of heat templates:

1. [A very basic example](.working-heat.md#a-very-basic-example)
2. [An improved stack example](./working-heat.md#an-improved-stack-example)
3. [A networking resource example](./working-heat.md#a-networking-resource-example)
4. [A volume resource stack example](./working-heat.md#a-volume-resource-stack-example)
5. [A user data stack example](./working-heat.md#a-user-data-stack-example)
6. [A nested stack example](./working-heat.md#a-nested-stack-example)

Also we introduce the Heat Environment files and work with a [Simple environment example](./working-heat.md#simple-environment-example)

#### A very basic example
In this section we are going to implement a simple HOT template example for starting an instance. This is not so useful but it helps to understand the basics. Here the example:

```
# vi heat-template-01.yaml
heat_template_version: 2015-10-15
  description: Simple template to deploy a single compute instance
  
  resources:
    my_instance:
      type: OS::Nova::Server
      properties:
        image: cirros
        flavor: small
        key_name: demokey
        networks:
          - network: tenant_network
```

First, each template has to include a valid version and an optional description telling the user what the template is doing

    heat_template_version: 2015-10-15
      description: Simple template to deploy a single compute instance


In the **resources** section, there is just one type of resource: a server. We know it is a server because its type tells us it is an ``OpenStack Nova Server (OS::Nova::Server)``. We have given it a name: *“my_instance”*.

We want our server to have certain properties:
* image: this instance is based on a certain image that we have in our OpenStack glance repository
* flavor: it uses certain size or flavor
* key-name: we also want to control access to this instance by injecting a keyname
* network: the instance has to be placed on a specific network

To create the stack, make sure the user has the Heat Stack Owner role
```
# source keystonerc_admin
# openstack role list --user demo --project demo
+----------------------------------+------------------+---------+------+
| ID                               | Name             | Project | User |
+----------------------------------+------------------+---------+------+
| 9fe2ff9ee4384b1894a90878d3e92bab | _member_         | demo    | demo |
| be2de476946c43dfb71196a961eebc6a | heat_stack_owner | demo    | demo |
+----------------------------------+------------------+---------+------+
```

Login as user to the Controller node and run the stack creation
```
# source keystonerc_demo
# heat stack-create first_heat_stack -f first_heat_stack.yaml
# heat stack-list
+--------------------------------------+------------------+-----------------+---------------------+--------------+
| id                                   | stack_name       | stack_status    | creation_time       | updated_time |
+--------------------------------------+------------------+-----------------+---------------------+--------------+
| b546cd6d-38b6-4ef0-91fa-54314093e305 | first_heat_stack | CREATE_COMPLETE | 2016-08-19T13:34:48 | None         |
+--------------------------------------+------------------+-----------------+---------------------+--------------+
```

After stack creation completes, check the instance created
```
# nova list --fields name
+--------------------------------------+-------------------------------------------+
| ID                                   | Name                                      |
+--------------------------------------+-------------------------------------------+
| c58fc820-a2a4-403e-b128-c473d960856b | first_heat_stack-my_instance-7fnnigpd6s3b |
+--------------------------------------+-------------------------------------------+
```

The stack can be deleted including all resources created
```
# heat stack-delete first_heat_stack
# nova list --fields name
+--------------------------------------+-------+
| ID                                   | Name  |
+--------------------------------------+-------+
+--------------------------------------+-------+
```
The complete first_heat_stack.yaml file can be found [here](../heat/first-heat-stack.yaml)


#### An improved Stack example
In the above example, all the properties are hardcoded into the **resources** section. Alternatively, we can specify those values as input parameters in the **parameters** section and ask the user to provide the values. In this way, the user can start different stacks without change the template file

```
# vi second_heat_stack.yaml
heat_template_version: 2015-10-15
description: Simple template to deploy a single compute instance

parameters:
  image:
    type: string
    label: Image name or ID
    description: Image to be used for compute instance
  flavor:
    type: string
    label: Flavor
    description: Type of instance (flavor) to be used
  key:
    type: string
    label: Key name
    description: Name of key-pair to be used for compute instance
  private_network:
    type: string
    label: Private network name or ID
    description: Network to attach instance to.
    default: tenant-network

resources:
  my_instance:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      key_name: { get_param: key }
      flavor: { get_param: flavor }
      networks:
        - network: { get_param: private_network }
```

Login as user to the Controller node and run the stack creation by providing the requested input parameters
```
# source keystonerc_demo
# heat stack-create second_heat_stack -f second_heat_stack.yaml
ERROR: The Parameter (image) was not provided.

# heat stack-create second_heat_stack \
-f second_heat_stack.yaml \
-P "image=cirros;key=demokey;flavor=small;private_network=tenant-network"

# heat stack-list
+--------------------------------------+----------------+-----------------+---------------------+--------------+
| id                                   | stack_name     | stack_status    | creation_time       | updated_time |
+--------------------------------------+----------------+-----------------+---------------------+--------------+
| 1d1f4824-91a3-48c4-9828-b9fbf49e28f8 | second_heat_st | CREATE_COMPLETE | 2016-08-19T14:03:42 | None         |
+--------------------------------------+----------------+-----------------+---------------------+--------------+
```

We can also define default values for input parameters which will be used in case the user does not provide the parameter during deployment. In addition, we can restrict the imput value to a specific set of predefined values. For example, the following definition for the flavor parameter will select the *"small"* flavor unless specified otherwise by the user. Also the user is forced to choise between a predefined list of flavors
```
parameters:
  flavor:
    type: string
    label: Flavor
    description: Type of instance (flavor) to be used
    constraints:
    - allowed_values: [small, medium, large]
    default: small
```

In the **output** section, we can specify outputs to the user: the name of the instance and the IP address. Otherwise, the user would have to look it up themselves once the stack has been created.
```
 outputs:
  instance_name:
    description: Name of the instance
    value: { get_attr: [my_instance, name] }
  instance_ip:
    description: IP address of the instance
    value: { get_attr: [my_instance, first_address] }
```

After stack creation, check the output from the stack
```
# heat stack-show second_heat_stack | grep output
|     "output_value": "second_heat_stack-my_instance-5q5knjktvy5b",                                           
|     "output_key": "instance_name"                                                                           
|     "output_value": "192.168.1.25",                                                                         
|     "output_key": "instance_ip"                                                                             
```

The complete second_heat_stack.yaml file can be found [here](../heat/second-heat-stack.yaml)


#### A networking resource example
In the examples above, we defined a template creating a single resource of Compute type ``OS::Nova::Server``. In this section, we are going to define other resources of Network type. The new template requires an external network parameter that can be used as a source of floating IP addresses. With just this information (along with the server parameters), the template creates its own private network, plus a router that connects it to the outside world.

Add the external network parameter to the template:
```
parameters:
...
  public_network:
    type: string
    label: Public network name or ID
    description: Public network with floating IP addresses.
    default: provider-network
  ...
```

Add the new resources to create private network, subnet and router
```
resources:
  ...
  private_network:
    type: OS::Neutron::Net

  private_subnet:
    type: OS::Neutron::Subnet
    properties:
      network_id: { get_resource: private_network }
      cidr: 192.168.100.0/24
      dns_nameservers:
        - 8.8.8.8

  router:
    type: OS::Neutron::Router
    properties:
      external_gateway_info:
        network: { get_param: public_network }

  router-interface:
    type: OS::Neutron::RouterInterface
    properties:
      router_id: { get_resource: router }
      subnet: { get_resource: private_subnet }
```

Add other resources to create the instance, the floating IP and the related association between them

```
resources:
  ...
  floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: { get_param: public_network }

  my_instance:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      key_name: { get_param: key }
      flavor: { get_param: flavor }
      networks:
        - network: { get_resource: private_network }

  floating_ip_assoc:
    type: OS::Nova::FloatingIPAssociation
    properties:
      floating_ip: { get_resource: floating_ip }
      server_id: { get_resource: my_instance }
```

Add some useful output to the user
```
outputs:
 instance_name:
   description: Name of the instance
   value: { get_attr: [my_instance, name] }
 instance_ip:
   description: IP address of the instance
   value: { get_attr: [my_instance, first_address] }
 floating_ip:
   description: Floating IP address assigned to the instance
   value: { get_attr: [floating_ip, floating_ip_address] }
 private_network:
   description: Private network name assigned to the instance
   value: { get_attr: [private_network, name] }
```

Create the stack and check the output
```
# heat stack-create net-heat-stack -f net-heat-stack.yaml -P "public_network=provider-network"
# heat stack-show net-heat-stack | grep output
|     "output_value": "net-heat-stack-my_instance-xefchgawjhsl", 
|     "output_key": "instance_name" 
|     "output_value": "net-heat-stack-private_network-464lgqhodqc6", 
|     "output_key": "private_network" 
|     "output_value": "192.168.100.3",
|     "output_key": "instance_ip" 
|     "output_value": "172.120.1.213",
|     "output_key": "floating_ip" 
```
Note that we created the stack providing only the ``public_network `` parameter. The server related parameters, as image, flavor, etc. are set with their own default values specified into the **parameters** section of the template.

The complete net-heat-stack.yaml file can be found [here](../heat/net-heat-stack.yaml)


#### A volume resource stack example
With the Heat templates, users can create any type of stacks and any type of cloud resources. In this example we are going to setup a stack made of two server instance, a Web Server and a Database Server with a persistent volume of storage attached. The new template requires (along the servers parameters) the following parameters: private network name where the servers are running and the volume size in GB of the attached storage.

Add the private network parameter to the template:
```
parameters:
...
  private_network:
    type: string
    label: Private network name or ID
    description: Network to attach instance to.
```

Add the volume size parameter
```
parameters:
...
  volume_size:
    type: string
    label: size of volume
    description: This is the size GB of the volume to be attached to the Database Server for persistent storage
```

Add the resources for the Web Server, the Dabase Server and the Volume
```
resources:
  my_web:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      flavor: { get_param: flavor }
      key_name: { get_param: key }
      networks:
        - network: { get_param: private_network }

  my_db:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      flavor: { get_param: flavor }
      key_name: { get_param: key }
      networks:
        - network: { get_param: private_network }

  my_vol:
   type: OS::Cinder::Volume
   properties:
      size: { get_param: volume_size }

  vol_attachment:
   type: OS::Cinder::VolumeAttachment
   properties:
      instance_uuid: { get_resource: my_db }
      volume_id: { get_resource: my_vol }
```

Add some useful output to the user
```
outputs:
 web_instance_name:
   description: Name of the web instance
   value: { get_attr: [my_web, name] }
 web_instance_ip:
   description: IP address of the web instance
   value: { get_attr: [my_web, first_address] }
 db_instance_name:
   description: Name of the db instance
   value: { get_attr: [my_db, name] }
 db_instance_ip:
   description: IP address of the db instance
   value: { get_attr: [my_db, first_address] }
 volume_name:
   description: Volume name attached to the db instance
   value: { get_attr: [my_vol, display_name] }
 volume_type:
   description: Volume type attached to the db instance
   value: { get_attr: [my_vol, volume_type] }
```

Create the stack and check the output
```
# heat stack-create vol-heat-stack \
-f vol-heat-stack.yaml \
-P "private_network=tenant-network;volume_size=10"

# heat stack-show vol-heat-stack | grep output

|     "output_key": "web_instance_name"
|     "output_value": "vol-heat-stack-my_web-6gesxcofz5ai",
|     "output_key": "web_instance_ip"
|     "output_value": "192.168.1.13",                                                                         
|     "output_key": "db_instance_name"
|     "output_value": "vol-heat-stack-my_db-snpzic7w4z3w",
|     "output_key": "db_instance_ip"
|     "output_value": "192.168.1.12", 
|     "output_key": "volume_name"
|     "output_value": "vol-heat-stack-my_vol-7qujfgwhcshc",
|     "output_key": "volume_type"
|     "output_value": "nfs",
```

The complete vol-heat-stack.yaml file can be found [here](../heat/vol-heat-stack.yaml)

#### A user data stack example
In the previous example, we create a volume resource and attached it to a compute server. However, that volume is still not usable from the compute server since it needs to be formatted before to be available. In this section, we are going to create a user data script to format the volume after it has been attached to the instance. To accomplish this task, we are use the *cloud-init* capabilities.

The cloud-init package is the standard for initialization of cloud instances. It comes pre-installed on the cloud images, including the Cirros images used for OpenStack testing. Cloud-init runs during the initial boot of an instance, and it contacts the Nova metadata service to get any actions that need to be executed at boot time. The standard way to interact with the cloud-init service is by providing a bash script to run. The script is executed as the root user, so it has full access to the instance.

We change the above volume template by forcing the device where the volume has to be attached

```
resources:
...
  my_vol:
   type: OS::Cinder::Volume
   properties:
      size: { get_param: volume_size }

  vol_attachment:
   type: OS::Cinder::VolumeAttachment
   properties:
      instance_uuid: { get_resource: my_db }
      volume_id: { get_resource: my_vol }
      mountpoint: /dev/vdc
```


Providing a script to format the attached volume as xfs file system
```
resources:
...
  my_db:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      flavor: { get_param: flavor }
      key_name: { get_param: key }
      networks:
        - network: { get_param: private_network }
      user_data: |
        #!/bin/bash
         mkfs.ext4 /dev/vdc
         echo '/dev/vdc /mnt ext4 defaults 1 1' >> /etc/fstab
         mount -a
```

Create the stack, login to the server and check the volume is correctly formatted and mounted as espected
```
# heat stack-create userdata-heat-stack \
-f _userdata-heat-stack.yaml \
-P "image=ubuntu;private_network=tenant-network;volume_size=1"

# ssh -i demokey.pem ubuntu@userdata-heat-stack-my-db
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-93-generic x86_64)
ubuntu@userdata-heat-stack-my-db:~$ df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/vda1      ext4      7.9G  787M  6.8G  11% /
/dev/vdb       ext4      976M  1.3M  908M   1% /mnt
...
```

The complete template can be found here: [userdata-heat-stack.yaml](../heat/userdata-heat-stack.yaml)

#### A nested stack example
When need to deploy a multi resources application with Heat, is to just put several resources into the template, along with their parameters and outputs. This approach can work, but it leads to very large template files that are very hard to debug or update. To keep things easier, we can split complex templates into smaller sub-templates, and use nesting to combine the parts into the whole picture. In this section we are going to split a multiserver stack made of a webserver and a database into several small and reusable templates.

In the main template we define two custom resources, one for the webserver and another one for the database
```
resources:
  web:
    type: ./webserver.yaml
    properties:
      server_image: { get_param: image }
      server_flavor: { get_param: flavor }
      server_key: { get_param: key }
      server_network: { get_param: private_network }

  db:
    type: ./database.yaml
    properties:
      server_image: { get_param: image }
      server_flavor: { get_param: flavor }
      server_key: { get_param: key }
      server_network: { get_param: private_network }
      volume_size: { get_param: volume_size }
```

The above snippet creates two custom resources that have their type set into the YAML files of the two sub-templates: ``webserver.yaml`` and ``database.yaml``. In the context of the master template ``nested-heat-stack.yaml``, all the resources defined in the sub-template are encapsulated in single resources. Resources that reference a nested template also have properties and attributes, like other regular resources. We can specify the parameters to be assigned to the resource's properties
```
parameters:
  image:
    type: string
    label: Image name or ID
    description: Image to be used for compute instance
    constraints:
    - allowed_values: [cirros, centos, ubuntu]
    default: cirros
  flavor:
    type: string
    label: Flavor
    description: Type of flavor to be used
    constraints:
    - allowed_values: [small, medium, large]
    default: small
  key:
    type: string
    label: Key name
    description: Name of key-pair to be used for compute instance
    default: demokey
  private_network:
    type: string
    label: Private network name or ID
    description: Network to attach instance to.
  volume_size:
    type: string
    label: size of volume
    description: This is the size of the Volume
    default: 3
```

and reference the resource's attributes as output for the user
```
outputs:
 web_instance_name:
   description: Name of the web instance
   value: { get_attr: [web, server_name] }
 web_instance_ip:
   description: IP address of the web instance
   value: { get_attr: [web, server_address] }
 db_instance_name:
   description: Name of the db instance
   value: { get_attr: [db, server_name] }
 db_instance_ip:
   description: IP address of the db instance
   value: { get_attr: [db, server_address] }
 volume_name:
   description: Volume name attached to the db instance
   value: { get_attr: [db, volume_name] }
 volume_type:
   description: Volume type attached to the db instance
   value: { get_attr: [db, volume_type] }
```

The properties of a nested resource are the parameters defined in the sub-template, while the attributes of a nested resource are its outputs defined in the sub-template. This is extremely powerful, as a nested resource can be seen as a specialized resource that can be written to be a black-box through its inputs and outputs. So, we have:

|Main Template| |Sub Template|
|-------------|------|------------|
|resource's properties|>>|template's parameters|
|resource's attributes|<<|template's outputs|

With this in mind, we can define the webserver resource in the ``webserver.yaml`` sub-template:
```
resources:
  server:
    type: OS::Nova::Server
    properties:
      image: { get_param: server_image }
      flavor: { get_param: server_flavor }
      key_name: { get_param: server_key }
      networks:
        - network: { get_param: server_network }
```
as well as the parameters:
```
parameters:
  server_image:
    type: string
    label: Image name or ID
    description: Image to be used for compute instance
  server_flavor:
    type: string
    label: Flavor
    description: Type of flavor to be used
  server_key:
    type: string
    label: Key name
    description: Name of key-pair to be used for compute instance
  server_network:
    type: string
    label: Private network name or ID
    description: Network to attach instance to.
```

and the outputs:
```
outputs:
 server_name:
   description: Name of the server instance
   value: { get_attr: [server, name] }
 server_address:
   description: IP address of the server instance
   value: { get_attr: [server, first_address] }
```

Similarly, we can define the database resource by hyding the complexity of server and volume creation, the volume attachment and formatting into a black-box style ``database.yaml`` sub-template:
```
resources:
  server:
    type: OS::Nova::Server
    properties:
      image: { get_param: server_image }
      flavor: { get_param: server_flavor }
      key_name: { get_param: server_key }
      networks:
        - network: { get_param: server_network }
      user_data_format: RAW
      user_data: |
        #!/bin/bash
        mkfs.ext4 /dev/vdb
        echo '/dev/vdb /mnt ext4 defaults 1 1' >> /etc/fstab
        mount -a
  volume:
   type: OS::Cinder::Volume
   properties:
      size: { get_param: volume_size }

  volume_attachment:
   type: OS::Cinder::VolumeAttachment
   properties:
      instance_uuid: { get_resource: server }
      volume_id: { get_resource: volume }
      mountpoint: /dev/vdb
```

with input parameters:
```
parameters:
  server_image:
    type: string
    label: Image name or ID
    description: Image to be used for compute instance
  server_flavor:
    type: string
    label: Flavor
    description: Type of flavor to be used
  server_key:
    type: string
    label: Key name
    description: Name of key-pair to be used for compute instance
  server_network:
    type: string
    label: Private network name or ID
    description: Network to attach instance to.
  volume_size:
    type: string
    label: size of volume
    description: This is the size of the Volume
```

and outputs:
```
outputs:
 server_name:
   description: Name of the server instance
   value: { get_attr: [server, name] }
 server_address:
   description: IP address of the server instance
   value: { get_attr: [server, first_address] }
 volume_name:
   description: Volume name attached to the server instance
   value: { get_attr: [volume, display_name] }
 volume_type:
   description: Volume type attached to the server instance
   value: { get_attr: [volume, volume_type] }
```

All the main template and sub-templates can be found here [nested-heat-stack.yaml](../heat/nested-heat-stack.yaml), [webserver.yaml](../heat/webserver.yaml) and [database.yaml](../heat/database.yaml) 

#### Simple environment example
Heat provides an alternative way to nested templates by using the so called **Environment Files**. An environment file is a YAML file that has global definitions that are imported in the Heat engine before the template itself is parsed. An environment file can be used to define custom resource types defined into external template files.

Consider the following ``simple-environment.yaml`` file:
```
resource_registry:
  OS::Application::WebServer: ./webserver.yaml
  OS::Application::DataBase: ./database.yaml
```

The ``resource_registry`` section in the file above contains a mapping of custom resource types described in the external templates that implement them. We assigned a custom prefix as namespace ``OS::Application`` and the type: ``WebServer`` and ``DataBase``. After Heat imports this environment file, it can recognize the custom types, so the resource section of the main template used above can be rewritten as:
```
resources:
  web:
    type: OS::Application::WebServer
    properties:
      server_image: { get_param: image }
      server_flavor: { get_param: flavor }
      server_key: { get_param: key }
      server_network: { get_param: private_network }

  db:
    type: OS::Application::DataBase
    properties:
      server_image: { get_param: image }
      server_flavor: { get_param: flavor }
      server_key: { get_param: key }
      server_network: { get_param: private_network }
      volume_size: { get_param: volume_size }
```

To create the stack using environment files use the following notation:
```
# heat stack-create env-heat-stack \
-f env-heat-stack.yaml \
-e simple-environment.yaml \
-P "private_network=tenant-network"
```

In the Heat environment files we can define default values for all templates, including the main-template and all the nested sub-templates. This is accomplished by defining default values in the *parameter_defaults:* section of the environment file. These defaults are passed into all template resources.

For example, we can force the image type to be used for all servers by defining the default value in the ``simple-environment.yaml`` file:
```
resource_registry:
  OS::Application::WebServer: ./webserver.yaml
  OS::Application::DataBase: ./database.yaml

parameter_defaults:
  image: ubuntu
```

This value will be loaded by the Heat engine before the main template file parsing and applied to all server resources defined in the sub-templates along with the main-template.

The environment file and the main template file can be found here: [main-heat-stack.yaml](../heat/main-heat-stack.yaml) and [simple-environment.yaml](../heat/simple-environment.yaml).
