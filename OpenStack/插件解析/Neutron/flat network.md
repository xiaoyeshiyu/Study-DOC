# flat network 原理与配置

flat network 是不带 tag 的网络，要求宿主机的物理网卡直接与 linux bridge 连接，这意味着：

**每个 flat network 都会独占一个物理网卡**。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201327.jpg)

上图中 eth1 桥接到 brqXXX，为 instance 提供 flat 网络。如果需要创建多个 flat network，就得准备多个物理网卡，如下图所示。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201349.jpg)

接下来讨论如何配置 flat 网络。

## 在 ML2 配置中 enable flat network

在 /etc/neutron/plugins/ml2/ml2_conf.ini 设置 flat network 相关参数。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201426.jpg)

``` shel
tenant_network_types = flat
```

指定普通用户创建的网络类型为 flat。

需要注意的是：因为 flat 网络与物理网卡一一对应，一般情况下租户网络**不会**采用 flat，这里只是示例。

接着需要指明 flat 网络与物理网卡的对应关系。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201503.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201526.png)

如上所示：

1. 在 [ml2_type_flat] 中通过 flat_networks 定义了一个 flat 网络，label 为 “default”。
2. 在 [linux_bridge] 中通过 physical_interface_mappings 指明 default 对应的物理网卡为 eth1。

### 理解 label 与 ethX 的关系

label 是 flat 网络的标识，在创建 flat 时需要指定 label（后面演示）。label 的名字可以是任意字符串，只要确保各个节点 ml2_conf.ini 中的 label 命名一致就可以了。

各个节点中 label 与物理网卡的对应关系可能不一样。这是因为每个节点可以使用不同的物理网卡将 instance 连接到 flat network。

例如对于 label 为 “default” 的 flat network，节点 A 可能使用 eth1，配置为：

``` xml
physical_interface_mappings = default:eth1
```

而节点 B 则可能使用 eth2，配置为：

``` xml
physical_interface_mappings = default:eth2s
```

### 支持多个 flat

如果要创建多个 flat 网络，需要定义多个 label，用逗号隔开，当然也需要用到多个物理网卡，如下所示：

```xml
[ml2_type_flat]
flat_networks = flat1,flat2

[linux_bridge]
physical_interface_mappings = flat1:eth1,flat2:eth2
```

# 创建 flat 网络

打开菜单 Admin -> Networks，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201726.jpg)

显示创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201732.jpg)

Provider Network Type 选择 “Flat”。Physical Network 填写 “default”，与 ml2_conf.ini 中 flat_networks 参数保持一致。

点击 “Create Network”，flat_net 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201746.jpg)

点击 flat_net 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201825.jpg)

设置 IP 地址为 “172.16.1.0/24”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201829.jpg)

点击 “Next”，设置 IP 地址范围为 172.16.1.101-172.16.1.200。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201900.jpg)

点击 “Create”，subnet 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201922.jpg)

## 底层网络发生了什么变化

执行 brctl show，查看控制节点当前的网络结构。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207201956.jpg)

Neutron 自动新建了 flat_net 对应的网桥 brqf153b42f-c3，以及 dhcp 的 tap 设备 tap19a0ed3d-fe。另外，tap19a0ed3d-fe 和物理网卡 eth1 都已经连接到 bridge。

此时 flat_net 结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202009.jpg)

# 部署 instance 到此 flat 网络

launch 新的 instance “cirros-vm1”，选择网络 falt_net。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202050.png)

cirros-vm1 分配到的 IP 为 172.16.1.103。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202110.jpg)

cirros-vm1 被 schedule 到控制节点，对应的 tap 设备为 tapc1875c7f-cb，并且已连接到 bridge。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202122.jpg)

当前 flat_net 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202143.jpg)

继续用同样的方式 launch instance cirros-vm2，分配到的 IP 为 172.16.1.104。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202206.jpg)

cirros-vm2 被 schedule 到计算节点，对应的 tap 设备为 tapfb3fb197-24，并且连接到 bridge。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202232.jpg)

这里有两点需要提醒：

1. 因为计算节点上没有 dhcp 服务，所以 brctl show 中没有 dhcp 对应的 tap 设备。
2. 计算节点上 bridge 的名称与控制节点上一致，都是 brqf153b42f-c3，表明是同一个 network。

当前 flat_net 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202244.jpg)

cirros-vm1（172.16.1.103） 与 cirros-vm2（172.16.1.104） 位于不同节点，通过 flat_net 相连，下面执行 PING 验证连通性。在 cirros-vm1 控制台中执行 ping 172.16.1.104：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202256.jpg)

如我们预料，ping 成功。





