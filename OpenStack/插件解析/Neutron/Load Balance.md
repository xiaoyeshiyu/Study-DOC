# 理解 Neutorn LBaaS

Load Balance as a Service（LBaaS）是 Neutron 提供的一项高级网络服务。LBaaS 允许租户在自己的网络中创建和管理 load balancer。

load balancer 可以说是分布式系统中比较基础的组件。它接收前端发来的请求，然后将请求按照某种均衡策略转发给后端资源池中的某个处理单元，以完成处理。load balancer 可以实现系统高可用和横向扩展。

LBaaS 有三个主要的概念：Pool Member，Pool 和 Virtual IP

**Pool Member**

Pool Member 是 layer 4 的实体，拥有 IP 地址并通过监听端口对外提供服务。
例如 Pool Member 可以是一个 web server，IP 为 172.16.100.9 并通过 80 端口提供 HTTP 服务。

**Pool**

Pool 由一组 Pool Member 组成。
这些 Pool Member 通常提供同一类服务。
例如一个 web server pool，包含：
web1：172.16.100.9：80
web2：172.16.100.10：80

**Virtual IP**

Virtual IP 也称作 VIP，是定义在 load balancer 上的 IP 地址。
每个 pool member 都有自己的 IP，但对外服务则是通过 VIP。

**load balancer 负责监听外部的连接，并将连接分发到 pool member。外部 client 只知道 VIP，不知道也不需要关心是否有 pool 或者有多少个 pool member。**

OpenStack Neutron 目前默认通过 HAProxy 软件来实现 LBaaS。HAProxy 是一个流行的开源 load balancer。Neutron 也支持其他一些第三方 load balancer。

下图展示了 HAProxy 实现 load balancer 的方式。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211170350.jpg)

左图是 client 发送请求到 web server 的数据流：

1. Client 10.10.10.4 通过浏览器访问服务器的外网 IP 10.10.10.7。

2. 请求首先到达路由器，将目的地址设置为服务器的内网 VIP 172.16.100.11

3. VIP 设置在 load balancer 上，load balancer 收到请求后选择 pool member WEB1，

   将数据包的目的 IP 设为 WEB1 的地址 172.16.100.9。

4. 在将数据包转发给 WEB1 之前，load balancer 将数据包的源 IP 修改为自己的 VIP 地址 172.16.100.11，

   其目的是保证 WEB1 能够将应答数据发送回 load balancer。

5. WEB1 收到请求数据包。

右图是 web server 应答的数据流：

1. WEB1 将数据包发送给 load balancer。
2. load balancer 收到 WEB1 发回的数据后，将目的 IP 修改为 Client 的地址 10.10.10.4。同时也将数据包的源 IP 修改为 VIP 地址 172.16.100.11，保证 Client 能够将后续数据发送给自己。
3. load balancer 将数据发送给路由器。
4. 路由器将数据包的源地址恢复成服务器的外网 IP 10.10.10.7，然后发送给 Client。
5. Client 收到应答数据。

以上就是 Load Balance as a Service 的工作原理。
下节我们将开始实践 Neutron LBaaS。

# 配置 LBaaS

首先在配置中启用 LBaaS 服务。Neutron 通过 lbaas plugin 和 lbaas agent 提供 LBaaS 服务。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171237.jpg)

lbaas plugin 与 Neutron Server 一起运行在控制节点上。lbaas agent 运行在网络节点上。对于验环境，控制节点和网络节点是一个，都是 devstack-controller。

**配置 LBaaS agent**

配置 LBaaS agent 的地方是` /etc/neutron/services/loadbalancer/haproxy/lbaas_agent.ini`。

定义 interface_driver：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171247.jpg)

interface_driver 的作用是设置 load balancer 的网络接口驱动，可以有两个选项：

Linux Bridge

``` shell
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
```

Open vSwitch

``` shell
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```

**配置 LBaaS plugin**

在` /etc/neutron/neutron.conf `中设置启用 LBaaS plugin

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171321.jpg)

在` /etc/neutron/neutron_lbaas.conf `中设置 service provider

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171333.jpg)

从注释信息可以看到，除了默认的 HAProxy，Neutron 也支持第三方 provider，比如 radware，VMWareEdge 等。

重启 neutron 服务，确保 LBaaS 正常运行。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171338.jpg)

