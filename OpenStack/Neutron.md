[TOC]

# Neutron网络解析

## 图示如下

![Neutron](https://cl.ly/1x2q1y3Y2k3K/Image%202017-08-30%20at%206.25.38%20PM.png)

## TAP device

查看`instance`硬件信息

```
[root@node2 ~]# virsh list
 Id    Name                           State
----------------------------------------------------
 14944 instance-00000070              running
 18862 instance-0000002d              running
 18893 instance-00000076              running

```

以实例`14944`为例子，查看实例`14944`具体硬件信息
```
[root@node2 ~]# virsh edit 14944
<interface type='bridge'>
  <mac address='fa:16:3e:52:43:c5'/>
  <source bridge='qbr85bcbfb0-bc'/>
  <target dev='tap85bcbfb0-bc'/>
  <model type='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>

```

可以看到`instance`的虚拟网卡MAC地址为`fa:16:3e:52:43:c5`，连接到的`bridge`为`qbr85bcbfb0-bc`，`target device`也就是对应物理机上的虚拟显卡为`tap85bcbfb0-bc`。

查看物理机上的网卡
```
[root@node2 ~]# ip a
42: tap85bcbfb0-bc: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master qbr85bcbfb0-bc state UNKNOWN qlen 1000
    link/ether fe:16:3e:52:43:c5 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc16:3eff:fe52:43c5/64 scope link
       valid_lft forever preferred_lft forever
```

可以看到物理机上`tap85bcbfb0-bc`网卡的MAC地址为`fe:16:3e:52:43:c5`，对应的也就是`instance 14944`的虚拟网卡。

- TAP设备主要是用来让宿主机可以接收到数据帧。

## Linux Bridge
上面看到`instance 14944`的网卡连接的bridge是`qbr85bcbfb0-bc`
查看网桥
```
[root@node2 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
qbr09d052bf-16		8000.b216e6c26e13	no		qvb09d052bf-16
qbr263b3957-09		8000.d6a957a5b378	no		qvb263b3957-09
							tap263b3957-09
qbr2d4a7a05-ea		8000.2afc1c44c8a4	no		qvb2d4a7a05-ea
							tap2d4a7a05-ea
qbr85bcbfb0-bc		8000.c67808b20c2e	no		qvb85bcbfb0-bc
							tap85bcbfb0-bc
qbrf010b8f6-df		8000.de5f220adb0c	no		qvbf010b8f6-df
							tapf010b8f6-df
```
`qbr85bcbfb0-bc`上，有两个接口，一个是`qvb85bcbfb0-bc`，一个是`tap85bcbfb0-bc`。也就是说，`instance 14944`连接在虚拟网桥`qbr85bcbfb0-bc`上。

- Linux Bridge类似一个集线器，用来过渡物理机到宿主机的数据帧。

## VETH
查看物理机网络`qvb85bcbfb0-bc`网卡的详细信息
```
[root@node2 ~]# ip a | grep qvb85bcbfb0-bc
40: qvo85bcbfb0-bc@qvb85bcbfb0-bc: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP qlen 1000
41: qvb85bcbfb0-bc@qvo85bcbfb0-bc: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1450 qdisc noqueue master qbr85bcbfb0-bc state UP qlen 1000
```
很明显，这两个网卡有关联。
```
[root@node2 ~]# ethtool -S qvo85bcbfb0-bc
NIC statistics:
     peer_ifindex: 41
[root@node2 ~]# ethtool -S qvb85bcbfb0-bc
NIC statistics:
     peer_ifindex: 40
```
`qvo85bcbfb0-bc`和`qvb85bcbfb0-bc`属于一对peer，可以理解为一个网线的两端，也就是说一根网线的一端`qvb85bcbfb0-bc`连接在虚拟网桥`qbr85bcbfb0-bc`上，而另外的一端`qvo85bcbfb0-bc`则是连接在`OpenvSwitch`的`br-int`上。

- VETH可以直接理解为一个网线，一端连接在虚拟网桥上，另一端连接在虚拟交换机上，实现数据流的传递

## OpenvSwitch
查看OpenvSwitch的路由器的详细信息
```
[root@node2 ~]# ovs-vsctl show
4ece5774-21aa-4ff6-8e7e-343983396eee
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port "vxlan-0a0a0a01"
            Interface "vxlan-0a0a0a01"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="10.10.10.2", out_key=flow, remote_ip="10.10.10.1"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
        Port "qvoe0f204ea-12"
            tag: 4095
            Interface "qvoe0f204ea-12"
                error: "could not open network device qvoe0f204ea-12 (No such device)"
        Port br-int
            Interface br-int
                type: internal
        Port "qr-764aecf0-56"
            tag: 1
            Interface "qr-764aecf0-56"
                type: internal
        Port "tapd87312fd-34"
            tag: 1
            Interface "tapd87312fd-34"
                type: internal
        Port "qvo09d052bf-16"
            tag: 2
            Interface "qvo09d052bf-16"
        Port "qvof010b8f6-df"
            tag: 2
            Interface "qvof010b8f6-df"
        Port "qvo85bcbfb0-bc"
            tag: 1
            Interface "qvo85bcbfb0-bc"
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "tap3287c19b-e3"
            tag: 2
            Interface "tap3287c19b-e3"
                type: internal
        Port "qvo2d4a7a05-ea"
            tag: 2
            Interface "qvo2d4a7a05-ea"
        Port "qvo263b3957-09"
            tag: 1
            Interface "qvo263b3957-09"
    Bridge br-ex
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
        Port br-ex
            Interface br-ex
                type: internal
        Port "enp4s0f1"
            Interface "enp4s0f1"
    Bridge br-vlan
        Port br-vlan
            Interface br-vlan
                type: internal
    ovs_version: "2.5.0"
```
上面的信息主要包含这么几点：

        qvo85bcbfb0-bc连接在桥br-int上；
        br-int通过int-br-ex和phy-br-ex这对配对接口相连；
        br-ex中有物理网卡enp4s0f1接口，如果enp4s0f1网口连接外部网络设备交换机，则可以实现虚拟机br-ex和外网直通；
        br-int和br-tun通过patch-int和patch-tun这对配对接口相连；
        br-tun中的接口vxlan-0a0a0a01通过ip tunnel以及内部IP，连接到集群中另外一台物理设备上，实现不同宿主机上虚拟机网络的互通。

- OpenvSwitch主要是通过不同功能的交换机之间的相互连接，从而实现虚拟机和外部网络的通信，进一步实现SDN，即软件定义网络。

上文中一些网络接口可以这么理解：

        软件+ID

比如说：

      qbr85bcbfb0-bc：qbr为qemu bridge + ID
      tap85bcbfb0-bc：tap为tap设备  + ID
      qvb85bcbfb0-bc：qvb为qemu virtual bridge  +ID
      qvo85bcbfb0-bc：qvo为qemu virtual OpenvSwitch +ID

## DHCP

先查看到DHCP的空间位置

``` shel
[root@hyhive01 ~]# ip netns ls
qdhcp-624031b6-981c-480c-94ee-f9459d1c07ed
qdhcp-d72f0c1e-cc46-4d8f-a918-60c0088cc60a
```

选取上面一个DHCP查看详细信息

``` shell
[root@hyhive01 ~]# ip netns exec qdhcp-624031b6-981c-480c-94ee-f9459d1c07ed ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
28: tap42681808-04: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN qlen 1000
    link/ether fa:16:3e:1c:f9:38 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global tap42681808-04
       valid_lft forever preferred_lft forever
    inet 169.254.169.254/16 brd 169.254.255.255 scope global tap42681808-04
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe1c:f938/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到的是IP地址会从`tap42681808-04`发送出去，并且网段为`192.168.100.0/24`网段

查看`OpenvSwitch`上`tap42681808-04`的信息

```shell
    Bridge br-int
        Port "tap42681808-04"
            tag: 3
            Interface "tap42681808-04"
                type: internal
```

`tap42681808-04`打的标签为3号，新建虚拟机的时候选择网卡网段，对应的就是tag

*目前笔者比较好奇的是：多节点情况下，不同节点的tag ID只是本地有效，节点之间通过vxlan连接，而不是像交换机直接用`trunk`连接，每台交换机的vlan ID是互通的。*

## ROUTER

先查看ROUTER的网络位置

``` shell
[root@hyhive02 ~]# ip netns ls
qrouter-8800b853-2938-4ecc-89ce-a7189f78ef0c
qdhcp-0be33604-c845-4e71-9e88-ba1facec20e9
```

查看ROUTER详细信息

``` shell
[root@hyhive02 ~]# ip netns exec qrouter-8800b853-2938-4ecc-89ce-a7189f78ef0c ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
15: qr-0fd24acc-81: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN qlen 1000
    link/ether fa:16:3e:c0:e6:7e brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.1/24 brd 192.168.0.255 scope global qr-0fd24acc-81
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fec0:e67e/64 scope link 
       valid_lft forever preferred_lft forever
27: qg-ae1e7daf-58: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN qlen 1000
    link/ether fa:16:3e:4c:e7:29 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.9/24 brd 192.168.10.255 scope global qg-ae1e7daf-58
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe4c:e729/64 scope link 
       valid_lft forever preferred_lft forever
```

查看ROUTER路由详细信息

``` shell
[root@hyhive02 ~]# ip netns exec qrouter-8800b853-2938-4ecc-89ce-a7189f78ef0c route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    0      0        0 qg-ae1e7daf-58
192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 qr-0fd24acc-81
192.168.10.0    0.0.0.0         255.255.255.0   U     0      0        0 qg-ae1e7daf-58
```

可以看到的是，路由器有两个接口，一个连接内部网络，图上的为`192.168.0.0/24`网段，一个为外部网络，图上为`192.168.10.0/24`，其中，内部网络可以连接多个，但是外部网络只能连接一个。外部网络可以是直接连接外网的`br-ex`网桥上的物理网卡接口；也可以是带有vlan ID的子网网段，相对应的，连接的外部网络设备上需要有带上该ID的vlan接口。



**DHCP服务是如何跨国tag对虚拟机提供IP地址的？虚拟机的网络流量是如果可以从`br-int`流向`br-ex`再流向外网的？**

诸如此类问题，需要对OpenvSwitch和neutron源码有更深入的了解，后面有机会再往下写。