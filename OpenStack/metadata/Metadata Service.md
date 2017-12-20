# metadata service 应用

instance 是通过 image 部署出来的，image 中包含了操作系统（例如 Ubuntu 16.04），最常用的软件（例如 SSH）以及最通用的配置（例如 eth0 dhcp）。然而在创建 instance 的时候，如果需要对 instance 进行一些额外的配置，比如：安装某些包、开启一些服务、添加 SSH 秘钥、配置 hostname 等等。如果直接将这些配置部署到image中，工作量太大，而且image变得非常庞杂，不易管理。

**推荐方法：**

由 OpenStack Metadata Service 提供 instance 的配置信息（这些信息被统称为 metadata）。instance 启动时向 Metadata Service 请求并获得自己的 metadata，instance 的 cloud-init（或 cloudbase-init）根据 metadata 完成个性化配置工作。

## 使用实例

将 ssh public key 添加到 instance。

首先在 “Project -> Compute -> Access & Security” 中创建 Key Pair。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219163746.jpg)

OpenStack 会创建一对 ssh pulbic key 和 private key，public key 存放在 OpenStack 数据库中，private key 会在点击 “Create Key Pair” 按钮时自动下载。

现在 "cloudman" 这个 key pair 就是要用的 metadata 了。部署 instance 时，选择 "cloudman"。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219163802.jpg)

instance 启动后，可以看到这个 cloudman 的 public key 已经保存到` .ssh/authorized_keys `中了。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219163810.jpg)

这样就可以用 cloudman 的 private key 直接登录 instance。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219163818.jpg)

# Metadata Service 架构详解

下面是 Metadata Service 的架构图，本节我们详细讨论各个组件以及它们之间的关系。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219164803.jpg)

**nova-api-metadata**

nova-api-metadata 是 nova-api 的一个子服务，它是 metadata 的提供者，instance 可以通过 nova-api-metadata 的 REST API 来获取 metadata 信息。

nova-api-metadata 运行在控制节点上，服务端口是 8775。

``` shell
# netstat -tanple|grep 8775
tcp        0      0 0.0.0.0:8775            0.0.0.0:*               LISTEN      162        637585076  82947/python2
```

通过进程 ID 82947 查看该启动程序。

``` shell
# ps aux | grep 82947 | grep -v grep 
nova      82947  0.7  0.0 450800 144472 ?       Ss   03:57   2:04 /usr/bin/python2 /usr/bin/nova-api
```

这个环境是 N 版 OpenStack，nova-api-metadata 的程序名称就是 nova-api，nova-api-metadata 与常规的 nova-api 服务是合并在一起的。不过在 OpenStack 的其他发行版中可能有单独的 nova-api-metadata 进程存在。

nova.conf 通过参数 enabled_apis 指定是否启用 nova-api-metadata。

``` shell
# cat /etc/nova/nova.conf
[DEFAULT]
enabled_apis = ec2,osapi_compute,metadata
```

osapi_compute 是常规的 nova-api 服务，metadata 就是 nova-api-metadata 服务。

**neutron-metadata-agent**

