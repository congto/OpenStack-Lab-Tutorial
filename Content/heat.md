# Heat Orchestration Service
Orchestration is the ability to automate the deployment and configuration of IT infrastructure. OpenStack Heat is the orchestration service that allow to spin up multiple instances, logical networks, and other cloud services in an automated fashion. More than just standing up virtual servers, Heat also manages scripts used to add VMs to networks, stands up multiple servers together as a stack and even installs application software.

Heat major components are:

  1. The **heat-api** component implements an OpenStack-native RESTful API. This components processes API requests by sending them to the Heat engine via AMQP.

  2. The **heat-api-cfn** component provides an API compatible with AWS CloudFormation, by forwarding API requests to the Heat engine over AMQP.

  3. The **heat-engine** component provides the main orchestration functionality.

All of these components would typically be installed on the Controller node even if there is nothing that requires them to be installed on other nodes. Like other OpenStack services, Heat uses a back-end MySQL database for maintaining state information.

The Heat Orchestration service is compatible with AWS CloudFormation by running the same CFN templates simplifying application portability between AWS and OpenStack. CFN templates are usually written in JSON language. The Heat Orchestration service also runs **Heat Orchestration Template** templates that are written in **YAML**. YAML is a non procedural notation that are similar to Python or Ruby. Therefore, it is easier to write, parse, grep, generate with tools, and maintain source-code management systems.

## Authorization model for orchestration
The Orchestration authorization model defines the authorization process for requests during its operations. For example during the auto-scaling procedure, the Orchestration service requests resources of other components, such as servers from compute or networks to extend or reduce the capacity of an auto-scaling group. The orchestration service needs for authorization to accomplish deferred operations like an auto-scaling. Authorization is based on two different models:

  1. Password authorization
  2. OpenStack Identity trusts authorization

The default is based on trusts authorization as specified in the ``heat.conf`` configuration file. To enable the trusts authorization model set ``deferred_auth_method=trusts`` or to enable the password authorization model set ``deferred_auth_method=password``.

## Implementing Heat
On the Controller node, install the Heat components

    # yum -y install openstack-heat-common openstack-heat-api openstack-heat-api-cfn openstack-heat-engine

Source the keystonerc_admin file to enable authentication with administrative privileges

    # source keystonerc_admin

Add user and roles for Heat services in Keystone

    # openstack user create --domain default --project service --password <servicepassword> heat
    # openstack role add --project service --user heat admin
    # openstack role create heat_stack_owner
    # openstack role create heat_stack_user
    # openstack role add --project admin --user admin heat_stack_owner

Create Heat services and the related end-points

    # openstack service create --name heat --description "orchestration" orchestration
    openstack endpoint create \
    --region RegionOne \
    --publicurl http://<controller_node>:8004/v1/%\(tenant_id\)s \
    --adminurl http://<controller_node>:8004/v1/%\(tenant_id\)s \
    --internalurl http://<controller_node>:8004/v1/%\(tenant_id\)s \
    orchestration

    # openstack service create --name heat-cfn --description "cloudformation" cloudformation
    openstack endpoint create \
    --region RegionOne \
    --publicurl http://<controller_node>:8000/v1 \
    --adminurl http://<controller_node>:8000/v1 \
    --internalurl http://<controller_node>:8000/v1 \
    cloudformation

Create a database for Heat 

    # openstack-db --init --service heat --password <password>

Configure Heat services by editing the ``/etc/heat/heat.conf`` configuration file

    # vi /etc/heat/heat.conf
    [DEFAULT]
    deferred_auth_method = trusts
    trusts_delegated_roles = heat_stack_owner
    # Heat installed server
    heat_metadata_server_url = http://10.10.10.30:8000
    heat_waitcondition_server_url = http://10.10.10.30:8000/v1/waitcondition
    heat_watch_server_url = http://10.10.10.30:8003
    heat_stack_user_role = heat_stack_user
    # Heat domain name
    stack_user_domain_name = heat
    # Heat domain admin name
    stack_domain_admin = heat_admin
    # Heat domain admin's password
    stack_domain_admin_password = <service_password>
    rpc_backend = rabbit
    [database]
    # MariaDB connection info
    connection = mysql://heat:password@10.10.10.30/heat
    # RabbitMQ connection info
    [oslo_messaging_rabbit]
    rabbit_host = 10.10.10.30
    rabbit_port = 5672
    rabbit_userid = guest
    rabbit_password = guest
    [ec2authtoken]
    # specify Keystone server
    auth_uri = http://10.10.10.30:5000/v2.0
    [heat_api]
    bind_host = 0.0.0.0
    bind_port = 8004
    workers = 0
    [heat_api_cfn]
    bind_host = 0.0.0.0
    bind_port = 8000
    workers = 0
    [keystone_authtoken]
    # specify Keystone auth info
    admin_user=heat
    admin_password=<admin_password>
    admin_tenant_name=services
    identity_uri=http://10.10.10.30:35357
    auth_uri=http://10.10.10.30:5000/v2.0

Start and enable Heat services

    # systemctl start openstack-heat-api openstack-heat-api-cfn openstack-heat-engine 
    # systemctl enable openstack-heat-api openstack-heat-api-cfn openstack-heat-engine 
