# 如何使用 OpenStack CLI

OpenStack 服务都有自己的 CLI。

命令很好记，就是服务的名字，比如 Glance 就是` glance`，Nova 就是` nova`。

但 Keystone 比较特殊，现在是用OpenStack来代替老版的 `keystone` 命令。 比如查询用户列表，如果用`openstack user list`

```shell
# openstack user list 
+----------------------------------+-------------+
| ID                               | Name        |
+----------------------------------+-------------+
| 137a1d13bd2a423ab8b9ef265c634c4b | nova        |
| 2ce8cae1f1364864b78dbe6018862a4f | demo        |
| 3fae06e8fc87413285306daec4f3efa1 | cloudmatrix |
| 5fd13261f2fd4144832d18e3fb581395 | neutron     |
| 6fad54a4b37b4b2884853ae41893795d | glance      |
| a892d6809f9646db8d12776cd11e4248 | cinder      |
| f9e86728148b4cffafeff374fca280d8 | admin       |
+----------------------------------+-------------+
```

不同服务用的命令虽然不同，但这些命令使用方式却非常类似，可以举一反三。

1. 执行命令之前，需要设置环境变量。

   这些变量包含用户名、Project、密码等； 如果不设置，每次执行命令都必须设置相关的命令行参数

2. 各个服务的命令都有增、删、改、查的操作

   其格式是

   ```shell
   CMD <obj>-create [parm1] [parm2]…
   CMD <obj>-delete [parm]
   CMD <obj>-update [parm1] [parm2]…
   CMD <obj>-list CMD <obj>-show [parm]
   ```

   例如 glance 管理的是 image，那么:
   CMD 就是 glance，obj 就是 image，对应的命令就有

   ```shell
   glance image-create
   glance image-delete
   glance image-update
   glance image-list
   glance image-show
   ```

   再比如 neutron 负责管理网络和子网，那么：
   CMD 就是 neutron；obj 就是 net 和 subnet 对应的命令就有

   网络相关操作

   ```shell
   neutron net-create
   neutron net -delete
   neutron net -update
   neutron net -list
   neutron net –show
   ```

   子网相关操作

   ```shell
   neutron subnet-create
   neutron subnet -delete
   neutron subnet -update
   neutron subnet -list
   neutron subnet–show
   ```

   有的命令 <obj> 可以省略，比如 nova 下面的操作都是针对 instance

   ```shell
   nova boot
   nova delete
   nova list nova show
   ```

3. 每个对象都有 ID

   delete，show 等操作都以 ID 为参数，例如

   ```shell
   # openstack user list 
   +----------------------------------+-------------+
   | ID                               | Name        |
   +----------------------------------+-------------+
   | 137a1d13bd2a423ab8b9ef265c634c4b | nova        |
   +----------------------------------+-------------+
   [root@hyhive01 ~]# openstack user show 137a1d13bd2a423ab8b9ef265c634c4b
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | domain_id           | default                          |
   | enabled             | True                             |
   | id                  | 137a1d13bd2a423ab8b9ef265c634c4b |
   | name                | nova                             |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+
   ```

4. 可用 help 查看命令的用法

   除了 delete，show 等操作只需要 ID 一个参数，其他操作可能需要更多的参数，用 help 查看所需的参数，格式是

   ```shell
   CMD help [SUB-CMD]
   ```

   例如查看 glance 都有哪些 SUB-CMD

   ```shell
   # glance help 
   usage: glance [--version] [-d] [-v] [--get-schema] [--no-ssl-compression] [-f]
                 [--os-image-url OS_IMAGE_URL]
                 [--os-image-api-version OS_IMAGE_API_VERSION]
                 [--profile HMAC_KEY] [--insecure] [--os-cacert <ca-certificate>]
                 [--os-cert <certificate>] [--os-key <key>] [--timeout <seconds>]
                 [--os-auth-url OS_AUTH_URL] [--os-domain-id OS_DOMAIN_ID]
                 [--os-domain-name OS_DOMAIN_NAME]
                 [--os-project-id OS_PROJECT_ID]
                 [--os-project-name OS_PROJECT_NAME]
                 [--os-project-domain-id OS_PROJECT_DOMAIN_ID]
                 [--os-project-domain-name OS_PROJECT_DOMAIN_NAME]
                 [--os-trust-id OS_TRUST_ID] [--os-user-id OS_USER_ID]
                 [--os-username OS_USERNAME]
                 [--os-user-domain-id OS_USER_DOMAIN_ID]
                 [--os-user-domain-name OS_USER_DOMAIN_NAME]
                 [--os-password OS_PASSWORD] [--key-file OS_KEY]
                 [--ca-file OS_CACERT] [--cert-file OS_CERT]
                 [--os-tenant-id OS_TENANT_ID] [--os-tenant-name OS_TENANT_NAME]
                 [--os-region-name OS_REGION_NAME]
                 [--os-auth-token OS_AUTH_TOKEN]
                 [--os-service-type OS_SERVICE_TYPE]
                 [--os-endpoint-type OS_ENDPOINT_TYPE]
                 <subcommand> ...

   Command-line interface to the OpenStack Images API.

   Positional arguments:
     <subcommand>
       explain             Describe a specific model.
       image-create        Create a new image.
       image-deactivate    Deactivate specified image.
       image-delete        Delete specified image.
       image-download      Download a specific image.
       image-list          List images you can access.
       image-reactivate    Reactivate specified image.
   ```

   查看` glance image-update` 的用法

   ```shell
   # glance help image-update
   usage: glance image-update [--architecture <ARCHITECTURE>]
                              [--protected [True|False]] [--name <NAME>]
                              [--instance-uuid <INSTANCE_UUID>]
                              [--min-disk <MIN_DISK>] [--visibility <VISIBILITY>]
                              [--kernel-id <KERNEL_ID>]
                              [--os-version <OS_VERSION>]
                              [--disk-format <DISK_FORMAT>]
                              [--os-distro <OS_DISTRO>] [--owner <OWNER>]
                              [--ramdisk-id <RAMDISK_ID>] [--min-ram <MIN_RAM>]
                              [--container-format <CONTAINER_FORMAT>]
                              [--property <key=value>] [--remove-property key]
                              <IMAGE_ID>

   Update an existing image.

   Positional arguments:
     <IMAGE_ID>            ID of image to update.

   Optional arguments:
     --architecture <ARCHITECTURE>
                           Operating system architecture as specified in
                           http://docs.openstack.org/user-guide/common/cli-
                           manage-images.html
     --protected [True|False]
                           If true, image will not be deletable.
     --name <NAME>         Descriptive name for the image
   ```

   ​