nova-api-metadata 在控制节点上，走 OpenStack 内部管理网络，instance 是无法通过 [http://controller_ip:8775]() 直接访问 metadata service 的，因为网络不通。

答案是：借助 neutron-metadata-agent。

neutron-metadata-agent 运行在网络节点上。instance 先将 metadata 请求发给 neutron-metadata-agent，neutron-metadata-agent 再将请求转发到 nova-api-metadata。

``` shell
# ps aux | grep neutron-metadata-agent | grep -v grep 
neutron   82948  1.9  0.0 320692 57396 ?        Ss   03:57   5:57 /usr/bin/python2 /usr/bin/neutron-metadata-agent --config-file /usr/share/neutron/neutron-dist.conf --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/metadata_agent.ini --config-dir /etc/neutron/conf.d/common --config-dir /etc/neutron/conf.d/neutron-metadata-agent --log-file /var/log/neutron/metadata-agent.log
```

这里还有个问题需要解释清楚：instance 如何将请求发送到 neutron-metadata-agent？
实际上 instance 是不能直接与 neutron-metadata-agent 通信的，因为 neutron-metadata-agent 也是在 OpenStack 内部管理网络上的。不过好在网络节点上有另外两个组件，dhcp agent 和 l3 agent，它们两兄弟与 instance 可以位于同一 OpenStack network 中，这样就引出了下一个组件：` neutron-ns-metadata-proxy `。

**neutron-ns-metadata-proxy**

neutron-ns-metadata-proxy 是由 dhcp-agent 或者 l3-agent 创建的，也运行在网络节点。更精确的说法是：运行在网络节点的 namespace 中。
如果由 dhcp-agent 创建，neutron-ns-metadata-proxy 就运行在 dhcp-agent 所在的 namespace 中；如果由 l3-agent 创建，neutron-ns-metadata-proxy 就运行在 neutron router 所在的 namespace 中。“neutron-ns-metadata-proxy” 中间的 ns 就是 namespace 的意思。neutron-ns-metadata-proxy 与 neutron-metadata-agent 通过 unix domain socket 直接相连。

``` shell
# ps aux | grep neutron-ns-metadata-proxy | grep -v grep 
neutron    9102  0.0  0.0 297368 49608 ?        S    Oct17   0:09 /usr/bin/python2 /bin/neutron-ns-metadata-proxy --pid_file=/var/lib/neutron/external/pids/7425418d-7156-4827-a4d4-1eb604bb823e.pid --metadata_proxy_socket=/var/lib/neutron/metadata_proxy --router_id=7425418d-7156-4827-a4d4-1eb604bb823e --state_path=/var/lib/neutron --metadata_port=9697 --metadata_proxy_user=991 --metadata_proxy_group=988 --log-file=neutron-ns-metadata-proxy-7425418d-7156-4827-a4d4-1eb604bb823e.log --log-dir=/var/log/neutron
```

这样整个链路就打通了：

1. instance 通过 neutron network（Project 网络）将 metadata 请求发送到 neutron-ns-metadata-proxy。

2. neutron-ns-metadata-proxy 通过 unix domain socket 将请求发给 neutron-metadata-agent。

3. neutron-metadata-agent 通过内部管理网络将请求发送给 nova-api-metadata。

可能大家对于 neutron-ns-metadata-proxy 还会有些疑虑：既然 dhcp-agent 和 l3-agent 都可以创建和管理 neutron-ns-metadata-proxy，使用的时候该如何选择呢？

简单的说：各有各的使用场景，并且两种方案可以共存。

## 实例

**环境介绍**

1. 一个 all-in-one 环境（多节点类似）。
2. 已创建 neutron 网络 test_net，DHCP 已启动。在这个 metadata 实验中， test_net 的 type 不重要，flat、vlan、vxlan 都可以。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170703.jpg)

3. 暂无 neutron router。

准备就绪，开始实验。

**启动 instance**

通过 cirros 镜像部署一个 instance，命名为 c1，选择网络 test_net。启动过程中，查看 instance 的启动日志。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170721.jpg)

上面的 log 中我们看到两个信息：

① instance 从 DHCP 拿到了 IP 17.17.17.5，这个好理解，因为我们在test_net 上开启的 DHCP 服务。

② instance 会去访问 [http://169.254.169.254/2009-04-04/instance-id]()，尝试了 20 次都失败了。

**169.254.169.254**

169.254.169.254 是 metadata service 的 IP。

这个地址来源于 AWS，当年亚马逊在设计公有云的时候，为了让 instance 能够访问 metadata，就将 169.254.169.254 这个特殊的 IP 作为 metadata 服务器的地址，instance 启动时就会向 169.254.169.254 请求 metadata。OpenStack 之后也沿用了这个设计。

现在遇到的问题是 169.254.169.254 没法访问。cirros 的 cloud-init 显然是没有拿到 metadata 的，这点至少可以从 instance 的 hostname 没有被设置为 c1 判断出来。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170735.jpg)

前面在 Metadata Service 架构部分介绍了，instance 首先会将 metadata 请求发送给 DHCP agent 或者 L3_agent 管理的 neutron-ns-metadata-proxy。那目前到底是谁在管理 neutron-ns-metadata-proxy 呢？先在控制节点上查看一下 neutron-ns-metadata-proxy 的进程。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170743.jpg)

