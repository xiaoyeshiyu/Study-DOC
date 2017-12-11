# VXLAN

前面讨论了 local, flat, vlan这几类网络，OpenStack 还支持 vxlan 和 gre 这两种 overlay network。

overlay network 是指建立在其他网络上的网络。overlay network 中的节点可以看作通过虚拟（或逻辑）链路连接起来的。overlay network 在底层可能由若干物理链路组成，但对于节点，不需要关心这些底层实现。

例如 P2P 网络就是 overlay network，隧道也是。vxlan 和 gre 都是基于隧道技术实现的，它们也都是 overlay network。

目前 linux bridge 只支持 vxlan，不支持 gre；open vswitch 两者都支持。vxlan 与 gre 实现非常类似，而且 vxlan 用得较多，所以目前只介绍 vxlan。

VXLAN 为 Virtual eXtensible Local Area Network。正如名字所描述的，VXLAN 提供与 VLAN 相同的以太网二层服务，但拥有更强的扩展性和灵活性。与 VLAN 相比，VXLAN 有下面几个优势：

1. 支持更多的二层网段。

   VLAN 使用 12-bit 标记 VLAN ID，最多支持 4094 个 VLAN，这对大型云部署会成为瓶颈。VXLAN 的 ID （VNI 或者 VNID）则用 24-bit 标记，支持 16777216 个二层网段。

2. 能更好地利用已有的网络路径。

   VLAN 使用 Spanning Tree Protocol 避免环路，这会导致有一半的网络路径被 block 掉。VXLAN 的数据包是封装到 UDP 通过三层传输和转发的，可以使用所有的路径。

3. 避免物理交换机 MAC 表耗尽。

   由于采用隧道机制，TOR (Top on Rack) 交换机无需在 MAC 表中记录虚拟机的信息。

**VXLAN 封装和包格式**

VXLAN 是将二层建立在三层上的网络。通过将二层数据封装到 UDP 的方式来扩展数据中心的二层网段数量。

VXLAN 是一种在现有物理网络设施中支持大规模多租户网络环境的解决方案。VXLAN 的传输协议是 IP + UDP。

VXLAN 定义了一个 MAC-in-UDP 的封装格式。在原始的 Layer 2 网络包前加上 VXLAN header，然后放到 UDP 和 IP 包中。通过 MAC-in-UDP 封装，VXLAN 能够在 Layer 3 网络上建立起了一条 Layer 2 的隧道。

VXLAN 包的格式如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211112115.jpg)

如上图所示，VXLAN 引入了 8-byte VXLAN header，其中 VNI 占 24-bit。VXLAN 和原始的 L2 frame 被封装到 UDP 包中。这 24-bit 的 VNI 用于标示不同的二层网段，能够支持 16777216 个 LAN。

**VXLAN Tunnel Endpoint**

VXLAN 使用 VXLAN tunnel endpoint (VTEP) 设备处理 VXLAN 的封装和解封。每个 VTEP 有一个 IP interface，配置了一个 IP 地址。VTEP 使用该 IP 封装 Layer 2 frame，并通过该 IP interface 传输和接收封装后的 VXLAN 数据包。

下面是 VTEP 的示意图：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211112135.jpg)

VXLAN 独立于底层的网络拓扑；反过来，两个 VTEP 之间的底层 IP 网络也独立于 VXLAN。VXLAN 数据包是根据外层的 IP header 路由的，该 header 将两端的 VTEP IP 作为源和目标 IP。

**VXLAN 包转发流程**

VXLAN 在 VTEP 间建立隧道，通过 Layer 3 网络传输封装后的 Layer 2 数据。下面例子演示了数据如何在 VXLAN 上传输：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211113213.jpg)

图中 Host-A 和 Host-B 位于 VNI 10 的 VXLAN，通过 VTEP-1 和 VTEP-2 之间建立的 VXLAN 隧道通信。数据传输过程如下：

1. Host-A 向 Host-B 发送数据时，Host-B 的 MAC 和 IP 作为数据包的目标 MAC 和 IP，Host-A 的 MAC 作为数据包的源 MAC 和 IP，然后通过 VTEP-1 将数据发送出去。
2. VTEP-1 从自己维护的映射表中找到 MAC-B 对应的 VTEP-2，然后执行 VXLAN 封装，加上 VXLAN 头，UDP 头，以及外层 IP 和 MAC 头。此时的外层 IP 头，目标地址为 VTEP-2 的 IP，源地址为 VTEP-1 的 IP。同时由于下一跳是 Router-1，所以外层 MAC 头中目标地址为 Router-1 的 MAC。
3. 数据包从 VTEP-1 发送出后，外部网络的路由器会依据外层 IP 头进行路由，最后到达与 VTEP-2 连接的路由器 Router-2。
4. Router-2 将数据包发送给 VTEP-2。VTEP-2 负责解封数据包，依次去掉外层 MAC 头，外层 IP 头，UDP 头 和 VXLAN 头。VTEP-2 依据目标 MAC 地址将数据包发送给 Host-B。

