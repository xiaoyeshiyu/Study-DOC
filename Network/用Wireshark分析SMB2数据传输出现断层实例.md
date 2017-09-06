# 通过Wireshark分析SMB2数据传输出现断层实例

今天碰到一个非编室五台非编工作站设备剪辑片子的时候出现花屏、卡屏的问题，排查之后觉得原因可能如下：

* SMB服务出现问题，在运行过程中出现自重启导致数据传输出现断层
* 网络问题，中途出现丢包或者数据帧被摧毁等情况
* 非编软件兼容性问题，在对容量较大的素材进行剪辑的时候出现花屏

经过排查，网络问题可能性最大。在非编机上安装Wireshark，进行剪辑的时候抓包查看

1. 过滤器设置为

``` shell
ip.src==192.168.0.50 and ip.dst=192.168.0.100 and tcp.port==445
```

抓去源为`192.168.0.50`目标为`192.168.0.100`之间`445`端口的数据包

2. 分析SMB数据包。

   SMB数据包正常的样子如下

   ![SMB正常情况](https://cl.ly/0B1C2T3n452f/Image%202017-09-06%20at%207.19.38%20PM.png)

   数据包详情（为和后面不正常的数据包进行比较，选取的为Response包）

   ``` yaml
   No.     Time           Source                Destination           Protocol Length Info
     19546 6.100517       192.168.0.100         192.168.0.50          SMB2     298    Create Response File: ???\??\L\???????\Transferred\5F037F5E-A924-4A37-8E27-D5F7A4A3BD7F.png

   Frame 19546: 298 bytes on wire (2384 bits), 298 bytes captured (2384 bits) on interface 0
   Ethernet II, Src: DawningI_18:b3:1e (e8:61:1f:18:b3:1e), Dst: AsustekC_16:c9:8b (10:7b:44:16:c9:8b)
   Internet Protocol Version 4, Src: 192.168.0.100, Dst: 192.168.0.50
   Transmission Control Protocol, Src Port: 445, Dst Port: 49842, Seq: 2995906, Ack: 2310166, Len: 244
   NetBIOS Session Service
   SMB2 (Server Message Block Protocol version 2)
       SMB2 Header
           Server Component: SMB2
           Header Length: 64
           Credit Charge: 1
           NT Status: STATUS_SUCCESS (0x00000000)
           Command: Create (5)
           Credits granted: 1
           Flags: 0x00000031, Response, Priority
               .... .... .... .... .... .... .... ...1 = Response: This is a RESPONSE
               .... .... .... .... .... .... .... ..0. = Async command: This is a SYNC command
               .... .... .... .... .... .... .... .0.. = Chained: This pdu is NOT a chained command
               .... .... .... .... .... .... .... 0... = Signing: This pdu is NOT signed
               .... .... .... .... .... .... .011 .... = Priority: This pdu contains a PRIORITY
               ...0 .... .... .... .... .... .... .... = DFS operation: This is a normal operation
               ..0. .... .... .... .... .... .... .... = Replay operation: This is NOT a replay operation
           Chain Offset: 0x00000000
           Message ID: Unknown (12217121)
           Process Id: 0x0000feff
           Tree Id: 0x513362de
           Session Id: 0x00000000b6b8df0a
           Signature: 00000000000000000000000000000000
           [Response to: 19545]
           [Time from request: 0.000240000 seconds]
       Create Response (0x05)
           StructureSize: 0x0059
           Oplock: No oplock (0x00)
           Response Flags: 0x00
           Create Action: The file existed and was opened (1)
           Create: Aug 30, 2017 12:37:20.000000000 中国标准时间
           Last Access: Sep  5, 2017 13:52:48.905407000 中国标准时间
           Last Write: Aug 30, 2017 12:37:20.000000000 中国标准时间
           Last Change: Aug 30, 2017 12:37:20.000000000 中国标准时间
           Allocation Size: 2097152
           End Of File: 1585585
           File Attributes: 0x00000020
               .... .... .... .... .... .... .... ...0 = Read Only: NOT read only
               .... .... .... .... .... .... .... ..0. = Hidden: NOT hidden
               .... .... .... .... .... .... .... .0.. = System: NOT a system file/dir
               .... .... .... .... .... .... .... 0... = Volume ID: NOT a volume ID
               .... .... .... .... .... .... ...0 .... = Directory: NOT a directory
               .... .... .... .... .... .... ..1. .... = Archive: Modified since last ARCHIVE
               .... .... .... .... .... .... .0.. .... = Device: NOT a device
               .... .... .... .... .... .... 0... .... = Normal: Has some attribute set
               .... .... .... .... .... ...0 .... .... = Temporary: NOT a temporary file
               .... .... .... .... .... ..0. .... .... = Sparse: NOT a sparse file
               .... .... .... .... .... .0.. .... .... = Reparse Point: Does NOT have an associated reparse point
               .... .... .... .... .... 0... .... .... = Compressed: Uncompressed
               .... .... .... .... ...0 .... .... .... = Offline: Online
               .... .... .... .... ..0. .... .... .... = Content Indexed: NOT content indexed
               .... .... .... .... .0.. .... .... .... = Encrypted: This is NOT an encrypted file
           Reserved: 00000000
           GUID handle File: ???\??\L\???????\Transferred\5F037F5E-A924-4A37-8E27-D5F7A4A3BD7F.png
               File Id: 3b36f0dc-0000-0000-bdb5-09d400000000
               [Frame handle opened: 19546]
               [Frame handle closed: 19547]
           ExtraInfo SMB2_CREATE_QUERY_MAXIMAL_ACCESS_REQUEST SMB2_CREATE_QUERY_ON_DISK_ID
               Offset: 0x00000098
               Length: 88
               Chain Element: SMB2_CREATE_QUERY_MAXIMAL_ACCESS_REQUEST "MxAc"
                   Chain Offset: 0x00000020
                   Tag: MxAc
                       Offset: 0x00000010
                       Length: 4
                   Data: MxAc INFO
                       Offset: 0x00000018
                       Length: 8
                       MxAc INFO
                           Query Status: STATUS_SUCCESS (0x00000000)
                           Access Mask: 0x001f01ff
                               .... .... .... .... .... .... .... ...1 = Read: READ access
                               .... .... .... .... .... .... .... ..1. = Write: WRITE access
                               .... .... .... .... .... .... .... .1.. = Append: APPEND access
                               .... .... .... .... .... .... .... 1... = Read EA: READ EXTENDED ATTRIBUTES access
                               .... .... .... .... .... .... ...1 .... = Write EA: WRITE EXTENDED ATTRIBUTES access
                               .... .... .... .... .... .... ..1. .... = Execute: EXECUTE access
                               .... .... .... .... .... .... .1.. .... = Delete Child: DELETE CHILD access
                               .... .... .... .... .... .... 1... .... = Read Attributes: READ ATTRIBUTES access
                               .... .... .... .... .... ...1 .... .... = Write Attributes: WRITE ATTRIBUTES access
                               .... .... .... ...1 .... .... .... .... = Delete: DELETE access
                               .... .... .... ..1. .... .... .... .... = Read Control: READ ACCESS to owner, group and ACL of the SID
                               .... .... .... .1.. .... .... .... .... = Write DAC: OWNER may WRITE the DAC
                               .... .... .... 1... .... .... .... .... = Write Owner: Can WRITE OWNER (take ownership)
                               .... .... ...1 .... .... .... .... .... = Synchronize: Can wait on handle to SYNCHRONIZE on completion of I/O
                               .... ...0 .... .... .... .... .... .... = System Security: System security is NOT set
                               .... ..0. .... .... .... .... .... .... = Maximum Allowed: Maximum allowed is NOT set
                               ...0 .... .... .... .... .... .... .... = Generic All: Generic all is NOT set
                               ..0. .... .... .... .... .... .... .... = Generic Execute: Generic execute is NOT set
                               .0.. .... .... .... .... .... .... .... = Generic Write: Generic write is NOT set
                               0... .... .... .... .... .... .... .... = Generic Read: Generic read is NOT set
               Chain Element: SMB2_CREATE_QUERY_ON_DISK_ID "QFid"
                   Chain Offset: 0x00000000
                   Tag: QFid
                       Offset: 0x00000010
                       Length: 4
                   Data: QFid INFO
                       Offset: 0x00000018
                       Length: 32
                       QFid INFO
                           Opaque File ID: b66b04000000000000fd0000000000000000000000000000...
   ```

   3. 从I/O图标中可以看到，中途43-61秒出现数据断层，SMB没有数据传输

![I/O图标](https://cl.ly/2U1F3f2d0029/Image%202017-09-06%20at%207.34.10%20PM.png)

4. 观察43-61秒数据传输过程，出现大量`NT Status: STATUS_INSUFFICIENT_RESOURCES (0xc000009a)`报错

![大量报错](https://cl.ly/1f110U0p0T0W/[471c4c821a012bb0b08f7f393774d9b4]_Image%202017-09-06%20at%207.37.56%20PM.png)

​	出现的报错基本如下 

* `NT Status: STATUS_NOTIFY_ENUM_DIR (0x0000010c)`
* `NT Status: STATUS_PENDING (0x00000103)`
* `NT Status: STATUS_FILE_IS_A_DIRECTORY (0xc00000ba)`
* `NT Status: STATUS_INSUFFICIENT_RESOURCES (0xc000009a)`

5. 报错数据包的解读

   ``` yaml
   No.     Time           Source                Destination           Protocol Length Info
    175097 42.315681      192.168.0.100         192.168.0.50          SMB2     131    Create Response, Error: STATUS_INSUFFICIENT_RESOURCES

   Frame 175097: 131 bytes on wire (1048 bits), 131 bytes captured (1048 bits) on interface 0
   Ethernet II, Src: SuperMic_9d:a8:ee (0c:c4:7a:9d:a8:ee), Dst: AsustekC_16:c9:8b (10:7b:44:16:c9:8b)
   Internet Protocol Version 4, Src: 192.168.0.100, Dst: 192.168.0.50
   Transmission Control Protocol, Src Port: 445, Dst Port: 49842, Seq: 19602360, Ack: 17867316, Len: 77
   NetBIOS Session Service
   SMB2 (Server Message Block Protocol version 2)
       SMB2 Header
           Server Component: SMB2
           Header Length: 64
           Credit Charge: 1
           NT Status: STATUS_INSUFFICIENT_RESOURCES (0xc000009a)
           Command: Create (5)
           Credits granted: 1
           Flags: 0x00000031, Response, Priority
               .... .... .... .... .... .... .... ...1 = Response: This is a RESPONSE
               .... .... .... .... .... .... .... ..0. = Async command: This is a SYNC command
               .... .... .... .... .... .... .... .0.. = Chained: This pdu is NOT a chained command
               .... .... .... .... .... .... .... 0... = Signing: This pdu is NOT signed
               .... .... .... .... .... .... .011 .... = Priority: This pdu contains a PRIORITY
               ...0 .... .... .... .... .... .... .... = DFS operation: This is a normal operation
               ..0. .... .... .... .... .... .... .... = Replay operation: This is NOT a replay operation
           Chain Offset: 0x00000000
           Message ID: Unknown (12288860)
           Process Id: 0x0000feff
           Tree Id: 0x513362de
           Session Id: 0x00000000b6b8df0a
           Signature: 00000000000000000000000000000000
           [Response to: 175095]
           [Time from request: 0.000300000 seconds]
       Create Response (0x05)
           StructureSize: 0x0009
           Error Context Count: 0
           Reserved: 0x00
           Byte Count: 0
           Error Data: 00
   ```

   可以看到二层帧头和三层包头以及TCP头部和SMB载荷头部都没有问题，问题处在SMB数据载荷，基本为空白，也就是说请求的数据从服务器端发送到客户端为空白数据。

6. microsoft官方解释

> | 0x80000006STATUS_NO_MORE_FILES          | No more files were found that match the file specification. |
> | --------------------------------------- | ---------------------------------------- |
> | 0xC00000BASTATUS_FILE_IS_A_DIRECTORY    | The file that was specified as a target is a directory and the caller specified that it could be anything but a directory. |
> | 0xC000009ASTATUS_INSUFFICIENT_RESOURCES | Insufficient system resources exist to complete the API. |

对解释的解读为服务器端资源不足，无法完成请求，或者资源不足请求的文件夹无法被服务器定位。造成上述原因在于SMB服务器内存资源不足，在同时对大量用户进行异步数据读取写入，并且数据量较大时会出现内存占用过多，无法及时回收，造成以上卡屏、花屏现象。