# Neutron 概述

传统的网络管理方式很大程度上依赖于管理员手工配置和维护各种网络硬件设备；而云环境下的网络已经变得非常复杂，特别是在多租户场景里，用户随时都可能需要创建、修改和删除网络，网络的连通性和隔离不已经太可能通过手工配置来保证了。

如何快速响应业务的需求对网络管理提出了更高的要求。传统的网络管理方式已经很难胜任这项工作，而“软件定义网络（software-defined networking, SDN）”所具有的灵活性和自动化优势使其成为云时代网络管理的主流。

Neutron 的设计目标是实现“网络即服务（Networking as a Service）”。为了达到这一目标，在设计上遵循了基于 SDN 实现网络虚拟化的原则，在实现上充分利用了 Linux 系统上的各种网络相关的技术。

# Neutron 功能

Neutron 为整个 OpenStack 环境提供网络支持，包括二层交换，三层路由，负载均衡，防火墙和 VPN 等。Neutron 提供了一个灵活的框架，通过配置，无论是开源还是商业软件都可以被用来实现这些功能。

## 二层交换 Switching

Nova 的 Instance 是通过虚拟交换机连接到虚拟二层网络的。Neutron 支持多种虚拟交换机，包括 Linux 原生的 Linux Bridge 和 Open vSwitch。 Open vSwitch（OVS）是一个开源的虚拟交换机，它支持标准的管理接口和协议。

利用 Linux Bridge 和 OVS，Neutron 除了可以创建传统的 VLAN 网络，还可以创建基于隧道技术的 Overlay 网络，比如 VxLAN 和 GRE（Linux Bridge 目前只支持 VxLAN）。

## 三层路由 Routing

Instance 可以配置不同网段的 IP，Neutron 的 router（虚拟路由器）实现 instance 跨网段通信。router 通过 IP forwarding，iptables 等技术来实现路由和 NAT。

## 负载均衡 Load Balancing

Openstack 在 Grizzly 版本第一次引入了 Load-Balancing-as-a-Service（LBaaS），提供了将负载分发到多个 instance 的能力。LBaaS 支持多种负载均衡产品和方案，不同的实现以 Plugin 的形式集成到 Neutron，目前默认的 Plugin 是 HAProxy。

## 防火墙 Firewalling

Neutron 通过下面两种方式来保障 instance 和网络的安全性。

**Security Group**

通过 iptables 限制进出 instance 的网络包。

**Firewall-as-a-Service**

FWaaS，限制进出虚拟路由器的网络包，也是通过 iptables 实现。

# Neutron 网络基本概念

Neutron 管理的网络资源包括 Network，subnet 和 port。

## network

network 是一个隔离的二层广播域。Neutron 支持多种类型的 network，包括 local, flat, VLAN, VxLAN 和 GRE。

**local**
local 网络与其他网络和节点隔离。local 网络中的 instance 只能与位于同一节点上同一网络的 instance 通信，local 网络主要用于单机测试。

**flat**
flat 网络是无 vlan tagging 的网络。flat 网络中的 instance 能与位于同一网络的 instance 通信，并且可以跨多个节点。

**vlan**
vlan 网络是具有 802.1q tagging 的网络。vlan 是一个二层的广播域，同一 vlan 中的 instance 可以通信，不同 vlan 只能通过 router 通信。vlan 网络可跨节点，是应用最广泛的网络类型。

**vxlan**
vxlan 是基于隧道技术的 overlay 网络。vxlan 网络通过唯一的 segmentation ID（也叫 VNI）与其他 vxlan 网络区分。vxlan 中数据包会通过 VNI 封装成 UDP 包进行传输。因为二层的包通过封装在三层传输，能够克服 vlan 和物理网络基础设施的限制。

**gre**
gre 是与 vxlan 类似的一种 overlay 网络。主要区别在于使用 IP 包而非 UDP 进行封装。

不同 network 之间在二层上是隔离的。

以 vlan 网络为例，network A 和 network B 会分配不同的 VLAN ID，这样就保证了 network A 中的广播包不会跑到 network B 中。当然，这里的隔离是指二层上的隔离，借助路由器不同 network 是可能在三层上通信的。

