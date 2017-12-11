# 访问外网 ML2 的配置

这里的外部网络是指的租户网络以外的网络。租户网络是由 Neutron 创建和维护的网络。外部网络不由 Neutron 创建。如果是私有云，外部网络通常指的是公司 intranet；如果是公有云，外部网络通常指的是 internet。

具体到实验网络环境：计算节点和控制节点 eth1 提供的是租户网络，IP 段租户可以自由设置。控制节点 eth2 连接的就是外部网络，IP 网段为 10.10.10.0/24。如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211094831.jpg)

## 配置准备

为了连接外部网络，需要在配置文件中告诉 Neutron 外部网络的类型以及对应的物理网卡。因为外部网络是已经存在的物理网络，一般都是 flat 或者 vlan 类型。

这里我们将外部网络的 label 命名为 “external”。如果类型为 flat，控制节点` /etc/neutron/plugins/ml2/ml2_conf.ini `配置如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095054.jpg)

如果类型为 vlan，配置如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095202.jpg)

修改配置后，需要重启 neutron 的相关服务。在实验的网络环境中，外部网络是 flat 类型。

虽然外部网络是已经存在的网络，但我们还是需要在 Neutron 中定义外部网络的对象，这样 router 才知道如何将租户网络和外部网络连接起来。

# 通过 UI 创建ext_net

进入 Admin -> Networks 菜单，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095436.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095454.jpg)

Provider Network Type 选择 “Flat”

Physical Network 填写 “external”，**与 ml2_conf.ini 中 flat_networks 参数的设置保持一致**。

勾选 External Network 选择框。

点击 “Create Network”，ext_net 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095530.jpg)

点击 ext_net 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095556.jpg)

创建 subnet\_10\_10\_10\_0，IP 地址为 10.10.10.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095601.jpg)

这里 Gateway 我们使用默认地址 10.10.10.1。通常我们需要询问网络管理员外网 subnet 的 Gateway IP，然后填在这里。

点击 “Next”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095632.jpg)

因为我们不会直接为 instance 分配外网 IP，所以一般不需要 enable DHCP。

点击 “Create”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095641.jpg)

subnet 创建成功，网关为 10.10.10.1。下面查看控制节点网络结构的变化，执行 brctl show：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211095719.jpg)

增加了一个网桥 brqe496d3d2-53，物理网卡 eth2 已经连接到该 bridge。

# 外网访问原理分析

将外网连接到 Neutron 的虚拟路由器，这样 instance 才能访问外网。点击菜单 Project -> Network -> Routers 进入 router 列表。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211100916.jpg)

点击 router\_100\_101 的 “Set Gateway” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211101005.jpg)

在 “External Network” 下拉列表中选择 ext_net，点击 “Set Gateway”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103559.jpg)

外网设置成功。

我们需要看看 router 发生了什么变化。点击 “router\_100\_101” 链接，打开 “Interfaces” 标签页。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103603.jpg)

router 多了一个 interface，IP 为 10.10.10.2。该 interface 用于连接外网 ext_net。查看控制节点的网络结构，外网 bridge 上已经连接了 router 的 tap 设备 tapb8b32a88-03。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103640.png)

在 router 的 namespace 中查看 tapb8b32a88-03 的 veth pair 设备。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103727.jpg)

该 veth pair 命名为 qg-b8b32a88-03，上面配置了 IP 10.10.10.2。

router 的每个 interface 在 namespace 中都有对应的 veth。如果 veth 用于连接租户网络，命名格式为 qr-xxx，比如 qr-d568ba1a-74 和 qr-e17162c5-00。如果 veth 用于连接外部网络，命名格式为 qg-xxx，比如 qg-b8b32a88-03。

查看 router 的路由表信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103731.jpg)

可以看到默认网关为 10.10.10.1。意味着对于访问 vlan100 和 vlan101 租户网络以外的所有流量，router\_100\_101 都将转发给 ext_net 的网关 10.10.10.1。

现在 router\_100\_101 已经同时连接了 vlan100, vlan101 和 ext_net 三个网络，如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103735.jpg)

下面我们在 cirros-vm3 上测试一下。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103824.jpg)

cirros-vm3 位于计算节点，现在已经可以 Ping 到 ext_net 网关 10.10.10.1 了。通过 traceroute 查看一下 cirros-vm3 到 10.10.10.1 的路径

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103834.jpg)

