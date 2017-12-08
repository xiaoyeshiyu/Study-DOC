# Neutron 搭建物理环境

第一步就是准备实验用的物理环境，考虑如下几个问题：

> 需要几个节点？
> 如何分配节点的角色？
> 节点上部署哪些服务？
> 配几个网卡？
> 物理网络如何连接？

## 1 控制节点 + 1 计算节点 的部署方案

实验环境应尽量贴近典型的部署方案；但同时，由于是个人学习使用，受物理条件的限制需要尽量利用有限的资源，所以采用下面的部署方案：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207143405.jpg)

## 配置多个网卡区分不同类型的网络数据

OpenStack 至少包含下面几类网络流量

> Management
> API
> VM
> External

**Management 网络**

用于节点之间 message queue 内部通信以及访问 database 服务，所有的节点都需要连接到 management 网络。

**API 网络**

OpenStack 各组件通过该网络向用户暴露 API 服务。Keystone, Nova, Neutron, Glance, Cinder, Horizon 的 endpoints 均配置在 API 网络上。通常，管理员也通过 API 网络 SSH 管理各个节点。

**VM 网络**

VM 网络也叫 tenant 网络，用于 instance 之间通信。
VM 网络可以选择的类型包括 local, flat, vlan, vxlan 和 gre。
VM 网络由 Neutron 配置和管理。

**External 网络**

External 网络指的是 VM 网络之外的网络，该网络不由 Neutron 管理。 Neutron 可以将 router attach 到 External 网络，为 instance 提供访问外部网络的能力。 External 网络可能是企业的 intranet，也可能是 internet。

这几类网络只是逻辑上的划分，物理实现上有非常大的自由度。

可以为每种网络分配单独的网卡；也可以多种网络共同使用一个网卡；为提高带宽和硬件冗余，可以使用 bonding 技术将多个物理网卡绑定成一个逻辑的网卡

我们的实验环境采用下面的网卡分配方式：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207143536.jpg)

1. 控制节点 3 网卡（eth0, eth1, eth2），计算节点 2 网卡（eth0, eth1）。

2. 合并 Management 和 API 网络，使用 eth0，IP 段为 192.168.104.0/24

3. VM 网络使用 eht1。

4. 控制节点的 eth2 与 External 网络连接，IP 段为 10.10.10.0/24。

## 网络拓扑

实验环境的网络拓扑如下图所示

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207143613.jpg)

分割线上方的网络由网络管理员配置。 主要涉及 Management, API 和 external 网络。 配置的内容包括节点上的物理网卡，物理交换机和外部路由器，防火墙以及物理连线等。

分割线下方是 VM 网络，由 Neutron 管理。 只需要通过 Web GUI 或者 CLI 操作，Neutron 会负责实现。

# 配置 linux-bridge mechanism driver

首先需要配置 linux-bridge mechanism driver。

Neutorn ML2 plugin 默认使用的 mechanism driver 是 open vswitch 而不是 linux bridge。

研究linux bridge原因如下：

1. linux bridge 技术非常成熟，而且高效，所以业界很多 OpenStack 方案选择 linux bridge，比如 Rackspace 的 private cloud。
2. open vswitch 实现的 Neutron 虚拟网络较为复杂，不易理解；而 linux bridge 方案更直观。先理解 linux bridge 方案后再学习 open vswitch 方案会更容易。并且可以通过两种方案的对比更加深入地理解 Neutron 网络。

先看下 linux bridge 实现虚拟交换节的基本原理。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207150633.jpg)

上图中，br0 是 linux bridge，br0 充当虚拟交换机的作用，负责将物理网卡 eth0 和虚拟网卡 tap 设备 vnet0/vent1 连接到同一个二层网络，实现虚拟机 VM1 和 VM2，以及虚拟机与外网之间的通信。

# 配置 linux-bridge mechanism driver

要在 Neutron 中使用 linux bridge，首先需要配置 linux-bridge mechanism driver。 Neutron 默认使用 ML2 作为 core plugin，其配置位于 /etc/neutron/neutron.conf。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207150709.jpg)