network 必须属于某个 Project（ Tenant 租户），Project 中可以创建多个 network。 network 与 Project 之间是 1对多 关系。

## subnet

subnet 是一个 IPv4 或者 IPv6 地址段。instance 的 IP 从 subnet 中分配。每个 subnet 需要定义 IP 地址的范围和掩码。

network 与 subnet 是 1对多 关系。一个 subnet 只能属于某个 network；一个 network 可以有多个 subnet，这些 subnet 可以是不同的 IP 段，但不能重叠。下面的配置是有效的：

``` json
network A   subnet A-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}
            subnet A-b: 10.10.2.0/24  {"start": "10.10.2.1", "end": "10.10.2.50"}
```

但下面的配置则无效，因为 subnet 有重叠

``` json
networkA    subnet A-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}
            subnet A-b: 10.10.1.0/24  {"start": "10.10.1.51", "end": "10.10.1.100"}
```

这里不是判断 IP 是否有重叠，而是 subnet 的 CIDR 重叠（都是 10.10.1.0/24）。但是，如果 subnet 在不同的 network 中，CIDR 和 IP 都是可以重叠的，比如

``` json
network A   subnet A-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}
networkB    subnet B-a: 10.10.1.0/24  {"start": "10.10.1.1", "end": "10.10.1.50"}
```

上面的IP地址是可以重叠的，那么就可能存在具有相同 IP 的两个 instance，但是这样并不冲突。

具体原因： 因为 Neutron 的 router 是通过 Linux network namespace 实现的。network namespace 是一种网络的隔离机制。通过它，每个 router 有自己独立的路由表。上面的配置有两种结果：

1. 如果两个 subnet 是通过同一个 router 路由，根据 router 的配置，只有指定的一个 subnet 可被路由。
2. 如果上面的两个 subnet 是通过不同 router 路由，因为 router 的路由表是独立的，所以两个 subnet 都可以被路由。

## port

port 可以看做虚拟交换机上的一个端口。port 上定义了 MAC 地址和 IP 地址，当 instance 的虚拟网卡 VIF（Virtual Interface） 绑定到 port 时，port 会将 MAC 和 IP 分配给 VIF。

subnet 与 port 是 1对多 关系。一个 port 必须属于某个 subnet；一个 subnet 可以有多个 port。

## 小结

Project 1 : m Network 1 : m Subnet 1 : m Port 1 : 1 VIF m : 1Instance

# Neutron 架构

与 OpenStack 的其他服务的设计思路一样，Neutron 也是采用分布式架构，由多个组件（子服务）共同对外提供网络服务。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207104950.jpg)

Neutron 由如下组件构成：

**Neutron Server**

对外提供 OpenStack 网络 API，接收请求，并调用 Plugin 处理请求。

**Plugin**

处理 Neutron Server 发来的请求，维护 OpenStack 逻辑网络状态， 并调用 Agent 处理请求。

**Agent**

处理 Plugin 的请求，负责在 network provider 上真正实现各种网络功能。

**network provider**

提供网络服务的虚拟或物理网络设备，例如 Linux Bridge，Open vSwitch 或者其他支持 Neutron 的物理交换机。

**Queue**

Neutron Server，Plugin 和 Agent 之间通过 Messaging Queue 通信和调用。

**Database**

存放 OpenStack 的网络状态信息，包括 Network, Subnet, Port, Router 等。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207105015.jpg)

Neutron 架构非常灵活，层次较多，目的是：

1. 为了支持各种现有或者将来会出现的优秀网络技术。
2. 支持分布式部署，获得足够的扩展性。

通常鱼和熊掌不能兼得，虽然获得了这些优势，但这样使得 Neutron 更加复杂，更不容易理解。 先通过一个例子了解这些组件各自的职责以及是如何协同工作。

以创建一个 VLAN100 的 network 为例，假设 network provider 是 linux bridge， 流程如下：

> 1. Neutron Server 接收到创建 network 的请求，通过 Message Queue（RabbitMQ）通知已注册的 Linux Bridge Plugin。
> 2. Plugin 将要创建的 network 的信息（例如名称、VLAN ID等）保存到数据库中，并通过 Message Queue 通知运行在各节点上的 Agent。
> 3. Agent 收到消息后会在节点上的物理网卡（比如 eth2）上创建 VLAN 设备（比如 eth2.100），并创建 bridge （比如 brqXXX） 桥接 VLAN 设备。

