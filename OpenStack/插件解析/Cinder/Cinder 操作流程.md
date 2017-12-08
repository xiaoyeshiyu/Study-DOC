# 准备LVM Volume Provider

Cinder 真正负责 Volume 管理的组件是 volume provider。Cinder 支持多种 volume provider，LVM 是默认的 volume provider。Devstack 安装之后，/etc/cinder/cinder 已经配置好了 LVM，如下图所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206155932.jpg)

上面的配置定义了名为“lvmdriver-1”的 volume provider，也称作 back-end。其 driver 是 LVM，LVM 的 volume group 名为“stack-volumes-lvmdriver-1”。Devstack 安装时并没有自动创建 volume group，所以需要我们手工创建。 如下步骤演示了在 /dev/sdb 上创建 VG “stack-volumes-lvmdriver-1"

1. 首先创建 physical volume /dev/sdb

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206160146.jpg)

   Linux 的 lvm 默认配置不允许在 /dev/sdb 上创建 PV，需要将 sdb 添加到 /etc/lvm.conf 的 filter 中。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206160218.jpg)

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206160249.jpg)

2. 然后创建 VG stack-volumes-lvmdriver-1

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206160332.jpg)

打开 Web GUI，可以看到 OpenStack 已经创建了 Volume Type “lvmdriver-1”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206160412.jpg)

其 Extra Specs volume_backend_name 为 lvmdriver-1

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206160439.jpg)

# Create Volume

Create 操作流程如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206160607.jpg)

1. 客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（cinder-api）发送请求：“帮我创建一个 volume”。
2. API 对请求做一些必要处理后，向 Messaging（RabbitMQ）发送了一条消息：“让 Scheduler 创建一个 volume”。
3. Scheduler（cinder-scheduler）从 Messaging 获取到 API 发给它的消息，然后执行调度算法，从若干计存储点中选出节点 A。
4. Scheduler 向 Messaging 发送了一条消息：“让存储节点 A 创建这个 volume”。
5. 存储节点 A 的 Volume（cinder-volume）从 Messaging 中获取到 Scheduler 发给它的消息，然后通过 driver 在 volume provider 上创建 volume。

因为 Create Volume 操作比较复杂，将分三次讨论：

* 第一部分，cinder-api 的处理过程；
* 第二部分， cinder-scheduler；
* 第三部分， cinder-volume 的操作。

## 向cinder-api发送请求

客户（可以是 OpenStack最终用户，也可以是其他程序）向 cinder-api发送请求：“帮我创建一个 volume。GUI 上操作的菜单为 Project -> Compute -> Volumes -> Create Volume

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206161402.jpg)

设置 volume 的名称，volume type，大小，Availability Zone 等基本信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206161437.jpg)

这里我们没有设置 Volume Source，这样会创建一个空白的 volume。点击“Create Volume” 按钮，cinder-api 将接收到创建 volume 的请求，查看 cinder-api 日志 /opt/stack/logs/c-api.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206161504.jpg)

日志显示 cinder-api 接收到一个 POST 类型的 REST API，经过对 HTTP body 的分析，该请求是：创建一个 1GB 的 volume。

紧接着，cinder-api 启动了一个 Flow（工作流）volume_create_api。 Flow 的执行状态依次为 PENDING, RUNNING 和 SUCCESS。volume_create_api 当前的状态由 PENDING 变为 RUNNING。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206161610.jpg)

volume_create_api 工作流包含若干 Task，每个 Task 完成特定的任务。 这些任务依次为 ExtractVolumeRequestTask, QuotaReserveTask, EntryCreateTask, QuotaCommitTask, VolumeCastTask。 Task 的执行状态也会经历 PENDING, RUNNING 和 SUCCESS 三个阶段。

Task 的名称基本上说明了任务的工作内容，前面几个 Task 主要是做一些创建 volume 的准备工作，比如：

**ExtractVolumeRequestTask** 获取 request 信息

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206161653.jpg)

**QuotaReserveTask** 预留配额

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206162034.jpg)

**EntryCreateTask** 在数据库中创建 volume 条目

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206162130.jpg)

**QuotaCommitTask** 确认配额

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206162527.jpg)

最后 **VolumeCastTask** 是向 cinder-sheduler 发送消息，开始调度工作

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206162556.jpg)

至此，Flow volume_create_api 已经完成，状态由 RUNNING 变为 SUCCESS，volume 创建成功。日志如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206162625.jpg)

