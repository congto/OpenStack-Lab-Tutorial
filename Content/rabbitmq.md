# RabbitMQ Message Broker
All OpenStack services use the RabbitMQ messaging system to communicate. There are two ways to secure RabbitMQ communication: require a username and password (authentication), which ensures that only machines with this username and password can communicate with the other OpenStack services, or use SSL to encrypt communication, which helps to prevent snooping and injection of rogue commands in the communication channels.

When configuring RabbitMQ to use SSL, you may have to change the ports that the services listen on. TCP/5672 is the standard port for RabbitMQ, but SSL will use TCP/5671. You will likely have to change the rabbit_port to 5671 for Glance, Cinder, etc.

###Setup the RabbitMQ service
Install and start the RabbitMQ:
```
# yum install -y rabbitmq-server
# systemctl start rabbitmq-server
```
Confirm that RabbitMQ is listening on default ports 5671 (SSL enabled) or 5672 (unsecure).
``# netstat -nlp | grep 567.*``

Configure the service to start at boot
``# systemctl enable rabbitmq-server``

The rabbitmq-management plugin provides an HTTP-based API for management and monitoring of your RabbitMQ server, along with a browser-based UI
```
# rabbitmq-plugins enable rabbitmq_management
# systemctl restart rabbitmq-server
```

The web UI is located at: http://server-name:15672/

To use the web UI you will need to authenticate as a RabbitMQ user (on a fresh installation the user "guest" is created with password "guest"). From here you can manage exchanges, queues, bindings, virtual hosts, users and permissions. Hopefully the UI is fairly self-explanatory.