1. plugin 解决的是 What 的问题，即网络要配置成什么样子？而至于如何配置 How 的工作则交由 agent 完成。
2. plugin，agent 和 network provider 是配套使用的，比如上例中 network provider 是 linux bridge，那么就得使用 linux bridge 的 plungin 和 agent；如果 network provider 换成了 OVS 或者物理交换机，plugin 和 agent 也得替换。
3. plugin 的一个主要的职责是在数据库中维护 Neutron 网络的状态信息，这就造成一个问题：所有 network provider 的 plugin 都要编写一套非常类似的数据库访问代码。为了解决这个问题，Neutron 在 Havana 版本实现了一个 ML2（Modular Layer 2）plugin，对 plgin 的功能进行抽象和封装。有了 ML2 plugin，各种 network provider 无需开发自己的 plugin，只需要针对 ML2 开发相应的 driver 就可以了，工作量和难度都大大减少。
4. plugin 按照功能分为两类： core plugin 和 service plugin。core plugin 维护 Neutron 的 netowrk, subnet 和 port 相关资源的信息，与 core plugin 对应的 agent 包括 linux bridge, OVS 等； service plugin 提供 routing, firewall, load balance 等服务，也有相应的 agent。

# Neutron 物理部署方案

不同节点部署不同的 Neutron 服务组件。

## 方案1：控制节点 + 计算节点

在这个部署方案中，OpenStack 由控制节点和计算节点组成。

**控制节点**
部署的服务包括：neutron server, core plugin 的 agent 和 service plugin 的 agent。

**计算节点**
部署 core plugin 的agent，负责提供二层网络功能。

这里有几点需要说明： 

1. core plugin 和 service plugin 已经集成到 neutron server，不需要运行独立的 plugin 服务。
2. 控制节点和计算节点都需要部署 core plugin 的 agent，因为通过该 agent 控制节点与计算节点才能建立二层连接。
3. 可以部署多个控制节点和计算节点。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207105830.jpg)

## 方案2：控制节点 + 网络节点 + 计算节点

在这个部署方案中，OpenStack 由控制节点，网络节点和计算节点组成。

**控制节点**

部署 neutron server 服务。

**网络节点**

部署的服务包括：core plugin 的 agent 和 service plugin 的 agent。

**计算节点**

部署 core plugin 的agent，负责提供二层网络功能。

这个方案的要点是将所有的 agent 从控制节点分离出来，部署到独立的网络节点上。

1. 控制节点只负责通过 neutron server 响应 API 请求。
2. 由独立的网络节点实现数据的交换，路由以及 load balance等高级网络服务。
3. 可以通过增加网络节点承担更大的负载。
4. 可以部署多个控制节点、网络节点和计算节点。

该方案特别适合规模较大的 OpenStack 环境。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207110459.jpg)

以上就是 Neutron 两种典型的部署方案.

# Neutron 服务组件

## Neutron Server 分层模型

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207110945.jpg)

上图是 Neutron Server 的分层结构，至上而下依次为：

**Core API**

对外提供管理 network, subnet 和 port 的 RESTful API。

**Extension API**

对外提供管理 router, load balance, firewall 等资源 的 RESTful API。

**Commnon Service**

认证和校验 API 请求。

**Neutron Core**

Neutron server 的核心处理程序，通过调用相应的 Plugin 处理请求。

**Core Plugin API**

定义了 Core Plgin 的抽象功能集合，Neutron Core 通过该 API 调用相应的 Core Plgin。

**Extension Plugin API**

定义了 Service Plgin 的抽象功能集合，Neutron Core 通过该 API 调用相应的 Service Plgin。

**Core Plugin**

实现了 Core Plugin API，在数据库中维护 network, subnet 和 port 的状态，并负责调用相应的 agent 在 network provider 上执行相关操作，比如创建 network。

**Service Plugin**

实现了 Extension Plugin API，在数据库中维护 router, load balance, security group 等资源的状态，并负责调用相应的 agent 在 network provider 上执行相关操作，比如创建 router。

