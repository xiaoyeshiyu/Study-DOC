# Cinder 组件

## cinder-api

cinder-api 是整个 Cinder 组件的门户，所有 cinder 的请求都首先由 cinder-api 处理。cinder-api 向外界暴露若干 HTTP REST API 接口。在 keystone 中可以查询 cinder-api 的 endponits。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206153240.png)

客户端可以将请求发送到 endponits 指定的地址，向 cinder-api 请求操作。 当然，作为最终用户的我们不会直接发送 Rest API 请求。OpenStack CLI，Dashboard 和其他需要跟 Cinder 交换的组件会使用这些 API。

cinder-api 对接收到的 HTTP API 请求会做如下处理：

1. 检查客户端传入的参数是否合法有效
2. 调用 cinder 其他子服务的处理客户端请求
3. 将 cinder 其他子服务返回的结果序列号并返回给客户端

cinder-api 接受的请求简单的说，只要是 Volume 生命周期相关的操作，cinder-api 都可以响应。大部分操作都可以在 Dashboard 上看到。

打开 Volume 管理界面

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206154155.jpg)

点击下拉箭头，列表中就是 cinder-api 可执行的操作。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206154701.jpg)

## cinder-scheduler

创建 Volume 时，cinder-scheduler 会基于容量、Volume Type 等条件选择出最合适的存储节点，然后让其创建 Volume。

## cinder-volume

cinder-volume 在存储节点上运行，OpenStack 对 Volume 的操作，最后都是交给 cinder-volume 来完成的。cinder-volume 自身并不管理真正的存储设备，存储设备是由 volume provider 管理的。cinder-volume 与 volume provider 一起实现 volume 生命周期的管理。

### 通过 Driver 架构支持多种 Volume Provider

现在市面上有很多块存储产品和方案（volume provider），cinder-volume 通过Driver架构与它们连接。

cinder-volume 为这些 volume provider 定义了统一的接口，volume provider 只需要实现这些接口，就可以 Driver 的形式即插即用到 OpenStack 系统中。下面是 Cinder Driver 的架构示意图：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206155045.jpg)

我们可以在 /opt/stack/cinder/cinder/volume/drivers/ 目录下查看到 OpenStack 源代码中已经自带了很多 volume provider 的 Driver：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206155112.jpg)

存储节点在配置文件 /etc/cinder/cinder.conf 中用 volume_driver 选项配置使用的driver：

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206155137.jpg)

这里 LVM 是我们使用的 volume provider。

### 定期向 OpenStack 报告计算节点的状态

在前面 cinder-scheduler 会用到 CapacityFilter 和 CapacityWeigher，它们都是通过存储节点的空闲容量来做筛选。空闲容量这些信息则是cinder-volume 定期向 Cinder 报告

从 cinder-volume 的日志 /opt/stack/logs/c-vol.log 可以发现每隔一段时间，cinder-volume 就会报告当前存储节点的资源使用情况。

![](http://oydlbqndl.bkt.clouddn.com/微信图片_20171206155219.jpg)

在实验环境中存储节点使用的是 LVM，所以在上面的日志看到存储节点通过“vgs”和”lvs”这两个命令获取 LVM 的容量使用信息。

### 实现 volume 生命周期管理

Cinder 对 volume 的生命周期的管理最终都是通过 cinder-volume 完成的，包括 volume 的 create、extend、attach、snapshot、delete 等。