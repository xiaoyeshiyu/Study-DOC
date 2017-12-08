# Routing 功能概述

路由服务（Routing）提供跨 subnet 联通功能。

> cirros-vm1      172.16.100.3        vlan100
>
> cirros-vm3      172.16.101.3        vlan101

这两个 instance 要通信必须借助 router。可以是物理 router 或者虚拟 router。

## 物理 router

使用物理 router，如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105038.jpg)

接入的物理 router 有两个 interface ip：

172.16.100.1 对应 vlan100 的网关。

172.16.101.1 对应 vlan101 的网关。

当 cirros-vm1 要跟 cirros-vm3 通信时，数据包的流向是这样的：

1. 因为 cirros-vm1 的默认网关指向 172.16.100.1，cirros-vm1 发送到 cirros-vm3 的数据包首先通过 vlan100 的 interface 进入物理 router。
2. router 发现目的地址 172.16.101.3 与 172.16.101.1 为同一个 vlan，则从 vlan101 的 interface 发出。
3. 数据包经过 brq1d7040b8-01 最终到达 cirros-vm3。

## 虚拟 router

虚拟 router 的路由机制与物理 router 一样，只是由软件实现。Neutron 两种方案都支持。如果要使用虚拟 router，需要启用 L3 agent。L3 agent 会在控制节点或者网络节点上运行虚拟 router，为 subnet 提供路由服务。

Neutron 的路由服务是由 l3 agent 提供的。 除此之外，l3 agent 通过 iptables 提供

firewall 和 floating ip 服务。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105230.jpg)

l3 agent 需要正确配置才能工作，配置文件为 /etc/neutron/l3_agent.ini，位于控制节点或网络节点上。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105250.jpg)

interface_driver 是最重要的选项，如果 mechanism driver 是 linux bridge，则：

``` xml
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
```

如果选用 open vswitch，则：

``` xml
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```

l3 agent 运行在控制或网络节点上。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105343.jpg)

可以看到 l3 agnet 已经正常启动。

## 创建 router 连通 subnet

创建虚拟路由器

“router\_100\_101”，打通 vlan100 和 vlan101。

打开操作菜单 Project -> Network -> Routers。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105733.jpg)

点击 “Create Router” 按钮：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105757.jpg)

router 命名为 “router\_100\_101”，点击 “Create Router” 按钮确认。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105853.jpg)

router\_100\_101 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105826.jpg)

接下来需要将 vlan100 和 vlan101 连接到 router\_100\_101。点击 “router\_100\_101” 链接进入 router 的配置页面，在 “Interfaces” 标签中点击 “Add Interface” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105925.jpg)

选择 vlan101 的 subnet\_172\_16\_101\_0，点击 “Add Interface” 确认。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208105933.jpg)

用同样的方法添加 vlan100 的 subnet\_172\_16\_100\_0。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208110043.jpg)

完成后，可以看到 router\_100\_101 有了两个 interface，其 IP 正好是 subnet 的 Gateway IP 172.16.100.1 和 172.16.101.1。

到这里，我们可以预见：

1. router\_100\_101 已经连接了 subnet\_172\_16\_100\_0 和 subnet\_172\_16\_101\_0。
2. router\_100\_101 上已经设置好了两个 subnet 的 Gateway IP。
3. cirros-vm1 和 cirros-vm3 应该可以通信了。

通过 PING 测试一下。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208110054.jpg)

判断正确，cirros-vm1 和 cirros-vm3 能通信了。

查看 cirros-vm1 的路由表，默认网关为 172.16.100.1。同时 traceroute 告诉我们，cirros-vm1 确实是通过 router\_100\_101 访问到 cirros-vm3 的。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208110221.jpg)

# 虚拟 router 原理分析

查看控制节点的 linux bridge 结构发生了什么变化。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208111929.jpg)

vlan101 的 bridge 上多了一个 tape17162c5-00，从命名上可以推断该 TAP 设备对应 router_100_101 的 interface (e17162c5-00fa)。

vlan100 的 bridge 上多了一个 tapd568ba1a-74，从命名上可以推断该 TAP 设备对应 router_100_101 的 interface (d568ba1a-740e)。

当前网络结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208111945.jpg)

但发现一个问题：两个 TAP 设备上并没有配置相应的 Gateway IP。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208112011.jpg)

如果没有 Gateway IP，router\_100\_101 是如何完成路由的呢？

答案是：

l3 agent 会为每个 router 创建了一个 namespace，通过 veth pair 与 TAP 相连，然后将 Gateway IP 配置在位于 namespace 里面的 veth interface 上，这样就能提供路由了。

通过 ip netns 查看 namespace：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208112025.jpg)

router 对应的 namespace 命名为 qrouter-< router id >。