尽然没有 neutron-ns-metadata-proxy 在运行！

其原因是：默认配置下，neutron-ns-metadata-proxy 是由 L3_agent 管理的（后面会讨论让 DHCP 来管理），由于当前 test_net 并没有挂在 neutron router 上，所以没有启动 neutron-ns-metadata-proxy。

**添加 router**

要解决这个问题很简单：创建虚拟路由器 test_router 并连接 test_net。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170753.jpg)

现在控制节点上已经能够看到 test_router 管理的 neutron-ns-metadata-proxy 了。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170807.jpg)

重启 instance c1，看会发生怎样的变化。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170817.jpg)

instance 成功访问到 169.254.169.254。从结果看，cloud-init 已经获取到 metadata，因为 hostname 已经设置为 c1。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219170824.jpg)

# 获取 metadata 过程详解

启动 neutron router 后 instance c1 终于拿到了 metadata, 从下面 c1 的启动日志可知：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219171912.jpg)

c1 所认为的 metadata 服务地址是 169.254.169.254，端口为 80。我们在 c1 中尝试访问一下 metadata。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219171920.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219171928.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219171933.jpg)

确实能够拿到 metadata。但我 nova-api-metadata 是运行在控制节点上的，IP并不是 169.254.169.254。下面分析这个过程。

从 c1 的路由表得访问 169.254.169.254 的请求会走 17.17.17.1。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219171945.jpg)

17.17.17.1 实际上就是 test_router 在 test_net 上的 interface IP。这条路由是 OpenStack 自动添加到 instance 中的，这样就将 metadata 的请求转发到 neutron router。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219171955.jpg)

test_router 接收到 c1 的请求，会通过 iptable 规则转发到 9697 端口。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219172004.jpg)

9697 端口是 neutron-ns-metadata-proxy 的监听端口。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219172011.jpg)

到这里可以把思路重新理一下了：

1. instance 通过预定义的 169.254.169.254 请求 metadata。
2. 请求被转发到 neutron router。
3. router 将请求转发给 neutron-ns-metadata-proxy。
4. 再后面就简单了：neutron-ns-metadata-proxy 将请求通过 unix domain socket 发给 neutron-metadata-agent，后者再通过管理网络发给 nova-api-metadata。

OpenStack 默认通过 l3-agent 创建和管理 neutron-ns-metadata-proxy。但不是所有环境都有 l3-agent，比如直接用物理 router 的场景。这时就需要让 dhcp-agent 来管理 neutron-ns-metadata-proxy。

## 通过 dhcp-agent 访问 Metadata

OpenStack 默认通过 l3-agent 创建和管理 neutron-ns-metadata-proxy，进而与 nova-metadata-api 通信。但不是所有环境都有 l3-agent，比如直接用物理 router 的场景。这时就需要走另一条路：让 dhcp-agent 来创建和管理 neutron-ns-metadata-proxy。

打开` /etc/neutron/dhcp_agent.ini `，设置 `force_metadata`

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219183941.jpg)

重启 dhcp-agent 后，可以看到控制节点上多了一个 neutron-ns-metadata-proxy 进程。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219184204.jpg)

此进程通过 `--network_id` 关联到 `test_net`，这就是 dhcp-agent 启动的 neutron-ns-metadata-proxy，用于接收 `test_net` 网络上 instance 的 metadata 请求。每一个 network 都有一个与之对应的 neutron-ns-metadata-proxy。

重启 instance `c1`，查看路由表。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219183837.jpg)

请注意，现在访问 `169.254.169.254` 的路由已由之前的 `17.17.17.1`变为 `17.17.17.2`。这里的 `17.17.17.2` 是 dhcp-agent 在`test_net` 上的 IP。这条路由是由 dhcp-agent 添加进去的。正是因为这条路由的存在，即便 l3-agent 与 dhcp-agent 同时提供 neutron-ns-metadata-proxy 服务，metadata 请求也只会发送给 dhcp-agent。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219183849.jpg)