需要特别注意的是，“volume 创建成功”只是指 cinder-api 已经成功处理了 volume create 请求，将消息发给了 cinder-scheduler，但并不意味 volume 在存储节点上已经成功创建，这一点是容易引起误解的。我们可以通过 cinder-volume 创建 volume 日志的时间戳验证。

## cinder-api 发送消息

cinder-api 向 RabbitMQ 发送了一条消息：“让cinder-scheduler 创建一个 volume”

前面我们提到消息是由 VolumeCastTask 发出的，因为 VolumeCastTask 没有打印相关日志，我们只能通过源代码查看 `/opt/stack/cinder/cinder/volume/flows/api/create_volume.py `，方法为 create_volume。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206162843.jpg)

## cinder-scheduler 执行调度

cinder-scheduler 执行调度算法，通过 Filter 和 Weigher 挑选最优的存储节点，日志为 /opt/stack/logs/c-sch.log。cinder-scheduler 通过 Flow volume_create_scheduler 执行调度工作。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206163351.jpg)

该 Flow 依次执行 ExtractSchedulerSpecTask 和 ScheduleCreateVolumeTask。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206163420.jpg)

主要的 filter 和 weighting 工作由 ScheduleCreateVolumeTask 完成。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206163505.jpg)

经过 AvailabilityZoneFilter, CapacityFilter, CapabilitiesFilter 和 CapacityWeigher 的层层筛选，最终选择了存储节点 devstack-controller@lvmdriver-1#lvmdriver-1。

Flow volume_create_scheduler 完成调度，状态变为 SUCCESS。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206163532.jpg)

## cinder-scheduler 发送消息

cinder-scheduler 发送消息给 cinder-volume，让其创建 volume。源码 /opt/stack/cinder/cinder/scheduler/filter_scheduler.py，方法为 schedule_create_volume。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206163704.jpg)

## cinder-volume 的处理过程

cinder-volume 通过 driver 创建 volume，日志为 /opt/stack/logs/c-vol.log。

与 cinder-api 和 cinder-scheduler 执行方式类似，cinder-volume 也启动了一个 Flow 来完成 volume 创建工作，Flow 的名称为 volume_create_manager。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171204.jpg)

volume_create_manager 首先执行 ExtractVolumeRefTask, OnFailureRescheduleTask, ExtractVolumeSpecTask, NotifyVolumeActionTask 为 volume 创建做准备。

ExtractVolumeRefTask过程

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171241.jpg)

OnFailureRescheduleTask过程

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171448.jpg)

ExtractVolumeSpecTask过程

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171527.jpg)

NotifyVolumeActionTask过程

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171609.jpg)

接下来 CreateVolumeFromSpecTask 执行 volume 创建任务。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171650.jpg)

因为 volume provider 为 LVM， CreateVolumeFromSpecTask 通过 lvcreate 命令在 VG stack-volumes-lvmdriver-1 中创建了一个 1G 的 LV，cinder-volume 将这个 LV 作为volume。 新创建的 LV 命名为“volume-1e7f6bd7-ce11-4a73-b95e-aabd65a5b188”，其格式为“volume-< volume ID >”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171727.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171743.jpg)

最后，CreateVolumeOnFinishTask 完成扫尾工作。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171812.jpg)

至此，volume 成功创建，Flow volume_create_manager 结束。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171838.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206171854.jpg)

# Attach Volume

Volume 的最主要用途是作为虚拟硬盘提供给 instance 使用。Volume 是通过 Attach 操作挂载到 instance 上的。每个 volume 实际上是存储节点上 VG 中的一个 LV。

通常情况存储节点和计算节点是不同的物理节点。计算节点通过 iSCSI 将存储节点上本地的 LV 如何挂载到本节点的 instance 上

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172152.jpg)

iSCSI 是 Client-Server 架构，有 target 和 initiator 两个术语。

**Target**
提供 iSCSI 存储资源的设备，简单的说，就是 iSCSI 服务器。

**Initiator**
使用 iSCSI 存储资源的设备，也就是 iSCSI 客户端。

Initiator 需要与 target 建立 iSCSI 连接，执行 login 操作，然后就可以使用 target 上面的块存储设备了。 Target 提供的块存储设备支持多种实现方式，实验环境中使用的是 LV。 Cinder 的存储节点 cinder-volume 默认使用 tgt 软件来管理和监控 iSCSI target，在计算节点 nova-compute 使用 iscsiadm 执行 initiator 相关操作。

