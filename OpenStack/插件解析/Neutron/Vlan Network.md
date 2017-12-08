# Neutron Vlan Network 原理

vlan network 是带 tag 的网络，是实际应用最广泛的网络类型。

下图是 vlan100 网络的示例。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208102638.jpg)

1. 三个 instance 通过 TAP 设备连接到名为 “brqXXXX” linux bridge。
2. 在物理网卡 eth1 上创建了 eth1.100 的 vlan interface，eth1.100 连接到 brqXXXX。
3. instance 通过 eth1.100 发送到 eth1 的数据包就会打上 vlan100 的 tag。

如果再创建一个 network vlan101，eth1 上会相应的创建 vlan interface eth1.101，并且连接的新的 lingux bridge “brqYYYY”。每个 vlan network 有自己的 bridge，从而也就实现了基于 vlan 的隔离。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208102737.jpg)

这里有一点要注意：

因为物理网卡 eth1 上面可以走多个 vlan 的数据，那么物理交换机上与 eth1 相连的的 port 要设置成 trunk 模式，而不是 access 模式。

# 在 ML2 中配置 Vlan Network

首先在 /etc/neutron/plugins/ml2/ml2_conf.ini 中设置 vlan network 相关参数。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208102934.jpg)

``` shell
tenant_network_types = vlan
```

指定普通用户创建的网络类型为 vlan。然后指定 vlan 的范围：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208102947.jpg)

上面配置定义了 label 为 “default” 的 vlan network，vlan id 的范围是 3001 - 4000。这个范围是针对普通用户在自己的租户里创建 network 的范围。因为普通用户创建 network 时并不能指定 vlan id，Neutron 会按顺序自动从这个范围中取值。

对于 admin 则没有 vlan id 的限制，admin 可以创建 id 范围为 1-4094 的 vlan network。

接着需要指明 vlan network 与物理网卡的对应关系：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103024.jpg)

如上所示：

在` [ml2_type_vlan] `中定义了 lable “default”，`[linux_bridge] `中则指明 default 对应的物理网卡为 eth1。

这里 label 的作用与前面 flat network 中的 label 一样，只是一个标识，可以是任何字符串。

配置完成，重启 Neutron 服务后生效。

# 创建第一个 vlan network "vlan100"

打开菜单 Admin -> Networks，点击 “Create Network” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103206.jpg)

Provider Network Type 选择 “VLAN”。

Physical Network 填写 “default”，必须与 ml2_conf.ini network_vlan_ranges 保持一致。

Segmentation ID 即 VLAN ID，设置为 100。

点击 “Create Network”，vlan100 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103222.jpg)

点击 vlan100 链接，进入 network 配置页面，目前还没有 subnet，点击 “Create Subnet” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103311.jpg)

创建 subnet\_172\_16\_100\_0，IP 地址为 172.16.100.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103453.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103338.jpg)

## 底层网络发生了什么变化

控制节点上执行 brctl show，查看当前网络结构。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103517.jpg)

Neutron 自动新建了三个设备：

1. vlan100 对应的网桥 brq3fcfdb98-9d。
2. vlan interface eth1.100。
3. dhcp 的 tap 设备 tap1180bbe8-06。

eth1.100 和 tap19a0ed3d-fe 已经连接到了 brq3fcfdb98-9d，VLAN 100 的二层网络就绪，此时 vlan100 结构如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103532.jpg)

# 将 instance 连接到 vlan100

launch 新的 instance “cirros-vm1”，网络选择 vlan100。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103827.jpg)

cirros-vm1 分配到的 IP 为 172.16.100.3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103841.jpg)

cirros-vm1 被 schedule 到控制节点，对应的 tap 设备为 tapc1875c7f-cb，并且连接到 bridge。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103906.jpg)

当前 vlan100 的结构如下。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103915.jpg)

继续用同样的方式 launch instance cirros-vm2，分配到的 IP 为 172.16.100.104。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103929.jpg)

cirros-vm2 被 schedule 到计算节点，对应的 tap 设备为 tapac94e0e8-2b，并且连接到 bridge。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208103948.jpg)

因为计算节点上没有 hdcp 服务，所以没有相应的 tap 设备。另外，bridge 的名称与控制节点上一致，都是 brq3fcfdb98-9d，表明是同一个 network。

当前 vlan100 的结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104017.jpg)

cirros-vm1（172.16.100.3） 与 cirros-vm2（172.16.100.4） 位于不同节点，通过 vlan100 相连，下面执行 PING 验证连通性。

在 cirros-vm1 控制台中执行 ping 172.16.100.4。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104028.jpg)

如我们预料，ping 成功。

# 创建第二个 vlan network "vlan101"

创建 vlan network“vlan101”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104430.jpg)

subnet IP 地址为 172.16.101.0/24。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104505.jpg)

## 底层网络发生了什么变化

Neutron 自动创建了 vlan101 对应的网桥 brq1d7040b8-01，vlan interface eth1.101，以及 dhcp 的 tap 设备 tap5b1a2247-32。eth1.101 和 tap5b1a2247-32 已经连接到 brq1d7040b8-01，VLAN 101 的二层网络就绪。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104525.jpg)

现在，网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104550.jpg)

# 将 instance 连接到 vlan101

部署 instance 到该 vlan network，验证 instance 之间的连通性。

launch 新的 instance “cirros-vm3”，网络选择 vlan101。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104703.jpg)

cirros-vm3 分配到的 IP 为 172.16.101.103。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104650.jpg)

cirros-vm3 被 schedule 到计算节点，对应的 tap 设备为 tapadb5cc6a-7a，并且连接到 bridge。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104812.jpg)

当前网络结构如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171208104743.jpg)

cirros-vm1 位于控制节点，属于 vlan100。

cirros-vm2 位于计算节点，属于 vlan100。

cirros-vm3 位于计算节点，属于 vlan101。

cirros-vm1 与 cirros-vm2 都在 vlan100，它们之间能通信。

cirros-vm3 在 vlan101，不能与 cirros-vm1 和 cirros-vm2 通信。

如果 vlan100 与 vlan101 中的 instance 需要通信，单靠二层 vlan 是不行的，需要在三层通过路由器转发。







































