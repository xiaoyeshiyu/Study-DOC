# keystone

1. 管理用户及其权限
2. 维护OpenStack Services的Endpoint
3. Authentication和Authorization

![概念](http://oydlbqndl.bkt.clouddn.com/Snipaste_2017-10-26_10-51-29.png)

## User

User请求访问OpenStack，Keystone首先验。User除了admin、demo，还包括各项组件,nova、cinder、glance、neutron等

## Credentials

Credentials是User用来证明自己身份信息的，可以是:

1. 用户名/密码
2. Token
3. API Key
4. 其他高级方式

## Authentication

Authentication是Keystone验证User的过程。

首先，User访问OpenStack时向Keystone提交用户名密码形式的Credentials

然后，Keystone通过验证，返回User一个Token作为后续访问的Credentials

## Token

Token是由字母数字组成的字符串

1. Token用作访问Service的Credentail
2. Service会通过Keystone验证Token的有效性
3. Token的有效期默认是24小时

## Project

Project用于将OpenStack的资源（计算、存储和网络）进行分组和隔离，根据OpenStack服务的对象不同，Project可以是一个客户、部门或者项目组

1. 组员的所有权是属于Project的，而不是User
2. 在OpenStack的界面和文档中，Tenant/Project/Account这几个术语是想通的
3. 每个User必须挂在Project里才能访问该Project的资源
4. admin相当于root用户，具有最高权限

## Service

OpenStack的Service包括Compute（Nova）、Block Storage（Cinder）、Object Storage（Swift）、Image Service (Glance) 、Networking Service (Neutron) 等。每个 Service 都会提供若干个 Endpoint，User 通过 Endpoint 访问资源和执行操作。

## Endpoint

通过命令可以查看

```shell
# . keystonerc_admin
# openstack catalog list 
+----------+----------+--------------------------------------------------------------------------+
| Name     | Type     | Endpoints                                                                |
+----------+----------+--------------------------------------------------------------------------+
| neutron  | network  | RegionOne                                                                |
|          |          |   public: http://10.0.20.28:9696                                         |
|          |          | RegionOne                                                                |
|          |          |   internal: http://10.0.20.28:9696                                       |
|          |          | RegionOne                                                                |
|          |          |   admin: http://10.0.20.28:9696                                          |
|          |          |                                                                          |
| nova     | compute  | RegionOne                                                                |
|          |          |   internal: http://10.0.20.28:8774/v2.1/0623e803ba6f46a4aec7fe7be08a0266 |
|          |          | RegionOne                                                                |
|          |          |   public: http://10.0.20.28:8774/v2.1/0623e803ba6f46a4aec7fe7be08a0266   |
|          |          | RegionOne                                                                |
|          |          |   admin: http://10.0.20.28:8774/v2.1/0623e803ba6f46a4aec7fe7be08a0266    |
|          |          |                                                                          |
| cinderv2 | volumev2 | RegionOne                                                                |
|          |          |   admin: http://10.0.20.28:8776/v2/0623e803ba6f46a4aec7fe7be08a0266      |
|          |          | RegionOne                                                                |
|          |          |   public: http://10.0.20.28:8776/v2/0623e803ba6f46a4aec7fe7be08a0266     |
|          |          | RegionOne                                                                |
|          |          |   internal: http://10.0.20.28:8776/v2/0623e803ba6f46a4aec7fe7be08a0266   |
|          |          |                                                                          |
| cinder   | volume   | RegionOne                                                                |
|          |          |   internal: http://10.0.20.28:8776/v1/0623e803ba6f46a4aec7fe7be08a0266   |
|          |          | RegionOne                                                                |
|          |          |   admin: http://10.0.20.28:8776/v1/0623e803ba6f46a4aec7fe7be08a0266      |
|          |          | RegionOne                                                                |
|          |          |   public: http://10.0.20.28:8776/v1/0623e803ba6f46a4aec7fe7be08a0266     |
|          |          |                                                                          |
| keystone | identity | RegionOne                                                                |
|          |          |   internal: http://10.0.20.28:35357/v3/                                  |
|          |          | RegionOne                                                                |
|          |          |   public: http://10.0.20.28:5000/v3/                                     |
|          |          | RegionOne                                                                |
|          |          |   admin: http://10.0.20.28:35357/v3/                                     |
|          |          |                                                                          |
| glance   | image    | RegionOne                                                                |
|          |          |   internal: http://10.0.20.28:9292                                       |
|          |          | RegionOne                                                                |
|          |          |   admin: http://10.0.20.28:9292                                          |
|          |          | RegionOne                                                                |
|          |          |   public: http://10.0.20.28:9292                                         |
|          |          |                                                                          |
| cinderv3 | volumev3 | RegionOne                                                                |
|          |          |   public: http://10.0.20.28:8776/v3/0623e803ba6f46a4aec7fe7be08a0266     |
|          |          | RegionOne                                                                |
|          |          |   internal: http://10.0.20.28:8776/v3/0623e803ba6f46a4aec7fe7be08a0266   |
|          |          | RegionOne                                                                |
|          |          |   admin: http://10.0.20.28:8776/v3/0623e803ba6f46a4aec7fe7be08a0266      |
|          |          |                                                                          |
+----------+----------+--------------------------------------------------------------------------+
```

## Role

安全包含两部分：Authentication（认证）和 Authorization（鉴权）

Authentication 解决的是“你是谁？”的问题
Authorization 解决的是“你能干什么？”的问题

Keystone 借助 Role 实现 Authorization：

1. Keystone定义Role

```shell
# openstack role list 
+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 993a68f3ae2b4bb0a92fc503a59798f6 | admin  |
| c8f9a415c2e343488d165f41194293b2 | user   |
| c973ed938462417abaa3e7f9620c8435 | leader |
+----------------------------------+--------+
```

2. Service 决定每个 Role 能做什么事情。 Service 通过各自的 policy.json 文件对 Role 进行访问控制。 下面是 Nova 服务 /etc/nova/policy.json 中

```json
	"compute:create": "rule:admin_or_owner",
    "compute:create:attach_network": "rule:admin_or_owner",
    "compute:create:attach_volume": "rule:admin_or_owner",
    "compute:create:forced_host": "",
```

上面配置的含义是：对于 create、attach_network 和 attach_volume 操作，admin或者所有者可以执行；任何用户都能执行 forced_host 操作。

