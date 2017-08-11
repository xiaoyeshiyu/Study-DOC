# Neutron网络解析
## 图示如下

![Neutron](https://cl.ly/2F2g111B0Z3e/未命名图片.png)

## TAP device

查看instance硬件信息

```
[root@node2 ~]# virsh list
 Id    Name                           State
----------------------------------------------------
 14944 instance-00000070              running
 18862 instance-0000002d              running
 18893 instance-00000076              running

```

以实例14944为例子，查看实例14944具体硬件信息
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

可以看到instance的虚拟网卡MAC地址为fa:16:3e:52:43:c5，连接到的bridge为qbr85bcbfb0-bc，target device也就是对应物理机上的虚拟显卡为tap85bcbfb0-bc。

查看物理机上的网卡
```
[root@node2 ~]# ip a
42: tap85bcbfb0-bc: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc pfifo_fast master qbr85bcbfb0-bc state UNKNOWN qlen 1000
    link/ether fe:16:3e:52:43:c5 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc16:3eff:fe52:43c5/64 scope link
       valid_lft forever preferred_lft forever
```

可以看到物理机上tap85bcbfb0-bc网卡的MAC地址为fe:16:3e:52:43:c5，对应的也就是instance 14944的虚拟网卡。

- TAP设备主要是用来让宿主机可以接收到数据帧。

## Linux Bridge
上面看到instance 14944的网卡连接的bridge是qbr85bcbfb0-bc
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
qbr85bcbfb0-bc上，有两个接口，一个是qvb85bcbfb0-bc，一个是tap85bcbfb0-bc。也就是说，instance 14944连接在虚拟网桥qbr85bcbfb0-bc上。

- Linux Bridge类似一个集线器，用来过渡物理机到宿主机的数据帧。

## VETH
查看物理机网络qvb85bcbfb0-bc网卡的详细信息
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
qvo85bcbfb0-bc和qvb85bcbfb0-bc属于一对peer，可以理解为一个网线的两端，也就是说一根网线的一端qvb85bcbfb0-bc连接在虚拟网桥qbr85bcbfb0-bc上，而另外的一端qvo85bcbfb0-bc则是连接在OpenvSwitch的be-int上。

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
