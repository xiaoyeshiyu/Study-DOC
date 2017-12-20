# Open vSwitch

## Open vSwitch概述

Linux Bridge 和 Open vSwitch 是目前 OpenStack 中使用最广泛的两种虚机交换机技术。

实验环境两节点的网卡分配方式与 Linux Bridge 一致，如下所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212110745.jpg)

1. 控制节点三个网卡（eth0, eth1, eth2），计算节点两网卡（eth0, eth1）。
2. 合并 Management 和 API 网络，使用 eth0，IP 段为 192.168.104.0/24。
3. VM 网络使用 eht1。
4. 控制节点的 eth2 与 External 网络连接，IP 段为 10.10.10.0/24。

**网络拓扑**

实验环境的网络拓扑如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212110805.jpg)

这个图在 Linux Bridge 实现中也看到过，唯一的区别是：对于节点中的 “Virtual Network Switch” 我们将用 Open vSwitch 替换掉 Linux Bridge。

**配置 openvswitch mechanism driver**

要将 Liunx Bridge 切换成 Open vSwitch，首先需要安装 Open vSwitch 的 agent。修改 devstack 的 local.conf：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212110915.jpg)

重新运行 ./stack，devstack 会自动下载并安装 Open vSwitch。

接下来就可以修改 ML2 的配置文件 /etc/neutron/plugins/ml2/ml2_conf.ini，设置使用 openvswitch mechanism driver。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212110924.jpg)

控制节点和计算节点都需要按照上面的方法安装并配置 Open vSwitch。

Neutron 服务重启后，可以通过 neutron agent-list 命令查看到 neutron-openvswitch-agent 已经在两个节点上运行。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212110930.jpg)

## OVS 中的各种网络设备 

### 初始网络状态

查看一下当前的网络状态。

**控制节点**

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111301.jpg)

ifconfig 显示控制节点上有三个网桥 br-ex，br-int 和 br-tun。从命名上看大致能猜出他们的用途：

**br-ex**

连接外部（external）网络的网桥。

**br-int**

集成（integration）网桥，所有 instance 的虚拟网卡和其他虚拟网络设备都将连接到该网桥。

**br-tun**

隧道（tunnel）网桥，基于隧道技术的 VxLAN 和 GRE 网络将使用该网桥进行通信。

这些网桥都是 Neutron 自动创建的，但是通过 brctl show 命令却看不到它们。这是因为我们使用的是 Open vSwitch 而非 Linux Bridge，需要用 Open vSwitch 的命令 ovs-vsctl show 查看，如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111311.jpg)

**计算节点**

计算节点上也有 br-int 和 br-tun，但没有 br-ext。这是合理的，因为发送到外网的流量是通过网络节点上的虚拟路由器转发出去的，所以 br-ext 只会放在网络节点（devstack-controller）上。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111320.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111322.jpg)

### 了解 Open vSwitch 环境中的各种网络设备

在 Open vSwitch 环境中，一个数据包从 instance 发送到物理网卡大致会经过下面几个类型的设备：

**tap interface**

命名为 tapXXXX。

**linux bridge**

命名为 qbrXXXX。

**veth pair**

命名为 qvbXXXX, qvoXXXX。

**OVS integration bridge**

命名为 br-int。

**OVS patch ports**

命名为 int-br-ethX 和 phy-br-ethX（X 为 interface 的序号）。

**OVS provider bridge**

命名为 br-ethX（X 为 interface 的序号）。

**物理 interface**

命名为 ethX（X 为 interface 的序号）。

**OVS tunnel bridge**

命名为 br-tun。

OVS provider bridge 会在 flat 和 vlan 网络中使用；OVS tunnel bridge 则会在 vxlan 和 gre 网络中使用。

Open vSwitch 支持 local, flat, vlan, vxlan 和 gre 所有五种 network type。

## Local Network

local network 不会与宿主机的任何物理网卡连接，流量只被限制在宿主机内，同时也不关联任何的 VLAN ID。

### 创建第一个 local network

进入菜单 Admin -> Networks，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111720.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111727.jpg)

“Provider Network Type” 选择 “Local”，点击 “Create Network”，first_local_net 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111738.jpg)

点击 first_local_net，进入 network 配置页面。

目前还没有 subnet，点击 “Create Subnet”按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111801.jpg)

设置 IP 地址为 “172.16.1.0/24”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111803.jpg)

点击 “Next”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111830.jpg)

勾选 “Enable DHCP”，IP 池设置为 “172.16.1.2,172.16.1.99”。点击 “Create”，subnet 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111839.jpg)

同时 devstack-controler 针对此 subnet 的 DHCP 服务也已经 Active。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111847.jpg)

#### 底层网络

创建 OVS local network 的过程与 Linux Bridge 没有什么区别。这是因为 Neutron 已经对不同 driver 进行了抽象，但底层肯定是有区别的。

打开控制节点的 shell 终端，用 ovs-vsctl show 查看当前 Open vSwitch 的状态。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111939.jpg)

可以看到 Neutron 自动在 br-int 网桥上创建了` port “tap7970bdcd-f2”`。从命名可知，该 port 对应 local_net 的 dhcp 接口。

与 linux bridge driver 一样，dhcp 设备也是放在命名空间里的。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111948.jpg)

目前网络结构如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212111955.jpg)

### 将 instance 部署到 OVS Local Network

launch 一个 instance，选择 first_local_net 网络：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112345.jpg)

instance 部署成功，分配的 IP 地址为 172.16.1.3

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112352.jpg)

