# Nova组件

## 部署方案

对于 Nova，这些服务会部署在两类节点上：计算节点和控制节点。

1. 只有 nova-compute 需要放在计算节点上。
2. 其他子服务则是放在控制节点上的。

用 `nova service-list` 查看` nova-* `子服务都分布在哪些节点上

```shell
# nova service-list 
+----+------------------+----------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host     | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+----------+----------+---------+-------+----------------------------+-----------------+
| 22 | nova-consoleauth | hyhive01 | internal | enabled | up    | 2017-10-26T17:56:38.000000 | -               |
| 24 | nova-scheduler   | hyhive01 | internal | enabled | up    | 2017-10-26T17:56:34.000000 | -               |
| 26 | nova-conductor   | hyhive01 | internal | enabled | up    | 2017-10-26T17:56:33.000000 | -               |
| 29 | nova-consoleauth | hyhive02 | internal | enabled | up    | 2017-10-26T17:56:32.000000 | -               |
| 31 | nova-scheduler   | hyhive02 | internal | enabled | up    | 2017-10-26T17:56:31.000000 | -               |
| 35 | nova-conductor   | hyhive02 | internal | enabled | up    | 2017-10-26T17:56:31.000000 | -               |
| 37 | nova-compute     | hyhive02 | nova     | enabled | up    | 2017-10-26T17:56:31.000000 | -               |
| 39 | nova-compute     | hyhive01 | nova     | enabled | up    | 2017-10-26T17:56:39.000000 | -               |
| 43 | nova-compute     | hyhive03 | nova     | enabled | up    | 2017-10-26T17:56:35.000000 | -               |
+----+------------------+----------+----------+---------+-------+----------------------------+-----------------+
```

## 从虚机创建流程看 nova-\* 子服务如何协同工作