数据包经过两跳到达 10.10.10.1 网关。

1. 数据包首先发送到 router\_100\_101 连接 vlan101 的 interface（172.16.101.1）。
2. 然后通过连接 ext_net 的 interface（10.10.10.2） 转发出去，最后到达 10.10.10.1。

当数据包从 router 连接外网的接口 qg-b8b32a88-03 发出的时候，会做一次 Source NAT，即将包的源地址修改为 router 的接口地址 10.10.10.2，这样就能够保证目的端能够将应答的包发回给 router，然后再转发回源端 instance。

可以通过 iptables 命令查看 SNAT 的规则。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103844.jpg)

当 cirros-vm3（172.16.101.3） Ping 10.10.10.1 时，可用通过 tcpdump 分别观察 router 两个 interface 的 icmp 数据包来验证 SNAT 的行为。

vlan101 interface qr-e17162c5-00 的 tcpdump 输出：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103913.jpg)

ext_net interface qg-b8b32a88-03 的 tcpdump 输出：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211103928.jpg)

SNAT 让 instance 能够直接访问外网，但外网还不能直接访问 instance。因为 instance 没有外网 IP。这里 “直接访问 instance” 是指通信连接由外网发起，例如从外网 SSH cirros-vm3。

# floating IP

当租户网络连接到 Neutron router，通常将 router 作为默认网关。当 router 接收到 instance 的数据包，并将其转发到外网时:

1. router 会修改包的源地址为自己的外网地址，这样确保数据包转发到外网，并能够从外网返回。
2. router 修改返回的数据包，并转发给真正的 instance。

这个行为被称作 Source NAT 。

如果需要从外网直接访问 instance，则可以利用 floating IP。下面是关于 floating IP 必须知道的事实：

1. floating IP 提供静态 NAT 功能，建立外网 IP 与 instance 租户网络 IP 的一对一映射。
2. floating IP 是配置在 router 提供网关的外网 interface 上的，而非 instance 中。
3. router 会根据通信的方向修改数据包的源或者目的地址。

点击 Project -> Compute -> Access & Security 菜单，打开 Floating IPs 标签页。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211110204.jpg)

点击 “Allocate IP To Project” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211110211.jpg)

floating IP Pool 为 ext_net，点击 “Allocate IP” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211110249.jpg)

从 Pool 中成功分配了一个 IP 10.10.10.3。下面我们将它分配给 cirror-vm3，点击 “Associate” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211110303.jpg)

在下拉列表中选择 cirror-vm3，点击 “Associate” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211110329.jpg)

分配成功，floating IP 10.10.10.3 已经对应到 cirros-vm3 的租户 IP 172.16.101.3。

# floating IP 原理分析

首先查看 router 的 interface 配置：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211110810.jpg)

可以看到，floating IP 已经配置到 router 的外网 interface qg-b8b32a88-03 上。查看 router 的 NAT 规则：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211110819.jpg)

iptables 增加了两条处理 floating IP 的规则：

1. 当 router 接收到从外网发来的包，如果目的地址是 floating IP 10.10.10.3，将目的地址修改为 cirros-vm3 的 IP 172.16.101.3。这样外网的包就能送达到 cirros-vm3。
2. 当 cirros-vm3 发送数据到外网，源地址 172.16.101.3 将被修改为 floating IP 10.10.10.3。

通过 PING 测试一下。在实验环境中，10.10.10.1 是外网中的物理交换机，现在让它 PING cirros-vm3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211111039.jpg)

能够 PING 通。我们通过 tcpdump 可用在 router 的 interface 上观察 floating IP 的行为。ext_net interface qg-b8b32a88-03 的 tcpdump 输出：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211111117.jpg)

可见，在外网接口 qg-b8b32a88-03 上，始终是通过 floating IP 10.10.10.3 与外网通信。vlan101 interface qr-e17162c5-00 的 tcpdump 输出：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211111125.jpg)

当数据转发到租户网络，地址已经变为 cirros-vm3 的租户 IP 172.16.101.3 了。

小结一下：

1. floating IP 能够让外网直接访问租户网络中的 instance。这是通过在 router 上应用 iptalbes 的 NAT 规则实现的。
2. floating IP 是配置在 router 的外网 interface 上的，而非 instance，这一点需要特别注意。



