#### 底层网络

对于 instance “cirros-vm1”，Neutron 会在 subnet 中创建一个 port，分配 IP 和 MAC 地址，并将 port 分配给 cirros-vm1。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112402.jpg)

如上图所示，port 列表中增加了一个 port “(fc1c6ebb-719d)”，IP 为 172.16.1.3，点击 port 名称查看 MAC 信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112410.jpg)

cirros-vm1 部署到了控制节点，通过 ovs-vsctl show 查看 bridge 的配置：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112452.jpg)

在 br-int 上多了一个 qvofc1c6ebb-71。用 brctl show 查看一下 linux bridge 的配置：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112718.jpg)

可以看到有一个新建的网桥 qbrfc1c6ebb-71，上面连接了两个设备 qvbfc1c6ebb-71 和 tapfc1c6ebb-71。从命名上看，他们都应该与 cirros-vm1 的虚拟网卡有关。

通过 virsh edit 查看 cirros-vm1 的配置：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112506.jpg)

确实 tapfc1c6ebb-71 是 cirros-vm1 的虚拟网卡。

用 ethtool -S 分别查看 qvbfc1c6ebb-71 和 qvofc1c6ebb-71 的 statistics。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112511.jpg)

原来 qvbfc1c6ebb-71 和 qvofc1c6ebb-71 都是 veth 设备，它们对应的另一端 veth 设备 的 index 分别是 12 和 13。通过 ip a 命令找到 index 12 和 13 的设备。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112613.jpg)

这里可以看到：qvbfc1c6ebb-71 和 qvofc1c6ebb-71 组成了一个 veth pair。，veth pair 是一种成对出现的特殊网络设备，它们象一根虚拟的网线连接两个网络设备。这里 qvbfc1c6ebb-71 和 qvofc1c6ebb-71 的作用就是连接网桥 qbrfc1c6ebb-71 和 br-int。

将前面梳理好的信息通过图片展示出来。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212112621.jpg)

由图所示，tapfc1c6ebb-71 通过 qbrfc1c6ebb-71 间接连接到 br-int。

那问题来了，为什么 tapfc1c6ebb-71 不能像左边的 DHCP 设备 tap7970bdcd-f2 那样直接连接到 br-int 呢？

其原因是： Open vSwitch 目前还不支持将 iptables 规则放在与它直接相连的 tap 设备上。

如果做不到这一点，就无法实现 Security Group 功能。为了支持 Security Group，不得不多引入一个 Linux Bridge 支持 iptables。

这样的后果就是网络结构更复杂了，路径上多了一个 linux bridge 和 一对 veth pair 设备。

### 部署cirros_vm2

### 连接第二个 instance 到 first_local_net

以同样的方式 launch instance “cirros-vm2”，分配的 IP 为 172.16.1.4。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113630.jpg)

cirros-vm2 也被 schedule 到控制节点，ovs-vsctl show 的输出如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113637.jpg)

cirros-vm2 对于的 tap 设备为 tapddbbb728-93。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113648.jpg)

从 cirros-vm2 能够 Ping 通 cirros-vm1 的 IP 地址  172.16.1.3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113656.jpg)

当前宿主机的网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113702.jpg)

两个 instance 都挂在 br-int 上，可以相互通信。

#### 部署second_local_net

为了分析 local network 的连通性，再创建一个 "second_local_net"。

second_local_net 的绝大部分属性与 first_local_net 相同，除了 IP 池范围为 172.16.1.101-172.16.1.200

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113800.jpg)

second_local_net 的 DHCP 设备也已经就绪。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113808.jpg)

DHCP 对应的 tap 设备为 tap2c1b3c58-4e，已经连接到 br-int。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212113816.jpg)

### OVS local network 连通性分析

#### 部署 instance 到 second_local_network

launch 新的 instance “cirros-vm3”，网络选择 second_local_net

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114056.jpg)

cirros-vm3 分配到的 IP 为 172.16.1.102

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114104.jpg)

cirros-vm3 被 schedule 到控制节点，其虚拟网卡也连接到 br-int。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114108.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114110.jpg)

当前的控制节点上的网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114126.jpg)

测试：cirros-vm3 能否 Ping 到 cirros-vm1 

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114135.jpg)

实验证明 cirros-vm3 无法 Ping 到 cirros-vm1。

#### 网络连通性分析

重新审视一下 br-int 上各个 port 的配置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114143.jpg)

可以看到，虚拟网卡和 DHCP 对应的 port 都有一个特殊的 tag 属性。

first_local_net 相关 port 其 tag 为 1；

second_local_net 相关 port 其 tag 为 2。

**分析 **

Open vSwitch 的每个网桥都可以看作一个真正的交换机，**可以支持 VLAN**，这里的 tag 就是 VLAN ID。

br-int 中标记 tag 1 的 port 和 标记 tag 2 的 port 分别属于不同的 VLAN，它们之间是隔离的。

需要特别说明的是： Open vSwitch 中的 tag 是内部 VLAN，用于隔离网桥中的 port，与物理网络中的 VLAN 没有关系。

将 tag 信息添加到网络结构图中，如下所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114154.jpg)

## flat network

flat network 是不带 tag 的网络，宿主机的物理网卡通过网桥与 flat network 连接，每个 flat network 都会占用一个物理网卡。

### 在 ML2 配置中 enable flat network

在控制节点` /etc/neutron/plugins/ml2/ml2_conf.ini `中设置 flat network 相关参数：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114736.jpg)

``` shell
tenant_network_types = flat
```

