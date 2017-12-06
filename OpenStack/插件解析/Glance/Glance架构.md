# Glance

Image Service 的功能是管理 Image，让用户能够发现、获取和保存 Image。在 OpenStack 中，提供 Image Service 的是 Glance，其具体功能如下：

1. 提供 REST API 让用户能够查询和获取 image 的元数据和 image 本身
2. 支持多种方式存储 image，包括普通的文件系统、Swift、Amazon S3 等
3. 对 Instance 执行 Snapshot 创建新的 image

## Glance架构

![Glance架构](http://oydlbqndl.bkt.clouddn.com/glance.jpg)

### glance-api

glance-api 是系统后台运行的服务进程。 对外提供 REST API，响应 image 查询、获取和存储的调用。

glance-api 不会真正处理请求。 如果操作是与 image metadata（元数据）相关，glance-api 会把请求转发给 glance-registry； 如果操作是与 image 自身存取相关，glance-api 会把请求转发给该 image 的 store backend。

在控制节点上可以查看 glance-api 进程

```shell
# ps aux | grep glance-api
glance    47536  0.6  0.0 763156 103300 ?       Ss   Oct20  54:32 /usr/bin/python2 /usr/bin/glance-api
glance    47583  0.0  0.0 1420400 146516 ?      S    Oct20   0:10 /usr/bin/python2 /usr/bin/glance-api
glance    47584  0.0  0.0 787488 118580 ?       S    Oct20   0:04 /usr/bin/python2 /usr/bin/glance-api
glance    47585  0.0  0.0 1407976 129292 ?      S    Oct20   0:09 /usr/bin/python2 /usr/bin/glance-api
```

### glance-registry

glance-registry 是系统后台运行的服务进程。 负责处理和存取 image 的 metadata，例如 image 的大小和类型。在控制节点上可以查看 glance-registry 进程

```shell
# ps aux | grep glance-registry
glance   158881  0.0  0.0 354140 80068 ?        Ss   Oct19   0:02 /usr/bin/python2 /usr/bin/glance-registry
glance   159122  0.0  0.0 354140 75400 ?        S    Oct19   0:00 /usr/bin/python2 /usr/bin/glance-registry
glance   159123  0.0  0.0 354140 75264 ?        S    Oct19   0:00 /usr/bin/python2 /usr/bin/glance-registry
```

Glance 支持多种格式的 image，包括

![Glance支持image格式](http://oydlbqndl.bkt.clouddn.com/glance支持格式.jpg)

### Database

Image 的 metadata 会保持到 database 中，默认是 MySQL。 在控制节点上可以查看 glance 的 database 信息

```shell
MySQL: root [glance]> show tables;
+----------------------------------+
| Tables_in_glance                 |
+----------------------------------+
| artifact_blob_locations          |
| artifact_blobs                   |
| artifact_dependencies            |
| artifact_properties              |
| artifact_tags                    |
| artifacts                        |
| image_locations                  |
| image_members                    |
| image_properties                 |
| image_tags                       |
| images                           |
| metadef_namespace_resource_types |
| metadef_namespaces               |
| metadef_objects                  |
| metadef_properties               |
| metadef_resource_types           |
| metadef_tags                     |
| migrate_version                  |
| task_info                        |
| tasks                            |
+----------------------------------+
```

### Store backend

Glance 自己并不存储 image。 真正的 image 是存放在 backend 中的。 Glance 支持多种 backend，包括：

1. A directory on a local file system（这是默认配置）
2. GridFS
3. Ceph RBD
4. Amazon S3
5. Sheepdog
6. OpenStack Block Storage (Cinder)
7. OpenStack Object Storage (Swift)
8. VMware ESX

具体使用哪种 backend，是在 `/etc/glance/glance-api.conf `中配置的，例子中使用Ceph RBD作为backend。

```shell
# cat /etc/glance/glance-api.conf
[glance_store]
stores = rbd
default_store = rbd
filesystem_store_datadir = /var/lib/glance/images/

#ceph rbd config
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_pool = hyhive_images
rbd_store_chunk_size = 4
```

其他 backend 的配置可参考[Glance image backend choices](http://docs.openstack.org/liberty/config-reference/content/configuring-image-service-backends.html)

查看目前已经存在的 image

```shell
# . keystonerc_admin
# glance image-list 
+--------------------------------------+---------------------+
| ID                                   | Name                |
+--------------------------------------+---------------------+
| d20b8f77-7ede-45aa-976f-fc7f0fe9b833 | centos6.4           |
+--------------------------------------+---------------------+
```

查看具体RBD

```shell
# rados -p hyhive_images ls | grep d20b8
rbd_id.d20b8f77-7ede-45aa-976f-fc7f0fe9b833
```

## 创建image

1. 将 image 上传到控制节点的文件系统中，例如 `/infinityfs1/images/CentOS-6.4-x86_64.qcow2`

2. 设置环境变量

   ```shell
   # . keystonerc_admin
   ```

3. 执行 image 创建命令

   ```shell
   # glance image-create --name test --file /infinityfs1/images/CentOS-6.4-x86_64.qcow2 --disk-format qcow2 --container-format bare --progress
   [========>                     ] 27%
   ```

   在创建 image 的 CLI 参数中我们用` --progress` 让其显示文件上传的百分比 `%`

   ```shell
   # glance image-create --name test --file /infinityfs1/images/CentOS-6.4-x86_64.qcow2 --disk-format qcow2 --container-format bare --progress
   [=============================>] 100%
   +------------------+----------------------------------------------------------------------------------+
   | Property         | Value                                                                            |
   +------------------+----------------------------------------------------------------------------------+
   | checksum         | 32e1cef67fee9f11b9e074bd938afa22                                                 |
   | container_format | bare                                                                             |
   | created_at       | 2017-10-26T01:45:11Z                                                             |
   | direct_url       | rbd://3318ecb9-65c8-4ca1-aa03-dda6247c2435/hyhive_images/f4a83450-e975-4052-a702 |
   |                  | -c19386fafd5f/snap                                                               |
   | disk_format      | qcow2                                                                            |
   | id               | f4a83450-e975-4052-a702-c19386fafd5f                                             |
   | locations        | [{"url": "rbd://3318ecb9-65c8-4ca1-aa03-dda6247c2435/hyhive_images/f4a83450-e975 |
   |                  | -4052-a702-c19386fafd5f/snap", "metadata": {}}]                                  |
   | min_disk         | 0                                                                                |
   | min_ram          | 0                                                                                |
   | name             | test                                                                             |
   | owner            | 9304cf6daa224c0c8b0a5c7527f01927                                                 |
   | protected        | False                                                                            |
   | size             | 1343881216                                                                       |
   | status           | active                                                                           |
   | tags             | []                                                                               |
   | updated_at       | 2017-10-26T01:45:41Z                                                             |
   | virtual_size     | None                                                                             |
   | visibility       | private                                                                          |
   +------------------+----------------------------------------------------------------------------------+
   ```

4. 查询现有镜像信息

   ```shell
   # glance image-list
   +--------------------------------------+---------------------+
   | ID                                   | Name                |
   +--------------------------------------+---------------------+
   | f4a83450-e975-4052-a702-c19386fafd5f | test                |
   | 07ce2f35-3c5c-4c13-a3bc-e9e47d8d47d8 | win7                |
   +--------------------------------------+---------------------+
   ```

5. 查看镜像具体信息

   ```shell
   # glance image-show 07ce2f35-3c5c-4c13-a3bc-e9e47d8d47d8
   +------------------+----------------------------------------------------------------------------------+
   | Property         | Value                                                                            |
   +------------------+----------------------------------------------------------------------------------+
   | checksum         | 74cd5c3b29adbff5258627414f100b7e                                                 |
   | container_format | bare                                                                             |
   | created_at       | 2017-09-25T01:11:23Z                                                             |
   | direct_url       | rbd://3318ecb9-65c8-4ca1-aa03-dda6247c2435/hyhive_images/07ce2f35-3c5c-4c13      |
   |                  | -a3bc-e9e47d8d47d8/snap                                                          |
   | disk_format      | qcow2                                                                            |
   | id               | 07ce2f35-3c5c-4c13-a3bc-e9e47d8d47d8                                             |
   | locations        | [{"url": "rbd://3318ecb9-65c8-4ca1-aa03-dda6247c2435/hyhive_images/07ce2f35      |
   |                  | -3c5c-4c13-a3bc-e9e47d8d47d8/snap", "metadata": {}}]                             |
   | min_disk         | 20                                                                               |
   | min_ram          | 0                                                                                |
   | name             | win7                                                                             |
   | os_distro        |                                                                                  |
   | os_type          | windows                                                                          |
   | owner            | 9304cf6daa224c0c8b0a5c7527f01927                                                 |
   | protected        | False                                                                            |
   | size             | 3580690432                                                                       |
   | status           | active                                                                           |
   | tags             | []                                                                               |
   | updated_at       | 2017-09-25T01:12:50Z                                                             |
   | virtual_size     | None                                                                             |
   | visibility       | public                                                                           |
   +------------------+----------------------------------------------------------------------------------+
   ```

6. 删除image

   ```shell
   # glance image-delete f4a83450-e975-4052-a702-c19386fafd5f
   ```

   ​