现在将实践如下 LBaaS 环境。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171347.jpg)

该环境描述如下：

1. 创建一个 Pool “web servers”。
2. 两个 pool member “WEB1” 和 “WEB2”，均为运行 Ubuntu cloud image 的 instance。
3. load balancer VIP 与 floating IP 关联。
4. 位于外网的 client 通过 floating IP 访问 web server。

# 创建 Pool & VIP

开始实现如下 LBaaS 环境。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171832.jpg)

环境描述如下：

1. 创建一个 Pool “web servers”。
2. 两个 pool member “WEB1” 和 “WEB2”，均为运行 Ubuntu cloud image 的 instance。
3. load balancer VIP 与 floating IP 关联。
4. 位于外网的 client 通过 floating IP 外网访问 web server。

我们从第一步开始。

## 创建 Pool

点击菜单 Project -> Network -> Load Balancers，点击 Pools 标签页中的 “Add Pool” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171857.jpg)

显示 Pool 创建页面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171915.jpg)

将 Pool 命名为“web servers”。

Provider 选择默认的 “haproxy”。

Subnet 选择 “172.16.100.0/24”。

Protocol 选择 “HTTP”。

Load Balancing Method 选择 “ROUND_ROBIN”。

点击 “Add” 按钮，“web servers” 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171932.jpg)

这里对 Pool 的几个属性进行一下说明。

LBaaS 支持如下几种 Protocol:

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211171948.jpg)

因为用 web server 做实验，所以这里需要选择 “HTTP”

LBaaS 支持多种 load balance method:

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211172002.jpg)

**ROUND_ROUBIN**

如果采用 round robin 算法，load balancer 按固定的顺序从 pool 中选择 member 相应 client 的连接请求。这种方法的不足是缺乏机制检查 member 是否负载过重。有可能出现某些 member 由于处理能力弱而不得不继续处理新连接的情况。如果所有 pool member 具有相同处理能力、内存容量，并且每个连接持续的时间大致相同，这种情况非常适合 round robin，每个 member 的负载会很均衡。

**LEAST_CONNECTIONS**

如果采用 least connections 算法，load balancer 会挑选当前连接数最少的 pool  member。这是一种动态的算法，需要实时监控每个 member 的连接数量和状态。计算能力强的 member 能够更快的处理连接进而会分配到更多的新连接。

**SOURCE_IP**

如果采用 source IP 算法，具有相同 source IP 的连接会被分发到同一个 pool member。source IP 算法对于像购物车这种需要保存状态的应用特别有用，因为我们希望用同一 server 来处理某个 client 连续的在线购物操作。

在此次实验中选择的是 ROUND_ROUBIN 算法。

## 为 Pool 添加 VIP

现在 Pool 已经就绪，接下需要为其设置 VIP。在 “web servers” 的操作列表中点击 “Add VIP”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211172040.jpg)

VIP 命名为 “VIP for web servers”。

Subnet 选择 “172.16.100.0/24”，与 pool 一致。

指定 VIP 为 172.16.100.11，如果不指定，系统会自动从 subnet 中分配。

指定 HTTP 端口 80。

Session Persistence 选择 “SOURCE IP”。

可以通过 Connection Limit 限制连接的数量，如果不指定则为不加限制。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211172049.jpg)

点击 “Add”，VIP 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211172147.jpg)

通常我们希望让同一个 server 来处理某个 client 的连续请求。否则 client 可能会由于丢失 session 而不得不重新登录。这个特性就是 Session Persistence。VIP 支持如下几种 Session Persistence 方式：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211172104.jpg)

**SOURCE_****IP**

这种方式与前面 load balance 的 SOURCE_IP 效果一样。初始连接建立后，后续来自相同 source IP 的 client 请求会发送给同一个 member。当大量 client 通过同一个代理服务器访问 VIP 时（比如在公司和学校上网），SOURCE_IP 方式会造成 member 负载不均。

**HTTP_COOKIE**

HTTP_COOKIE 的工作方式如下：

当 client 第一次连接到 VIP 时，HAProxy 从 pool 中挑选出一个 member。

当此 member 响应请求时，HAProxy 会在应答报文中注入命名为 “SRV” 的 cookie，这个 cookie 包含了该 member 的唯一标识。client 的后续请求都会包含这个 “SRV” cookie。HAProxy 会分析 cookie 的内容，并将请求转发给同一个 member。HTTP_COOKIE 优于 SOURCE_IP，因为它不依赖 client 的 IP。