指定普通用户创建的网络类型为 flat。

需要注意的是：因为 flat 网络与物理网卡一一对应，一般情况下租户网络不会采用 flat，这里只是示例。

接着需要指明 flat 与物理网络的对应关系：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114745.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114751.jpg)

如上所示：

在` [ml2_type_flat] `中通过` flat_networks `定义了一个 flat 网络，label 为 “default”。

在` [ovs] `中通过` bridge_mappings `指明 default 对应的 Open vSwitch 网桥为 br-eth1。

label 是 flat 网络的标识，在创建 flat 时会用到，label 命名可以是任意字符串，只要确保各个节点 ml2_conf.ini 中的 label 命名一致就可以了。各个节点中 label 与物理网卡的对于关系可能不一样。

这是因为每个节点可以使用不同的物理网卡将 instance 连接到 flat network。

与 linux bridge 实现的 flat 网络不同，ml2 中并不会直接指定 label 与物理网卡的对应关系，而是指定 label 与 ovs bridge 的对应关系。

``` shell 
[ovs]
bridge_mappings = default:br-eth1
```

这里的 ovs bridge 是 br-eth1，我们需要提前通过 ovs-ovctl 命令：

1. 创建 br-eth1。
2. 将物理网卡 eth1 桥接在 br-eth1 上。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114805.jpg)

如果要创建多个 flat 网络，需要定义多个 label，用逗号隔开，当然也需要用到多个 ovs bridge，如下所示：

``` shell
[ml2_type_flat] 
flat_networks = flat1,flat2
[ovs]
bridge_mappings = flat1:br-eth1,flat2:br-eth2
```

通过以上步骤控制节点的 flat 网络就准备好了。计算节点也需要做相同的配置，然后重启所有节点的 Neutron 服务。

通过 ovs-vsctl show 检视一下当前的网络结构。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114827.jpg)

对于 ovs bridge “br-eth1” 和其上桥接的 port “eth1” 我们应该不会感到意外，这是前面配置的结果。然而除此之外，br-int 和 br-eth1 分别多了一个 port “int-br-eth1” 和 “phy-br-eth1”，而且这两个 port 都是 “patch” 类型，同时通过 “peer” 指向对方。

上面的配置描述了这样一个事实：br-int 与 br-eht1 这两个网桥通过 int-br-eth1 和 phy-br-eth1 连接在一起了。

目前控制节点网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114832.jpg)

### veth pair VS patch port

在前面 local network 我们看到，br-int 与 linux bridge 之间可以通过 veth pair 连接。

![1](http://oydlbqndl.bkt.clouddn.com/微信图片_20171212114855.jpg)

而这里两个 ovs bridge 之间是用 patch port 连接的。看来 veth pair 和 patch port 都可以连接网桥，使用的时候如何选择。

patch port 是 ovs bridge 自己特有的 port 类型，只能在 ovs 中使用。如果是连接两个 ovs bridge，优先使用 patch port，因为性能更好。

所以：

1. 连接两个 ovs bridge，优先使用 patch port。技术上veth pair 也能实现，但性能不如 patch port。
2. 连接 ovs bridge 和 linux bridge，只能使用 veth pair。
3. 连接两个 linux bridge，只能使用 veth pair。


### 创建 OVS flat network

Admin -> Networks，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218111742.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218111752.jpg)

Provider Network Type 选择 “Flat”。Physical Network 填写 “default”，与 ml2_conf.ini 中 flat_networks 参数值保持一致。

点击 “Create Network”，flat_net 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218111857.jpg)

点击 flat_net 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218111912.jpg)

设置 IP 地址为 “172.16.1.0/24”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218111920.jpg)

点击 “Next”，勾选 “Enable DHCP”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218112021.jpg)

点击 “Create”，subnet 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218112109.jpg)

**底层网络的变化**

查看控制节点的网络结构，执行 ovs-vsctl show：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218112039.jpg)

Neutron 自动在 br-int 网桥上创建了 flat-net dhcp 的接口 “tap83421c44-93”。

此时 flat_net 结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218112156.jpg)

#### 部署 instance

launch 新的 instance “cirros-vm1”，网络选择 falt_net。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218112418.jpg)

cirros-vm1 分配到的 IP 为 172.16.1.3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218112427.jpg)

cirros-vm1 被 schedule 到控制节点，其虚拟网卡也连接到 br-int。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218114304.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218114307.jpg)

虚拟网卡与 br-int 的连接方式与 local 网络是一样的。

当前 flat_net 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218114312.jpg)

继续用同样的方式 launch instance cirros-vm2，分配到的 IP 为 172.16.1.4。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218114328.jpg)

cirros-vm2 被 schedule 到计算节点，虚拟网卡已经连接到 br-int。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218114544.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218114547.jpg)

因为计算节点上没有 hdcp 服务，所以没有相应的 tap 设备。

当前 flat_net 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218115022.jpg)

cirros-vm1（172.16.1.3） 与 cirros-vm2（172.16.1.4） 位于不同节点，通过 flat_net 相连，下面验证连通性。

在 cirros-vm2 控制台中 ping 172.16.1.3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218115038.jpg)

## vlan network

vlan network 是带 tag 的网络。

在 Open vSwitch 实现方式下，不同 vlan instance 的虚拟网卡都接到 br-int 上。

这一点与 linux bridge 非常不同，linux bridge 是不同 vlan 接到不同的网桥上。

在实验环境中，收发 vlan 数据的物理网卡为 eth1，上面可以走多个 vlan，