同时也看到，dhcp-agent 已经将 IP `169.254.169.254` 配置到了自己身上。也就是说：`c1` 访问 metadata 的请求 [http://169.254.169.254]() 实际上是发送到了 dhcp-agent 的 80 端口。而监听 80 端口的正是 dhcp-agent 启动的 neutron-ns-metadata-proxy 进程。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219183858.jpg)

metadata-proxy 将请求通过 unix domain socket 发给 neutron-metadata-agent，后者再通过管理网络发给 nova-api-metadata。

到这里，已经分别讨论了通过 l3-agent 和 dhcp-agent 访问 metadata 的实现方法。对于 `169.254.169.254`：

1. l3-agent 用 iptables 规则来转发。
2. dhcp-agent 则是将此 IP 配置到自己的 interface 上。

nova-api-metadata 是怎么知道应该返回哪个 instance 的 metadata？`c1` 只是向 `169.254.169.254` 发送了一个 http 请求，nova-api-metadata 怎么就知道应该返回 `c1` 的 metadata 呢？

# instance 获得自己的 Metadata

要想从 nova-api-metadata 获得 metadata，需要指定 instance 的 id。但 instance 刚启动时无法知道自己的 id，所以 http 请求中不会有 instance id 信息，id 是由 neutron-metadata-agent 添加进去的。针对 l3-agent 和 dhcp-agent 这两种情况在实现细节上有所不同，下面分别讨论。

## l3-agent

下面是 l3-agent 参与情况下 metadata http 请求的处理流程图。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185300.jpg)

大的流程为：instance -> neutron-ns-metadata-proxy -> neutron-metadata-agent -> nova-api-metadata，处理细节说明如下：

① neutron-ns-metadata-proxy 接收到请求，在转发给 neutron-metadata-agent 之前会将 instance ip 和 router id 添加到 http 请求的 head 中，这两个信息对于 l3-agent 来说很容易获得。

② neutron-metadata-agent 接收到请求后，会查询 instance 的 id，具体做法是：

1) 通过 router id 找到 router 连接的所有 subnet，然后筛选出 instance ip 所在的 subnet。

2）在 subnet 中找到 instance ip 对应的 port。

3）通过 port 找到对应的 instance 及其 id。

③ neutron-metadata-agent 将 instance id 添加到 http 请求的 head 中，然后转发给 nova-api-metadata，这样 nova-api-metadata 就能返回指定 instance 的 metadata 了。

## dhcp-agent

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185316.jpg)

① neutron-ns-metadata-proxy 在转发请求之前会将 instance ip 和 network id 添加到 http 请求的 head 中，这两个信息对于 dhcp-agent 来说很容易获得。

② neutron-metadata-agent 接收到请求后，会查询 instance 的 id，具体做法是：

1) 通过 network id 找到 network 所有的 subnet，然后筛选出 instance ip 所在的 subnet。

2）在 subnet 中找到 instance ip 对应的 port。

3）通过 port 找到对应的 instance 及其 id。

③ neutron-metadata-agent 将 instance id 添加到 http 请求的 head 中，然后转发给 nova-api-metadata，这样 nova-api-metadata 就能返回指定 instance 的 metadata 了。

这样，不管 instance 将请求发给 l3-agent 还是 dhcp-agent，nova-api-metadata 最终都能获知 instance 的 id，进而返回正确的 metadata。

从获取 metadata 的流程上看，有一步是至关重要的：**instance 必须首先能够正确获取 DHCP IP**，否则请求发送不到 `169.254.169.254`。但不是所有环境都会启用 dhcp，更极端的，有些环境可能连 nova-api-metadata 服务都不会启用。

## 实践 config drive

如果 instance 无法通过 metadata service 获取 metadata（无 DHCP 或者 nova-api-metadata 服务），instance 还可以通过 config drive 获得 metadata。

config drive 是一个特殊的文件系统，OpenStack 会将 metadata 写到 config drive，并在 instance 启动时挂载给 instance。如过 instance 安装了 cloud-init，config drive 会被自动 mount 并从中读取 metadata，进而完成后续的初始化工作。

**配置**

config drive 默认是 disable 的，所以首先得启用。有两种方法启用 config drive：

1. 启动 instance 时指定 `--config-drive true`。
2. 在计算节点的` /etc/nova/nova.conf `中配置 `force_config_drive = true`，这样部署到此计算节点的 instance 都会使用 config drive。实验中使用的就是这种方法。