归纳起来，Neutron Server 包括两部分：

1. 提供 API 服务。
2. 运行 Plugin。

即 **Neutron Server = API + Plugins**

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207111021.jpg)

## 支持多种 network provider

Neutron 的架构是非常开放的，可以支持多种 network provider，只要遵循一定的设计原则和规范

先讨论一个简单的场景：在 Neutorn 中使用 linux bridge 这一种 network provider。

**linux bridge core plugin**

1. 与 neutron server 一起运行。
2. 实现了 core plugin API。
3. 负责维护数据库信息。
4. 通知 linux bridge agent 实现具体的网络功能。

**linux bridge agent**

1. 在计算节点和网络节点（或控制节点）上运行。
2. 接收来自 plugin 的请求。
3. 通过配置本节点上的 linux bridge 实现 neutron 网络功能。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207111306.jpg)

同样的道理，如果要支持 open vswitch，只需要实现 open vswitch plugin 和 open vswitch agent。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207111406.jpg)

Neutron 可以通过开发不同的 plugin 和 agent 支持不同的网络技术。这是一种相当开放的架构。不过随着支持的 network provider 数量的增加，开发人员发现了两个突出的问题：

1. 只能在 OpenStack 中使用一种 core plugin，多种 network provider 无法共存。
2. 不同 plugin 之间存在大量重复代码，开发新的 plugin 工作量大。

## ML2 Core Plugin

Core Plugin，其功能是维护数据库中 network, subnet 和 port 的状态，并负责调用相应的 agent 在 network provider 上执行相关操作，比如创建 network。

Moduler Layer 2（ML2）是 Neutron 在 Havana 版本实现的一个新的 core plugin，用于替代原有的 linux bridge plugin 和 open vswitch plugin。

**传统 core plugin 的问题**

之所以要开发 ML2，主要是因为传统 core plugin 存在两个突出的问题。

### 问题1：无法同时使用多种 network provider

Core plugin 负责管理和维护 Neutron 的 network, subnet 和 port 的状态信息，这些信息是全局的，只需要也只能由一个 core plugin 管理。

只使用一个 core plugin 本身没有问题。但问题在于传统的 core plugin 与 core plugin agent 是一一对应的。也就是说，如果选择了 linux bridge plugin，那么 linux bridge agent 将是唯一选择，就必须在 OpenStack 的所有节点上使用 linux bridge 作为虚拟交换机（即 network provider）。

同样的，如果选择 open vswitch plugin， 所有节点上只能使用 open vswitch，而不能使用其他的 network provider。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207111728.jpg)

### 问题2：开发新的 core plugin 工作量大

所有传统的 core plugin 都需要编写大量重复和类似的数据库访问的代码，大大增加了 plugin 开发和维护的工作量。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207112418.jpg)

## ML2 能解决传统 core plugin 的问题

ML2 作为新一代的 core plugin，提供了一个框架，允许在 OpenStack 网络中同时使用多种 Layer 2 网络技术，不同的节点可以使用不同的网络实现机制。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207113411.jpg)

如上图所示，采用 ML2 plugin 后，可以在不同节点上分别部署 linux bridge agent, open vswitch agent, hyper-v agent 或其他第三方 agent。

ML2 不但支持异构部署方案，同时能够与现有的 agent 无缝集成：以前用的 agent 不需要变，只需要将 Neutron server 上的传统 core plugin 替换为 ML2。

有了 ML2，要支持新的 network provider 就变得简单多了：无需从头开发 core plugin，只需要开发相应的 mechanism driver，大大减少了要编写和维护的代码。

## ML2 架构

ML2 对二层网络进行抽象和建模，引入了 type driver 和 mechansim driver。这两类 driver 解耦了 Neutron 所支持的网络类型（type）与访问这些网络类型的机制（mechanism），其结果就是使得 ML2 具有非常好的弹性，易于扩展，能够灵活支持多种 type 和 mechanism。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207113756.jpg)

### Type Driver

Neutron 支持的每一种网络类型都有一个对应的 ML2 type driver。

type driver 负责维护网络类型的状态，执行验证，创建网络等。 ML2 支持的网络类型包括 local, flat, vlan, vxlan 和 gre。 