所以物理交换机上与 eth1 相连的的 port 要设置成 trunk 模式，而不是 access 模式。

### 在 ML2 配置中 enable vlan network

在` /etc/neutron/plugins/ml2/ml2_conf.ini `设置 vlan network 相关参数：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218162359.jpg)

``` shell     
tenant_network_types = vlan
```

指定普通用户创建的网络类型为 vlan。

然后指定 vlan 的范围：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218162419.jpg)

上面配置定义了 label 为 “default” 的 vlan network，vlan id 的范围是 3001 - 4000。

这个范围是针对普通用户在自己的租户里创建 network 的范围。

因为普通用户创建 network 时并不能指定 vlan id，Neutron 会按顺序自动从这个范围中取值。

对于 admin 则没有 vlan id 的限制，admin 可以创建 id 范围为 1-4094 的 vlan network。

接下来指明 vlan 网络与物理网络的对应关系：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218162438.jpg)

如上所示：

在` [ml2_type_vlan] `中定义了 lable “default”，

[ovs] 中则通过` bridge_mappings `指明 default 对应的 Open vSwitch 网桥为 br-eth1。

这里 label 的作用与前面 flat network 中的 label 一样，只是一个标示，可以是任何字符串。

需要提前通过 ovs-ovctl 命令：

1. 创建 br-eth1。
2. 将物理网卡 eth1 桥接在 br-eth1 上。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218162446.jpg)

### 创建 OVS vlan100 netwrok

打开菜单 Admin -> Networks，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218163210.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218163734.jpg)

Provider Network Type 选择 “VLAN”。Physical Network 填写 “default”，与 ml2_conf.ini 中 network_vlan_ranges 参数值保持一致。Segmentation ID 即 VLAN ID，设置为 100。

点击 “Create Network”，vlan100 创建成功

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218163741.jpg)

点击 vlan100 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218163752.jpg)

创建 subnet\_172\_16\_100\_0，IP 地址为 172.16.100.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218163800.jpg)

**底层网络**

在控制节点上执行 ovs-vsctl show，查看网络结构：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164022.jpg)

Neutron 自动在 br-int 网桥上创建了 vlan100 dhcp 的接口 “tap43567363-50”。
此时 vlan100 结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164032.jpg)

### 部署 instance 到 OVS vlan100

launch 新的 instance “cirros-vm1”，网络选择 vlan100。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164659.jpg)

cirros-vm1 分配到的 IP 为 172.16.100.3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164739.jpg)

cirros-vm1 被 schedule 到控制节点，其虚拟网卡也连接到 br-int。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164747.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164930.jpg)

虚拟网卡与 br-int 的连接方式与 local 和 flat 网络没有任何区别。

当前 vlan100 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164758.jpg)

继续用同样的方式 launch instance cirros-vm2，分配到的 IP 为 172.16.100.104。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164807.jpg)

cirros-vm2 被 schedule 到计算节点，虚拟网卡已经连接到 br-int。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164814.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164822.jpg)

因为计算节点上没有 hdcp 服务，所以没有相应的 tap 设备。

当前 vlan100 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164826.jpg)

cirros-vm1（172.16.100.3） 与 cirros-vm2（172.16.100.4） 位于不同节点，通过 vlan100 相连，下面执行 PING 验证连通性。

在 cirros-vm1 控制台中执行 ping 172.16.100.4：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218164835.jpg)

### 创建 OVS vlan101并部署 instance

前面创建了 OVS vlan100 并部署了 instance，现在再创建 vlan101。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165343.jpg)

subnet IP 地址为 172.16.101.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165351.jpg)

**底层网络**

Neutron 自动在 br-int 网桥上创建了 vlan100 dhcp 的接口 “tap1820558c-0a”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165411.jpg)

现在，网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165421.jpg)

**将 instance 连接到 vlan101**

launch 新的 instance “cirros-vm3”，网络选择 vlan101。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165438.jpg)

cirros-vm3 分配到的 IP 为 172.16.101.103。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165446.jpg)

cirros-vm3 被 schedule 到计算节点，虚拟网卡已经连接到 br-int。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165453.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165500.jpg)

当前网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218165514.jpg)

cirros-vm1 位于控制节点，属于 vlan100。

cirros-vm2 位于计算节点，属于 vlan100。

cirros-vm3 位于计算节点，属于 vlan101。

cirros-vm1 与 cirros-vm2 都在 vlan100，它们之间能通信。

cirros-vm3 在 vlan101，不能与 cirros-vm1 和 cirros-vm2 通信。

### 分析 OVS 如何实现 vlan 隔离

**Open vSwitch ****是如何实现**** vlan100 和 vlan101 的隔离**？

当前拓扑结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218171949.jpg)

cirros-vm1 位于控制节点，属于 vlan100。

cirros-vm2 位于计算节点，属于 vlan100。

cirros-vm3 位于计算节点，属于 vlan101。

今天详细分析 OVS 如何实现 vlan100 和 vlan101 的隔离。

与 Linux Bridge driver 不同，Open vSwitch driver 并不通过 eth1.100, eth1.101 等 VLAN interface 来隔离不同的 VLAN。所有的 instance 都连接到同一个网桥 br-int，**Open vSwitch 通过 flow rule（流规则）来指定如何对进出 br-int 的数据进行转发，进而实现 vlan 之间的隔离**。

具体来说：当数据进出 br-int 时，flow rule 可以修改、添加或者剥掉数据包的 VLAN tag，Neutron 负责创建这些 flow rule 并将它们配置到 br-int，br-eth1 等 Open vSwitch 上。

