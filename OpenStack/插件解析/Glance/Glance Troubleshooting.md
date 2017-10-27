# Troubleshooting 

OpenStack 排查问题的方法主要是通过日志，Service 都有自己单独的日志。Glance 主要有两个日志，`api.log` 和 `registry.log`，保存在`/var/log/glance/` 目录里

`glance-api `日志，记录 REST API 调用情况

` glance-registry `日志，记录 Glance 服务处理请求的过程以及数据库操作。

如果需要得到最详细的日志信息，可以在` /etc/glance/*.conf `中打开 debug 选项。