上面的流程是 VTEP 是 VXLAN 的最核心组件，负责数据的封装和解封。

隧道也是建立在 VTEP 之间的，VTEP 负责数据的传送。

**Linux 对 VXLAN 的支持**

VTEP 可以由专有硬件来实现，也可以使用纯软件实现。目前比较成熟的 VTEP 软件实现包括：

1. 带 VXLAN 内核模块的 Linux
2. Open vSwitch

先来看 Linux 如何支持 VXLAN。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211113223.jpg)

实现方式：

1. Linux vxlan 创建一个 UDP Socket，默认在 8472 端口监听。
2. Linux vxlan 在 UDP socket 上接收到 vxlan 包后，解包，然后根据其中的 vxlan ID 将它转给某个 vxlan interface，然后再通过它所连接的 linux bridge 转给虚机。
3. Linux vxlan 在收到虚机发来的数据包后，将其封装为多播 UDP 包，从网卡发出。

# 配置 VXLAN

介绍如何在 ML2 中启用 VXLAN。在 /etc/neutron/plugins/ml2/ml2_conf.ini 设置 vxlan network 相关参数。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211114055.jpg)

``` xml
tenant_network_types = vxlan
```

指定普通用户创建的网络类型为 vxlan。这里还使用了一个名为 “l2population” mechanism driver

然后指定 vxlan 的范围。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211114115.jpg)

上面的配置定义了 vxlan vni 的范围是 1001 - 2000。这个范围是针对普通用户在自己的租户里创建 vxlan network 的范围。因为普通用户创建 network 时并不能指定 vni，Neutron 会按顺序自动从这个范围中取值。

对于 admin 则没有 vni 范围的限制，admin 可以创建 vni 范围为 1-16777216 的 vxlan network。

接着需要在 [VXLAN] 中配置 VTEP。控制节点 devstack_controller 的 ml2_conf.ini 配置如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211114146.jpg)

计算节点 devstack_compute01 的 ml2_conf.ini 配置如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211114154.jpg)

local_ip 指定节点上用作 VTEP 的 IP 地址。

devstack_controller 的 VTEP IP 是 166.66.16.10，网卡为 eth1。

devstack_compute01 的 VTEP IP 是 166.66.16.11，网卡为 eth1。

**注意**：作为准备工作，这两个 VTEP IP 需要提前配置到节点的 eht1 上，Neutron 并不会分配这个 IP。

# 创建 VXLAN

通过 Web UI 创建 vxlan100_net 并观察节点网络结构的变化。

打开菜单 Admin -> Networks，点击 “Create Network” 按钮

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211152820.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211152829.jpg)

Provider Network Type 选择 “VXLAN”。Segmentation ID 即 VNI，设置为 100。点击 “Create Network”，vxlan100 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211152943.jpg)

点击 vxlan100 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211152954.jpg)

创建 subnet\_172\_16\_100\_0，IP 地址为 172.16.100.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153016.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153023.jpg)

## 底层网络发生了什么变化

在控制节点上执行 brctl show，查看当前的网络结构。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153331.jpg)

Neutron 创建了：

1. vxlan100 对应的网桥 brq1762d312-d4
2. vxlan interface vxlan-100
3. dhcp 的 tap 设备 tap4df76d0e-59

vxlan-100 和 tap4df76d0e-59 已经连接到 brq1762d312-d4，vxlan-100 的二层网络就绪。执行` ip -d link show dev vxlan-100 `查看 vxlan interface 的详细配置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153503.jpg)

可见，vxlan-100 的 VNI 是 100，对应的 VTEP 网络接口为 eth1。此时 vxlan100 结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153516.jpg)

## 部署 instance 到 VXLAN

launch 新的 instance “cirros-vm1”，网络选择 vxlan100。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153656.jpg)

cirros-vm1 分配到的 IP 为 172.16.100.3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153703.jpg)

cirros-vm1 被 schedule 到控制节点，对应的 tap 设备为 tap099caa87-cd，并且连接到 bridge brq1762d312-d4。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153729.jpg)