下面我们就来研究一下当前的 flow rule。

查看 flow rule 的命令是` ovs-ofctl dump-flow < bridge > `

首先查看计算节点 br-eth1 的 flow rule:

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218171954.jpg)

br-eth1 上配置了四条 rule，每条 rule 有不少属性，其中比较重要的属性有：

**priority**

rule 的优先级，值越大优先级越高。Open vSwitch 会按照优先级从高到低应用规则。

**in_port**

inbound 端口编号，每个 port 在 Open vSwitch 中会有一个内部的编号。

可以通过命令` ovs-ofctl show < bridge> `查看 port 编号。

比如 br-eth1：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218172005.jpg)

eth1 编号为 1；phy-br-eth1 编号为 2。

**dl_vlan**

数据包原始的 VLAN ID。

**actions**

对数据包进行的操作。

br-eth1 跟 VLAN 相关的 flow rule 是前面两条，下面来详细分析。

清晰起见，只保留重要的信息，如下：

``` shell
priority=4,in_port=2,dl_vlan=1 actions=mod_vlan_vid:100,NORMAL
priority=4,in_port=2,dl_vlan=5 actions=mod_vlan_vid:101,NORMAL
```

第一条的含义是：

从 br-eth1 的端口 phy-br-eth1（in_port=2）接收进来的包，如果 VLAN ID 是 1（dl_vlan=1），那么需要将 VLAN ID 改为 100（actions=mod_vlan_vid:100）

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218172016.jpg)

从上面的网络结构我们可知，phy-br-eth1 连接的是 br-int，phy-br-eth1 的 inbound 包实际上就是 instance 通过 br-int 发送给物理网卡的数据。

将 VLAN ID 1 改为 VLAN ID 100的解释是

看下面计算节点 ovs-vsctl show 的输出：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218172025.jpg)

br-int 通过 tag 隔离不同的 port，这个 tag 可以看成内部的 VLAN ID。

从 qvo4139d09b-30（对应 cirros-vm2，vlan100）进入的数据包会被打上 1 的 VLAN tag。

从 qvo98582dc9-db（对应 cirros-vm3，vlan101）进入的数据包会被打上 5 的 VLAN tag。

因为 br-int 中的 VLAN ID 跟物理网络中的 VLAN ID 并不相同，所以当 br-eth1 接收到 br-int 发来的数据包时，需要对 VLAN 进行转换。Neutron 负责维护 VLAN ID 的对应关系，并将转换规则配置在 flow rule 中。

理解了 br-eth1 的 flow rule，我们再来分析 br-int 的 flow rule。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218172035.jpg)

最关键的是下面两条

``` shell
priority=3,inport=1,dl_vlan=100 actions=mod_vlan_vid:1,NORMAL
priority=3,inport=1,dl_vlan=101 actions=mod_vlan_vid:5,NORMAL
```

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218172043.jpg)

port 1 为 int-br-eth1，那么这两条规则的含义就应该是：

1. 从物理网卡接收进来的数据包，如果 VLAN 为 100，则改为内部 VLAN 1。
2. 从物理网卡接收进来的数据包，如果 VLAN 为 101，则将为内部 VLAN 5。

简单的说，数据包在物理网络中通过 VLAN 100 和 VLAN 101 隔离，在计算节点 OVS br-int 中则是通过内部 VLAN 1 和 VLAN 5 隔离。

控制节点的 flow rule 非常类似。

## Route

Neutron Routing 服务提供跨 subnet 互联互通的能力。

例如前面搭建了实验环境：

| VM         | IP           | Vlan    |
| ---------- | ------------ | ------- |
| cirros-vm1 | 172.16.100.3 | vlan100 |
| cirros-vm3 | 172.16.101.3 | vlan101 |

这两个 instance 要通信必须借助 router。可以是物理 router 或者虚拟 router。

下面详细讨论 Neutron 的虚拟 router 实现。

### 配置 l3 agent

Neutron 的路由服务是由 l3 agent 提供的。

l3 agent 需要正确配置才能工作，配置文件为` /etc/neutron/l3_agent.ini `，位于控制节点或网络节点。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174440.jpg)

devstack 已经配置默认的属性，大部分情况下不需要修改就可以使用。

external_network_bridge 指定连接外网的网桥，默认是 br-ex。

interface_driver 是最重要的选项，如果 mechanism driver 是 open vswitch，则：

``` shell
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```

如果选用 linux bridge，则：

``` shell
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
```

l3 agent 运行在控制或网络节点。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174500.jpg)

下面将创建虚拟 router “router\_100\_101”，打通 vlan 100 和 vlan 101。

**创建 router**

进入操作菜单 Project -> Network -> Routers。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174509.jpg)

点击 “Create Router” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174517.jpg)

router 命名为 “router\_100\_101”，点击 “Create Router” 按钮确认。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174525.jpg)

router\_100\_101 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174824.jpg)

接下来需要将 vlan100 和 vlan101 连接到 router\_100\_101。

点击 “router\_100\_101” 链接进入 router 的配置页面，在 “Interfaces” 标签中点击 “Add Interface” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174546.jpg)

选择 vlan100 的 subnet\_172\_16\_100\_0，点击 “Add Interface” 确认。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174913.jpg)

用同样的方法添加 vlan101 的 subnet\_172\_16\_101\_0。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174605.jpg)