config drive 支持两种格式，iso9660 和 vfat，默认是 iso9660，但这会导致 instance 无法在线迁移，必须设置成 `config_drive_format=vfat` 才能在线迁移，这一点需要注意。

配置完成后，重启 nova-compute 服务。

**过程分析**

部署一个新的 cirros instance ：`c2`，先到计算节点的 instances 目录下看看 c1 与 c2 的区别。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185733.jpg)

c2 的目录下会多一个 `disk.config` 文件，这就是 config drive。通过`virsh edit` 可以看到 disk.config 已经挂载到 instance 上了。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185741.jpg)

打开 c2 的控制台，hostname 已经配置好，说明 metadata 拿到了。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185749.jpg)

为了确保 metadata 不是从 nova-api-metadata 获取，测试环境中已经提前关闭了 DHCP 服务，可以看到当前 c2 是没有 IP 的。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185756.jpg)

`lsblk` 查看块设备，iso 设备 `sr0` 就是 config drive。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185809.jpg)

mount sr0，查看 config drive 的内容。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185951.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185825.jpg)

`meta_data.json` 中存放了 ssh public key, hostname 等信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219185839.jpg)

instance 可以通过 nova-api-metadata 或者 config drive 这两种途径拿到 metadata，如何使用这些 data 是 cloud-init 要完成的工作。

# cloud-init 工作原理

cloud-init 是 linux 的一个工具，当系统启动时，cloud-init 可从 nova metadata 服务或者 config drive 中获取 metadata，完成包括但不限于下面的定制化工作：

1. 设置 default locale
2. 设置 hostname
3. 添加 ssh keys到 .ssh/authorized_keys
4. 设置用户密码
5. 配置网络
6. 安装软件包

为了实现 instance 定制工作，cloud-init 会按 4 个阶段执行任务：

1. local
2. init
3. config
4. final

cloud-init 安装时会将这 4 个阶段执行的任务以服务的形式注册到系统中，比如在 systemd 的环境下，我们能够看到这4个阶段分别对应的服务：

1. local - cloud-init-local.service
2. init - cloud-init.service
3. config - cloud-config.service
4. final - cloud-final.service

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219190338.jpg)

#### **local 阶段**

作为 cloud-init 执行的第一个阶段，此时 instance 还不知道该如何配置网卡，cloud-init 的任务就是从 config drive 中获取配置信息，然后写入 /etc/network/interfaces 文件（如果是 centos 则写入 /etc/sysconfig/network-scripts/ifcfg-xxx）。

**如果没有 config drive，则将所有网卡配置成 dhcp 模式**。这是非常关键的一步，只有当网卡正确配置后，才能获取到 metadata。

关于 local 阶段下一节会通过实验详细分析。

#### **init, config 和 final 阶段**

正常情况下，在这三个阶段执行之前 instance 网络已经配置好了，并且已经成功获取到 metadata。cloud-init 的配置文件 ` /etc/cloud/cloud.cfg `定义了三个阶段分别要执行的任务，任务以 module 形式指定。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219190354.jpg)

instance 真正的定制工作就是由这些 module 完成的。module 决定做哪些定制化工作，而 metadata 则决定最终定制化的结果。

举个例子，如果 cloud.cfg 中指定了 `set_hostname` 这个 module，则意味着 cloud-int 会设置 instance 的主机名，而具体设置成哪个主机名则由 metadata 中 `hostname` 参数决定。

有些 module 是有默认行为的，比如 `growpart`，如果 metadata 中没有特别指定，它会自动扩展 `/` 分区。