**APP_COOKIE**

app cookie 依赖于服务器端应用定义的 cookie。比如 app 可以通过在 session 中创建 cookie 来区分不同的 client。HAProxy 会查看报文中的 app cookie，确保将包含 app cookie 的请求发送到同一个 member。如果没有 cookie（新连接或者服务器应用不创建 cookie），HAProxy 会采用 ROUND_ROUBIN 算法分配 member。

## 比较 Load Balance Method 和 Session Persistence

前面介绍了三种 Load Balance Method：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211172002.jpg)

这里还有三种 Session Persistence：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211172104.jpg)

因为两者都涉及到如何选择 pool member，所以很容易混淆。它们之间的最大区别在于选择 pool member 的阶段不同：

1. Load Balance Method 是为新连接选择 member 的方法
2. Session Persistence 是为同一个 client 的后续连接选择 member 的方法

例如这里我们的设置为：

Load Balance Method -- ROUND_ROUBIN

Session Persistence -- SOURCE_IP

当 client A 向 VIP 发送第一个请求时，HAProxy 通过 ROUND_ROUBIN 选择 member1。对于 client A 后续的请求，HAProxy 则会应用 SOURCE_IP 机制，仍然选择 member1 来处理请求。

## 添加 Pool Member

先准备两个 instance： “Web1” 和 “Web2”。

**使用 Ubuntu Cloud Image**

由于 cirros 镜像不能运行 HTTP 服务，实验使用 Ubuntu Cloud Image。下载地址为 http://uec-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211173253.jpg)

与以前的 instance 不同，Web1 和 Web2 使用了命名为“cloud”的 Key Pair。这是因为 Ubuntu Cloud Image 默认用户 “ubuntu” 的密码是随机生成的，通过 Key Pair 访问是一个比较方便的方法。

下面是 Key Pair 的配置方法：

### 1. 通过 ssh-keygen 生成 Key Pair。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211173304.jpg)

其中 cloud.key.pub 是公钥，需要添加到 instance 的 ~/.ssh/authorized_keys 文件中。 cloud.key 是私钥，用于访问 instance。

### 2. 将 Pair Key 导入 OpenStack

进入 Project -> Compute -> Access & Security 菜单，点击 Key Pairs 标签页的 “Import Key Pair” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211173321.jpg)

复制 cloud.key.pub 的内容。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211173401.jpg)

粘贴到 Public Key 输入框中，给 Key Pair 命名为 “cloud”，并点击 “Import Key Pair”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211173411.jpg)

Key Pair “cloud” 成功导入。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211174540.jpg)

### 3. 使用 Key Pair “cloud”。

launch instance 的时候，在 Access & Security 标签页中选择 Key Pair “cloud”。OpenStack 会自动将公钥添加进 instance 的  ~/.ssh/authorized_keys 文件。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211173438.jpg)

### 4. ssh instance。

用 -i cloud.key 指定私钥，并以 ubuntu 用户 ssh “Web1” 和 “Web2”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211174624.jpg)

### 5. 无需密码直接登录 instance。

注：为了便于演示，这里是在 router 的 namespace 中执行 ssh，其目的是网络可达。“Web1” 和 “Web2” 准备就绪了，可以添加到 Pool 中了。

在 Project -> Network -> Load Balancers 的 Members 标签页中 点击 “Add Member” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211174635.jpg)

Pool 选择 “web servers”。

Member(s) 中多选 “Web1” 和 “Web2”。

Protocol Port 设置为 “80”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211174654.jpg)

点击 “Add”，member 添加成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211174704.jpg)

Pool Member 有了，下面为 Pool 创建 Monitor，用于 Member 健康检查。

## 创建 Monitor

### 创建 Monitor

LBaaS 可以创建 monitor，用于监控 Pool Member 健康状态。

如果某个 member 不能正常工作，monitor 会将其状态设置为 down，从而避免将后续请求转发给它。

下面为 Pool 添加一个 monitor。在 Monitors 标签页中点击 “Add Monitor” 按钮

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175314.jpg)

Type 选择 “HTTP”，含义是通过 HTTP 检查 member 的健康状态。Delay 设置为 “10”，含义是 10 秒检查一次 member 的状态。Timeout 设置为 “5”，含义是如果 member 在 5 秒内无法应答，则超时。Max Reties 设置为 “3”，含义是如果尝试 3 次都超时或者失败，则将 member 状态设置为 down。