完成后，可以看到 router\_100\_101 有了两个 interface，其 IP 正好是 subnet 的 Gateway IP 172.16.100.1 和 172.16.101.1。

到这里，可以预见：

1. router\_100\_101 已经连接了 subnet\_172\_16\_100\_0 和 subnet\_172\_16\_101\_0。
2. router\_100\_101 上已经设置好了两个 subnet 的 Gateway IP。
3. cirros-vm1 和 cirros-vm3 应该可以通信了。

通过 PING 测试一下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218174612.jpg)

cirros-vm1 和 cirros-vm3 能通信了。

### Neutron Router 工作原理

首先查看控制节点的网络结构发生了什么变化：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180334.jpg)

br-int 上多了两个 port:

1. qr-d295b258-45，从命名上可以推断该 interface 对应 router\_100\_101 的 interface (d295b258-4586)，是 subnet\_172\_16\_100\_0 的网关。
2. qr-2ffdb861-73，从命名上可以推断该 interface 对应 router\_100\_101 的 interface (2ffdb861-731c)，是 subnet\_172\_16\_101\_0 的网关。

与 linux bridge 实现方式一样， router\_100\_101 运行在自己的 namespace 中。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180344.jpg)

如上所示，qrouter-a81cc110-16f4-4d6c-89d2-8af91cec9714 为 router 的 namespace，两个 Gateway IP 分别配置在 qr-2ffdb861-73 和 qr-d295b258-45 上。

当前网络结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180354.jpg)

route\_101\_101 上配置了 vlan100 和 vlan101 的网关，两个网络在三层上就打通了。

### 访问 Neutron 外部网络

这里的外部网络是指的租户网络以外的网络。租户网络是由 Neutron 创建和维护的网络。 外部网络不由 Neutron 创建。如果是私有云，外部网络通常指的是公司 intranet；如果是公有云，外部网络通常指的是 internet。

具体到我们的实验网络环境： 计算节点和控制节点 eth1 提供的是租户网络，IP 段租户可以自由设置。 控制节点 eth2 连接的就是外部网络，IP 网段为 10.10.10.2/24。如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180644.jpg)

###### **配置准备**

为了连接外部网络，需要预先在配置文件中告诉 Neutron 外部网络的类型以及对应的 Open vSwitch 网桥。 外部网络是已经存在的物理网络，一般都是 flat 或者 vlan 类型。

这里将外部网络的 label 命名为 “external”，网桥为 br-ex。 如果类型为 flat，控制节点 ` /etc/neutron/plugins/ml2/ml2_conf.ini ` 配置如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180654.jpg)

如果类型为 vlan，配置如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180701.jpg)

在我们的网络环境中，外部网络是 flat 类型。 修改配置后，需要重启 neutron 的相关服务。另外，需要提前准备好 br-ex，将 eth2 添加到 br-ex。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180708.jpg)

br-ex 已经存在，只需要添加 eth2。

### 创建 OVS 外部网络 ext_net

进入 Admin -> Networks 菜单，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180902.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180910.jpg)

Provider Network Type 选择 “Flat”。 

Network 填写 “external”，与 ml2_conf.ini 中 flat_networks 的参数值保持一致。 

勾选 External Network 选择框。 

点击 “Create Network”，ext_net 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180920.jpg)

点击 ext_net 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180929.jpg)

创建 subnet\_10\_10\_10\_0，IP 地址为 10.10.10.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180938.jpg)

这里 Gateway 使用默认地址 10.10.10.1。 通常生产换进下需要询问网络管理员外网 subnet 的 Gateway IP，然后填在这里。

点击 “Next”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218180952.jpg)

因为我们不会直接为 instance 分配外网 IP，所以不需要 enable DHCP。点击 “Create”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181001.jpg)

subnet 创建成功，网关为 10.10.10.1。下面查看控制节点网络结构的变化，执行 ovs-vsctl show：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181009.jpg)

上图所示，br-ex 与 br-int 通过 patch port “phy-br-ex” 和 “int-br-ex” 连接。

### 将 ext_net 连接到 router

点击菜单 Project -> Network -> Routers 进入 router 列表。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181331.jpg)

点击 router\_100\_101 的 “Set Gateway” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181339.jpg)

在 “External Network” 下拉列表中选择 ext_net，点击 “Set Gateway”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181347.jpg)

外网设置成功。看看 router 发生了什么变化。 点击 “router\_100\_101” 链接，打开 “Interfaces” 标签页。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181355.jpg)

router 多了一个新 interface，IP 为 10.10.10.2。 该 interface 用于连接外网 ext_net，对应的 br-ex 的 port “qg-cf54d3ea-6a”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181404.jpg)

在 router 的 namespace 中查可以看到 qg-cf54d3ea-6a 已经配置了 IP 10.10.10.2。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181412.jpg)

router interface 的命名规则如下： 

1. 如果 interface 用于连接租户网络，命名格式为 qr-xxx。 

2. 如果 interface 用于连接外部网络，命名格式为 qg-xxx。

查看 router 的路由表信息：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181614.jpg)

可以看到默认网关为 10.10.10.1。 

意味着对于访问 vlan100 和 vlan101 租户网络以外的所有流量，router\_100\_101 都将转发给 ext_net 的网关 10.10.10.1。

现在 router\_100\_101 已经同时连接了 vlan100, vlan101 和 ext_net 三个网络，如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181429.jpg)

我们在 cirros-vm3 上测试一下。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181436.jpg)

cirros-vm3 位于计算节点，现在已经可以 Ping 到 ext_net 网关 10.10.10.1 了。 通过 traceroute 查看一下 cirros-vm3 到 10.10.10.1 的路径：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218181445.jpg)

