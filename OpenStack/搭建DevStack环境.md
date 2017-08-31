# 搭建DevStack环境

## 快速开始

### 安装Linux

DevStack环境适用于Ubuntu 16.04/17.04,Fedore 24/25,CentOS/RHEL 7,Debian和OpenSUSE。

如果没有更好的选择，推荐Ubuntu 16.04。

下面使用的是CentOS 7.3

``` shell
[root@Mitaka ~]# cat /etc/redhat-release 
CentOS Linux release 7.3.1611 (Core) 
[root@Mitaka ~]# uname -a 
Linux Mitaka 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```

网络状态

``` shell
[root@Mitaka ~]# ifconfig 
ens160: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.50.97  netmask 255.255.255.0  broadcast 192.168.50.255
        inet6 fe80::7314:f764:fe81:8e4a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:34:45:7b  txqueuelen 1000  (Ethernet)
        RX packets 104399  bytes 151291538 (144.2 MiB)
        RX errors 0  dropped 32  overruns 0  frame 0
        TX packets 55598  bytes 4009855 (3.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### 环境搭建

安装git

``` shell
[root@Mitaka ~]# yum install -y git 
```

关闭selinux、iptables

``` shell
[root@Mitaka keystone]# systemctl stop firewalld 
[root@Mitaka keystone]# systemctl disable firewalld
[root@Mitaka keystone]# setenforce 0 
[root@Mitaka keystone]# cat /etc/sysconfig/selinux 

	SELINUX=disabled
```

使用豆瓣pip源头

``` shell
[root@Mitaka ~]# mkdir -p ~/.pip
[root@Mitaka ~]# cat ~/.pip/pip.conf 
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/  
[install] 
trusted-host = mirrors.aliyun.com
```

使用阿里yum源

``` shell
[root@Mitaka ~]# yum install -y wget 
[root@Mitaka ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@Mitaka ~]# yum clean all 
[root@Mitaka ~]# yum makecache 
[root@Mitaka ~]# yum install epel-release -y
[root@Mitaka ~]# yum install python-pip -y
[root@Mitaka ~]# pip install --upgrade pip
[root@Mitaka ~]# pip install -U os-testr
```

### 安装阶段

创建stack用户

```shell
[root@Mitaka ~]# useradd -s /bin/bash -d /opt/stack -m stack
[root@Mitaka ~]# echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
```

下载DevStack

``` shell
[root@Mitaka ~]# sudo su - stack
[stack@Mitaka ~]$ cd
[stack@Mitaka ~]$ git clone https://git.openstack.org/openstack-dev/devstack
```

### 创建local.conf配置文件

``` shell
[stack@Mitaka ~]$ cd devstack/
[stack@Mitaka devstack]$ cat local.conf 
[[local|localrc]]
ADMIN_PASSWORD=daemon
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
```

### 开始安装

``` shell
[stack@Mitaka devstack]$ ./stack.sh 
```

这个过程会持续15-20分钟，这个时间取决于网络下载速度。



## 错误总结

### 出现 generate-subunit

``` shell
2016-07-27 08:40:04.711 | ./stack.sh:686:git_clone
2016-07-27 08:40:04.711 | /devstack/functions-common:444:git_timed
2016-07-27 08:40:04.711 | /devstack/functions-common:510:die
2016-07-27 08:40:04.713 | [ERROR] /devstack/functions-common:510 git call failed: [git clone git://git.openstack.org/openstack/requirements.git /opt/stack/requirements]
2016-07-27 08:40:05.715 | Error on exit
2016-07-27 08:40:05.716 | ./stack.sh: line 463: generate-subunit: command not found
```

解决办法：

``` shell
[root@Mitaka ~]# yum install python-pip -y
[root@Mitaka ~]# pip install --upgrade pip
[root@Mitaka ~]# pip install -U os-testr
```

### 出现网络不通

原因：在安装neutron之后，网卡ens160会加入到br-ex虚拟交换机上，导致网络出现故障（具体故障原因还需深究）

解决办法：注销ens160配置文件中的IP地址，新建br-ex配置文件，并配置上IP地址

``` shell
[root@Mitaka network-scripts]# cat ifcfg-ens160 
TYPE=Ethernet
BOOTPROTO=no
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
NAME=ens160
UUID=24117a8a-62ef-4415-ac3e-e274bb0f7ad3
DEVICE=ens160
ONBOOT=yes
#IPADDR=192.168.50.97
#PREFIX=24
#GATEWAY=192.168.50.1
#DNS1=114.114.114.114
TYPE=OVSPort
OVS_BRIDGE=br-ex
DEVICETYPE=ovs

[root@Mitaka network-scripts]# cat ifcfg-br-ex 
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.50.97
PREFIX=24
GATEWAY=192.168.50.1
DNS1=114.114.114.114
```

### 出现某个域名无法解析

``` shell
2017-08-31 02:23:30.452 | 
2017-08-31 02:23:30.453 | 
2017-08-31 02:23:30.456 | curl: (6) Could not resolve host: git.openstack.org; Unknown error
2017-08-31 02:23:30.457 | 
2017-08-31 02:23:30.458 | ERROR: could not install deps [-r/opt/stack/tempest/requirements.txt]; v = InvocationError('/opt/stack/tempest/tools/tox_install.sh https://git.openstack.org/cgit/openstack/requirements/plain/upper-constraints.txt -r/opt/stack/tempest/requirements.txt (see /opt/stack/tempest/.tox/tempest/log/venv-tempest-1.log)', 6)
2017-08-31 02:23:30.463 | ___________________________________ summary ____________________________________
```

解决办法：