module资料：[https://cloudinit.readthedocs.io/en/latest/topics/modules.html](https://cloudinit.readthedocs.io/en/latest/topics/modules.html)

## instance 网卡

instance 的网卡是如何被配置并拉起的？这是理解和用好 cloud-init 非常关键的一步。先讨论一个最简单基础的场景：镜像中没有安装 cloud-init。

此时 instance 启动时网卡能不能被拉起来完全 **靠运气**！是的，就是运气。

因为这种情况下网卡的配置是死的，完全依赖于镜像中 /etc/network/interfaces 原有的配置。比如原镜像中的配置是：
auto eth0
iface eth0 inet dhcp
instance 只有满足下面所有条件网卡才能被拉起来：

1. 正好只有一块网卡
2. 正好网卡就叫 eth0
3. 正好 subnet 开了 DHCP

只要出现下面任意一种情况就会失败：

1. 还有其他网卡，比如 eth1，或者
2. 网卡不叫 eth0 ，比如 ens3，或者
3. 没有 DHCP

不同 instance 的网络配置差别很大，在 image 中写死的方法几乎是无效的，只能依靠 cloud-init 动态写入，接下来详细分析 cloud-init 的解决方案。

**DHCP**

先考虑 subnet 有 DHCP 服务的情况。

测试使用的镜像是 ubuntu 的 cloud image，已经预装的 cloud-init，下载地址为 [http://cloud-images.ubuntu.com/]() ，国内镜像[http://mirrors.ustc.edu.cn/ubuntu-cloud-images/]()

部署成功后，登录 instance，`ip a` 显示网卡 `ens3` 已经正确配置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219191254.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219191257.jpg)

下面分析这个 IP 是怎样配置上去的。

cloud-init 是在 local 阶段完成网络配置的，cloud-init 的执行过程被详细记录在 /var/log/cloud-init.log 中，看看相关操作。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219191308.jpg)

这里可以看到，cloud-init 会做如下工作：

① 扫描出 instance 中的所有网卡（这里是 ens3）

② 获取该网卡的配置信息。 因为没有 config drive，无法得知网卡的详细配置信息，只能采用默认的 fallback 配置，即 dhcp 配置。

③ 将配置信息写入 /etc/network/interfaces.d/50-cloud-init.cfg，内容为：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219191321.jpg)

这样网卡就以 dhcp 模式拉起来，正好与 subnet 的 dhcp 服务对接上，IP、网关等信息就配上去了。

几点说明：

1. instance 上的每一块网卡都会被 cloud-init 扫描出来。
2. 如果没有 config drive 将采用 fallback 配置，将扫描出来的**第一块** （只有这一块）网卡配置成 dhcp 模式。请注意：这是 cloud-init 默认行为，跟这块网卡对应的 subnet 是否开启了 DHCP 没有任何关系。
3. cloud-init 会根据 instance 操作系统类型生成网卡配置文件。例如操作系统是 centos 的话则会将配置写到 /etc/sysconfig/network-scripts 目录下。

## 用 config drive 配置网络

在其他条件不变的情况下，cloud-init 依然会完成那 3 个步骤，也就是说网卡还是会被配置成 dhcp 模式，只是最后网卡没办法获得 IP 而已。

不开 DHCP 也是一个常见的场景，为了让 instance 的网卡在这种情况下也能够被正确配置，需要借助 config drive，下面开始实践。

在计算节点 /etc/nova/nova.conf 中需要添加一个配置，然后重启 nova-compute 服务。

``` shell
[DEFAULT]
flat_injected = True
```

flat_injected = True

flat_injected 的作用是让 config drive 能够在 instance 启动时将网络配置信息动态注入到操作系统中。

当前网络的 DHCP 已经关闭。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219193253.jpg)

instance 部署时指定使用 config drive。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219193302.jpg)

Neutron 为 instance 分配的 IP 为 `18.18.18.5`。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219193309.jpg)

instance 启动后登录系统，`ip a` 验证 IP 已经成功配置，说明 config drive 起作用了。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219193318.jpg)

重要的是弄懂 `18.18.18.5` 这个 IP 是如何配置上去的。打开 /var/log/cloud-init.log，分析如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219193331.jpg)

① 扫描出 instance 中的所有网卡，这一步与不使用 config drive 的情况完全一样。

② 获取该网卡的配置信息。 日志显示配置信息是从 `ds` 获取。ds 是 datasource 的缩写，在这里指的就是 config drive。在不使用 config drive 的情况下采用的是 `fallback` 配置。网卡配置信息记录在 config drive ` openstack/latest/network_data.json `文件里，内容如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219193536.jpg)

③ 将配置信息写入` /etc/network/interfaces.d/50-cloud-init.cfg `，内容为：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219193345.jpg)

可以看到 IP 以 `static` 方式配置。

总结一下：