### Mechansim Driver

Neutron 支持的每一种网络机制都有一个对应的 ML2 mechansim driver。 

mechanism driver 负责获取由 type driver 维护的网络状态，并确保在相应的网络设备（物理或虚拟）上正确实现这些状态。

type 和 mechanisim 都太抽象，现在我们举一个具体的例子： type driver 为 vlan，mechansim driver 为 linux bridge，我们要完成的操作是创建 network vlan100，那么：

1. vlan type driver 会确保将 vlan100 的信息保存到 Neutron 数据库中，包括 network 的名称，vlan ID 等。
2. linux bridge mechanism driver 会确保各节点上的 linux brige agent 在物理网卡上创建 ID 为 100 的 vlan 设备 和 brige 设备，并将两者进行桥接。

mechanism driver 有三种类型：

**Agent-based**

包括 linux bridge, open vswitch 等。

**Controller-based**

包括 OpenDaylight, VMWare NSX 等。

**基于物理交换机**

包括 Cisco Nexus, Arista, Mellanox 等。 比如前面那个例子如果换成 Cisco 的 mechanism driver，则会在 Cisco 物理交换机的指定 trunk 端口上添加 vlan100。

linux bridge 和 open vswitch 的 ML2 mechanism driver 作用是配置各节点上的虚拟交换机。linux bridge driver 支持的 type 包括 local, flat, vlan, and vxlan。open vswitch driver 除这 4 种 type 还支持 gre。

L2 population driver 作用是优化和限制 overlay 网络中的广播流量。 vxlan 和 gre 都属于 overlay 网络。

ML2 core plugin 已经成为 OpenStack Neutron 的首选 plugin。

## Service Plugin / Agent

Core Plugin/Agent 负责管理核心实体：net, subnet 和 port。而对于更高级的网络服务，则由 Service Plugin/Agent 管理。

Service Plugin 及其 Agent 提供更丰富的扩展功能，包括路由，load balance，firewall等，如图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207114335.jpg)

**DHCP**

dhcp agent 通过 dnsmasq 为 instance 提供 dhcp 服务。

**Routing**

l3 agent 可以为 project（租户）创建 router，提供 Neutron subnet 之间的路由服务。路由功能默认通过 IPtables 实现。

**Firewall**

l3 agent 可以在 router 上配置防火墙策略，提供网络安全防护。另一个与安全相关的功能是 Security Group，也是通过 IPtables 实现。 Firewall 与 Security Group 的区别在于：

1. Firewall 安全策略位于 router，保护的是某个 project 的所有 network。
2. Security Group 安全策略位于 instance，保护的是单个 instance。

Firewall 与 Security Group 后面会详细分析。

**Load Balance**

Neutron 默认通过 HAProxy 为 project 中的多个 instance 提供 load balance 服务。

# 总结

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207140408.jpg)

与 OpenStack 其他服务一样，Neutron 采用的是分布式架构，包括 Neutorn Server、各种 plugin/agent、database 和 message queue。

1. Neutron server 接收 api 请求。
2. plugin/agent 实现请求。
3. database 保存 neutron 网络状态。
4. message queue 实现组件之间通信。

metadata-agent 补充：

> instance 在启动时需要访问 nova-metadata-api 服务获取 metadata 和 userdata，这些 data 是该 instance 的定制化信息，比如 hostname, ip， public key 等。
>
> 但 instance 启动时并没有 ip，那如何通过网络访问到 nova-metadata-api 服务呢？
>
> 答案就是 neutron-metadata-agent 该 agent 让 instance 能够通过 dhcp-agent 或者 l3-agent 与 nova-metadata-api 通信

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207140442.jpg)

1. Neutron 通过 plugin 和 agent 提供的网络服务。
2. plugin 位于 Neutron server，包括 core plugin 和 service plugin。
3. agent 位于各个节点，负责实现网络服务。
4. core plugin 提供 L2 功能，ML2 是推荐的 plugin。
5. 使用最广泛的 L2 agent 是 linux bridage 和 open vswitch。
6. service plugin 和 agent 提供扩展功能，包括 dhcp, routing, load balance, firewall, vpn 等。



















