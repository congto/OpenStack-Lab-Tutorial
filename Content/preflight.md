# Initial server setup for an OpenStack deployment

The x86_64 platform is currently the only supported architecture. Machine with at least 4GB RAM, processors with hardware virtualization extensions (for compute nodes), and one or more network adapters. Both Intel and AMD CPU support virtualization technology which allows multiple operating systems to run simultaneously on an x86 computer in a safe and efficient manner using hardware virtualization. Use the following commands to verify that if hardware virtualization extensions is enabled or not in your BIOS. On the compute nodes type the following commands as root to verify that host cpu has support for Intel VT or AMD-V technology.

```
# grep --color vmx /proc/cpuinfo
# grep --color svm /proc/cpuinfo
```

## Install CentOS 7.2 Operating System
Install CentOS 7.2 minimal on all the target servers. This tutorial refers to a setup made of the following machines

|Management IP|Hostname|Usage|
|------------|-----------|-------------------|
|10.10.10.30 |oscontroller |Controller Node|
|10.10.10.32 |oscompute01 |Compute Node|
|10.10.10.34 |oscompute02 |Compute Node|
|10.10.10.36 |osstorage |Storage Node|
|10.10.10.38 |osnetwork |Network Node |

Update the system

``# yum update -y``

Since CentOS minimal comes without common tools installed, install them:

```
# yum install net-tools
# yum install bind-utils
```

Stop and disable the NetworkManager service, because we don’t need it on the server:
```
# systemctl stop NetworkManager 
# systemctl disable NetworkManager
```
Restart the network service

```
# systemctl restart network.service
```

Set the hostnames and update the ``/etc/hosts`` file
```
# hostnamectl set-hostname <hostname>
# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.10.30 oscontroller
10.10.10.32 oscompute01
10.10.10.34 oscompute02
10.10.10.36 osstorage
10.10.10.38 osnetwork
```

Install, configure and start the NTP service
```
# yum install -y ntp
# systemctl enable ntpd.service
# systemctl start ntpd.service
# ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ntp1.inrim.it   .CTD.            1 u    2   64    1   20.958    3.063   1.029
+ntp2.inrim.it   .CTD.            1 u    1   64    1   27.671    3.232   0.299
```

Setup the RDO repository

```
yum install -y https://rdoproject.org/repos/rdo-release.rpm
```