下面是 Attach 操作的流程图：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172307.jpg)

1. 向 cinder-api 发送 attach 请求
2. cinder-api 发送消息
3. cinder-volume 初始化 volume 的连接
4. nova-compute 将 volume attach 到 instance

## 向 cinder-api 发送 attach 请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 cinder-api 发送请求：“请将这个 volume attach 到指定的 instance 上。”

将 volume “vol-1” attach 到 instance ”c2”上。 attach 操作之前，c2 上的虚拟磁盘如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172403.jpg)

进入 GUI 操作菜单 Project -> Compute -> Volumes

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172426.jpg)

选择 volume “vol-1”，点击“Manage Attachments”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172450.jpg)

在 “Attach to Instance”下拉列表中，选择instance “c2”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172517.jpg)

点击 ”Attach Volume”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172545.jpg)

cinder-api 将接收到 attach volume 的请求，attach 请求实际上包含两个步骤：

1. **初始化 volume 的连接**

   Volume 创建后，只是在 volume provider 中创建了相应存储对象（比如 LV），这时计算节点是无法使用的。Cinder-volume 需要以某种方式将 volume export 出来，计算节点才能够访问得到。这个 export 的过程就是“初始化 volume 的连接”。 下面是 cinder-api 的日志文件 /opt/stack/logs/c-api.log 中记录的相关信息

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172756.jpg)

   Initialize_connection 的具体工作主要由 cinder-volume 完成。

2. **Attach volume**

   初始化 volume 连接后，计算节点将 volume 挂载到指定的 instance，完成 attach 操作。下面是 cinder-api 的日志文件 /opt/stack/logs/c-api.log 中记录的相关信息

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206172933.jpg)

   Attach 的具体工作主要由 nova-compute 完成，也将在后面详细讨论。

## cinder-api 发送消息

cinder-api 分两步完成 attach 操作，所以会先后向 RabbitMQ 发送了两条消息：

1. **初始化 volume 的连接**

   cinder-api 没有打印发送消息的日志，只能通过源代码查看 /opt/stack/cinder/cinder/volume/api.py，方法为 initialize_connection

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173210.jpg)

2. **Attach volume **

   cinder-api 没有打印发送消息的日志，只能通过源代码查看 /opt/stack/cinder/cinder/volume/api.py，方法为 attach

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173308.jpg)

## cinder-volume 初始化 volume 的连接

cinder-volume 接收到 initialize_connection 消息后，会通过 tgt 创建 target，并将 volume 所对应的 LV 通过 target export 出来。日志为 /opt/stack/logs/c-vol.log

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173423.jpg)

下面的日志显示：通过命令 tgtadm --lld iscsi --op show --mode target 看到已经将 1GB（1024MB）的 LV /dev/stack-volumes-lvmdriver-1/volume-1e7f6bd7-ce11-4a73-b95e-aabd65a5b188 通过 Target 1 export 出来了。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173537.jpg)

Initialize connection 完成。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173606.jpg)

## nova-compute 将 volume attach 到 instance

计算节点作为 iSCSI initiator 访问存储节点 Iscsi Target 上的 volume，并将其 attach 到 instance。日志文件为 /opt/stack/logs/n-cpu.log

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173705.jpg)

nova-compute 依次执行 iscsiadm 的 new, update, login, rescan 操作访问 target 上的 volume

new 操作

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173844.jpg)

update 操作

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173907.jpg)

login 操作

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206173922.jpg)

rescan 操作

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174004.jpg)

计算节点将 iSCSI target 上的 volume 识别为一个磁盘文件。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174033.jpg)

然后通过更新 instance 的 XML 配置文件将 volume 映射给 instance。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174141.jpg)

也可以通过 virsh edit查看更新后的 XML。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174222.jpg)

可以看到，instance 增加了一个类型为 block 的虚拟磁盘，source 就是要 attach 的 volume，该虚拟磁盘的设备名为 vdb。

手工 Shut off 并 Start instance，通过 fdisk -l 查看到 volume 已经 attach 上来，设备为 vdb

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174254.jpg)

GUI 界面也会更新相关 attach 信息

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174322.jpg)

现在如果在存储节点执行 tgt-admin --show --mode target，会看到计算节点作为 initiator 已经连接到 target 1。cinder-volume 刚刚创建 target 的时候是没有 initiator 连接的，可以将下面的截图与之前的日志做个对比。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174401.jpg)

