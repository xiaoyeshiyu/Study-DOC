# Wireshark分析TCP ACK错误实例

## 情况复现

上传文件出现报错，报错如下

![上传文件出现报错](https://cl.ly/2u161P463K11/Image%202017-10-16%20at%2010.59.21%20AM.png)

`testv.mp4上传失败unexpected error -- SyntaxError: Unexpected end of JSON input`

然而，用VPN连接到校园网进行上传没有出现报错，同时检查服务器各项服务配置文件，均为发现有异常，判断问题出现在网络。

## Wireshark抓包分析

抓包分析可以看到大量的TCP传输失败

``` scss
No.     Time           Source                Destination           Protocol Length Info
  44072 67.553776      192.168.166.91        192.168.88.168        TCP      60     80 → 62862 [RST, ACK] Seq=64527 Ack=905514 Win=4210688 Len=0
  44073 67.553777      192.168.166.91        192.168.88.168        TCP      60     80 → 62862 [RST, ACK] Seq=64527 Ack=906974 Win=4210688 Len=0
  44074 67.553777      192.168.166.91        192.168.88.168        TCP      60     80 → 62862 [RST, ACK] Seq=64527 Ack=908434 Win=4210688 Len=0
  44075 67.553778      192.168.166.91        192.168.88.168        TCP      60     80 → 62862 [RST, ACK] Seq=64527 Ack=909894 Win=4210688 Len=0
  44076 67.553778      192.168.166.91        192.168.88.168        TCP      60     80 → 62862 [RST, ACK] Seq=64527 Ack=911354 Win=4210688 Len=0
```

可以看到，出现TCP传输失败的包都是从192.168.166.91（服务器）传到192.168.88.168（客户端）的包，并且序列号均为`64527`，确认号均不一样，从Wiresharek回复的结果来看，是服务器发送重置连接的请求给客户端。

包详细内容如下

``` scss
No.     Time           Source                Destination           Protocol Length Info
  44075 67.553778      192.168.166.91        192.168.88.168        TCP      60     80 → 62862 [RST, ACK] Seq=64527 Ack=909894 Win=4210688 Len=0

Frame 44075: 60 bytes on wire (480 bits), 60 bytes captured (480 bits) on interface 0
Ethernet II, Src: Hangzhou_32:3a:01 (b0:f9:63:32:3a:01), Dst: Micro-St_d4:22:dd (8c:89:a5:d4:22:dd)
Internet Protocol Version 4, Src: 192.168.166.91, Dst: 192.168.88.168
Transmission Control Protocol, Src Port: 80, Dst Port: 62862, Seq: 64527, Ack: 909894, Len: 0
    Source Port: 80
    Destination Port: 62862
    [Stream index: 130]
    [TCP Segment Len: 0]
    Sequence number: 64527    (relative sequence number)
    Acknowledgment number: 909894    (relative ack number)
    0101 .... = Header Length: 20 bytes (5)
    Flags: 0x014 (RST, ACK)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Nonce: Not set
        .... 0... .... = Congestion Window Reduced (CWR): Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...1 .... = Acknowledgment: Set
        .... .... 0... = Push: Not set
        .... .... .1.. = Reset: Set
            [Expert Info (Warning/Sequence): Connection reset (RST)]
        .... .... ..0. = Syn: Not set
        .... .... ...0 = Fin: Not set
        [TCP Flags: ·······A·R··]
    Window size value: 8224
    [Calculated window size: 4210688]
    [Window size scaling factor: 512]
    Checksum: 0xcf9c [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    [SEQ/ACK analysis]
        [This is an ACK to the segment in frame: 43651]
        [The RTT to ACK the segment was: 0.048690000 seconds]
        [iRTT: 0.004349000 seconds]
```

出现报错的时候的数据包如下

``` scss
No.     Time           Source                Destination           Protocol Length Info
  44793 67.886898      192.168.88.168        192.168.166.91        TCP      874    [TCP Retransmission] 62887 → 80 [PSH, ACK] Seq=1 Ack=1 Win=65536 Len=820

Frame 44793: 874 bytes on wire (6992 bits), 874 bytes captured (6992 bits) on interface 0
Ethernet II, Src: Micro-St_d4:22:dd (8c:89:a5:d4:22:dd), Dst: Hangzhou_32:3a:01 (b0:f9:63:32:3a:01)
Internet Protocol Version 4, Src: 192.168.88.168, Dst: 192.168.166.91
Transmission Control Protocol, Src Port: 62887, Dst Port: 80, Seq: 1, Ack: 1, Len: 820
    Source Port: 62887
    Destination Port: 80
    [Stream index: 155]
    [TCP Segment Len: 820]
    Sequence number: 1    (relative sequence number)
    [Next sequence number: 821    (relative sequence number)]
    Acknowledgment number: 1    (relative ack number)
    0101 .... = Header Length: 20 bytes (5)
    Flags: 0x018 (PSH, ACK)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Nonce: Not set
        .... 0... .... = Congestion Window Reduced (CWR): Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...1 .... = Acknowledgment: Set
        .... .... 1... = Push: Set
        .... .... .0.. = Reset: Not set
        .... .... ..0. = Syn: Not set
        .... .... ...0 = Fin: Not set
        [TCP Flags: ·······AP···]
    Window size value: 256
    [Calculated window size: 65536]
    [Window size scaling factor: 256]
    Checksum: 0x83a3 [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    [SEQ/ACK analysis]
        [iRTT: 0.003539000 seconds]
        [Bytes in flight: 820]
        [Bytes sent since last PSH flag: 820]
        [TCP Analysis Flags]
            [Expert Info (Note/Sequence): This frame is a (suspected) retransmission]
            [The RTO for this segment was: 0.299346000 seconds]
            [RTO based on delta from frame: 44270]
    TCP payload (820 bytes)
    Retransmitted TCP segment data (820 bytes)
```

这个数据包是产生报错的时候发生的，从括号中的说明中可以看到的是这个数据包是客户端发送给服务器的，分析TCP的ACK有问题。

## 分析

首先从服务器端大量发送相同序列号和不同Ack到客户端可以分析出由于Ack没有回到发送的地方，所以进行重传，所有的重传包都是由服务器发送，表明客户端配置应该存储问题。

从Wireshark中的analysis分析可以看到，客户端接收到其他网络信息中，BAD TCP也会产生。

![BAD TCP](https://cl.ly/1f392W2t1q3x/Image%202017-10-16%20at%203.16.33%20PM.png)