HTTP Method 设置为 “GET”URL 设置为 “/”Expected HTTP Status Codes 设置为 “200”

上面三项的含义是通过 HTTP GET 请求 member “/” URL，如果返回码为 200，则认为 member 状态正常。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175323.jpg)

点击 “Add”，monitor 创建成功。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175333.jpg)

下面将新建的 monitor 添加到 pool 。
在 “web servers” 的操作列表中点击 “Associate Monitor”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175344.jpg)

选择我们刚刚创建的 monitor。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175352.jpg)

点击 “Associate”。

### 测试 LBaaS

经过上面的设置，现在创建了包含 member “Web1” 和 “Web2” 的 Pool “web servers”，并添加了 monitor。

准备就绪，可以测试 load balancer 是否正常工作了。

首先在 Web1 和 Web2 中启动 HTTP 服务，在 80 端口监听

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175524.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175530.jpg)

这里我们使用 python 提供的 SimpleHTTPServer 模块启动了 HTTP 服务。web server 的 index.html 显示当前访问的是哪个 member。

在 router 的 namespace 上多次执行 curl 172.16.100.11（VIP）

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175537.jpg)

测试结果显示每次访问的都是 Web2 这个 member。

**为什么没有访问到 Web1 呢？**

前面讨论的内容中可以看到：Load Balance Method -- ROUND_ROUBINSession Persistence -- SOURCE_IP

在这种配置下，第一个 curl 请求 HAProxy 通过 ROUND_ROUBIN 选择了 Web2。而后续的请求，HAProxy 则会应用 SOURCE_IP 机制，仍然选择 Web2。

下面我们修改一下配置。在 “web servers” 的操作列表中点击 “Edit VIP”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175628.jpg)

选择 “No session persistence” 并保存。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175637.jpg)

再进行 curl 测试。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211175646.jpg)

可以看到已经在 “Web1” 和 “Web2” 之间 round robin 了。

# LBaaS 实现机制

在控制节点上运行 ip netns，可以看到 Neutron 创建了新的 namespace qlbaas-xxx。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180232.jpg)

该 namespace 对应我们创建的 pool “web servers”。其命名格式为 qlbaas-< pool ID>。可以通过 ip a 查看其设置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180236.jpg)

VIP 172.16.100.11 已经配置在 namespace interface 上。在 subnet 的 Port 列表中也可以找到该 interface 的相应配置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180306.jpg)

对于每一个 pool，Neutron 都会启动一个 haproxy 进程提供 load balancering 功能。通过 ps 命令查找 haproxy 进程：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180310.jpg)

haproxy 配置文件保存在` /opt/stack/data/neutron/lbaas/< pool ID>/conf `中。查看 “web servers” 的配置内容：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180324.jpg)

可以看到：

1. frontend 使用的 HTTP 地址为 VIP:80
2. backend 使用的 HTTP 地址为 172.16.100.10:80 和 172.16.100.9:80
3. balance 方法为 roundrobin

这些内容与前面的配置一致。

以上就是 Neutron 实现 LBaaS 的理。

# 使用 floating IP 访问 VIP

访问 Project -> Compute -> Access & Security，打开 Floating IPs 标签页，点击 “Allocate IP to Project” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180635.jpg)

在下拉列表中选择 “ext_net”，Neutron 将从该网络中分配 floating IP。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180915.jpg)

点击 “Allocate IP”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180649.jpg)

分配到的 IP 为 “10.10.10.7”。点击 “Associate” 按钮。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180658.jpg)

在 “Port to be associated” 列表中选择 “VIP for web servers: 172.16.100.11” 并点击 “Associate”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180709.jpg)

成功将外网 IP 10.10.10.7 关联到 VIP。

下面是在 IP 为 10.10.10.4 的 instance 中进行 curl 测试。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171211180725.jpg)

floating IP 生效，load balaner 工作正常。

# LBaaS 总结

LBaaS 为租户提供了横向扩展应用的能力。

租户可以将外部请求 balancing 到多个 instance 上，并通过 monitor 实现应用的高可用。

LBaaS 当前的实现是基于 HAProxy，其功能已经能够满足普通业务需求。



































































































