当前 vxlan100 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153738.jpg)

继续用同样的方式 launch instance cirros-vm2，分配到的 IP 为 172.16.100.4。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153758.jpg)

cirros-vm2 被 schedule 到计算节点，对应的 tap 设备为 tap457cc048-aa，并且连接到 bridge brq1762d312-d4。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153815.jpg)

因为计算节点上没有 hdcp 服务，所以没有相应的 tap 设备。另外，bridge 的名称与控制节点上一致，都是 brq1762d312-d4，表明是同一个 network。

当前 vxlan100 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153902.jpg)

cirros-vm1（172.16.100.3） 与 cirros-vm2（172.16.100.4） 位于不同节点，通过 vxlan100 相连，下面用 PING 验证连通性。在 cirros-vm1 控制台中执行 ping 172.16.100.4：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211153912.jpg)

如我们预料，ping 成功。

对于多 vxlan 之间的 routing 以及 floating ip ，实现方式与 vlan 非常类似，这里不再赘述。

# L2 Population

L2 Population 是用来提高 VXLAN 网络 Scalability 的。

通常我们说某个系统的 Scalability 好，其意思是：当系统的规模变大时，仍然能够高效地工作。

L2 Population 到底解决了怎样的 Scalability 问题？请看下图：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211155718.jpg)

这是一个包含 5 个节点的 VXLAN 网络，每个节点上运行了若干 VM。

现在假设 Host 1 上的 VM A 想与 Host 4 上的 VM G 通信。VM A 做的第一步是获知 VM G 的 MAC 地址。于是 VM A 需要在整个 VXLAN 网络中广播 APR 报文：“VM G 的 MAC 地址是多少？”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211155722.jpg)

如果 VXLAN 网络的节点很多，广播的成本会很大，这样 Scalability 就成问题了。幸好 L2 Population 出现了。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211155834.jpg)

L2 Population 的作用是在 VTEP 上提供 Porxy ARP 功能，使得 VTEP 能够预先获知 VXLAN 网络中如下信息：

1. VM IP -- MAC 对应关系
2. VM -- VTEP 的对应关系

当 VM A 需要与 VM G 通信时：

1. Host 1 上的 VTEP 直接响应 VM A 的 APR 请求，告之 VM G 的 MAC 地址。
2. 因为 Host 1 上的 VTEP 知道 VM G 位于 Host 4，会将封装好的 VXLAN 数据包直接发送给 Host 4 的 VTEP。

这样就解决了 MAC 地址学习和 APR 广播的问题，从而保证了 VXLAN 的 Scalability。

那么下一个关键问题是：**VTEP 是如何提前获知 IP -- MAC -- VTEP 相关信息的呢？**

答案是：

1. Neutron 知道每一个 port 的状态和信息； port 保存了 IP，MAC 相关数据。
2. instance 启动时，其 port 状态变化过程为：down -> build -> active。
3. 每当 port 状态发生变化时，Neutron 都会通过 RPC 消息通知各节点上的 Neutron agent，使得 VTEP 能够更新 VM 和 port 的相关信息。

VTEP 可以根据这些信息判断出其他 Host 上都有哪些 VM，以及它们的 MAC 地址，这样就能直接与之通信，从而避免了不必要的隧道连接和广播。

# 配置 L2 Population

目前 L2 Population 支持 VXLAN with Linux bridge 和 VXLAN/GRE with OVS。

可以通过以下配置启用 L2 Population。

在 ` /etc/neutron/plugins/ml2/ml2_conf.ini `设置 l2population mechanism driver。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211160131.jpg)

``` shell
mechanism_drivers = linuxbridge,l2population
```

同时在 [VXLAN] 中配置 enable L2 Population。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211160147.jpg)

L2 Population 生效后，创建的 vxlan-100 会多一个 Proxy ARP 功能。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211160216.jpg)

查看控制节点上的 forwarding database，可以看到 VTEP 保存了 cirros-vm2 的 port 信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211160223.jpg)

cirros-vm2 的 MAC 为 fa:16:3e:1d:23:a3。VTEP IP 为 166.66.16.11。

当需要与 cirros-vm2 通信时，控制节点 VTEP 166.66.16.10 会将封装好的 VXLAN 数据包直接发送给计算节点的 VTEP 166.66.16.11。

我们再查看一下计算节点上的 forwarding database：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211160254.jpg)

fdb 中保存了 cirros-vm1 和 dhcp 的 port 信息。当需要与它们通信时，计算节点 VTEP 知道应该将数据包直接发送给控制节点的 VTEP。



