控制节点和计算节点都需要在各自的 neutron.conf 中配置 core_plugin 选项。然后需要让 ML2 使用 linux-bridge mechanism driver。 ML2 的配置文件位于 /etc/neutron/plugins/ml2/ml2_conf.ini。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207150749.jpg)

mechanism_drivers 选项指明当前节点可以使用的 mechanism driver，这里可以指定多种 driver，ML2 会负责加载。 上面配置指明我们只使用 linux-bridge driver。

控制节点和计算节点都需要在各自的 ml2_conf.ini 中配置 mechanism_drivers 选项。Neutron 服务正常启动后，所有节点上都会运行 neutron-linuxbridge-agent

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207150757.jpg)

linux-bridge mechanism driver 已经配置完毕，随着各种 Neutron 功能的实践，这个网络状态会不断的演变和发展。

# 准备工作

两个准备工作：

1. 检视初始的网络状态。
2. 了解 linux bridge 环境中的各种网络设备。

## 初始网络状态

首先考察实验环境最初始的网络状态。在实验环境中，当前节点上只存在物理网卡设备 ethX，还没有 bridge 和 tap，状态如下：

### 控制节点

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207152100.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207152129.jpg)

### 计算节点

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207152159.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207152223.jpg)

## linux bridge 环境中的各种网络设备

在配置 linux bridge driver 之前先了解几种网络设备，后面会经常用到。

在 linux bridge 环境中，一个数据包从 instance 发送到物理网卡会经过下面几个类型的设备：

1. tap interface命名为 tapN (N 为 0, 1, 2, 3......)
2. linux bridge命名为 brqXXXX。
3. vlan interface命名为 ethX.Y（X 为 interface 的序号，Y 为 vlan id）
4. vxlan interface命名为 vxlan-Z（z 是 VNI）
5. 物理 interface命名为 ethX（X 为 interface 的序号）

vlan interface 会在 vlan 网络中使用；vxlan interface 会在 vxlan 网络中使用。linux-bridge 支持 local, flat, vlan 和 vxlan 四种 network type，目前不支持 gre。

# enable local network

local network 的特点是不会与宿主机的任何物理网卡相连，也不关联任何的 VLAN ID。对于每个 local netwrok，ML2 linux-bridge 会创建一个 bridge，instance 的 tap 设备会连接到 bridge。位于同一个 local network 的 instance 会连接到相同的 bridge，这样 instance 之间就可以通信了。

因为 bridge 没有与物理网卡连接，所以 instance 无法与宿主机之外的网络通信。 同时因为每个 local network 有自己的 bridge，bridge 之间是没有连通的，所以两个 local network 之间也不能通信，即使它们位于同一宿主机上。

下图是 local network 的示例：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207160635.jpg)

- 创建了两个 local network，分别对应两个网桥 brqXXXX 和 brqYYYY。
- VM0 和 VM1 通过 tap0 和 tap1 连接到 brqXXXX。
- VM2 通过 tap2 连接到 brqYYYY。
- VM0 与 VM1 在同一个 local network中，它们之间可以通信。
- VM2 位于另一个 local network，由于 brqXXXX 和 brqYYYY 没有联通，所以 VM2 无法与 VM0 和 VM1 通信。

## 在 ML2 配置中 enable local network

创建 local 网络之前请先确保 ML2 已经加载了 local type driver。 ML2 的配置文件位于 /etc/neutron/plugins/ml2/ml2_conf.ini。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207160814.jpg)

type_drivers 告诉 ML2 加载所有 5 种网络的 type driver。

``` xml
type_drivers = local,flat,vlan,gre,vxlan
```

这样所有类型的网络我们都可以创建（虽然在本节只创建 local 网络）。普通用户和 admin 都可以通过 CLI 或者 Web GUI 创建网络，但只有 amdin 才能指定网络的 type，所以需要用 tenant_network_types 告诉 ML2 当普通用户在自己的 Tenant（Project）中创建网络时，默认创建哪种 type 的网络，这里 type 是 local。

``` xml
tenant_network_types = local
```

tenant_network_types 可以指定多种 type，比如：

``` xml
tenant_network_types = vlan, local
```