通过 ip netns exec < namespace name >ip a 命令查看 router\_100\_101 namespace 中的 veth interface 配置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208112040.jpg)

namespace 中有两个 interface：

1. qr-e17162c5-00 上设置了 Gateway IP 172.16.101.1，与 root namespace 中的 tape17162c5-00 组成 veth pair。
2. qr-d568ba1a-74 上设置了 Gateway IP 172.16.100.1，与 root namespace 中的 tapd568ba1a-74 组成 veth pair。

网络结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208112100.jpg)

namespace 中的路由表也保证了 subnet\_172\_16\_100\_0 和 subnet\_172\_16\_101\_0 之间是可以路由的。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208112149.jpg)

# namespace 封装 router

回顾一下前面的网络逻辑结构图：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208162835.jpg)

为什么不直接在 tape17162c5-00 和 tapd568ba1a-74 上配置 Gateway IP，而是引入一个 namespace，在 namespace 里面配置 Gateway IP 呢？

首先考虑另外一个问题：

如果不用 namespace，直接 Gareway IP 配置到 tape17162c5-00 和 tapd568ba1a-74 上，能不能连通 subnet\_172\_16\_100\_0 和 subnet\_172\_16\_101\_0 呢？

答案是可以的，只要控制节点上配置了类似下面的路由。

``` shell
Destination Gateway Genmask Flags Metric Ref Iface
172.16.100.0 * 255.255.255.0 U 0 0  tapd568ba1a-74
172.16.101.0 * 255.255.255.0 U 0 0  tape17162c5-00
```

既然不需要 namespace 也可以路由，为什么还要加一层 namespace 增加复杂性呢？

其根本原因是：**为了支持网络重叠**。

云环境下，租户可以按照自己的规划创建网络，不同租户的网络是可能重叠的。将路由功能放到 namespace 中，就能隔离不同租户的网络，从而支持网络重叠。

下面通过例子进一步解释。

``` shell
Tenant A  vlan100 subnet A-1: 10.10.1.0/24    {"start": "10.10.1.1", "end": "10.10.1.254"}
Tenant A  vlan101 subnet A-2: 10.10.2.0/24    {"start": "10.10.2.1", "end": "10.10.2.254"}

Tenant B  vlan102 subnet B-1: 10.10.1.0/24    {"start": "10.10.1.1", "end": "10.10.1.254"}
Tenant B  vlan103 subnet B-2: 10.10.2.0/24    {"start": "10.10.2.1", "end": "10.10.2.254"}
```

A，B 两个租户定义了完全相同的两个 subnet，网络完全重叠。

## 不使用 namespace 的场景

如果不使用 namespace，网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208163330.jpg)

其特征是网关 IP 配置在 TAP interface 上。因为没有 namespace 隔离，router\_100\_101 和 router\_102\_103 的路由条目都只能记录到控制节点操作系统（root namespace）的路由表中，内容如下：

``` shell
Destination Gateway Genmask Flags Metric Use Iface
10.10.1.0  * 255.255.255.0   U     0      0      tap1
10.10.2.0  * 255.255.255.0   U     0      0      tap2
10.10.1.0  * 255.255.255.0   U     0      0      tap3
10.10.2.0  * 255.255.255.0   U     0      0      tap4
```

这样的路由表是无法工作的。

按照路由表优先匹配原则，Tenant B 的数据包总是错误地被 Tenant A 的 router 路由。例如 vlan102 上有数据包要发到 vlan103。选择路由时，会匹配路由表的第二个条目，结果数据被错误地发到了 vlan101。

## 使用 namespace 的场景

如果使用 namespace，网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208163530.jpg)

其特征是网关 IP 配置在 namespace 中的 veth interface 上。每个 namespace 拥有自己的路由表。

router\_100\_101 的路由表内容如下：

``` shell
Destination Gateway Genmask Flags Metric Use Iface
10.10.1.0 * 255.255.255.0   U     0      0    qr-1
10.10.2.0 * 255.255.255.0   U     0      0    qr-2
```

router\_102\_103 的路由表内容如下：

``` shell
Destination Gateway Genmask Flags Metric Use Iface
10.10.1.0 * 255.255.255.0   U     0      0     qr-3
10.10.2.0 * 255.255.255.0   U     0      0     qr-4
```

这样的路由表是可以工作的。

例如 vlan102 上有数据包要发到 vlan103。选择路由时，会查看 router\_102\_103 的路由表, 匹配第二个条目，数据通过 qr-4 

被正确地发送到 vlan103。

同样当 vlan100 上有数据包要发到 vlan101时，会匹配 router\_100\_101 路由表的第二个条目，数据通过 qr-2 被正确地发送到 vlan101。

可见，namespace 使得每个 router 有自己的路由表，而且不会与其他 router 冲突，所以能很好地支持网络重叠。



