数据包经过两跳到达 10.10.10.1 网关。 

1. 数据包首先发送到 router\_100\_101 连接 vlan101 的 interface（172.16.101.1）。
2. 然后通过连接 ext_net 的 interface（10.10.10.2） 转发出去，最后到达 10.10.10.1。

当数据包从 router 连接外网的接口 qg-cf54d3ea-6a 发出的时候，会做一次 Source NAT，将包的源地址修改为 router 的接口地址 10.10.10.2，这样就能够保证目的端能够将应答的包发回给 router，然后再转发回源端 instance。

**floating IP**

通过 SNAT 使得 instance 能够直接访问外网，但外网还不能直接访问 instance。直接访问 instance 指的是通信连接由外网发起，例如从外网 SSH instance。 如果需要从外网直接访问 instance，可以利用 floating IP。

Open vSwitch driver 环境中 floating IP 的实现与 Linux Bridge driver 完全一样：都是通过在 router 提供网关的外网 interface 上配置 iptables NAT 规则实现。有关 floating IP 的详细分析可以参考 Linux Bridge 中 floating IP 的相关章节。

## OVS VxLAN

Open vSwitch 支持 VXLAN 和 GRE 这两种 overlay network。因为 OpenStack 对于 VXLAN 与 GRE 配置和实现差别不大，这里只讨论如何实施 VXLAN。

**在 ML2 配置中 enable vxlan network**

在` /etc/neutron/plugins/ml2/ml2_conf.ini `设置 vxlan network 相关参数。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218182359.jpg)

指定普通用户创建的网络类型为 vxlan，同时 enable l2population mechanism driver，然后指定 vxlan 的范围。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218182410.jpg)

上面配置定义了 vxlan vni 的范围是 1001 - 2000，这个范围是针对普通用户在自己的租户里创建 vxlan network 的范围。 因为普通用户创建 network 时不能指定 vni，Neutron 会按顺序自动从这个范围中取值。

对于 admin 则没有 vni 范围的限制，admin 可以创建 vni 范围为 1-16777216 的 vxlan network。

在 [agent] 中配置启用 vxlan 和 l2population。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218182420.jpg)

最后在 [ovs] 中配置 VTEP。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218182428.jpg)

vxlan tunnel 对应的网桥为 br-tun。 
local_ip 指定 VTEP 的 IP 地址。 
devstack-controller 使用 166.66.16.10，此 IP 配置在网卡 eth1 上。 
devstack-compute01 则使用 166.66.16.11，此 IP 配置在网卡 eth1 上。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218182436.jpg)

**初始网络结构**

Neutron 服务重启后，通过 ovs-vsctl show 查看网络配置：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218182446.jpg)

br-int 与 br-tun 通过 patch port “patch-tun” 和 “br-tun” 连接。 目前网络结构如下所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218182455.jpg)

### 创建 vxlan 并部署 instance

**创建 vxlan100_net**

打开菜单 Admin -> Networks，点 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183605.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183615.jpg)

Provider Network Type 选择 “VXLAN”。 

Segmentation ID 即 VNI，设置为 100。

点击 “Create Network”，vxlan100 创建成功

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183625.jpg)

点击 vxlan100 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183633.jpg)

创建 subnet\_172\_16\_100\_0，IP 地址为 172.16.100.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183643.jpg)

**将 instance 连接到 vxlan100_net**

launch 新的 instance “cirros-vm1”，“cirros-vm2” 网络选择 vxlan100。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183653.jpg)

cirros-vm1，cirros-vm2 分别部署到控制节点和计算节点，IP 如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183702.jpg)

测试 cirros-vm1 和 cirros-vm2 的连通性。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218183809.jpg)

与我们预期相同，cirros-vm1 能 Ping 通 cirros-vm2。

### OVS vxlan 底层结构分析

**控制节点**

执行 ovs-vsctl show：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218184510.jpg)

**br-int**

br-int 连接了如下 port:

1. tap0d4cb13a-7a 是 vxlan100_net 的 DHCP 服务对应的 interface。
2. qvoa2ac3b9a-24 将 cirros-vm1 虚拟网卡连接到 vxlan100_net。

**br-tun**

br-tun 上创建了一个特殊的 port “vxlan-a642100b”，它是 VXLAN 的隧道端点，指定了本地（devstack-controller）节点和远端（devstack-compute1）节点 VTEP 的 IP。

计算节点

执行 ovs-vsctl show：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218184523.jpg)

**br-int**

br-int 上 qvoab219616-01 将 cirros-vm2 虚拟网卡连接到 vxlan100_net。

**br-tun**

br-tun 上也创建了 port “vxlan-a642100b”，配置内容与控制节点相对，指定了本地（devstack-compute1）节点和远端（devstack-controller）节点 VTEP 的 IP。

当前网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218184557.jpg)

需要特别注意的是：无论存在多少个 VXLAN，devstack-controller 与 devstack-compute1 之间所有的数据都只通过 “vxlan-a642100b” 这对 port 上建立的隧道传输。

### OVS VxLAN Flow 分析

OVS 的数据流向都是由 Flow 规则控制的，先分析 VxLAN 的 Flow 规则。

下面分析控制节点上的 flow rule，计算节点类似。

**br-int 的 flow rule**

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218184801.jpg)

br-int 的 rule 看上去虽然多，其实逻辑很简单，br-int 被当作一个二层交换机，其重要的 rule 是下面这条：