# Detach Volume 

下图是 Detach 操作的流程图

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174447.jpg)

1. 向 cinder-api 发送 detach 请求
2. cinder-api 发送消息
3. nova-compute detach volume
4. cinder-volume 删除 target

## 向 cinder-api 发送detach 请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 cinder-api 发送请求：“请 detach 指定 instance 上的 volume。”

这里将 detach instance “c2”上的 volume “vol-1” 进入 GUI 操作菜单Project -> Compute -> Volumes。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174533.jpg)

选择 volume “vol-1”，点击“Manage Attachments”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174605.jpg)

点击 “Detach Volume”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174630.jpg)

再次确认。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174656.jpg)

cinder-api 将接收到 detach volume 的请求。日志文件在 /opt/stack/logs/c-api.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174723.jpg)

## cinder-api 发送消息

cinder-api 发送消息 detach 消息。cinder-api 没有打印发送消息的日志，只能通过源代码查看 /opt/stack/cinder/cinder/volume/api.py，方法为 detach。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206174816.jpg)

Detach 的操作由 nova-compute 和 cinder-volume 共同完成：

1. 首先 nova-compute 将 volume 从 instance 上 detach，然后断开与 iSCSI target 的连接。
2. 最后 cinder-volume 删除 volume 相关的 iSCSI target。

## nova-compute detach volume

nova-compute 首先将 volume 从 instance 上 detach 。日志为 /opt/stack/logs/n-cpu.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175054.jpg)

这时通过 virsh edit可以看到 XML 配置文件中已经不在有 volume 的虚拟磁盘。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175120.jpg)

接下来断开与 iSCSI target 的连接。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175139.jpg)

具体有下面几个步骤：

1. 将缓存中的数据 Flush 到 volume。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175213.jpg)

2. 删除计算节点上 volume 对应的 SCSI 设备。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175437.jpg)

3. 通过 iscsiadm 的 logout，delete 操作断开与 iSCSI target 的连接。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175515.jpg)

compue-nova 完成了 detach 工作，接下来 cinder-volume 就可以删除 volume 相关的 target 了。

## cinder-volume 删除 target 

存储节点 cinder-volume 通过 tgt-admin 命令删除 volume 对应的 target。日志文件为 /opt/stack/logs/c-vol.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175605.jpg)

至此 detach volume 操作已经完成，GUI 也会更新 volume 的 attach 信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206175632.jpg)

# Extend Volume

Extend 操作用于扩大 Volume 的容量，状态为 Available 的 volume 才能够被 extend。如果 volume 当前已经 attach 给 instance，需要先 detach 后才能 extend（为了保护现有数据，cinder 不允许缩小 volume。）。Extend 实现比较简单，流程图如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206180034.jpg)

## 向 cinder-api 发送 extend 请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 cinder-api 发送请求：“请 extend 指定的 volume。”

这里我们将 extend volume “vol-2”。进入 GUI 操作菜单 Project -> Compute -> Volumes。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206180114.jpg)

vol-2 当前大小为 1GB。其在存储节点上对应的 LV 信息如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206180147.jpg)

选择 volume “vol-2”，点击“Extend Volume”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206181146.jpg)

指定新容量为 3GB，点击 “Extend Volume”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206181218.jpg)

cinder-api 将接收到 extend volume 的请求。日志文件在 /opt/stack/logs/c-api.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206181242.jpg)

## cinder-api 发送消息

cinder-api 发送消息 extend 消息。cinder-api 没有打印发送消息的日志，只能通过源代码查看。/opt/stack/cinder/cinder/volume/api.py，方法为 extend。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206181327.jpg)

## cinder-volume extend volume

cinder-volume 执行 lvextend 命令 extend volume。日志为 /opt/stack/logs/c-vol.log

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206181400.jpg)

LV 被 extend 到 3GB。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206181427.jpg)

Extend 操作完成后，GUI 也会更新 volume 的状态信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206182525.jpg)

# Delete Volume

状态为 Available 的 volume 才能够被 delete。 如果 volume 当前已经 attach 到 instance，需要先 detach 后才能 delete。Delete操作实现比较简单，流程图如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206182620.jpg)

1. 向 cinder-api 发送 delete 请求
2. cinder-api 发送消息
3. cinder-volume 执行 delete 操作

