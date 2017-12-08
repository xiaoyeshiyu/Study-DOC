# 搭建DevStack环境

## 快速开始

### 安装Linux

DevStack环境适用于Ubuntu 16.04/17.04,Fedore 24/25,CentOS/RHEL 7,Debian和OpenSUSE。

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

当安装出现问题的时候，首先清除当前环境，然后修改配置文件之后再次执行

```
[stack@Mitaka devstack]$ ./unstack.sh 
[stack@Mitaka devstack]$ ./stack.sh 
```

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

### 出现nova-api did not start

``` shell
[ERROR] /opt/stack/devstack/lib/nova:816 nova-api did not start
```

根据

> http://openstack.10931.n7.nabble.com/can-t-install-devstack-nova-api-did-not-start-td17280.html  

所描述

![nova-api did not start](https://cl.ly/3R1U1o191m30/Image%202017-09-01%20at%205.27.37%20PM.png)

解决办法：

``` shell
[root@Mitaka nova]# pip install oslo.config
```

### 出现keystone not start 

``` shell
[ERROR] /opt/stack/devstack/lib/keystone:567 keystone did not start
```

解决办法：

在local.conf中添加

``` shell
HOST_IP=192.168.50.97
SERVICE_HOST=127.0.0.1
```

### 第二个节点出现

``` shell
2017-12-08 03:39:15.552 | +functions:wait_for_compute:455            echo 'Didn'\''t find service registered by hostname after 60 seconds'
2017-12-08 03:39:15.552 | Didn't find service registered by hostname after 60 seconds
2017-12-08 03:39:15.558 | +functions:wait_for_compute:456            openstack --os-cloud devstack-admin --os-region RegionOne compute service list
2017-12-08 03:39:18.400 | +----+------------------+---------+----------+---------+-------+----------------------------+
2017-12-08 03:39:18.400 | | ID | Binary           | Host    | Zone     | Status  | State | Updated At                 |
2017-12-08 03:39:18.400 | +----+------------------+---------+----------+---------+-------+----------------------------+
2017-12-08 03:39:18.400 | |  3 | nova-scheduler   | stack01 | internal | enabled | up    | 2017-12-07T09:25:01.000000 |
2017-12-08 03:39:18.400 | |  5 | nova-consoleauth | stack01 | internal | enabled | up    | 2017-12-07T09:24:59.000000 |
2017-12-08 03:39:18.401 | |  6 | nova-conductor   | stack01 | internal | enabled | up    | 2017-12-07T09:24:57.000000 |
2017-12-08 03:39:18.401 | |  1 | nova-conductor   | stack01 | internal | enabled | up    | 2017-12-07T09:24:57.000000 |
2017-12-08 03:39:18.401 | |  2 | nova-compute     | stack01 | nova     | enabled | up    | 2017-12-07T09:25:05.000000 |
2017-12-08 03:39:18.401 | +----+------------------+---------+----------+---------+-------+----------------------------+
2017-12-08 03:39:18.488 | +functions:wait_for_compute:458            return 124
2017-12-08 03:39:18.496 | +lib/nova:is_nova_ready:1                  exit_trap
2017-12-08 03:39:18.504 | +./stack.sh:exit_trap:493                  local r=124
2017-12-08 03:39:18.512 | ++./stack.sh:exit_trap:494                  jobs -p
2017-12-08 03:39:18.520 | +./stack.sh:exit_trap:494                  jobs=
2017-12-08 03:39:18.528 | +./stack.sh:exit_trap:497                  [[ -n '' ]]
2017-12-08 03:39:18.535 | +./stack.sh:exit_trap:503                  '[' -f /tmp/tmp.rcd8I7zW4x ']'
2017-12-08 03:39:18.541 | +./stack.sh:exit_trap:504                  rm /tmp/tmp.rcd8I7zW4x
2017-12-08 03:39:18.549 | +./stack.sh:exit_trap:508                  kill_spinner
2017-12-08 03:39:18.556 | +./stack.sh:kill_spinner:407               '[' '!' -z '' ']'
2017-12-08 03:39:18.563 | +./stack.sh:exit_trap:510                  [[ 124 -ne 0 ]]
2017-12-08 03:39:18.569 | +./stack.sh:exit_trap:511                  echo 'Error on exit'
2017-12-08 03:39:18.569 | Error on exit
```

解决办法

·

## 安装成功

安装成功之后，界面会显示

![DevStack安装成功图](https://cl.ly/17303l261C2E/Image%202017-09-03%20at%207.44.23%20PM.png)

## stack.sh中的执行顺序：

1. 支持OS类型包括Ubuntu 12.04或以上；Fedora F18或以上
2. 禁止使用root运行
3. 读取local.conf
4. 检查stackrc文件是否存在
5. 检查Devstack是不是已经在运行。如果在运行，则退出
6. 配置目标安装目录，包括创建目录，设置权限
7. 配置hostname，logging等
8. 读取各组件的安装和启动script
9. 如果没有配置密码，则需要用户输入各密码
10. 配置数据库
11. 配置Keystone
12. 安装各pre-condition包
13. 安装client包
14. 安装和配置keystone，swift，glance，cinder，neutron，nova，horizon，ceilometer，heat，CA
15. 配置数据库
16. 配置screen
17. 创建个组件使用的账号
18. 初始化和启动horizon
19. 启动swift，glance，
20. 安装images
21. 启动swift，nova_api，neutron，nova，cinder，ceilometer，heat



参考资料：

> http://blog.csdn.net/zhaihaifei/article/details/40893823

