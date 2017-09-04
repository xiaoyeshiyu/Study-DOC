`实例为制作CentOS 6.9镜像，其他镜像该方法同样适用。`

* 在文件夹中存放CentOS 6.9镜像文件

``` shell
[root@hyhive01 infinityfs1]# ll CentOS-6.9-x86_64-bin-DVD1.iso 
-rw-r--r-- 1 root root 3972005888 Sep  4 08:27 CentOS-6.9-x86_64-bin-DVD1.iso
[root@hyhive01 infinityfs1]# md5sum CentOS-6.9-x86_64-bin-DVD1.iso 
2e227fa14c8a9791293b6525289c7dad  CentOS-6.9-x86_64-bin-DVD1.iso
```

* 创建一个qcow2格式的文件，用于将这个文件制作成qcow2镜像，文件大小为制作之后的镜像最大使用容量，该容量在创建虚拟机的时候可调整

``` shell
[root@hyhive01 infinityfs1]# qemu-img create -f qcow2 centos6.9.qcow2 20G
Formatting 'centos6.9.qcow2', fmt=qcow2 size=21474836480 encryption=off cluster_size=65536 lazy_refcounts=off 
```

* 创建一个xml模板，用于启动虚拟机

``` xml
[root@hyhive01 infinityfs1]# cat centos6.9.xml 
<domain type='kvm'>
        <name>centos6.9</name>
        <memory ballonable='no'>8388608</memory>
        <vcpu>6</vcpu>
        <os>
          <type arch='x86_64' machine='pc'>hvm</type>
          <boot dev='cdrom'/>
          <boot dev='hd'/>
       </os>
<clock offset='localtime'>
    <timer name='hypervclock' present='yes'/>
  </clock>
       <on_poweroff>destroy</on_poweroff>
       <on_reboot>restart</on_reboot>
       <on_crash>destroy</on_crash>
       <devices>
         <emulator>/usr/libexec/qemu-kvm</emulator>
         <disk type='file' device='cdrom'>
           <source file='/infinityfs1/CentOS-6.9-x86_64-bin-DVD1.iso'/>
           <target dev='hda' bus='ide'/>
         </disk>
         <disk type='file' device='disk'>
          <driver name='qemu' type='qcow2'/>
           <source file='/infinityfs1/centos6.9.qcow2'/>
           <target dev='vda' bus='virtio'/>
         </disk>
        <interface type='bridge'>
          <model type="virtio"/>
		  <virtualport type='openvswitch'/>
          <source bridge='br-ex'/>
        </interface>
        <input type='tablet' bus='usb' />
 	<video>
      		<model type='cirrus' vram='16384' heads='1' primary='yes'/>
    	</video>
	<graphics type='vnc' autoport='yes' listen='0.0.0.0' keymap='en-us'>
      		<listen type='address' address='0.0.0.0'/>
    	</graphics>
       </devices>
     </domain>
```

xml文件解读：

``` xml
        <name>centos6.9</name>
        <memory ballonable='no'>8388608</memory>
        <vcpu>6</vcpu>
        <os>
          <type arch='x86_64' machine='pc'>hvm</type>
          <boot dev='cdrom'/>
          <boot dev='hd'/>
       </os>
```

虚拟机的名称、内存、CPU配置、启动项

``` xml
         <disk type='file' device='cdrom'>
           <source file='/infinityfs1/CentOS-6.9-x86_64-bin-DVD1.iso'/>
           <target dev='hda' bus='ide'/>
         </disk>
         <disk type='file' device='disk'>
          <driver name='qemu' type='qcow2'/>
           <source file='/infinityfs1/centos6.9.qcow2'/>
           <target dev='vda' bus='virtio'/>
         </disk>
```

虚拟机的虚拟硬盘，需要注意的是`qcow2`格式文件名称为`qwmu`，`bus`为`virtio`。

另：`windows`平台在安装过程中无法识别虚拟硬盘，需要安装专门的驱动。

``` xml
        <interface type='bridge'>
          <model type="virtio"/>
		  <virtualport type='openvswitch'/>
          <source bridge='br-ex'/>
        </interface>
```

虚拟机的网卡配置，该网卡会直接桥接在`openvswitch`的`br-ex`网卡上。

``` shell
	<graphics type='vnc' autoport='yes' listen='0.0.0.0' keymap='en-us'>
      		<listen type='address' address='0.0.0.0'/>
    	</graphics>
```

虚拟机远程桌面vnc的配置，虚拟机启动之后，可直接通过vnc打开虚拟机图形界面。

* 通过xml文件创建一个虚拟机

``` shell
[root@hyhive01 infinityfs1]# virsh create centos6.9.xml 
Domain centos6.9 created from centos6.9.xml
```

* 正常添加之后，可以看到刚才添加的虚拟机

``` shell
[root@hyhive01 infinityfs1]# virsh list --all 
 Id    Name                           State
----------------------------------------------------
 1     instance-00000012              running
 10    instance-00000016              running
 32    centos6.9                      running
```

* 查看spice协议端口

``` shell
[root@hyhive01 infinityfs1]# virsh domdisplay centos6.9
vnc://localhost:0
```

* 通过vnc连接查看安装过程是否顺利

![vnc截图](https://cl.ly/441I3y0e151N/Image%202017-09-04%20at%206.41.27%20PM.png)

剩下的过程如同CentOS安装过程，安装结束之后，文件路径中`/infinityfs1/centos6.9.qcow2`即为做好的qcow2镜像。