## 向 cinder-api 发送 delete 请求 

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 cinder-api 发送请求：“请 delete 指定的 volume。” 这里我们将 delete volume “vol-2”

进入 GUI 操作菜单 Project -> Compute -> Volumes。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206182752.jpg)

选择volume “vol-2”，点击“Delete Volume”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206182822.jpg)

再次确认。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206182845.jpg)

cinder-api 将接收到 delete volume 的请求。日志文件在 /opt/stack/logs/c-api.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206182911.jpg)

## cinder-api 发送消息

cinder-api 发送消息 delete 消息。cinder-api 没有打印发送消息的日志，只能通过源代码查看 /opt/stack/cinder/cinder/volume/api.py，方法为 delete。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206182941.jpg)

## cinder-volume delete volume

cinder-volume 执行 lvremove 命令 delete volume。日志为 /opt/stack/logs/c-vol.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183017.jpg)

这里比较有意思的是：cinder-volume 执行的是“安全”删除。 所谓“安全”实际上就是将 volume 中的数据抹掉，LVM driver 使用的是 dd 操作将 LV 的数据清零。日志如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183050.jpg)

然后删除 LV。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183210.jpg)

# Snapshot Volume

Snapshot 可以为 volume 创建快照，快照中保存了 volume 当前的状态，以后可以通过 snapshot 回溯。snapshot 操作实现比较简单，流程图如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183248.jpg)

1. 向 cinder-api 发送 snapshot 请求
2. cinder-api 发送消息
3. cinder-volume 执行 snapshot 操作

## 向 cinder-api 发送 snapshot 请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 cinder-api 发送请求：“请 snapshot 指定的 volume。

这里我们将 snapshot volume “vol-1”。进入 GUI 操作菜单 Project -> Compute -> Volumes。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183331.jpg)

选择“vol-1”，点击 “Create Snapshot”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183357.jpg)

为 snapshot 命名。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183422.jpg)

这里看到界面提示当前 volume 已经 attach 到某个 instance，创建 snapshot 可能导致数据不一致。可以先 pause instance，或者确认当前 instance 没有大量的磁盘 IO，处于相对稳定的状态，则可以创建 snapshot，否则还是建议先 detach volume 在做 sanpshot。

cinder-api 将接收到 snapshot volume 的请求。日志文件在 /opt/stack/logs/c-api.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183503.jpg)

## cinder-api 发送消息

cinder-api 发送消息 snapshot 消息。cinder-api 没有打印发送消息的日志，只能通过源代码查看 /opt/stack/cinder/cinder/volume/api.py，方法为 _create_snapshot。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183548.jpg)

## cinder-volume 执行 snapshot 操作

cinder-volume 执行 lvcreate 创建 snapshot。日志为 /opt/stack/logs/c-vol.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183627.jpg)

对于 LVM volume provider，snapshot 实际上也是一个 LV，同时记录了与源 LV 的 snapshot 关系，可以通过 lvdisplay 查看。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183656.jpg)

GUI 的 Volume Snapshots 标签中可以看到新创建的 “vol-1-snapshot”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183718.jpg)

有了 snapshot，我们就可以将 volume 回溯到创建 snapshot 时的状态。方法是通过 snapshot 创建新的 volume。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183743.jpg)

新创建的 volume 容量必须大于或等于 snapshot 的容量。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206183816.jpg)

其过程与 Create Volume 类似，不同之处在于 LV 创建之后会通过 dd 将 snapshot 的数据 copy 到新的 volume。

如果一个 volume 存在 snapshot，则这个 volume 是无法删除的。这是因为 snapshot 依赖于 volume，snapshot 无法独立存在。在 LVM 作为 volume provider 的环境中，snapshot 是从源 volume 完全 copy 而来，所以这种依赖关系不强。但在其他 volume provider（比如商业存储设备或者分布式文件系统），snapshot 通常是源 volume 创建快照时数据状态的一个引用（指针），占用空间非常小，在这种实现方式里 snapshot 对源 volume 的依赖就非常明显了。

# Backup Volume

Backup 是将 volume 备份到别的地方（备份设备），将来可以通过 restore 操作恢复。

## Backup VS Snapshot

初看 backup 功能好像与 snapshot 很相似，都可以保存 volume 的当前状态，以备以后恢复。但二者在用途和实现上还是有区别的，具体表现在：0