其作用是先创建 vlan 网络，当没有 vlan 可创建时（比如 vlan id 用完），创建 local 网络。当配置文件发生了变化，需要重启 Neutron 相关服务。

# 创建第一个 local network

## GUI创建第一个local network

下面通过 Web GUI 创建第一个 local network。

首先确保各个节点上的 neutorn agent 状态正常。GUI 菜单 为 Admin -> System -> System Infomation -> Neutron Agents

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207162538.jpg)

GUI 中有两个地方可以创建 network：

1. Project -> Network -> Networks这是普通用户在自己的 tenant 中创建 network 的地方。
2. Admin -> Networks 这是 admin 创建 network 的地方。

先用第一种方式创建，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207162722.jpg)

点击 “Next”，创建 subnet，命名为 “subnet\_172\_16\_1\_0”，地址为 “172.16.1.0/24”。如果 Gateway IP 不设置，默认为 subnet 的第一个 IP，即 172.16.1.1。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207162759.jpg)

点击 “Next”，设置 subnet 的 IP 地址范围为 172.16.1.2-172.16.1.100，instance 的 IP 会从这里分配。默认会 “Enable DHCP”，同时还可以设置 subnet 的 DNS 和添加静态路由条目。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207162852.jpg)

点击 “Create”，network 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207162924.jpg)

通过 GUI 创建 local network 相对比较简单。

## 底层网络的变化

点击 “first_local_net” 链接，显示 network 的 subnet 和 port 信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163259.jpg)

在 Ports 列表中已经创建了一个 port，名称为 “(a5bd3746-3f89)”，IP 为 172.16.1.2， Attached Device 是 network:dhcp。

打开控制节点的 shell 终端，用 `brctl show` 查看当前 linux bridge 的状态。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163400.jpg)

可以看到 Neutron 自动创建了如下两个设备：

**bridge 设备 brqbb9b6d21-c6**

brqbb9b6d21-c6 对应 local network “first_local_net”，命名规则为 brqXXX，XXX 为 network ID 的前 11 个字符。

**tap 设备 tapa5bd3746-3f**

tapa5bd3746-3f 对应 port (a5bd3746-3f89)，命名为 tapYYY, YYY 是 port ID 的前 11 个字符。该 tap 设备已经连接到 bridge，即连接到该 local 网络。

# 将 instance 连接到 first_local_net

## 将 instance 连接到 first_local_net

launch 一个 instance，在“Networking”标签页面选择 first_local_net 网络。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163717.jpg)

instance 部署成功，分配的 IP 地址为 172.16.1.3

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163748.jpg)

## 底层网络发生的变化

对于 instance “cirros-vm1”，Neutron 会在 subnet 中创建一个 port，分配 IP 和 MAC 地址，并将 port 分配给 cirros-vm1。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163826.jpg)

如上图所示，port 列表中增加了一个 port “(fa7e090e-a29c)”，IP 为 172.16.1.3。点击 port 名称查看 MAC 信息：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163855.jpg)

当 cirros-vm1 启动时：

1. 宿主机上的 neutron-linuxbridge-agent 会根据 port 信息创建 tap 设备，并连接到 local 网络所在的 bridge
2. 同时该 tap 会映射成 cirros-vm1 的虚拟网卡，即 virtual interface （VIF）。

下面验证一下以上信息：

cirros-vm1 部署到了控制节点，通过 brctl show 查看 bridge 的配置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163927.jpg)

可以看到 bridge brqbb9b6d21-c6 上连接了一个新的 tap 设备 tapfa7e090e-a2

从命名上可知 tapfa7e090e-a2 对应着 port “(fa7e090e-a29c)”。

`virsh list `中显示的虚拟机 instance-00000001 即为 “cirros-vm1”，命名方式有所不同，需注意。

通过` virsh edit `命令查看 cirros-vm1 的配置，确认 VIF 就是 tapfa7e090e-a2。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207163943.jpg)

另外，VIF 的 MAC 地址为` fa:16:3e:c1:66:a5`，这个数据就是从 port “(fa7e090e-a29c)” 取过来的。

