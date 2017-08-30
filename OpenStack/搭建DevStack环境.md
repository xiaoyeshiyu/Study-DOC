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

ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.2  netmask 255.255.255.0  broadcast 10.0.0.255
        inet6 fe80::20c:29ff:fe34:4585  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:34:45:85  txqueuelen 1000  (Ethernet)
        RX packets 1902  bytes 174498 (170.4 KiB)
        RX errors 0  dropped 1  overruns 0  frame 0
        TX packets 28  bytes 2688 (2.6 KiB)
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
 index-url = http://pypi.douban.com/simple/
 [install] 
 trusted-host = pypi.douban.com
```

使用阿里yum源

``` shell
[root@Mitaka ~]# yum install -y wget 
[root@Mitaka ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@Mitaka ~]# yum clean all 
[root@Mitaka ~]# yum makecache 
```

### 安装阶段

下载DevStack

``` shell
[root@Mitaka ~]# cd /home/
[root@Mitaka home]# git clone http://git.trystack.cn/openstack-dev/devstack.git -b stable/ocata
```

创建stack用户

``` shell
[root@Mitaka home]# mkdir -p /home/stack/logs
[root@Mitaka home]# cd devstack/tools/
[root@Mitaka tools]# ./create-stack-user.sh
[root@Mitaka tools]# passwd stack
[root@Mitaka tools]# chown stack:stack -R /home/stack
[root@Mitaka tools]# chown stack:stack -R /home/devstack
```

授权stack用户，在第98行添加一行

``` shell
[root@Mitaka tools]# vim /etc/sudoers

99 stack ALL=(ALL:ALL)     ALL
```

### 创建local.conf配置文件

``` shell
[stack@Mitaka devstack]$ cat local.conf 
[[local|localrc]]

# use TryStack git mirror
GIT_BASE=http://git.trystack.cn
NOVNC_REPO=http://git.trystack.cn/kanaka/noVNC.git
SPICE_REPO=http://git.trystack.cn/git/spice/spice-html5.git

#OFFLINE=True
RECLONE=True

# Define images to be automatically downloaded during the DevStack built process.
#DOWNLOAD_DEFAULT_IMAGES=False
#IMAGE_URLS="http://images.trystack.cn/cirros/cirros-0.3.4-x86_64-disk.img"

HOST_IP=192.168.50.97


# Credentials
DATABASE_PASSWORD=daemon
ADMIN_PASSWORD=daemon
SERVICE_PASSWORD=daemon
SERVICE_TOKEN=daemon
RABBIT_PASSWORD=daemon

HORIZON_BRANCH=stable/ocata
KEYSTONE_BRANCH=stable/ocata
NOVA_BRANCH=stable/ocata
NEUTRON_BRANCH=stable/ocata
GLANCE_BRANCH=stable/ocata
CINDER_BRANCH=stable/ocata


#keystone
KEYSTONE_TOKEN_FORMAT=UUID

##Heat
HEAT_BRANCH=stable/ocata
enable_service h-eng h-api h-api-cfn h-api-cw


## Swift
SWIFT_BRANCH=stable/ocata
ENABLED_SERVICES+=,s-proxy,s-object,s-container,s-account
SWIFT_REPLICAS=1
SWIFT_HASH=011688b44136573e209e


# Enabling Neutron (network) Service
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service q-metering
enable_service neutron

## Neutron options
Q_USE_SECGROUP=True
FLOATING_RANGE="192.168.50.0/24"
FIXED_RANGE="10.0.0.0/24"
NETWORK_GATEWAY="10.0.0.1"
Q_FLOATING_ALLOCATION_POOL=start=192.168.50.100,end=192.168.50.200
PUBLIC_NETWORK_GATEWAY="192.168.50.1"
Q_L3_ENABLED=True
PUBLIC_INTERFACE=ens160
Q_USE_PROVIDERNET_FOR_PUBLIC=True
OVS_PHYSICAL_BRIDGE=br-ex
PUBLIC_BRIDGE=br-ex
OVS_BRIDGE_MAPPINGS=public:br-ex

# #VLAN configuration.
Q_PLUGIN=ml2
ENABLE_TENANT_VLANS=True

# Logging
LOGFILE=/home/stack/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=True
SCREEN_LOGDIR=/home/stack/logs
```

### 开始安装

``` shell
[stack@Mitaka devstack]$ ./stack.sh 
```

这个过程会持续15-20分钟，这个时间取决于网络下载速度。