1. Snapshot 依赖于源 volume，不能独立存在；而 backup 不依赖源 volume，即便源 volume 不存在了，也可以 restore。
2. Snapshot 与源 volume 通常存放在一起，都由同一个 volume provider 管理；而 backup 存放在独立的备份设备中，有自己的备份方案和实现，与 volume provider 没有关系。
3. 上面两点决定了 backup 具有容灾功能；而 snapshot 则提供 volume provider 内便捷的回溯功能。

## 配置 cinder-backup

Cinder 的 backup 功能是由 cinder-backup 服务提供的，devstack 默认没有启用该服务，需要手工启用。与 cinder-volume 类似，cinder-backup 也通过 driver 架构支持多种备份 backend，包括 POSIX 文件系统、NFS、Ceph、GlusterFS、Swift 和 IBM TSM。支持的driver 源文件放在 /opt/stack/cinder/cinder/backup/drivers/

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206184338.jpg)

实验通过 NFS 为 backend 来研究 backup 操作。

在实验环境中，存放 volume backup 的 NFS 远程目录为 192.168.104.11:/backup cinder-backup 服务节点上 mount point 为 /backup_mount。需要在 /etc/cinder/cinder.conf 中相应配置。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206184453.jpg)

然后手工启动 cinder-backup 服务。

``` shell
/usr/bin/python /usr/local/bin/cinder-backup --config-file /etc/cinder/cinder.conf
```

backup 操作的流程

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206184806.jpg)

1. 向 cinder-api 发送 backup 请求
2. cinder-api 发送消息
3. cinder-backup 执行 backup 操作

## 向 cinder-api 发送 backup 请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 cinder-api 发送请求：“请 backup 指定的 volume。” 这里我们将 backup volume “vol-1”，目前 backup 只能在 CLI 中执行。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206185005.jpg)

这里因为 vol-1 已经 attach 到 instance，需要使用 --force 选项。cinder-api 接收到 backup volume 的请求。日志文件在 /opt/stack/logs/c-api.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206185040.jpg)

## cinder-api 发送消息

cinder-api 发送 backup 消息。cinder-api 没有打印发送消息的日志，只能通过源代码查看 /opt/stack/cinder/cinder/backup/api.py，方法为 create。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206185126.png)

## cinder-backup 执行 backup 操作

cinder-backup 收到消息后，通过如下步骤完成 backup 操作，日志为 /opt/stack/logs/c-vol.log。

1. 启动 backup 操作，mount NFS。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206185236.jpg)

2. 创建 volume 的临时快照。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206190810.jpg)

3. 创建存放 backup 的 container 目录。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206190840.jpg)

4. 对临时快照数据进行压缩，并保存到 container 目录。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206190926.jpg)

5. 创建并保存 sha256（加密）文件和 metadata 文件。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191011.jpg)

6. 删除临时快照。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191107.jpg)

里面有三个文件，根据前面的日志可以知道：

1. backup-00001，压缩后的 backup 文件。
2. backup_metadata，metadata 文件。
3. backup_sha256file，加密文件。

可以通过 cinder backup-list 查看当前存在的 backup。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191218.jpg)

另外查看一下 cinder backup-create 的用法。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191237.jpg)

这里有 --incremental 选项，表示可以执行增量备份。 如果之前做过普通（全量）备份，之后可以通过增量备份大大减少需要备份的数据量，是个很不错的功能。

# Restore Volume

restore 的过程其实很简单，两步走：

1. 在存储节点上创建一个空白 volume。
2. 将 backup 的数据 copy 到空白 voluem 上。

下面我们来看 restore 操作的详细流程：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191345.jpg)

1. 向 cinder-api 发送 backup 请求
2. cinder-api 发送消息
3. cinder-scheduler 挑选最合适的 cinder-volume
4. cinder-volume 创建空白 volume
5. cinder-backup 将 backup 数据 copy 到空白 volume 上

## 向 cinder-api 发送 backup 请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 cinder-api 发送请求：“请 restore 指定的 backup。这里我们将 restore 之前创建的 backup。

目前 restore 只能在 CLI 中执行。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191445.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191459.jpg)

cinder-api 接收到 restore 请求。日志文件在 /opt/stack/logs/c-api.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191517.jpg)

这里看到 cinder-api 转发请求，为 restore 创建 volume。 之后 cinder-scheduler 和 cinder-volume 将创建空白 volume，这个过程与 create volume 一样。

接下来分析数据恢复的过程。 首先，在 cinder-api 日志中可看到相关信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191610.jpg)

