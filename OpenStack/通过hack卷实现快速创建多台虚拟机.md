# 通过hack卷ID实现快速克隆多台虚拟机

通常克隆的方法如下：

* 首先将虚拟机导出成`raw`格式文件，
* 然后将`raw`格式转化成qcow2格式镜像文件
* 最后将qcow2镜像格式文件加入到glance中，虚拟机从该镜像启动

这种方式后面有测试环境之后，会进行演示。这种方法缺点为raw格式文件普遍比较大，导出过程比较费事，之后做成镜像之后镜像文件容量也非常大，创建虚拟机的时候会非常耗时，甚至超时导致创建失败。

现在的办法为：

* 创建一个和模板相同大小的空白卷，记录下卷ID
* 删除该卷，拷贝模板卷，并重命名为删除卷ID
* 直接通过拷贝之后的卷，从硬盘启动虚拟机

确定需要克隆的卷

``` shell
[root@hyhive03 ~]# cinder list --all 
+--------------------------------------+----------------------------------+-----------+------------+------+-------------+----------+--------------------------------------+
| ID                                   | Tenant ID                        | Status    | Name       | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+----------------------------------+-----------+------------+------+-------------+----------+--------------------------------------+
| 406f1649-99f7-4947-bd83-83482a55a6b6 | d75491425d0b455db53c170fc163d675 | available |            | 120  | infinity    | true     |                                      |
+--------------------------------------+----------------------------------+-----------+------------+------+-------------+----------+--------------------------------------+
```

通过ID`406f1649-99f7-4947-bd83-83482a55a6b6`确定卷在存储中的位置（本文cinder存储服务由ceph提供）

``` shell
[root@hyhive03 ~]# rbd info hyhive_volumes/volume-406f1649-99f7-4947-bd83-83482a55a6b6
rbd image 'volume-406f1649-99f7-4947-bd83-83482a55a6b6':
	size 120 GB in 30720 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.3b8506b8b4567
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 
```

查看卷实际容量

``` shell
[root@hyhive03 ~]# rbd diff hyhive_volumes/volume-406f1649-99f7-4947-bd83-83482a55a6b6  | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
20872 MB
```

创建空白的卷

``` shell
[root@hyhive03 ~]# cinder create --name test 120
+--------------------------------+--------------------------------------+
| Property                       | Value                                |
+--------------------------------+--------------------------------------+
| attachments                    | []                                   |
| availability_zone              | nova                                 |
| bootable                       | false                                |
| consistencygroup_id            | None                                 |
| created_at                     | 2017-09-07T11:04:14.000000           |
| description                    | None                                 |
| encrypted                      | False                                |
| id                             | ea068a78-5c57-4a91-999c-e7273f12625f |
| metadata                       | {}                                   |
| migration_status               | None                                 |
| multiattach                    | False                                |
| name                           | test                                 |
| os-vol-host-attr:host          | None                                 |
| os-vol-mig-status-attr:migstat | None                                 |
| os-vol-mig-status-attr:name_id | None                                 |
| os-vol-tenant-attr:tenant_id   | 0dbdf41493bc418eb4b6fb572ddd5617     |
| replication_status             | disabled                             |
| size                           | 120                                  |
| snapshot_id                    | None                                 |
| source_volid                   | None                                 |
| status                         | creating                             |
| updated_at                     | None                                 |
| user_id                        | c829aeaf6f5245fbbcf07be1815344de     |
| volume_type                    | infinity                             |
+--------------------------------+--------------------------------------+
[root@hyhive03 ~]# rbd diff hyhive_volumes/volume-ea068a78-5c57-4a91-999c-e7273f12625f  | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
0 MB
```

在ceph块存储中定位该卷

``` shell
[root@hyhive03 ~]# rbd -p hyhive_volumes ls | grep ea068a78-5c57-4a91-999c-e7273f12625f
volume-ea068a78-5c57-4a91-999c-e7273f12625f
```

删除该卷

``` shell
[root@hyhive03 ~]# rbd -p hyhive_volumes rm volume-ea068a78-5c57-4a91-999c-e7273f12625f
Removing image: 100% complete...done.
```

拷贝该卷

``` shell
[root@hyhive03 ~]# date
Thu Sep  7 19:10:26 CST 2017
[root@hyhive03 ~]# rbd -p hyhive_volumes cp volume-406f1649-99f7-4947-bd83-83482a55a6b6 hyhive_volumes/volume-ea068a78-5c57-4a91-999c-e7273f12625f
Image copy: 100% complete...done.
[root@hyhive03 ~]# date
Thu Sep  7 19:15:38 CST 2017
```

当然，这个速度也跟ceph IOPS有关，但是这个速度绝对比导出raw格式，转成qcow2格式再创建虚拟机快得多

查看拷贝之后的卷

``` shell
[root@hyhive03 ~]# rbd diff hyhive_volumes/volume-ea068a78-5c57-4a91-999c-e7273f12625f  | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
20872 MB
```

将该卷的`bootable`属性设置为True，

``` shell
[root@hyhive03 ~]# cinder set-bootable ea068a78-5c57-4a91-999c-e7273f12625f True
[root@hyhive03 ~]# cinder show ea068a78-5c57-4a91-999c-e7273f12625f
+--------------------------------+--------------------------------------+
| Property                       | Value                                |
+--------------------------------+--------------------------------------+
| attachments                    | []                                   |
| availability_zone              | nova                                 |
| bootable                       | true                                 |
| consistencygroup_id            | None                                 |
| created_at                     | 2017-09-07T11:04:14.000000           |
| description                    | None                                 |
| encrypted                      | False                                |
| id                             | ea068a78-5c57-4a91-999c-e7273f12625f |
| metadata                       | {}                                   |
| migration_status               | None                                 |
| multiattach                    | False                                |
| name                           | test                                 |
| os-vol-host-attr:host          | cluster@infinity#infinity            |
| os-vol-mig-status-attr:migstat | None                                 |
| os-vol-mig-status-attr:name_id | None                                 |
| os-vol-tenant-attr:tenant_id   | 0dbdf41493bc418eb4b6fb572ddd5617     |
| replication_status             | disabled                             |
| size                           | 120                                  |
| snapshot_id                    | None                                 |
| source_volid                   | None                                 |
| status                         | available                            |
| updated_at                     | 2017-09-07T12:02:02.000000           |
| user_id                        | c829aeaf6f5245fbbcf07be1815344de     |
| volume_type                    | infinity                             |
+--------------------------------+--------------------------------------+
```

直接从该卷启动虚拟机，在dashboard界面即可选择从该卷启动。