![流程图](http://oydlbqndl.bkt.clouddn.com/Nova流程图.png)

1. 客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我创建一个虚机”
2. API 对请求做一些必要处理后，向 Messaging（RabbitMQ）发送了一条消息：“让 Scheduler 创建一个虚机”
3. Scheduler（nova-scheduler）从 Messaging 获取到 API 发给它的消息，然后执行调度算法，从若干计算节点中选出节点 A
4. Scheduler 向 Messaging 发送了一条消息：“在计算节点 A 上创建这个虚机”
5. 计算节点 A 的 Compute（nova-compute）从 Messaging 中获取到 Scheduler 发给它的消息，然后在本节点的 Hypervisor 上启动虚机。
6. 在虚机创建的过程中，Compute 如果需要查询或更新数据库信息，会通过 Messaging 向 Conductor（nova-conductor）发送消息，Conductor 负责数据库访问。

## Nova各组件详解

### nova-api

nova-api 是整个 Nova 组件的门户，所有对 Nova 的请求都首先由 nova-api 处理。nova-api 向外界暴露若干 HTTP REST API 接口 在 keystone 中我们可以查询 nova-api 的 endponits。

```shell
# openstack endpoint list | grep nova 
| 2aa1157d77214f399c0cd72bd6c37478 | RegionOne | nova         | compute      | True    | admin     | http://10.124.241.89:8774/v2.1/%(tenant_id)s |
| a3f619429d1946779e0e38c328bc477e | RegionOne | nova         | compute      | True    | internal  | http://10.124.241.89:8774/v2.1/%(tenant_id)s |
| b7ffe9a8a7dd43bdb0dc4479d8d63e5d | RegionOne | nova         | compute      | True    | public    | http://10.124.241.89:8774/v2.1/%(tenant_id)s |
# openstack endpoint show 2aa1157d77214f399c0cd72bd6c37478
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 2aa1157d77214f399c0cd72bd6c37478             |
| interface    | admin                                        |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | 28cec47162654fbda3618e40e1df4632             |
| service_name | nova                                         |
| service_type | compute                                      |
| url          | http://10.124.241.89:8774/v2.1/%(tenant_id)s |
+--------------+----------------------------------------------+
```

客户端就可以将请求发送到 endponits 指定的地址，向 nova-api 请求操作。 当然，作为最终用户的我们不会直接发送 Rest AP I请求。 OpenStack CLI，Dashboard 和其他需要跟 Nova 交换的组件会使用这些 API。

Nova-api 对接收到的 HTTP API 请求会做如下处理：

1. 检查客户端传入的参数是否合法有效
2. 调用 Nova 其他子服务的处理客户端 HTTP 请求
3. 格式化 Nova 其他子服务返回的结果并返回给客户端

### nova-conductor

nova-compute 需要获取和更新数据库中 instance 的信息。但 nova-compute 并不会直接访问数据库，而是通过 nova-conductor 实现数据的访问。

这样做有两个显著好处：

1. 更高的系统安全性
2. 更好的系统伸缩性

#### 更高的安全性

在 OpenStack 的早期版本中，nova-compute 可以直接访问数据库，但这样存在非常大的安全隐患。 因为 nova-compute 这个服务是部署在计算节点上的，为了能够访问控制节点上的数据库，就必须在计算节点的`/etc/nova/nova.conf` 中配置访问数据库的连接信息，比如

```shell
# cat /etc/nova/nova.conf
[database]
connection = mysql+pymysql://nova:nova.daemon@10.124.241.89:3306/nova
```

试想任意一个计算节点被黑客入侵，都会导致部署在控制节点上的数据库面临极大风险。为了解决这个问题，从 G 版本开始，Nova 引入了一个新服务 nova-conductor，将 nova-compute 访问数据库的全部操作都放到 nova-conductor 中，而且 nova-conductor 是部署在控制节点上的。 这样就避免了 nova-compute 直接访问数据库，增加了系统的安全性。

#### 更好的伸缩性

nova-conductor 将 nova-compute 与数据库解耦之后还带来另一个好处：提高了 nova 的伸缩性。nova-compute 与 conductor 是通过消息中间件交互的。
这种松散的架构允许配置多个 nova-conductor 实例。 在一个大规模的 OpenStack 部署环境里，管理员可以通过增加 nova-conductor 的数量来应对日益增长的计算节点对数据库的访问。

### nova-scheduler

创建 Instance 时，用户会提出资源需求，例如 CPU、内存、磁盘各需要多少。OpenStack 将这些需求定义在 flavor 中，用户只需要指定用哪个 flavor 就可以了。

```shell
# nova flavor-list 
+--------------------------------------+--------------------------------------+-----------+------+-----------+------+-------+-------------+-----------+
| ID                                   | Name                                 | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+--------------------------------------+--------------------------------------+-----------+------+-----------+------+-------+-------------+-----------+
| 00f46381-bbbe-46a2-9286-6dfbc0670b7f | 00f46381-bbbe-46a2-9286-6dfbc0670b7f | 512       | 0    | 0         |      | 1     | 1.0         | True      |
| 03292da9-a158-408e-850e-8918eabfa6f3 | 03292da9-a158-408e-850e-8918eabfa6f3 | 512       | 0    | 0         |      | 1     | 1.0         | True      |
| 04dbd80c-4075-4d7a-bd4f-b889de20b809 | 04dbd80c-4075-4d7a-bd4f-b889de20b809 | 2048      | 0    | 0         |      | 2     | 1.0         | True      |
```

Flavor 主要定义了 VCPU，RAM，DISK 和 Metadata 这四类。 nova-scheduler 会按照 flavor 去选择合适的计算节点。 

下面介绍 nova-scheduler 是如何实现调度的。

在` /etc/nova/nova.conf `中，nova 通过` scheduler_driver`，`scheduler_available_filters `和 `scheduler_default_filters `这三个参数来配置 nova-scheduler。

#### Filter scheduler

Filter scheduler 是 nova-scheduler 默认的调度器，调度过程分为两步：

1. 通过过滤器（filter）选择满足条件的计算节点（运行 nova-compute）
2. 通过权重计算（weighting）选择在最优（权重值最大）的计算节点上创建 Instance。

```shell
scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
```

Nova 允许使用第三方 scheduler，配置 scheduler_driver 即可。 这又一次体现了OpenStack的开放性。Scheduler 可以使用多个 filter 依次进行过滤，过滤之后的节点再通过计算权重选出最适合的节点。

![通过节点权重选择最优节点](http://oydlbqndl.bkt.clouddn.com/Nova选合适节点.jpg)

上图是调度过程的一个示例：

1. 最开始有 6 个计算节点 Host1-Host6
2. 通过多个 filter 层层过滤，Host2 和 Host4 没有通过，被刷掉了
3. Host1，Host3，Host5，Host6 计算权重，结果 Host5 得分最高，最终入选

#### Filter

当 Filter scheduler 需要执行调度操作时，会让 filter 对计算节点进行判断，filter 返回 True 或 False。Nova.conf 中的 scheduler_available_filters 选项用于配置 scheduler 可用的 filter，默认是所有 nova 自带的 filter 都可以用于滤操作。

```shell
scheduler_available_filters = nova.scheduler.filters.all_filters
```

另外还有一个选项 scheduler_default_filters，用于指定 scheduler 真正使用的 filter，默认值如下

```shell
scheduler_default_filters=RetryFilter,AvailabilityZoneFilter,PciPassthroughFilter,CoreFilter,RamFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter
```

scheduler 按照列表中的顺序依次过滤。 下面依次介绍每个 filter。

#####RetryFilter

RetryFilter 的作用是刷掉之前已经调度过的节点。

假设 A,B,C 三个节点都通过了过滤，最终 A 因为权重值最大被选中执行操作。 但由于某个原因，操作在 A 上失败了。 默认情况下，nova-scheduler 会重新执行过滤操作（重复次数由 scheduler_max_attempts 选项指定，默认是 3）。 那么这时候 RetryFilter 就会将 A 直接刷掉，避免操作再次失败。 RetryFilter 通常作为第一个 filter。

##### AvailabilityZoneFilter

为提高容灾性和提供隔离服务，可以将计算节点划分到不同的Availability Zone中。例如把一个机架上的机器划分在一个 Availability Zone 中。 OpenStack 默认有一个命名为 “Nova” 的 Availability Zone，所有的计算节点初始都是放在 “Nova” 中。 用户可根据需要创建自己的 Availability Zone。

```shell
# nova availability-zone-list
+-----------------------+----------------------------------------+
| Name                  | Status                                 |
+-----------------------+----------------------------------------+
| internal              | available                              |
| |- hyhive01           |                                        |
| | |- nova-conductor   | enabled :-) 2017-10-26T19:23:12.000000 |
| | |- nova-consoleauth | enabled :-) 2017-10-26T19:23:18.000000 |
| | |- nova-scheduler   | enabled :-) 2017-10-26T19:23:15.000000 |
| |- hyhive02           |                                        |
| | |- nova-conductor   | enabled :-) 2017-10-26T19:23:11.000000 |
| | |- nova-consoleauth | enabled :-) 2017-10-26T19:23:12.000000 |
| | |- nova-scheduler   | enabled :-) 2017-10-26T19:23:11.000000 |
| nova                  | available                              |
| |- hyhive01           |                                        |
| | |- nova-compute     | enabled :-) 2017-10-26T19:23:19.000000 |
| |- hyhive02           |                                        |
| | |- nova-compute     | enabled :-) 2017-10-26T19:23:11.000000 |
| |- hyhive03           |                                        |
| | |- nova-compute     | enabled :-) 2017-10-26T19:23:15.000000 |
+-----------------------+----------------------------------------+
```

创建 Instance 时，需要指定将 Instance 部署到在哪个 Availability Zone中。

nova-scheduler 在做 filtering 时，会使用 AvailabilityZoneFilter 将不属于指定 Availability Zone 的计算节点过滤掉。

##### RamFilter

RamFilter 将不能满足 flavor 内存需求的计算节点过滤掉。

对于内存有一点需要注意： 为了提高系统的资源使用率，OpenStack 在计算节点可用内存时允许 overcommit，也就是可以超过实际内存大小。 超过的程度是通过 `nova.conf `中 ram_allocation_ratio 这个参数来控制的，默认值为 1.5

```shell
# cat /etc/nova/nova.conf
ram_allocation_ratio=1.5
```

其含义是：如果计算节点的内容为 10GB，OpenStack 则会认为它有 15GB（10*1.5）内存。

##### DiskFilter

DiskFilter 将不能满足 flavor 磁盘需求的计算节点过滤掉。Disk 同样允许 overcommit，通过 nova.conf 中 disk_allocation_ratio 控制，默认值为 1

```shell
# cat /etc/nova/nova.conf
disk_allocation_ratio = 1.0
```

##### CoreFilter

CoreFilter 将不能满足 flavor vCPU 需求的计算节点过滤掉。vCPU 同样允许 overcommit，通过 `nova.conf` 中 cpu_allocation_ratio 控制，默认值为 16

```shell
# cat /etc/nova/nova.conf
cpu_allocation_ratio = 16
```

这意味着一个 8 vCPU 的计算节点，nova-scheduler 在调度时认为它有 128 个 vCPU。 需要提醒的是： nova-scheduler 默认使用的 filter 并没有包含 CoreFilter。 如果要用，可以将 CoreFilter 添加到 nova.conf 的 scheduler_default_filters 配置选项中。

##### ComputeCapabilitiesFilter

ComputeCapabilitiesFilter 根据计算节点的特性来筛选。这个比较高级。
例如节点有 x86\_64 和 ARM 架构的，如果想将 Instance 指定部署到 x86_64 架构的节点上，就可以利用 ComputeCapabilitiesFilter。在 flavor 中有个 Metadata ，Compute 的 Capabilitie s在 Metadata中 指定。

![ComputeCapabilitiesFilter 1](http://oydlbqndl.bkt.clouddn.com/ComputeCapabilitiesFilter 1.png)



![ComputeCapabilitiesFilter 2](http://oydlbqndl.bkt.clouddn.com/ComputeCapabilitiesFilter 2.png)

![ComputeCapabilitiesFilter 3](http://oydlbqndl.bkt.clouddn.com/ComputeCapabilitiesFilter 3.png)

配置好后，ComputeCapabilitiesFilter 在调度时只会筛选出 x86_64 的节点。 如果没有设置 Metadata，ComputeCapabilitiesFilter 不会起作用，所有节点都会通过筛选。

##### ImagePropertiesFilter

ImagePropertiesFilter 根据所选 image 的属性来筛选匹配的计算节点。 跟 flavor 类似，image 也有 metadata，用于指定其属性。

例如某个 image 只能运行在 kvm 的 hypervisor 上，可以通过 “Hypervisor Type” 属性来指定。

```shell
# nova hypervisor-list 
+----+---------------------+-------+---------+
| ID | Hypervisor hostname | State | Status  |
+----+---------------------+-------+---------+
| 3  | hyhive02            | up    | enabled |
| 5  | hyhive01            | up    | enabled |
| 7  | hyhive03            | up    | enabled |
+----+---------------------+-------+---------+
```

##### ServerGroupAntiAffinityFilter

ServerGroupAntiAffinityFilter 可以尽量将 Instance 分散部署到不同的节点上。

例如有 inst1，inst2 和 inst3 三个 instance，计算节点有 A,B 和 C。 为保证分散部署，进行如下操作：

1. 创建一个 anti-affinity 策略的 server group “group-1”

   ```shell
   #nova server-group-create --policy anti-affinity group-1
   ```

   请注意，这里的 server group 其实是 instance group，并不是计算节点的 group。

2. 依次创建 Instance inst1, inst2和inst3并放到group-1中

   ```shell
   #nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst1
   #nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst2
   #nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst3
   ```

因为 group-1 的策略是 AntiAffinity，调度时 ServerGroupAntiAffinityFilter 会将 inst1, inst2 和 inst3 部署到不同计算节点 A, B 和 C。目前只能在 CLI 中指定 server group 来创建 instance。

创建 instance 时如果没有指定 server group，ServerGroupAntiAffinityFilter 会直接通过，不做任何过滤。

##### ServerGroupAffinityFilter

与 ServerGroupAntiAffinityFilter 的作用相反，ServerGroupAffinityFilter 会尽量将 instance 部署到同一个计算节点上。 方法类似

1. 创建一个 affinity 策略的 server group “group-2”

   ```shell
   #nova server-group-create --policy affinity group-2
   ```

2. 依次创建 instance inst1, inst2 和 inst3 并放到 group-2 中

   ```shell
   #nova boot --image IMAGE_ID --flavor 1 --hint group=group-2 inst1
   #nova boot --image IMAGE_ID --flavor 1 --hint group=group-2 inst2
   #nova boot --image IMAGE_ID --flavor 1 --hint group=group-2 inst3
   ```

因为 group-2 的策略是 Affinity，调度时 ServerGroupAffinityFilter 会将 inst1, inst2 和 inst3 部署到同一个计算节点。创建 instance 时如果没有指定 server group，ServerGroupAffinityFilter 会直接通过，不做任何过滤。

##### Weight

经过前面一堆 filter 的过滤，nova-scheduler 选出了能够部署 instance 的计算节点。如果有多个计算节点通过了过滤，那么最终选择哪个节点由weight决定

Scheduler 会对每个计算节点打分，得分最高的获胜。 打分的过程就是 weight，翻译过来就是计算权重值。

目前 nova-scheduler 的默认实现是根据计算节点空闲的内存量计算权重值： 空闲内存越多，权重越大，instance 将被部署到当前空闲内存最多的计算节点上。

#### 日志

回顾一下 nova-scheduler 的工作过程。整个过程都被记录到 nova-scheduler 的日志文件中。 比如部署一个 instance 时打开 nova-scheduler 的日志`/var/log/nova/scheduler.log`

![scheduler](http://oydlbqndl.bkt.clouddn.com/nova-scheduler.png)

日志显示初始有两个 host（在实验环境中就是 devstack-controller 和 devstack-compute1），依次经过 9 个 filter 的过滤（RetryFilter, AvailabilityZoneFilter, RamFilter, DiskFilter, ComputeFilter, ComputeCapabilitiesFilter, ImagePropertiesFilter, ServerGroupAntiAffinityFilter, ServerGroupAffinityFilter），两个计算节点都通过了。那么接下来就该 weight 了：

![ weight](http://oydlbqndl.bkt.clouddn.com/Nova_reweight.png)

可以看到因为 devstack-controller 的空闲内存比 devstack-compute1 多（7466 > 3434），权重值更大（1.0 > 0.4599），最终选择 devstack-controller。

注：要显示 DEBUG 日志，需要在 /etc/nova/nova.conf 中打开 debug 选项

```shell
# cat /etc/nova/nova.conf
[DEFAULT]
debug = True
```

### nova-compute

nova-compute 在计算节点上运行，负责管理节点上的 instance。 OpenStack 对 instance 的操作，最后都是交给 nova-compute 来完成的。 nova-compute 与 Hypervisor 一起实现 OpenStack 对 instance 生命周期的管理。

#### 通过 Driver 架构支持多种 Hypervisor

现在市面上有很多 Hypervisor，nova-compute 为这些 Hypervisor 定义了统一的接口，Hypervisor 只需要实现这些接口，就可以 Driver 的形式即插即用到 OpenStack 系统中。 Nova Driver的架构示意图：

![Nova Driver](http://oydlbqndl.bkt.clouddn.com/nova driver.png)

我们可以在 /opt/stack/nova/nova/virt/ 目录下查看到 OpenStack 源代码中已经自带了上面这几个 Hypervisor 的 Driver

```shell
# ll /usr/lib/python2.7/site-packages/nova/virt/ | grep "^d"
drwxr-xr-x 4 root root   169 Sep 11 05:09 disk
drwxr-xr-x 2 root root  4096 Sep 11 05:09 hyperv
drwxr-xr-x 2 root root   143 Sep 11 05:09 image
drwxr-xr-x 2 root root  4096 Sep 11 05:09 ironic
drwxr-xr-x 4 root root  4096 Sep 11 05:09 libvirt
drwxr-xr-x 2 root root  4096 Sep 11 05:09 vmwareapi
drwxr-xr-x 4 root root  4096 Sep 11 05:09 xenapi
```

某个特定的计算节点上只会运行一种 Hypervisor，只需在该节点 nova-compute 的配置文件 `/etc/nova/nova.conf `中配置所对应的 compute_driver 就可以了。在我们的环境中因为是 KVM，所以配置的是 Libvirt 的 driver。

```shell
compute_driver = libvirt.LibvirtDriver
```

nova-compute 的功能可以分为两类：

1. 定时向 OpenStack 报告计算节点的状态
2. 实现 instance 生命周期的管理

#### 定期向OpenStack报告计算节点的状态

前面有 nova-scheduler 的很多 Filter 是根据算节点的资源使用情况进行过滤的。比如 RamFilter 要检查计算节点当前可以的内存量；CoreFilter 检查可用的 vCPU 数量；DiskFilter 则会检查可用的磁盘空间。

nova-compute 会定期向 OpenStack 报告每个计算节点的这些信息。从 nova-compute 的日志 `/var/log/nova/nova-compute.log` 可以发现： 每隔一段时间，nova-compute 就会报告当前计算节点的资源使用情况和 nova-compute 服务状态。

```shell
2017-10-23 03:15:11.866 40200 INFO nova.compute.resource_tracker [req-734717c0-6e12-418d-a901-a8e0642feacf - - - - -] Final resource view: name=hyhive02 phys_ram=262048MB used_ram=76800MB phys_disk=2631GB used_disk=0GB total_vcpus=40 used_vcpus=35 pci_stats=[]
2017-10-23 03:15:11.905 40200 INFO nova.compute.resource_tracker [req-734717c0-6e12-418d-a901-a8e0642feacf - - - - -] Compute_service record updated for hyhive02:hyhive02
```

nova-compute 是如何获得当前计算节点的资源使用信息的

要得到计算节点的资源使用详细情况，需要知道当前节点上所有 instance 的资源占用信息。 这些信息最清楚的是 Hypervisor。

通过上面的 Nova Driver 架构，nova-compute 可以通过 Hypervisor 的 driver 拿到这些信息。举例来说，在实验环境下 Hypervisor 是 KVM，用的 Driver 是 LibvirtDriver。 LibvirtDriver 可以调用相关的 API 获得资源信息，这些 API 的作用相当于在 CLI 里执行 virsh nodeinfo、virsh dominfo 等命令。

#### 实现 instance 生命周期的管理

OpenStack 对 instance 最主要的操作都是通过 nova-compute 实现的，包括 instance 的 launch、shutdown、reboot、suspend、resume、terminate、resize、migration、snapshot 等。

首先看instance launch

当 nova-scheduler 选定了部署 instance 的计算节点后，会通过消息中间件 rabbitMQ 向选定的计算节点发出 launch instance 的命令。 该计算节点上运行的 nova-compute 收到消息后会执行 instance 创建操作。 日志`/var/log/nova/nova-compute.log` 记录了整个操作过程。

nova-compute 创建 instance 的过程可以分为 4 步：

1. 为 instance 准备资源
2. 创建 instance 的镜像文件
3. 创建 instance 的 XML 定义文件
4. 创建虚拟网络并启动虚拟机

下面依次讨论每个步骤。

##### 为instance准备资源

nova-compute 首先会根据指定的 flavor 依次为 instance 分配内存、磁盘空间和 vCPU。可以在日志中看到这些细节

```shell
2017-10-27 03:50:36.518 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] Attempting claim: memory 2048 MB, disk 0 GB, vcpus 2 CPU
2017-10-27 03:50:36.519 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] Total memory: 262048 MB, used: 76800.00 MB
2017-10-27 03:50:36.519 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] memory limit: 262048.00 MB, free: 185248.00 MB
2017-10-27 03:50:36.520 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] Total disk: 2631 GB, used: 0.00 GB
2017-10-27 03:50:36.520 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] disk limit not specified, defaulting to unlimited
2017-10-27 03:50:36.521 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] Total vcpu: 40 VCPU, used: 35.00 VCPU
2017-10-27 03:50:36.521 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] vcpu limit: 160.00 VCPU, free: 125.00 VCPU
2017-10-27 03:50:36.523 40200 INFO nova.compute.claims [req-ee7eb4e4-d6a8-41b2-a097-6eacb563be15 2ce8cae1f1364864b78dbe6018862a4f cae3c5c61f9f418bbad5535560508040 - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] Claim successful
```

网络资源也会提前分配

![提前分配网络资源](http://oydlbqndl.bkt.clouddn.com/nova网络资源提前分配.png)

##### 创建instance的镜像文件

资源准备好之后，nova-compute 会为 instance 创建镜像文件。OpenStack 启动一个 instance 时，会选择一个 image，这个 image 由 Glance 管理。 nova-compute会：

1. 首先将该 image 下载到计算节点
2. 然后将其作为 backing file 创建 instance 的镜像文件。

###### 从Glance下载imag

nova-compute 首先会检查 image 是否已经下载（比如之前已经创建过基于相同 image 的 instance）。如果没有，就从 Glance 下载 image 到本地。由此可知，如果计算节点上要运行多个相同 image 的 instance，只会在启动第一个 instance 的时候从 Glance 下载 image，后面的 instance 启动速度就大大加快了。 日志如下：

![下载镜像](http://oydlbqndl.bkt.clouddn.com/从Glance下载img.png)

可以看到

1. image（ID为 917d60ef-f663-4e2d-b85b-e4511bb56bc2）是 qcow2 格式，nova-compute 将其下载。Nova 默认会通过 qemu-img 转换成 raw 格式，以提高 IO 性能。
2. image 的存放目录是`/opt/stack/data/nova/instances/_base`，这是由 `/etc/nova/nova.conf `的下面两个配置选项决定的。

```shell
instances_path = /opt/stack/data/nova/instances
base_dir_name = _base
```

3. 下载的 image 文件被命名为 60bba5916c6c90ed2ef7d3263de8f653111dd35f，这是 image id 的 SHA1 哈希值。

###### 为instance创建镜像文件

有了 image 之后，instance 的镜像文件直接通过 qemu-img 命令创建，backing file 就是下载的 image。

![创建镜像](http://oydlbqndl.bkt.clouddn.com/创建镜像.png)

这里 instance 的镜像文件位于 /opt/stack/data/nova/instances/f1e22596-6844-4d7a-84a3-e41e6d7618ef/disk，格式为 qcow2，其中 f1e22596-6844-4d7a-84a3-e41e6d7618ef 就是 instance 的 id。可以通过 qume-info 查看 disk 文件的属性

```shell
# qemu-img info /infinityfs1/images/win7.qcow2 
image: /infinityfs1/images/win7.qcow2
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 3.3G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

这里有两个容易搞混淆的术语，在此特别说明一下：

1. image，指的是 Glance 上保存的镜像，作为 instance 运行的模板。 计算节点将下载的 image 存放在RBD 中。
2. 镜像文件，指的是 instance 启动盘所对应的文件
3. 二者的关系是：image 是镜像文件 的 backing file。image 不会变，而镜像文件会发生变化。比如安装新的软件后，镜像文件会变大。

因为英文中两者都叫 “image”，为避免混淆，我们用 “image” 和 “镜像文件” 作区分。

###### 创建instance的xml定义的文件

![定义的xml文件](http://oydlbqndl.bkt.clouddn.com/定义的xml文件.png)

截图为创建的 XML 文件会保存到该 instance 目录 /opt/stack/data/nova/instances/f1e22596-6844-4d7a-84a3-e41e6d7618ef，命名为 libvirt.xml

###### 创建虚拟机网络并启动instance

接下来便是为 instance 创建虚拟网络设备

![创建虚拟网络](http://oydlbqndl.bkt.clouddn.com/创建虚拟机网络.png)

本环境用的是 linux-bridge 实现的虚拟网络，在 Neutron 章节我们会详细讨论 OpenStack 虚拟网络的不同实现方式。一切就绪，接下来可以启动 instance 了。

```shell
2017-10-27 03:51:34.782 40200 INFO nova.compute.manager [req-5bdf54ba-0eef-4b44-9068-9c5761a26a64 - - - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] VM Paused (Lifecycle Event)
2017-10-27 03:51:34.874 40200 INFO nova.compute.manager [req-5bdf54ba-0eef-4b44-9068-9c5761a26a64 - - - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] During sync_power_state the instance has a pending task (spawning). Skip.
2017-10-27 03:51:37.188 40200 INFO nova.compute.manager [req-5bdf54ba-0eef-4b44-9068-9c5761a26a64 - - - - -] [instance: fffcc455-6026-4391-a368-e39d54524eb6] VM Resumed (Lifecycle Event)
2017-10-27 03:51:37.194 40200 INFO nova.virt.libvirt.driver [-] [instance: fffcc455-6026-4391-a368-e39d54524eb6] Instance spawned successfully.
```

至此，instance 已经成功启动。 OpenStack 图形界面和 KVM CLI 都可以查看到 instance 的运行状态。

```shell
# virsh list 
 Id    Name                           State
----------------------------------------------------
 14004 instance-0000009b              running
```

在计算节点上，instance 并不是以 OpenStack上的名字命名，而是采用 instance-xxxxx 的格式。