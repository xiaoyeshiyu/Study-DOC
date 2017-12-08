# 配置 DHCP 服务

前面章节我们看到 instance 在启动过程中能够从 Neutron 的 DHCP 服务获得 IP，本节将详细讨论其内部实现机制。

Neutron 提供 DHCP 服务的组件是 DHCP agent。DHCP agent 在网络节点运行上，默认通过 dnsmasq 实现 DHCP 功能。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202412.jpg)

## 配置 DHCP agent

DHCP agent 的配置文件位于 /etc/neutron/dhcp_agent.ini。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202431.jpg)

**dhcp_driver**

使用 dnsmasq 实现 DHCP。

**interface_driver**

使用 linux bridge 连接 DHCP namespace interface。

当创建 network 并在 subnet 上 enable DHCP 时，网络节点上的 DHCP agent 会启动一个 dnsmasq 进程为该 network 提供 DHCP 服务。

dnsmasq 是一个提供 DHCP 和 DNS 服务的开源软件。dnsmasq 与 network 是一对一关系，一个 dnsmasq 进程可以为同一 netowrk 中所有 enable 了 DHCP 的 subnet 提供服务。

回到我们的实验环境，之前创建了 flat_net，并且在 subnet 上启用了 DHCP，执行 ps 查看 dnsmasq 进程，如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202511.jpg)

DHCP agent 会为每个 network 创建一个目录 /opt/stack/data/neutron/dhcp/，用于存放该 network 的 dnsmasq 配置文件。

下面讨论 dnsmasq 重要的启动参数：

**--dhcp-hostsfile**

存放 DHCP host 信息的文件，这里的 host 在我们这里实际上就是 instance。dnsmasq 从该文件获取 host 的 IP 与 MAC 的对应关系。每个 host 对应一个条目，信息来源于 Neutron 数据库。

对于 flat_net，hostsfile 是 /opt/stack/data/neutron/dhcp/f153b42f-c3a1-4b6c-8865-c09b5b2aa274/host，记录了 DHCP，cirros-vm1 和 cirros-vm2 的 interface 信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202522.jpg)

**--interface**

指定提供 DHCP 服务的 interface。dnsmasq 会在该 interface 上监听 instance 的 DHCP 请求。

对于 flat_net，interface 是 ns-19a0ed3d-fe。或许大家还记得，之前我们看到的 DHCP interface 叫 tap19a0ed3d-fe（如下图所示），并非 ns-19a0ed3d-fe。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202538.jpg)

从名称上看，ns-19a0ed3d-fe 和 tap19a0ed3d-fe 应该存在某种联系，但那是什么呢？

要回答这个问题，需要先搞懂一个概念：Linux Network Namespace

# 用 namspace 隔离 DHCP 服务

Neutron 通过 dnsmasq 提供 DHCP 服务，而 dnsmasq 如何独立的为每个 network 服务呢？

答案是通过 Linux Network Namespace 隔离，本节将详细讨论。

在二层网络上，VLAN 可以将一个物理交换机分割成几个独立的虚拟交换机。类似地，在三层网络上，Linux network namespace 可以将一个物理三层网络分割成几个独立的虚拟三层网络。

每个 namespace 都有自己独立的网络栈，包括 route table，firewall rule，network interface device 等。

Neutron 通过 namespace 为每个 network 提供独立的 DHCP 和路由服务，从而允许租户创建重叠的网络。如果没有 namespace，网络就不能重叠，这样就失去了很多灵活性。

每个 dnsmasq 进程都位于独立的 namespace, 命名为 qdhcp-< network id >，例如 flat_net：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202639.jpg)

ip netns list 命令列出所有的 namespace。qdhcp-f153b42f-c3a1-4b6c-8865-c09b5b2aa274 就是 flat_net 的 namespace。

其实，宿主机本身也有一个 namespace，叫 root namespace，拥有所有物理和虚拟 interface device。物理 interface 只能位于 root namespace。

新创建的 namespace 默认只有一个 loopback device。管理员可以将虚拟 interface，例如 bridge，tap 等设备添加到某个 namespace。

对于 flat_net 的 DHCP 设备 tap19a0ed3d-fe，需要将其放到 namespace qdhcp-f153b42f-c3a1-4b6c-8865-c09b5b2aa274 中，但这样会带来一个问题：tap19a0ed3d-fe 将无法直接与 root namespace 中的 bridge 设备 brqf153b42f-c3 连接。

**Neutron 使用 veth pair 解决了这个问题。**

veth pair 是一种成对出现的特殊网络设备，它们象一根虚拟的网线，可用于连接两个 namespace。向 veth pair 一端输入数据，在另一端就能读到此数据。

tap19a0ed3d-fe 与 ns-19a0ed3d-fe 就是一对 veth pair，它们将 qdhcp-f153b42f-c3a1-4b6c-8865-c09b5b2aa274 连接到 brqf153b42f-c3。

如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202652.jpg)

可以通过 ip netns exec < network namespace name> < command>管理 namespace。例如查看 ns-19a0ed3d-fe 的配置：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202729.jpg)

# 获取 dhcp IP 过程分析

在创建 instance 时，Neutron 会为其分配一个 port，里面包含了 MAC 和 IP 地址信息。这些信息会同步更新到 dnsmasq 的 host 文件。如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202826.jpg)

同时 nova-compute 会设置 cirros-vm1 VIF 的 MAC 地址。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207202838.png)

一切准备就绪，instance 获取 IP 的过程如下：

1. cirros-vm1 开机启动，发出 DHCPDISCOVER 广播，该广播消息在整个 flat_net 中都可以被收到。
2. 广播到达 veth tap19a0ed3d-fe，然后传送给 veth pair 的另一端 ns-19a0ed3d-fe。dnsmasq 在它上面监听，dnsmasq 检查其 host 文件，发现有对应项，于是dnsmasq 以  DHCPOFFER 消息将 IP（172.16.1.103）、子网掩码（255.255.255.0）、地址租用期限等信息发送给 cirros-vm1。
3. cirros-vm1 发送 DHCPREQUEST 消息确认接受此 DHCPOFFER。
4. dnsmasq 发送确认消息 DHCPACK，整个过程结束。

这个过程我们可以在 dnsmasq 日志中查看。

dnsmasq 默认将日志记录到 /var/log/syslog。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207203921.jpg)