这里注意日志中的 volume_id 和 backup_id 与前面 backup-restore 命令的输出是一致的。下面来看 cinder-backup 是如何恢复数据的。

## cinder-backup 执行 backup 操作

日志为 /opt/stack/logs/c-vol.log。

1. 启动 restore 操作，mount NFS。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191722.jpg)

2. 读取 container 目录中的 metadata。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206191753.jpg)

3. 将数据解压并写到 volume 中。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192018.jpg)

4. 恢复 volume 的 metadata，完成 restore 操作。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192043.jpg)

此时，在 GUI 中已经可以看到 restore 创建的 volume。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192131.jpg)

以上就是 volume restore 的分析。

# Boot from Volume

Volume 除了可以用作 instance 的数据盘，也可以作为启动盘（Bootable Volume），打开 instance 的 launch 操作界面。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192226.jpg)

这里有一个下拉菜单“Instance Boot Source”。以前 launch instance 要么直接从 image launch（Boot from image），要么从 instance 的 snapshot launch（Boot from snapshot）。

这两种 launch 方式下，instance 的启动盘 vda 均为镜像文件，存放路径为计算节点 /opt/stack/data/nova/instances/< Instance ID >/disk，例如：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192330.jpg)

下拉列表的后三项则可以将 volume 作为 instance 的启动盘 vda，分别为：

**Boot from volume**
直接从现有的 bootable volume launch

**Boot from image (create a new volume)**
创建一个新的 volume，将 image 的数据 copy 到 volume，然后从该 volume launch

**Boot from volume snapshot (create a new volume)**
通过指定的 volume snapshot 创建 volume，然后从该 volume launch，当然前提是该snapshot 对应的源 volume 是 bootable 的。

下面以 Boot from image (create a new volume)为例，看如何从 volume 启动。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192432.jpg)

选择 cirros 作为 image，instance 命名为“c3” ，如果希望 terminate instant 的时候同时删除 volume，可以勾选“Delete on Terminate”

c3 成功 Launch 后，volume 列表中可以看到一个新 bootable volume，以 volume ID 命名，并且已经 attach 到 c3。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192528.jpg)

该 volume 已经配置为 c3 的启动盘 vda。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192556.jpg)

如果用该 volume 创建 snapshot，之后就可以通过 Boot from volume snapshot (create a new volume) 部署新的 instance。

boot from volume 的 instance 也可以执行 live miagrate

前面的实验使用的是 LVM provider，cinder 当然也支持其他 provider。

# NFS Volume Provider

cinder-volume 支持多种 volume provider，前面一直使用的是默认的 LVM，现在增加 NFS volume provider。

虽然 NFS 更多的应用在实验或小规模 cinder 环境，由于性能和缺乏高可用的原因在生产环境中不太可能使用，但是学习 NFS volume provider 的意义在于：

1. 理解 cinder-volume 如何支持多 backend
2. 更重要的，可以理解 cinder-volume，nova-compute 和 volume provider 是如何协同工作，共同为 instance 提供块存储。
3. 举一反三，能够快速理解并接入其他生产级 backend ，比如 Ceph，商业存储等。

下图展示了 cinder、nova 是如何与 NFS volume provider 协调工作的。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206192806.jpg)

**NFS Volume Provider**

就是我们通常说的 NFS Server，提供远程 NFS 目录，NFS Clinet 可以 mount 这些远程目录到本地，然后像使用本地目录一样创建、读写文件以及子目录。

**cinder-volume**

存储节点通过 NFS driver 管理 NFS volume provider 中的 volume，这些 volume 在 NFS 中实际上是一个个文件。

**nova-compute**

计算节点将 NFS volume provider 存放 volume 的目录 mount 到本地，然后将 volume 文件作为虚拟硬盘映射给 instance。

这里有几点需要强调：

1. 在 Cinder 的 driver 架构中，运行 cinder-volume 的存储节点和 Volume Provider 可以是完全独立的两个实体。 cinder-volume 通过 driver 与 Volume Provider 通信，控制和管理 volume。
2. Instance 读写 volume 时，数据流不需要经过存储节点，而是直接对 Volume Provider 中的 volume 进行读写。 正如上图所示，存储节点与 NFS Volume Provider 的连接只用作 volume 的管理和控制（绿色连线）；真正的数据读写，是通过计算节点和 NFS Volume Proiver 之间的连接完成的（紫色连线）。这种设计减少了中间环节，存储节点不直接参与数据传输，保证了读写效率。
3. 其他 Volume Provider（例如 ceph，swift，商业存储等）均遵循这种控制流与数据流分离的设计。