1. 在没有使用 config drive 的情况下，cloud-init 只会配置第一块网卡，且设置为 dhcp 模式，所以：

① 如果 instance 只有一块网卡，且启用了 DHCP，网卡能够被正常拉起。 

② 如果 instance 有多块网卡，第一块会尝试以 dhcp 方式拉起，其他网卡不作处理。

2. 使用 config drive 的情况下，无论是否启用 DHCP，所有网卡都能被正确配置且成功拉起（如果 dhcp 网卡 >= 2，CentOS 还是有问题，可能跟目前所用的 cloud-init 版本较低有关）。
3. 如果可能，尽量使用 config drive。

# cloud-init 典型应用

## 设置 hostname

cloud-init 默认会将 instance 的名字设置为 hostname。但这样不太方便，有时希望能够将二者分开，可利用 cloud-init 的`set_hostname` 模块实现。`set_hostname` 它会查询 metadata 中 hostname 信息，默认值就是 instance 的名字。我们可以指定自己的 hostname，方法是将下面的内容传给 cloud-init：

``` shell
#cloud-config
hostname: my1.cloudman.cc
manage_etc_hosts: true
```

说明如下：

1. cloud-init 只会读取以 `#cloud-config` 开头的数据，所以这一行一定要写对。
2. `hostname: my1.cloudman.cc` 告诉 cloud-init 将 hostname 设置为 my1.cloudman.cc。
3. `manage_etc_hosts: true` 告诉 cloud-init 更新 /etc/hosts 文件。

接下来的问题是：如何将这些信息传给 cloud-init？

有三几种方法：

① instance 部署时，直接将其粘贴到 `Customization Script` 输入框中。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219204430.jpg)

② 将其保存为文件，instance 部署时上传（上图所示）。

③ 将其保存为文件，命令行 `nova boot` 或者 `openstack server create` 部署 instance 时，使用参数 `--user-data` 传入。

部署成功后，hostname 正确设置，/etc/hosts 也相应更新。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219204441.jpg)

## 定制用户初始密码

官方的 cloud image 默认只能通过 ssh key 登录。可以利用`set-passwords` 模块为用户设置密码并启用密码登录。需要传入的脚本如下：

``` shell
#cloud-config
chpasswd:
   list: |
       root:123456
       ubuntu:123456
   expire: false
ssh_pwauth: true
```



说明如下：

1. root 和 ubuntu 用户密码设置为 123456。
2. `ssh_pwauth` 启用密码登录。

instance 启动后 ssh 验证：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219204449.jpg)

ubuntu 用户 ssh 密码登录成功，并且可通过密码切换到 root。

## 安装软件

标准镜像中不可能包含我们需要的所有软件，定制安装是不可避免的。一个办法是部署完后手动安装，另一个办法是通过 `package-update-upgrade-install` 模块让 cloud-init 自动为我们安装。

需要传入的脚本如下：

``` shell
#cloud-config
apt:
 primary:
   - arches: [default]
     search:
       - http://1.2.3.4
packages:
- pwgen
- pastebinit
- [libpython2.7, 2.7.3-0ubuntu3.1]
```

说明如下：

1. `apt` 指定安装源的位置，这里为 [http://1.2.3.4]() 。如果是 yum 源则用 `yum_repos` 模块指定，具体用法可参看官网文档。
2. `packages` 指定需要安装的软件包，还可以指定具体版本。

instance 启动后可看到` /etc/apt/sources.list `中安装源已经更新为[http://1.2.3.4。]()

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219204519.jpg)

由于 [http://1.2.3.4]() 不是一个有效的 apt 源，安装肯定会失败，我们可以在 /var/log/cloud-init.log 看到失败的信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171219204528.jpg)

虽然失败了，但我们至少可以确定如下事情：

1. 传入的脚本是有效的，cloud-init 确实在尝试安装指定的软件。
2. /var/log/cloud-init.log 会完整地记录 cloud-init 运行的所有细节，是 debug 最重要的工具。

cloud-init 的模块众多，功能很全，是实现 instance 定制的神器。由于篇幅所限，这里只讨论了几个典型用例。

更多用法以及示例可参看： [https://cloudinit.readthedocs.io]()

























































