在 cirros-vm1 中执行 ifconfig，通过 MAC 地址可以确认 eth0 与 tapfa7e090e-a2 对应。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207164006.jpg)

下图展示了创建 cirros-vm1 后宿主机当前的网络结构。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207164024.jpg)

## 连接第二个 insance 到 first_local_net

以同样的方式 launch instance “cirros-vm2”，分配的 IP 为 172.16.1.4

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207165654.jpg)

cirros-vm2 也被 schedule 到控制节点，virsh list 和 brctl show 输出如下，cirros-vm2 对于的 tap 设备为 tapa5bd3746-3f。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207165725.jpg)

在 cirros-vm2 的控制台运行 ifconfig，instance 已经拿到了 DCHP 的 IP 地址。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207165849.jpg)

能够 Ping 通 cirros-vm1 的 IP 地址  172.16.1.3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207165908.jpg)

当前宿主机的网络结构如下。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207165926.jpg)

两个 instance 的 VIF 挂在同一个 linux bridge 上，可以相互通信。

这里有一个问题：

> 如果 cirros-vm2 launch 时被 schedule 到计算节点而非控制节点，它能获得 DHCP 的 IP 吗？

答案：

> **不能**。
>
> 因为 DHCP agent 在控制节点上运行，cirros-vm2 在计算节点的 local 网络上，两者位于不同物理节点。由于 local 网络的流量只能局限在本节点之内，发送的 DHCP 请求无法到达控制节点。

# 创建第二个 local network

GUI 中有两个地方可以创建 network：

**1. Project -> Network -> Networks** 

这是普通用户在自己的 tenant 中创建 network 的地方。

**2. Admin -> Networks** 

这是 admin 创建 network 的地方。前面已经用第一种方式创建了 "first_local_net"，本节将以第二种方式创建 local network "second_local_net"。

菜单路径为 Admin -> Networks，此菜单只有 admin 用户才能够访问。点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207170415.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200324.jpg)



可以看到几个与普通用户创建 network 不同的地方：

> 1. 可以选择该 network 属于哪个 Project（租户）。
> 2. 可以选择 network type。
> 3. 可以指定 network 是否与其他 Project 共享。
> 4. 可以指定是否为 external network。

可见，这种方式赋予 admin 用户创建 network 更大的灵活性，后面都将采用这种方式创建 network。

点击 “Create Network”，second_local_net 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200428.jpg)

点击 second_local_net 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200507.jpg)

设置 IP 地址为 “172.16.1.0/24”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200531.jpg)

点击 “Next”，设置 IP 地址范围为 172.16.1.101-172.16.1.200。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200605.jpg)

点击 “Create”，subnet 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200555.jpg)

查看控制节点的网络结构，增加了 second_local_net 对应的网桥 brq161e0b25-58，以及 dhcp 的 tap 设备 tapae547b6b-2a。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200704.jpg)

# 将 instance 连接到 second_local_net

launch 新的 instance “cirros-vm3”，网络选择 second_local_net。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200753.jpg)

cirros-vm3 分配到的 IP 为 172.16.1.102。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200816.jpg)

cirros-vm3 被 schedule 到控制节点，对应的 tap 设备为 tap5395d19b-ed。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200847.jpg)

控制台显示 cirros-vm3 已经成功从 DHCP 拿到 IP 地址 172.16.1.102。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200855.jpg)

但是 cirros-vm3 无法 Ping 到 cirros-vm1。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200933.jpg)

这是在预料之中的，因为 cirros-vm3 和 cirros-vm1 位于不同的 local network，之间没有连通，即使都位于同一个宿主机也不能通信。

网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207200940.jpg)

# 小结

到这里 local network 的知识点已经讨论完毕。

1. 位于同一 local network 的 instance 可以通信。
2. 位于不同 local network 的 instance 无法通信。
3. 一个 local network 只能位于一个物理节点，无法跨节点。

虽然在实际应用中极少使用 local network，但学习 local network 的意义在于：

**local network 可作为学习 flat, vlan, vxlan 等更复杂网络类型的起点，降低 Neutron 的学习难度**。















