## 配置 NFS Volume Provider

在实验环境中，NFS volume provider 的 NFS 远程目录为 192.168.104.11:/storage

cinder-volume 服务节点上 mount point 为 /nfs_storage。在 /etc/cinder/cinder.conf 中添加 nfs backend

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193050.jpg)

1. enabled_backends = lvmdriver-1,nfs 

   让 cinder-volume 使用 nfs backend


2. [nfs] 中细配置 nfs backend。包括：

   a)	指定存储节点上 /nfs_storage 为 nfs 的 mount point。

   ``` shell
   nfs_mount_point_base = /nfs_storage
   ```

   b)	查看 /etc/cinder/nfs_shares 活动 nfs 共享目录列表。nfs_shares_config = /etc/cinder/nfs_shares，其内容为

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193210.jpg)

   c)	nfs volume driver。

   ``` shell
   volume_driver=cinder.volume.drivers.nfs.NfsDriver
   ```

   d)	设置 volume backend name。

   在 cinder 中需要根据这里的 volume_backend_name 创建对应的 volume type，这个非常重要。 

   ``` shell
   volume_backend_name = nfs
   ```

重启 cinder-volume，cinder service-list 确认 nfs cinder-volume 服务正常工作。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193321.jpg)

创建 nfs volume type。

打开GUI页面Admin -> System -> Volumes -> Volume Types，点击 “Create Volume Type”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193437.jpg)

命名 nfs，点击“Create Volume Type”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193455.jpg)

选择 nfs volume tyep，点击下拉菜单“View Extra Specs”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193531.jpg)

点击“Create”，Key 输入 volume_backend_name ；Value 输入 nfs。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193552.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206193622.jpg)

## 创建 volume

创建 NFS volume 操作方法与 LVM volume 一样，唯一区别是在 volume type 的下拉列表中选择“nfs”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207092624.jpg)

点击“Create Volume”，cinder-api，cinder-scheduler 和 cinder-volume 共同协作创建 volume “nfs-vol-1”。这个流程与 LVM volume 一样。

下面重点分析 cinder-volume 日志，看看 NFS volume provider 是如何创建 volume 的。 日志在 /opt/stack/logs/c-vol.log。

cinder-volume 也会启动 Flow 来完成 volume 创建工作，Flow 的名称为 volume_create_manager。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207092840.jpg)

volume_create_manager 首先执行 ExtractVolumeRefTask, OnFailureRescheduleTask, ExtractVolumeSpecTask, NotifyVolumeActionTask 为 volume创建做准备。然后由 CreateVolumeFromSpecTask 真正创建 volume。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207092938.jpg)

首先 mount 远程 NFS 目录。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093019.jpg)

执行 stat、du 命令检查 NFS 目录。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093117.jpg)

执行 truncate 创建 volume 文件。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093147.jpg)

设置 volume 文件为可读写。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093231.jpg)

create 操作完成。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093300.jpg)

Volume 在 NFS 上以文件存在，命名为“volume-< volume ID >”。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093335.jpg)

volume 列表中可以看到新创建的 volume。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093356.jpg)

# attach to instance

通过日志分析，nova-compute 会将存放 volume 文件的 NFS 目录 mount 到本地 /opt/stack/data/nova/mnt 目录下，然后修改 instance 的 XML 将 volume 文件配置为虚拟磁盘，日志为 /opt/stack/logs/n-cpu.log

通过 findmnt 和 mkdir 测试和创建 mount 点。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093710.jpg)

mount NFS 目录。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093738.jpg)

更新 instance 的 XML 配置文件，将 volume 文件映射给 instance。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093826.jpg)

也可以通过 virsh edit查看更新后的XML。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093849.jpg)

GUI 界面也会更新相关 attach 信息。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171207093915.jpg)

# 小结

Cinder 作为 OpenStack 的块存储服务，为 instance 提供虚拟磁盘。 本章我们首先学习了 Cinder 的架构，然后讨论了 Cinder 的各个服务组件，最后通过使用场景详细分析了 Volume 的各种操作。

操作中的详细日志和截图可以帮助我们更好地理解 Cinder 内部运行机制，并为故障分析提供了非常有益的线索。


















































































































