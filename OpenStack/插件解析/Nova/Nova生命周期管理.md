# Launch和Shut Off操作

详细分析 instance launch 和 shut off 操作，以及如何在日志中快速定位有用信息的技巧。（以下试验环境为DevStack环境）

## Launch

Launch instance 应该算 Nova 最重要的操作。

仔细研究 lanuch 操作可以充分理解 Nova 各个子服务的协调配合和运行机制。先回顾一下流程。

![Nova Launch](http://oydlbqndl.bkt.clouddn.com/Nova Launch.jpg)

1. 客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我创建一个 Instance”
2. API对请求做一些必要处理后，向 Messaging（RabbitMQ）发送了一条消息：“让 Scheduler 创建一个 Instance”
3. Scheduler（nova-scheduler）从 Messaging 获取到 API 发给它的消息，然后执行调度算法，从若干计算节点中选出节点 A。
4. Scheduler 向 Messaging 发送了一条消息：“在计算节点 A 上创建这个 Instance”
5. 计算节点 A 的 Compute（nova-compute）从 Messaging 中获取到 Scheduler 发给它的消息，然后通过本节点的 Hypervisor Driver 创建 Instance。
6. 在 Instance 创建的过程中，Compute 如果需要查询或更新数据库信息，会通过 Messaging 向 Conductor（nova-conductor）发送消息，Conductor 负责数据库访问。

## Shut Off

shut off instance 的流程图

![shut off instance](http://oydlbqndl.bkt.clouddn.com/Nova shut off instance.jpg)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

### 向 nova-api 发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我关闭这个 Instance”

查看日志` /opt/stack/logs/n-api.log`

![nova-api-log](http://oydlbqndl.bkt.clouddn.com/nova-api-log.jpg)

如何在日志文件中快速查找到有用的信息，这里给大家几个小窍门：

1. 先确定大的范围，比如在操作之前用` tail -f` 打印日志文件，这样需要查看的日志肯定在操作之后的打印输出的这些内容里。 另外也可以通过时间戳来确定需要的日志范围。

2. 利用 “代码模块” 快速定位有用的信息。 nova-* 子服务都有自己特定的代码模块：

   **nova-api**

   nova.api.openstack.compute.servers

   nova.compute.api

   nova.api.openstack.wsgi

   **nova-compute**

   nova.compute.managernova.virt.libvirt.

   **nova-scheduler**

   nova.scheduler.*

3. 利用 Request ID 查找相关的日志信息。 在上面的日志中个，可以利用` “req-1758b389-a2d0-44cc-a95a-6f75e4dc07fd”` 这个 Request ID 快速定位 n-api.log 中相与 shut off 操作的其他日志条目。 需要补充说明的是，Request ID 是跨日志文件的，这一个特性能在其他子服务的日志文件中定位到相关信息。

### nova-api发送请求

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“关闭这个 Instance”。nova-api 没有将发送消息的操作记录到日志中，不过我们可以通过查看源代码来验证。日志记录了需要查看的源代码在` /opt/stack/nova/nova/compute/api.py `的 1977 行，方法是` force_stop`。

![force_stop](http://oydlbqndl.bkt.clouddn.com/force_stop.jpg)

`force_stop` 方法最后调用的是对象` self.compute_rpcapi` 的` stop_instance `方法。 在 OpenStack 源码中，以 `xxx_rpcapi `命名的对象，表示的就是 xxx 的消息队列。 `xxx_rpcapi.yyy() `方法则表示向 xxx 的消息队列发送 yyy 操作的消息。所以 `self.compute_rpcapi.stop_instance()` 的作用就是向 RabbitMQ 上 `nova-compute` 的消息队列里发送一条 `stop instance `的消息。

这里补充说明一下： 关闭 instance 的前提是 instance 当前已经在某个计算节点上运行，所以这里不需要 nova-scheduler 再挑选合适的节点，这个跟 launch 操作不同。

### nova-compute执行操作

查看计算节点上的日志 /opt/stack/logs/n-cpu.log

![nova-compute-api](http://oydlbqndl.bkt.clouddn.com/nova-compute-api.jpg)

这里利用了 Request ID “`req-1758b389-a2d0-44cc-a95a-6f75e4dc07fd`” 在 n-cpu.log 中快速定位到` nova-compute `关闭 instance 的日志条目。

### 小结

分析某个操作时，首先要理清该操作的内部流程，然后再到相应的节点上去查看日志。例如shut off 的流程为：

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

1，2 两个步骤是在控制节点上执行的，查看 nova-api 的日志。 第 3 步是在计算节点上执行的，查看 nova-compute 的日志。

# Start Instance操作详解

本节通过日志文件详细分析 instance start 操作。 start instance 的流程图：

![Nova instance start](http://oydlbqndl.bkt.clouddn.com/nova instance start.jpg)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作 

## 向 nova-api 发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向API（nova-api）发送请求：“帮我启动这个 Instance”

查看日志 /opt/stack/logs/n-api.log

![Nova instance start api](http://oydlbqndl.bkt.clouddn.com/Nova instance start-api.jpg)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“启动这个 Instance”。查看源代码 `/opt/stack/nova/nova/compute/api.py `的 2002 行，方法是` start`。

![nova api start instance](http://oydlbqndl.bkt.clouddn.com/nova api start instance.jpg)

`self.compute_rpcapi.start_instance()` 的作用就是向 RabbitMQ 上 nova-compute 的消息队列里发送一条 start instance 的消息。

## nova-compute 执行操作

查看日志 /opt/stack/logs/n-cpu.log

开始启动

![nova-compute-start-instance](http://oydlbqndl.bkt.clouddn.com/nova-compute-start-instance.jpg)

准备虚拟网卡

![nova compute start instance 准备网卡](http://oydlbqndl.bkt.clouddn.com/nova compute start instance 准备网卡.jpg)

准备 instance 的 XML 文件

![准备 instance 的 XML 文件](http://oydlbqndl.bkt.clouddn.com/准备 instance 的 XML 文件.jpg)

准备 instance 镜像文件

![准备 instance 镜像文件](http://oydlbqndl.bkt.clouddn.com/准备 instance 镜像文件.jpg)

成功启动

![成功启动](http://oydlbqndl.bkt.clouddn.com/成功启动.jpg)

# Reboot

reboot分为soft reboot和hard reboot，它们的区别在于：

* soft reboot 只是重启操作系统，整个过程中，instance 依然处于运行状态。相当于在虚拟机中执行 reboot 命令
* hard reboot 是重启 instance，相当于关机之后再开机

对比而言，可以理解为挂载块设备这一项，软重启不会卸载再挂载块设备，而硬重启会。

# Lock

为了避免误操作，比如意外重启或删除 instance，可以将 instance  加锁。

对被加锁（Lock）的 instance 执行重启等改变状态的操作会提示操作不允许。执行解锁（Unlock）操作后恢复正常。Lock/Unlock 操作都是在 nova-api 中进行的。 操作成功后 nova-api 会更新 instance 加锁的状态。 执行其他操作时，nova-api 根据加锁状态来判断是否允许。

Lock/Unlock 不需要 nova-compute 的参与。

* admin 角色的用户不受 lock 的影响，及无论加锁与否都可以正常执行操作。
* 根据默认 policy 的配置，任何用户都可以 unlock。也就是说如果发现 instance 被加锁了，可以通过 unlock 解锁，然后再执行操作。

# Terminate

Terminate 操作就是删除 instance，下面是 terminate instance 的流程图：

![](http://oydlbqndl.bkt.clouddn.com/640.webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

## 向 nova-api 发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我删除这个 Instance”

查看日志 `/opt/stack/logs/n-api.log`

![](http://oydlbqndl.bkt.clouddn.com/640 (1).webp)

## nova-api 发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“删除这个 Instance”。源代码在 `/opt/stack/nova/nova/compute/api.py`，方法是 _do_force_delete。

![](http://oydlbqndl.bkt.clouddn.com/640 (2).webp)

## nova-compute执行操作

查看日志 /opt/stack/logs/n-cpu.log

关闭 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (3).webp)

删除 instance 的镜像文件

![](http://oydlbqndl.bkt.clouddn.com/640 (4).webp)

释放虚拟网络等其他资源

![](http://oydlbqndl.bkt.clouddn.com/640 (5).webp)

# Pause 

有时需要短时间暂停 instance，可以通过 Pause 操作将 instance 的状态保存到宿主机的内存中。当需要恢复的时候，执行 Resume 操作，从内存中读回 instance 的状态，然后继续运行 instance。

![](http://oydlbqndl.bkt.clouddn.com/640 (6).webp)



1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

## 向 nova-api 发送请求 

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我暂停这个 Instance”

查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/640 (7).webp)

注：对于 Pause 操作，日志没有前面 Start 记录得那么详细，例如这里就没有记录` nova.api.openstack.compute.servers `和 `nova.compute.api `代码模块的日志。

## nova-api 发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“暂停这个 Instance”。查看源代码 `/opt/stack/nova/nova/compute/api.py`，方法是 pause。

![](http://oydlbqndl.bkt.clouddn.com/640 (8).webp)

## nova-compute 执行操作

查看日志 /opt/stack/logs/n-cpu.log

![](http://oydlbqndl.bkt.clouddn.com/640 (9).webp)

暂停操作成功执行后，instance 的状态变为 Paused。

# Suspend/Resume

有时需要长时间暂停 instance，可以通过 Suspend 操作将 instance 的状态保存到宿主机的磁盘上。需要恢复的时候，执行 Resume 操作，从磁盘读回 instance 的状态，然后继续运行。

这里需要对 Suspend 和 Pause 做个比较：

**相同点**

两者都是暂停 instance 的运行，并保存当前状态，之后可以通过 Resume 操作恢复。

**不同点**

* Suspend 将 instance 的状态保存在磁盘；Pause 是保存在内存中，所以 Resume 被 Pause 的 instance 要比 Suspend 快。
* instance 被 Suspend 后，状态为 Shut Down；而被 Pause 的 instance 状态是 Paused。
* 虽然都是通过 Resume 操作恢复，Pause 对应的 Resume 在 OpenStack 内部被叫作 “Unpause”；Suspend 对应的 Resume 才是真正的 “Resume”。这个在日志中能体现出来。

# Rescue/Unrescue

有时候由于误操作或者突然断电，操作系统重启后却起不来了。 为了最大限度挽救数据，我们通常会使用一张系统盘将系统引导起来，然后在尝试恢复。 问题如果不太严重，完全可以通过这种方式让系统重新正常工作。 比如某个系统文件意外删除， root 密码遗忘等。Nova 也提供这种故障恢复机制，叫 Rescue。 我们来看看 rescue 的说明：

![](http://oydlbqndl.bkt.clouddn.com/640 (10).webp)

Rescue 用指定的 image 作为启动盘引导 instance，将 instance 本身的系统盘作为第二个磁盘挂载到操作系统上。下面是 rescue instance 的流程图

![](http://oydlbqndl.bkt.clouddn.com/640 (11).webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

## 向nova-api发送请求

目前 Rescue 操作只能通过 CLI 执行

![](http://oydlbqndl.bkt.clouddn.com/640 (12).webp)

这里没有指明用哪个 image 作为引导盘，nova 将使用 instance 部署时使用的 image。查看日志` /opt/stack/logs/n-api.log`

![](http://oydlbqndl.bkt.clouddn.com/640 (13).webp)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“Rescue 这个 Instance”。源代码在`/opt/stack/nova/nova/compute/api.py`，方法是 rescue。

![](http://oydlbqndl.bkt.clouddn.com/640 (14).webp)

## nova-compute执行操作

查看日志 /opt/stack/logs/n-cpu.log

关闭 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (15).webp)

![](http://oydlbqndl.bkt.clouddn.com/640 (16).webp)

通过 image 创建新的引导盘，命名为 disk.rescue

![](http://oydlbqndl.bkt.clouddn.com/640.png)

启动 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (17).webp)

Rescue 执行成功后，可通过 virsh edit < instance_name >查看 instance 的 XML 定义，disk.rescue 作为启动盘 vda，真正的启动盘 disk 作为第二个磁盘 vdb。

![](http://oydlbqndl.bkt.clouddn.com/640 (18).webp)

登录 instance，通过 fdisk 也可确认。

![](http://oydlbqndl.bkt.clouddn.com/640 (19).webp)

此时，instance 处于 Rescue 状态

![](http://oydlbqndl.bkt.clouddn.com/640 (20).webp)

Rescue 操作给我们机会修复损坏的操作系统。 修好之后，使用 Unrescue 操作从原启动盘重新引导 instance。

![](http://oydlbqndl.bkt.clouddn.com/640 (21).webp)

# snapshot

有时候操作系统损坏得很严重，通过 Rescue 操作无法修复，那么就得考虑通过备份恢复了。当然前提是之前对instance做过备份。Nova 备份的操作叫 Snapshot，其工作原理是对 instance 的镜像文件（系统盘）进行全量备份，生成一个类型为 snapshot 的 image，然后将其保存到 Glance 上。从备份恢复的操作叫 Rebuild。

![](http://oydlbqndl.bkt.clouddn.com/640 (22).webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

## 向nova-api发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“对这个 Instance 做个快照”

![](http://oydlbqndl.bkt.clouddn.com/640 (23).webp)

查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/640 (24).webp)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“对这个 Instance 做快照”。源代码在 `/opt/stack/nova/nova/compute/api.py`，方法是 snapshot。

![](http://oydlbqndl.bkt.clouddn.com/640 (25).webp)

## nova-compute执行操作

查看日志 /opt/stack/logs/n-cpu.log，暂停 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (26).webp)

对 instance 的镜像文件做快照

![](http://oydlbqndl.bkt.clouddn.com/640 (27).webp)

恢复 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (28).webp)

![](http://oydlbqndl.bkt.clouddn.com/640 (29).webp)

将快照上传到 Glance

![](http://oydlbqndl.bkt.clouddn.com/640 (30).webp)

![](http://oydlbqndl.bkt.clouddn.com/640 (31).webp)

Snapshot 成功保存在 Glance 中

![](http://oydlbqndl.bkt.clouddn.com/640 (32).webp)

instance 备份成功。

# Rebuild

snapshot 的一个重要作用是对 instance 做备份。如果 instance 损坏了，可以通过 snapshot 恢复，这个恢复的操作就是 Rebuild。Rebuild 会用 snapshot 替换 instance 当前的镜像文件，同时保持 instance 的其他诸如网络，资源分配属性不变。下面是 rebuild instance 的流程图

![](http://oydlbqndl.bkt.clouddn.com/640 (33).webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

## 向nova-api发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“Rebuild 这个 Instance”，选择用于恢复的 image

![](http://oydlbqndl.bkt.clouddn.com/640 (34).webp)

查看日志/opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/640 (35).webp)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“Rebuild 这个 Instance”。源代码在 `/opt/stack/nova/nova/compute/api.py`，方法是 rebuild。

![](http://oydlbqndl.bkt.clouddn.com/640 (36).webp)

## nova-compute执行操作

查看日志 /opt/stack/logs/n-cpu.log，关闭 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (37).webp)

![](http://oydlbqndl.bkt.clouddn.com/640 (38).webp)

下载新的 image，并准备 instance 的镜像文件

![](http://oydlbqndl.bkt.clouddn.com/640 (39).webp)

![](http://oydlbqndl.bkt.clouddn.com/640 (41).webp)

启动 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (42).webp)

![](http://oydlbqndl.bkt.clouddn.com/640 (43).webp)

Rebuild 后，GUI 显示 instance 已经使用新的 image

![](http://oydlbqndl.bkt.clouddn.com/640 (44).webp)

# Shelve

Instance 被 Suspend 后虽然处于 Shut Down 状态，但 Hypervisor 依然在宿主机上为其预留了资源，以便在以后能够成功 Resume。如果希望释放这些预留资源，可以使用 Shelve 操作。 Shelve 会将 instance 作为 image 保存到 Glance 中，然后在宿主机上删除该 instance。 下面是 shelve instance 的流程图

![](http://oydlbqndl.bkt.clouddn.com/640 (45).webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

## 向nova-api发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向API（nova-api）发送请求：“帮我 shelve 这个 Instance”，查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/640 (46).webp)

## 向nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“shelve 这个 Instance”。查看源代码 /opt/stack/nova/nova/compute/api.py，方法是 shelve。

![](http://oydlbqndl.bkt.clouddn.com/640 (47).webp)

## nova-compute执行操作

查看日志 /opt/stack/logs/n-cpu.log。首先，关闭 instance

![](http://oydlbqndl.bkt.clouddn.com/640 (48).webp)

然后对 instance 执行 snapshot 操作

![](http://oydlbqndl.bkt.clouddn.com/640 (49).webp)

成功后，snapshot 生成的 image 会保存在 Glance 上，命名为 < instance name >-shelved，最后删除 instance 在宿主机上的资源

![](http://oydlbqndl.bkt.clouddn.com/640 (50).webp)

暂停操作成功执行后，instance 的状态变为 Shelved Offloaded，电源状态是 Shut Down

![](http://oydlbqndl.bkt.clouddn.com/640 (51).webp)

# unshelve

因为 Glance 中保存了 instance 的 image，unshelve 的过程其实就是通过该 image launch 一个新的 instance，nova-scheduler 也会调度合适的计算节点来创建该 instance。instance unshelve 后可能运行在与 shelve 之前不同的计算节点上，但 instance 的其他属性（比如 flavor，IP 等）不会改变。下面是 Unshelve instance 的流程图

![](http://oydlbqndl.bkt.clouddn.com/640 (52).webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-scheduler 执行调度
4. nova-scheduler 发送消息
5. nova-compute 执行操作

## 向nova-api发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我 Unshelve 这个 Instance”，查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/640 (53).webp)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“unshelve 这个 Instance”。查看源代码 /opt/stack/nova/nova/compute/api.py，方法是 unshelve。

![](http://oydlbqndl.bkt.clouddn.com/640 (54).webp)

## nova-scheduler执行调度

nova-scheduler 收到消息后，会为 instance 选择合适的计算节点。查看日志 /opt/stack/logs/n-sch.log

![](http://oydlbqndl.bkt.clouddn.com/640 (55).webp)

![](http://oydlbqndl.bkt.clouddn.com/640 (56).webp)

经过筛选，最终 devstack-controller 被选中 launch instance。

## nova-scheduler发送消息

nova-scheduler 发送消息，告诉被选中的计算节点可以 launch instance 了

源代码在 `/opt/stack/nova/nova/scheduler/filter_scheduler.py `第 95 行，方法为 select_destinations

![](http://oydlbqndl.bkt.clouddn.com/640 (57).webp)

## nova-compute执行操作

nova-compute 执行 unshelve 的过程与 launch instance 非常类似。

一样会经过如下几个步骤：

1. 为 instance 准备 CPU、内存和磁盘资源
2. 创建 instance 镜像文件
3. 创建 instance 的 XML 定义文件
4. 创建虚拟网络并启动 instance

日志记录在 /opt/stack/logs/n-cpu.log。

# Migrate

Migrate 操作的作用是将 instance 从当前的计算节点迁移到其他节点上。Migrate 不要求源和目标节点必须共享存储，当然共享存储也是可以的。 Migrate 前必须满足一个条件：计算节点间需要配置 nova 用户无密码访问。下面是 Migrate instance 的流程图：

![](http://oydlbqndl.bkt.clouddn.com/640 (58).webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-scheduler 执行调度
4. nova-scheduler 发送消息
5. nova-compute 执行操作

## 向nova-api发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我迁移这个 Instance”。Migrate 操作是特权操作，只能在 Admin 的 instance 菜单中执行

![](http://oydlbqndl.bkt.clouddn.com/640 (59).webp)

查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/640 (60).webp)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“迁移这个 Instance”，查看源代码 `/opt/stack/nova/nova/compute/api.py`，方法是 resize。 **是 resize 而非 migrate**。这是由于 migrate 实际上是通过 resize 操作实现的。

![](http://oydlbqndl.bkt.clouddn.com/640 (61).webp)

## nova-scheduler执行调度

nova-scheduler 收到消息后，会为 instance 选择合适的目标计算节点。查看日志 /opt/stack/logs/n-sch.log

![](http://oydlbqndl.bkt.clouddn.com/640 (62).webp)

可以看到，因为 devstack-compute1 的权值比 devstack-controller 大，最终选择 devstack-compute1 作为目标节点。在分析这段日志的时候， scheduler 选出来的计算节点有可能是当前节点源节点！ 因为 scheduler 并没在初始的时候将源节点剔除掉，而是与其他节点放在一起做 filter，按照这个逻辑，只要源节点的权值足够大，是有可能成为目标节点的。那么问题来了：如果源节点和目标节点是同一个，migrate 操作会怎样进行呢？

实验得知，nova-compute 在做 migrate 的时候会检查目标节点，如果发现目标节点与源节点相同，会抛出 UnableToMigrateToSelf 异常。Nova-compute 失败之后，scheduler 会重新调度，由于有 RetryFilter，会将之前选择的源节点过滤掉，这样就能选到不同的计算节点了。  RetryFilter是nova-scheduler选择计算节点中的一种过滤条件。

在上面的操作中 sheduler 选择的目标节点是 devstack-compute1，意味着 instance 将从 devstack-controller 迁移到 devstack-compute1。

## nova-scheduler发送消息

nova-scheduler 发送消息，通知计算节点可以迁移 instance 了。源代码在 /opt/stack/nova/nova/scheduler/filter_scheduler.py 第 95 行，方法为 select_destinations

![](http://oydlbqndl.bkt.clouddn.com/640 (63).webp)

## nova-compute执行操作

nova-compute 会在源计算节点和目标计算节点上分别执行操作。

### 源计算节点

迁移操作在源节点上首先会关闭 instance，然后将 instance 的镜像文件传到目标节点上。日志在 `/opt/stack/logs/n-cpu.log`，具体步骤如下：

**开始migrate**

![](http://oydlbqndl.bkt.clouddn.com/640 (64).webp)

**在目标节点上创建 instance 的目录**

nova-compute 首先会尝试通过 ssh 在目标节点上的 instance 目录里 touch 一个临时文件，日志如下

![](http://oydlbqndl.bkt.clouddn.com/640 (65).webp)

如果 touch 失败，说明目标节点上还没有该 instance 的目录，也就是说，源节点和目标节点没有共享存储。那么接下来就要在目标节点上创建 instance 的目录，日志如下：

![](http://oydlbqndl.bkt.clouddn.com/640 (66).webp)

**关闭 instance**

![](http://oydlbqndl.bkt.clouddn.com/640 (68).webp)

**将 instance 镜像文件通过 scp 传到目标节点**

![](http://oydlbqndl.bkt.clouddn.com/640 (69).webp)

### 目标计算节点

在目标节点上启动 instance，过程与 launch instance 非常类似。会经过如下几个步骤：

1.	为 instance 准备 CPU、内存和磁盘资源
  2.创建 instance 镜像文件
  3.创建 instance 的 XML 定义文件
  4.创建虚拟网络并启动 instance

日志记录在 /opt/stack/logs/n-cpu.log

### Confirm

这时，instance 会处于 “Confirm or Revert Resize/Migrate”状态，需要用户确认或者回退当前的迁移操作，实际上给用户一个反悔的机会。

![](http://oydlbqndl.bkt.clouddn.com/640 (70).webp)

当按下 Confirm 按钮后，会发生如下事情：

1. nova-api 接收到 confirm 的消息

   ![](http://oydlbqndl.bkt.clouddn.com/640 (71).webp)

2. 源计算节点删除 instance 的目录，并在 Hypervisor 上删除 instance。

   ![](http://oydlbqndl.bkt.clouddn.com/640 (72).webp)

   ![](http://oydlbqndl.bkt.clouddn.com/640 (73).webp)

3. 目标计算节点不需要做任何事情

### Revert

![](http://oydlbqndl.bkt.clouddn.com/640 (74).webp)

1. nova-api 接收到 revert 的消息

   ![](http://oydlbqndl.bkt.clouddn.com/640 (75).webp)

2. 在目标计算节点上关闭 instance，删除 instance 的目录，并在 Hypervisor 上删除 instance。

   ![](http://oydlbqndl.bkt.clouddn.com/640 (76).webp)

   ![](http://oydlbqndl.bkt.clouddn.com/640 (77).webp)

3. 源计算节点上启动 instance 因为之前迁移的时候只是在源节点上关闭了该 instance，revert 操作只需重新启动 instance。

   ![](http://oydlbqndl.bkt.clouddn.com/640 (78).webp)

   ![](http://oydlbqndl.bkt.clouddn.com/640 (79).webp)

以上是 Migrate 操作的完整流程，有一点需要特别注意：

迁移过程中源和目标节点之前需要使用 ssh 和 scp，为了使操作顺利进行，必须要保证 nova-compute 进程的启动用户（通常是 nova，也可能是 root，可以通过 ps 命令确认）能够在计算节点之间无密码访问。否则 nova-compute 会等待密码输入，但后台服务是无法输入密码的，迁移操作会一直卡在那里。

# Resize

Resize 的作用是调整 instance 的 vCPU、内存和磁盘资源。Instance 需要多少资源是定义在 flavor 中的，resize 操作是通过为 instance 选择新的 flavor 来调整资源的分配。有了前面对 Migrate 的分析，再来看 Resize 的实现就非常简单了。 因为 instance 需要分配的资源发生了变化，在 resize 之前需要借助 nova-scheduler 重新为 instance 选择一个合适的计算节点，如果选择的节点与当前节点不是同一个，那么就需要做 Migrate。

所以本质上讲：Resize 是在 Migrate 的同时应用新的 flavor。 Migrate 可以看做是 resize 的一个特例： flavor 没发生变化的 resize，这也是为什么在Migrate日志中看到 migrate 实际上是在执行 resize 操作。

下面是 Resize instance 的流程图

![](http://oydlbqndl.bkt.clouddn.com/640 (80).webp)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-scheduler 执行调度
4. nova-scheduler 发送消息
5. nova-compute 执行操作

Resize 分两种情况：

1. nova-scheduler 选择的目标节点与源节点是不同节点。操作过程跟上一节 Migrate 几乎完全一样，只是在目标节点启动 instance 的时候按新的 flavor 分配资源。 同时，因为要跨节点复制文件，也必须要保证 nova-compute 进程的启动用户（通常是 nova，也可能是 root，可以通过 ps 命令确认）能够在计算节点之间无密码访问。 
2. 目标节点与源节点是同一个节点。则不需要 migrate。下面重点讨论这一种情况。

## 向nova-api发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向 API（nova-api）发送请求：“帮我 Resize 这个 Instance”

选择新的 flavor

![](http://oydlbqndl.bkt.clouddn.com/640 (81).webp)

点击 Resize 按钮，查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/640 (82).webp)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“Resize 这个 Instance”。查看源代码 `/opt/stack/nova/nova/compute/api.py`，方法是 resize_instance。

![](http://oydlbqndl.bkt.clouddn.com/640 (83).webp)

## nova-scheduler执行调度

nova-scheduler 收到消息后，会为 instance 选择合适的目标计算节点。查看日志 /opt/stack/logs/n-sch.log

![](http://oydlbqndl.bkt.clouddn.com/640 (84).webp)

在本例中，nova-scheduler 选择了 devstack-compute1 作为的目节点，与源节点相同。

## nova-scheduler发送消息

nova-scheduler 发送消息，通知计算节点可以迁移 instance 了。源代码在 /opt/stack/nova/nova/scheduler/filter_scheduler.py 第 95 行，方法为 select_destinations

![](http://oydlbqndl.bkt.clouddn.com/640 (85).webp)

## nova-compute执行操作

在目标节点上启动 instance，过程与 launch instance 非常类似。日志记录在 /opt/stack/logs/n-cpu.log，会经过如下几个步骤：

1. 按新的 flavor 为 instance 准备 CPU、内存和磁盘资源

   ![](http://oydlbqndl.bkt.clouddn.com/640 (86).webp)

2. 关闭instance

   ![](http://oydlbqndl.bkt.clouddn.com/640 (87).webp)

3. 创建instance镜像文件

   ![](http://oydlbqndl.bkt.clouddn.com/640 (88).webp)

4. 将 instance 的目录备份一份，命名为_resize，以便 revert。

   ![](http://oydlbqndl.bkt.clouddn.com/640 (89).webp)

5. 创建 instance 的 XML 定义文件

   ![](http://oydlbqndl.bkt.clouddn.com/640 (90).webp)

6. 准备虚拟网络

   ![](http://oydlbqndl.bkt.clouddn.com/640 (91).webp)

7. 启动instance

   ![](http://oydlbqndl.bkt.clouddn.com/640 (92).webp)

## Confirm

这时，instance 的状态处于“Confirm or Revert Resize/Migrate”状态，需要用户确认或者回退当前的迁移操作，实际上给了用户一个反悔的机会。

![](http://oydlbqndl.bkt.clouddn.com/640 (94).webp)

当按下 Confirm 按钮后，会发生如下事情：

1. nova-api 接收到 confirm 的消息

   ![](http://oydlbqndl.bkt.clouddn.com/640 (95).webp)

2. 删除计算节上备份的 instance 目录 < instance_id >_resize

   ![](http://oydlbqndl.bkt.clouddn.com/640 (96).webp)

   ![](http://oydlbqndl.bkt.clouddn.com/640 (97).webp)

## Revert

反过来，执行 Revert 操作可以看到

![](http://oydlbqndl.bkt.clouddn.com/640 (98).webp)

1. nova-api 接收到 revert 的消息

   ![](http://oydlbqndl.bkt.clouddn.com/640 (99).webp)

2. 在计算节点上关闭 instance

   ![](http://oydlbqndl.bkt.clouddn.com/640 (100).webp)

3. 通过备份目录 < instance_id >_resize 恢复 instance 目录

   ![](http://oydlbqndl.bkt.clouddn.com/640（101）.webp)

4. 重新启动 instance

   ![](http://oydlbqndl.bkt.clouddn.com/2017年11月22日102.webp)

以上是 Resize 操作的详细分析。

# Live Migrate

Migrate 操作会先将 instance 停掉，也就是所谓的“冷迁移”。而 Live Migrate 是“热迁移”，也叫“在线迁移”，instance不会停机。

Live Migrate 分两种：

1. 源和目标节点没有共享存储，instance 在迁移的时候需要将其镜像文件从源节点传到目标节点，这叫做 Block Migration（块迁移）
2. 源和目标节点共享存储，instance 的镜像文件不需要迁移，只需要将 instance 的状态迁移到目标节点。

源和目标节点需要满足一些条件才能支持 Live Migration：

1. 源和目标节点的 CPU 类型要一致。

2. 源和目标节点的 Libvirt 版本要一致。

3. 源和目标节点能相互识别对方的主机名称，比如可以在 /etc/hosts 中加入对方的条目。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206094135.jpg)

4. 在源和目标节点的 /etc/nova/nova.conf 中指明在线迁移时使用 TCP 协议。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206094322.jpg)

5. Instance 使用 config driver 保存其 metadata。在 Block Migration 过程中，该 config driver 也需要迁移到目标节点。由于目前 libvirt 只支持迁移 vfat 类型的 config driver，所以必须在 /etc/nova/nova.conf 中明确指明 launch instance 时创建 vfat 类型的 config driver。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206094817.jpg)

6. 源和目标节点的 Libvirt TCP 远程监听服务得打开，需要在下面两个配置文件中做一点配置。

   ``` shell
   # cat /etc/default/libvirt-bin
   start_libvirtd = "yes"
   libvirtd_opts = "-d -l"
   ```

   ``` shell
   # cat /etc/libvirt/libvirtd.conf
   listen_tls = 0
   listen_tcp = 1
   unix_sock_group = "libvirtd"
   unix_sock_ro_perms = "0777"
   unix_sock_rw_perms = "0770"
   auth_unix_ro = "none"
   auth_unix_rw = "none"
   auth_tcp = "none"
   ```

   然后重启 Libvirtd 服务

   ``` shell
   # service libvirt-bin restart
   ```

## 非共享存储 Blocl Migration

先讨论非共享存储的 Block Migration。流程图如下：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206100537.jpg)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-compute 执行操作

### 向nova-api发送请求

客户（可以是 OpenStack 最终用户，也可以是其他程序）向API（nova-api）发送请求：“帮我将这个 Instance 从节点 A Live Migrate 到节点 B”

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206100636.jpg)

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206100657.jpg)

这里源节点是 devstack-compute1，目标节点是 devstack-controller，因为是非共享存储，记得将“Block Migration”勾选上。

这里还有一个“Disk Over Commit”选项，如果勾选了此选项，nova 在检查目标节点的磁盘空间是否足够时，是以 instance 磁盘镜像文件定义的最大容量为准；否则，以磁盘镜像文件当前的实际大小为准。

查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206101155.jpg)

### nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“Live Migrate 这个 Instance”。源代码在 `/opt/stack/nova/nova/compute/api.py`，方法是 live_migrate。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206101253.jpg)

### nova-compute执行操作

源和目标节点执行 Live Migrate 的操作过程如下：

1. 目标节点执行迁移前的准备工作，首先将 instance 的数据迁移过来，主要包括镜像文件、虚拟网络等资源，日志在 devstack-controller:/opt/stack/logs/n-cpu.log

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102203.jpg)

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102231.jpg)

2. 源节点启动迁移操作，暂停 instance

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102646.jpg)

3. 在目标节点上 Resume instance

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102724.jpg)

4. 在源节点上执行迁移的后处理工作，删除 instance

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102753.jpg)

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102814.jpg)

5. 在目标节点上执行迁移的后处理工作，创建 XML，在 Hypervisor 中定义 instance，使之下次能够正常启动。

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102853.jpg)

Instance 在 Live Migrate 的整个过程中不会停机，通过 Ping 操作来观察

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206102923.jpg)

可见在迁移过程中，Ping 进程没有中断，只是有一个 ping 包的延迟增加了。下面我们再来看源和目标节点共享存储下的 Live Migrate。

## 共享存储Live Migration

有多种方式可以实现共享存储，比如可以将 instance 的镜像文件放在 NFS 服务器上，或者使用 NAS 服务器，或者分布式文件系统。

作为实验，这里采用 NFS 方案。其他共享存储方案对于 Live Migration 本质上是一样的，只是在性能和高可用性上更好。

### 搭建NFS环境

将 devstack-controller 作为 NFS 服务器，共享其目录 /opt/stack/data/nova/instances。 devstack-compute1 作为 NFS 客户端将此目录 mount 到本机，如下所示：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206104140.jpg)

这样，OpenStack 的 instance 在 devstack-controller 和 devstack-compute1 上就实现共享存储了。

共享存储的迁移过程与 Block Migrate 基本上一样，只是几个环节有点区别：

1. 向 nova-api 提交请求的时候，不能勾选“Block Migrate”

   ![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206104221.jpg)

2. 因为源和目标节点都能直接访问 instance 的镜像，所以目标节点在准备阶段不需要传输镜像文件，源节点在迁移后处理阶段也无需删除 instance 的目录。

3. 只有 instance 的状态需要从源节点传输到的目标节点，整个迁移速递比 Block Migration 快很多。

以上是 Live Migrate 操作的详细分析。

# Evacute

Rebuild 可以恢复损坏的 instance，但如果是宿主机坏了，比如硬件故障或者断电造成整台计算节点无法工作，该节点上运行的 instance 该如何恢复。（Shelve或者Migrate都可以管理虚拟机，但是这两个操作都要求 instance 所在计算节点的 nova-compute 服务正常运行。 ）

Evacuate 可在 nova-compute 无法工作的情况下将节点上的 instance 迁移到其他计算节点上。但有个前提： Instance 的镜像文件必须放在共享存储上。

Evacuate instance 的流程图

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206110728.jpg)

1. 向 nova-api 发送请求
2. nova-api 发送消息
3. nova-scheduler 执行调度
4. nova-scheduler 发送消息
5. nova-compute 执行操作

## 向 nova-api 发送请求

Instance c2 运行在 devstack-compute1 上。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206110916.jpg)

通过断电模拟计算节点故障，然后执行 Evacuate 操作恢复 instance c2。 目前 Evacuate 只能通过 CLI 执行。（测试OpenStack版本比较老旧）

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206110940.jpg)

这里需要指定` --on-shared-storage `这个参数

查看日志 /opt/stack/logs/n-api.log

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206111048.jpg)

## nova-api发送消息

nova-api 向 Messaging（RabbitMQ）发送了一条消息：“Evacuate 这个 Instance”。查看源代码 /opt/stack/nova/nova/compute/api.py，方法是 evacuate。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206111141.jpg)

evacuate 实际上是通过 rebuild 操作实现的。 这是可以理解的，因为 evacuate 是用共享存储上 instance 的镜像文件重新创建虚机

## nova-scheduler 执行调度

nova-scheduler 收到消息后，会为 instance 选择合适的计算节点。查看日志 /opt/stack/logs/n-sch.log。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206112451.jpg)

nova-scheduler 最后选择在 devstack-controller 计算节点上重建 instance。

## nova-scheduler 发送消息

nova-scheduler 发送消息，通知计算节点可以创建 instance 了。源代码在 /opt/stack/nova/nova/scheduler/filter_scheduler.py 第 95 行，方法为 select_destinations。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206112601.jpg)

## nova-compute执行操作

计算节点上的工作是用共享存储上的镜像文件重建 instance。日志在 devstack-controller:/opt/stack/logs/n-cpu.log。

为instance分配资源

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206112732.jpg)

使用共享存储上的镜像文件

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206112802.jpg)

启动instance

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206112854.jpg)

Evacuate 操作完成后，instance 在 devstack-controller 上运行。

以上是 Evacuate 操作的详细分析。

# Instance 操作总结

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206113124.jpg)

如上图所示，对 Instance 的管理按运维工作的场景分为两类：常规操作和故障处理。

## 常规操作

常规操作中，Launch、Start、Reboot、Shut Off 和 Terminate 都很好理解。

**Resiz**
通过应用不同的 flavor 调整分配给 instance 的资源。

**Lock/Unlock**
可以防止对 instance 的误操作。

**Pause/Suspend/Resume**
暂停当前 instance，并在以后恢复。 Pause 和 Suspend 的区别在于 Pause 将 instance 的运行状态保存在计算节点的内存中，而 Suspend 保存在磁盘上。Pause 的优点是 Resume 速度比 Suspend 快；缺点是如果计算节点因某种原因重启，内存数据丢失，就无法 Resume 了，而 Suspend 则没有这个问题。

**Snapshot**
备份 instance 到 Glance。 Snapshot 生成的 image 可用于故障恢复，或者以此为模板部署新的 instance。

## 故障处理

故障处理有两种场景：计划内和计划外。

计划内是指提前安排时间窗口做的维护工作，比如服务器定期微码升级，添加更换硬件等。 计划外是指发生了没有预料到的突发故障，比如强行关机造成 OS 系统文件损坏，服务器掉电，硬件故障等。

### 计划内故障处理

对于计划内的故障处理，可以在维护窗口中将 instance 迁移到其他计算节点。

涉及如下操作：

**Migrate**
将 instance 迁移到其他计算节点。 迁移之前，instance 会被 Shut Off，支持共享存储和非共享存储。

**Live Migrate**
与 Migrate 不同，Live Migrate 能不停机在线地迁移 instance，保证了业务的连续性。也支持共享存储和非共享存储（Block Migration）

**Shelve/Unshelve**

Shelve 将 instance 保存到 Glance 上，之后可通过 Unshelve 重新部署。 Shelve 操作成功后，instance 会从原来的计算节点上删除。 Unshelve 会重新选择节点部署，可能不是原节点。

### 计划外故障处理

计划外的故障按照影响的范围又分为两类：Instance 故障和计算节点故障。

#### Instance 故障

Instance 故障只限于某一个 instance 的操作系统层面，系统无法正常启动。

可以使用如下操作修复 instance：

**Rescue/Unrescue**
用指定的启动盘启动，进入 Rescue 模式，修复受损的系统盘。成功修复后，通过 Unrescue 正常启动 instance。

**Rebuild**
如果 Rescue 无法修复，则只能通过 Rebuild 从已有的备份恢复。Instance 的备份是通过 snapshot 创建的，所以需要有备份策略定期备份。

#### 计算节点故障

Instance 故障的影响范围局限在特定的 instance，计算节点本身是正常工作的。如果计算节点发生故障，OpenStack 则无法与节点的 nova-compute 通信，其上运行的所有 instance 都会受到影响。这个时候，只能通过 Evacuate 操作在其他正常节点上重建 Instance。

**Evacuate**
利用共享存储上 Instance 的镜像文件在其他计算节点上重建 Instance。 所以提前规划共享存储是关键。

# 小结

Nova 是 OpenStack 最重要的项目，处于 OpenStack 的中心。其他 Keystone，Glance，Cinder 和 Neutron 项目都是为 Nova 其服务的，一定要好好理解。