``` shell
cookie=0xaaa0e760a7848ec3, duration=52798.625s, table=0, n_packets=143, n_bytes=14594, idle_age=9415, priority=0 actions=NORMAL
```

此规则的含义是：根据 vlan 和 mac 进行转发。

**br-tun 的 flow rule**

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218184811.jpg)

这些才是真正处理 VXLAN 数据包的 rule，流程如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218184820.jpg)

上图各方块中的数字对应 rule 中 table 的序号，比如编号为0的方块对应下面三条 rule。

**table 0**

``` shell
cookie=0xaaa0e760a7848ec3, duration=76707.867s, table=0, n_packets=70, n_bytes=6600, idle_age=33324, hard_age=65534, priority=1,in_port=1 actions=resubmit(,2)
cookie=0xaaa0e760a7848ec3, duration=76543.287s, table=0, n_packets=56, n_bytes=4948, idle_age=33324, hard_age=65534, priority=1,in_port=2 actions=resubmit(,4)
cookie=0xaaa0e760a7848ec3, duration=76707.867s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
```

结合如下 port 编号：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171218184832.jpg)

table 0 flow rule 的含义为：

1. 从 port 1（patch-int）进来的包，扔给 table 2 处理：actions=resubmit(,2)
2. 从 port 2（vxlan-a642100b）进来的包，扔给 table 4 处理：actions=resubmit(,4)

即第一条 rule 处理来自内部 br-int（这上面挂载着所有的网络服务，包括路由、DHCP 等）的数据；第二条 rule 处理来自外部 VXLAN 隧道的数据。

**table 4**

``` shell
cookie=0xaaa0e760a7848ec3, duration=76647.039s, table=4, n_packets=56, n_bytes=4948, idle_age=33324, hard_age=65534, priority=1,tun_id=0x64 actions=mod_vlan_vid:1,resubmit(,10)
```

table 4 flow rule 的含义为： 如果数据包的 VXLAN tunnel ID 为 100（tun_id=0x64），action 是添加内部 VLAN ID 1（tag=1），然后扔给 table 10 去学习。

**table 10**

``` shell
cookie=0xaaa0e760a7848ec3, duration=76707.865s, table=10, n_packets=56, n_bytes=4948, idle_age=33324, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0xaaa0e760a7848ec3,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
```

table 10 flow rule 的含义为： 学习外部（从 tunnel）进来的包，往 table 20 中添加对返程包的正常转发规则，然后从 port 1（patch-int）扔给 br-int。

rule 中下面的内容为学习规则，这里就不详细讨论了。

``` shell
NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]
```

**table 2**

``` shell
cookie=0xaaa0e760a7848ec3, duration=76707.866s, table=2, n_packets=28, n_bytes=3180, idle_age=33324, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
cookie=0xaaa0e760a7848ec3, duration=76707.866s, table=2, n_packets=42, n_bytes=3420, idle_age=33379, hard_age=65534, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
```

table 2 flow rule 的含义为：

1. br-int 发过来数据如果是单播包，扔给 table 20 处理：resubmit(,20)
2. br-int 发过来数据如果是多播或广播包，扔 table 22 处理：resubmit(,22)

**table 20**

``` shell
cookie=0xaaa0e760a7848ec3, duration=76543.287s, table=20, n_packets=28, n_bytes=3180, idle_age=33324, hard_age=65534, priority=2,dl_vlan=1,dl_dst=fa:16:3e:fd:8a:ed actions=strip_vlan,set_tunnel:0x64,output:2
cookie=0xaaa0e760a7848ec3, duration=76707.865s, table=20, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=resubmit(,22)
```

table 20 flow rule 的含义为：

1. 第一条规则就是 table 10 学习来的结果。内部 VLAN 号为 1（tag=1），目标 MAC 是 fa:16:3e:fd:8a:ed（virros-vm2）的数据包，即发送给 virros-vm2 的包，action 是去掉 VLAN 号，添加 VXLAN tunnel ID 100(十六进制 0x64)，并从 port 2 (tunnel 端口 vxlan-a642100b) 发出。
2. 对于没学习到规则的数据包，则扔给 table 22 处理。

**table 22**

``` shell
cookie=0xaaa0e760a7848ec3, duration=76543.282s, table=22, n_packets=2, n_bytes=84, idle_age=33379, hard_age=65534, dl_vlan=1 actions=strip_vlan,set_tunnel:0x64,output:2
cookie=0xaaa0e760a7848ec3, duration=76707.82s, table=22, n_packets=40, n_bytes=3336, idle_age=65534, hard_age=65534, priority=0 actions=drop
```

table 22 flow rule 的含义为： 如果数据包的内部 VLAN 号为 1（tag=1），action 是去掉 VLAN 号，添加 VXLAN tunnel ID 100(十六进制 0x64)，并从 port 2 (tunnel 端口 vxlan-a642100b) 发出。

### VXLAN 的路由和 floating IP 支持

对于多 VXLAN 之间的 routing 以及 floating IP，实现方式与 vlan 非常类似。

## 总结

本章重点讨论 Neutron 的架构，并通过分析 Linux Bridge 和 Open vSwitch 两个 mechnism driver 的技术细节，实践了 local，flat，vlan，vxlan 四种网络类型，同时也讨论了 routing 以及 floating IP 的实现细节。

Linux Bridge 和 Open vSwitch 都支持 Securet Group，Firewall as a Service ，Load Balancing as a Service 等高级功能，其实现方式也大致相同。





























































