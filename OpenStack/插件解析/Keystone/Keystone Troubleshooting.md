# Troubleshooting

OpenStack 排查问题的方法主要是通过日志。

Keystone 主要有两个日志： `keystone.log `和` keystone_access.log`，分别保存在：

```shell
# ls -ltr /var/log/keystone/keystone.log
-rw-rw---- 1 root keystone 943331 Oct 26 06:53 /var/log/keystone/keystone.log
# ls -ltr /var/log/httpd/keystone_access.log
-rw-r--r-- 1 root root 21028795 Oct 26 06:53 /var/log/httpd/keystone_access.log
```

如果需要得到最详细的日志信息，可以在` /etc/keystone/keystone.conf `中打开` debug` 选项

```shell
# cat /etc/keystone/keystone.conf
[DEFAULT]
debug = False
```

