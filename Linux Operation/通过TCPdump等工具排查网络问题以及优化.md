# Tcpdump

​	作为技术支持，有时候会遇到与 LAN 或 WAN 中的问题相关或由其直接导致的问题，这是不可避免的。在这些情况下，在求助于二级支持或者现场网络管理员之前，最好对问题进行初步诊断，这有助于识别问题的根源，至少可以给出进一步研究的大方向。

​	在需要对网络问题进行简单的分析的时候，掌握一个重要的诊断工具是必须的，这里推荐的基于UNIX平台的上的Tcpdump，它可以帮助解决与TCP/IP网络相关的问题。

## 步骤分析

网络中相关问题大多数可以通过其他方法进行检测，例如二层链路中的arp，三层中的路由，四层中的端口等，而Tcpdump在能解决以上层面的问题的同时，也能针对上三层进行判断。首先介绍下四层问题诊断的几种工具。

### 分析网络连通性

​	诊断任何网络相关问题的第一步都是检查目标主机是否正在运行。可以使用 `ping`检查是否可以通过网络访问某一主机。这个命令向主机发送一个` Internet Control Message Protocol (ICMP) `回显请求数据包，然后等待回显回复。

``` shell
# ping infinity02 
PING infinity02 (10.124.241.12) 56(84) bytes of data.
64 bytes from infinity02 (10.124.241.12): icmp_seq=1 ttl=64 time=0.146 ms
64 bytes from infinity02 (10.124.241.12): icmp_seq=2 ttl=64 time=0.127 ms
64 bytes from infinity02 (10.124.241.12): icmp_seq=3 ttl=64 time=0.087 ms
64 bytes from infinity02 (10.124.241.12): icmp_seq=4 ttl=64 time=0.121 ms
^C
--- infinity02 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.087/0.120/0.146/0.022 ms
#
```

成功的 `ping` 意味着：

- 您的主机有一个活跃的网络适配器，可以使用它发送请求。
- 目标主机正在运行，而且在您使用的 IP 地址上配置了活跃的网络适配器。
- 如果使用主机名而不是 IP 地址，就说明名称解析对于这个主机是有效的。
- 在您的主机和目标主机之间有双向路由。
- 在两个主机之间的路由上或在这两个主机上运行的防火墙不会阻挡 ICMP 通信流。

成功的 `ping` 的输出还有助于判断网络延迟，因为它会报告收到回显回复花费的时间。响应时间长很可能意味着与目标主机交换数据的应用程序的性能会比较差。

如果没有收到回显回复，就说明不满足上述一个或多个条件，`ping` 失败了。当收到的数据包数量少于发送的数量，数据包损失大于 0 时，`ping` 失败。

``` shell
# ping test.com
PING test.com (69.172.200.235) 56(84) bytes of data.
From if-ae-30-2.tcore1.EQL-Los-Angeles.as6453.net (206.82.129.18) icmp_seq=5 Packet filtered
^C
--- test.com ping statistics ---
10 packets transmitted, 0 received, +1 errors, 100% packet loss, time 8999ms
# 
```

如果 `ping` 不成功，可以使用 `ifconfig` 检查用来发送请求的适配器是否启动了。

可以使用 `ifconfig` 命令显示某一适配器的状态，还可以使用 `-a` 选项检查所有适配器。应该确认用来向主机发送请求的适配器显示为 `UP` 和 `RUNNING`。如果不是这样，就需要进一步检查。

``` shell
# ifconfig enp8s0f0 
enp8s0f0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.124.241.11  netmask 255.255.255.0  broadcast 10.124.241.255
        inet6 fe80::21b:21ff:fea6:d37c  prefixlen 64  scopeid 0x20<link>
        ether 00:1b:21:a6:d3:7c  txqueuelen 1000  (Ethernet)
        RX packets 11331201767  bytes 5948435198177 (5.4 TiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 11460625972  bytes 6187803778951 (5.6 TiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
# ifconfig -a 
bond0: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet 192.168.1.160  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::21b:21ff:fea6:dc4d  prefixlen 64  scopeid 0x20<link>
        ether 00:1b:21:a6:dc:4d  txqueuelen 1000  (Ethernet)
        RX packets 577072427  bytes 117320042302 (109.2 GiB)
        RX errors 0  dropped 8  overruns 0  frame 0
        TX packets 564225166  bytes 101491562063 (94.5 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
#
```

### 二层链路检查

可以用 `ethtool` 显示适配器的以太网统计数据，包括与设备相关的统计数据。还可以用这个命令显示链路状态（打开或关闭）和介质速度（例如 1000Mbps Full Duplex）。如果需要根据适配器连接的链路伙伴或网络检查设置，介质速度是有用的，因为速度或双工不匹配会造成问题。

``` shell
#  ethtool enp8s0f0 
Settings for enp8s0f0:
	Supported ports: [ FIBRE ]
	Supported link modes:   10000baseT/Full 
	Supported pause frame use: No
	Supports auto-negotiation: No
	Advertised link modes:  10000baseT/Full 
	Advertised pause frame use: No
	Advertised auto-negotiation: No
	Speed: 10000Mb/s
	Duplex: Full
	Port: Direct Attach Copper
	PHYAD: 0
	Transceiver: external
	Auto-negotiation: off
	Supports Wake-on: d
	Wake-on: d
	Current message level: 0x00000007 (7)
			       drv probe link
	Link detected: yes
	#
```

### 三层路由检查

如果适配器已经打开了，可以使用 `route` 检查从您的主机到目标主机的路由是否正确。如果根本没有路由，`ping` 会通知您，但是如果有路由，就需要让网络管理员检查路由是否正确。根据主机使用的路由表中的信息。

``` shell
# route 
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    0      0        0 bond0
10.124.241.0    0.0.0.0         255.255.255.0   U     0      0        0 enp8s0f0
link-local      0.0.0.0         255.255.0.0     U     1004   0        0 enp8s0f0
link-local      0.0.0.0         255.255.0.0     U     1006   0        0 enp130s0f0
link-local      0.0.0.0         255.255.0.0     U     1014   0        0 bond0
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 bond0
#
```

如果路由是正确的，可以使用 `tracepath` 查明数据包通过网络发送到目标主机的准确路由。成功的 `tracepath` 的输出显示数据包经过的每个路由器，以及到达各个路由器花费的最小、平均和最大响应时间。

``` shell
# tracepath www.baidu.com
 1?: [LOCALHOST]                                         pmtu 1500
 1:  gateway                                               0.198ms 
 1:  gateway                                               0.241ms 
 2:  211.144.114.62                                        5.280ms 
 3:  172.16.55.1                                           0.822ms 
 4:  211.144.115.46                                        1.439ms
 #
```

不成功的 `traceroute`在时间字段中显示星号（`reply`），这是因为对下一个路由器的探测超时了，所以无法判断出响应时间。这个示例还演示 `-n` 选项的用途，它输出数字的主机地址，从而避免名称查询和解析，可以提高跟踪速度。

``` shell
# tracepath -n www.baidu.com/80
 1?: [LOCALHOST]                                         pmtu 1500
 1:  192.168.1.1                                           0.137ms 
 1:  192.168.1.1                                           0.108ms 
 2:  211.144.114.62                                       31.950ms 
 3:  172.16.55.1                                          11.917ms 
 4:  211.144.115.46                                        5.213ms 
 5:  211.144.99.153                                        5.802ms 
 6:  125.215.62.5                                          4.200ms 
 7:  211.144.99.190                                      144.144ms asymm  8 
 8:  211.144.99.177                                       26.731ms asymm  7 
 9:  220.248.104.37                                       11.713ms asymm 11 
10:  no reply
#
```

### 四层端口检测

在 TCP/IP 网络的应用层上运行的服务监听一个或多个端口，这些端口用于在客户机和由传输层管理的主机服务器之间交换数据。如果存在到主机的有效路由，而且它能够响应 ping，但是应用服务没有响应（可能ICMP协议被防火墙干掉），那么可以使用 `telnet` 检查到相关端口的连接。

基本形式的 `telnet` 建立到一个主机的终端连接。但是，也可以使用它建立到主机上特定端口的连接（默认端口是 23，telnet 服务）。标准端口的列表见 `/etc/services`。

如果连接成功，消息显示 telnet 转义序列。输入这个键序列（通常是 Control-]）回到 `telnet>` 提示，输入 `quit` 返回到 shell 提示。

``` shell
$ telnet 192.168.1.159 80


Connecting to 192.168.1.159:80...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.


Escape to local shell...... To return to remote host, enter "exit".
To close connection, enter "disconnect".

Type `help' to learn how to use Xshell prompt.
$
```

如果连接失败，那么会显示连接超时或拒绝连接消息。这个消息意味着目标主机上没有运行这个服务（因此没有服务监听这个端口），或者主机上（或路由上其他地方）运行的防火墙阻止连接这个端口。

``` shell
$ telnet 192.168.1.159 443


Connecting to 192.168.1.159:443...
Could not connect to '192.168.1.159' (port 443): Connection failed.

Type `help' to learn how to use Xshell prompt.
$
```

### 域名解析

当在应用程序或本文讨论的任何诊断命令中使用主机名时，主机名必须能够解析为 IP 地址。TCP/IP 网络的 Internet 层使用 IP 地址处理数据包。

必须通过` /etc/irs.conf `和` /etc/netsvc.conf `中指定的名称解析服务之一解析主机名。`hosts` 记录决定执行名称解析的次序。这里只讨论本地和 BIND/DNS 解析；其他方法超出了本文的范围。

如果指定 `local`，就使用` /etc/hosts `文件解析主机名。因此，要检查是否有针对目标主机的条目。

``` shell
# grep infinity02 /etc/hosts
10.124.241.12 infinity02
#
```

如果指定 `bind` 或 `dns`，那么使用 DNS 解析主机名，可以使用 `nslookup` 检查是否可以解析主机名。

``` shell
# nslookup infinity02 
Server:		223.5.5.5
Address:	223.5.5.5#53

** server can't find infinity02: NXDOMAIN
#
```

更强大的 DNS 检查工具是 `dig`。这个命令具有比 `nslookup` 多得多的选项和参数。后者有提供更多功能的交互模式。因此，对于复杂的查询，尤其是输出将由脚本解析的查询，最好使用 `dig`。

``` shell
# dig -x 8.8.8.8

; <<>> DiG 9.9.4-RedHat-9.9.4-38.el7_3 <<>> -x 8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32991
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;8.8.8.8.in-addr.arpa.		IN	PTR

;; ANSWER SECTION:
8.8.8.8.in-addr.arpa.	23041	IN	PTR	google-public-dns-a.google.com.

;; Query time: 10 msec
;; SERVER: 223.5.5.5#53(223.5.5.5)
;; WHEN: Tue Feb 27 16:28:12 CST 2018
;; MSG SIZE  rcvd: 82
#
```

### 二层ARP解析

主机使用` Address Resolution Protocol (ARP) `表或` arp `缓存记录其他网络设备的` media access control (MAC) `地址及其 IP 地址。TCP/IP 网络的链路层使用设备的 MAC 地址，所以要使用 ARP 表在 MAC 地址和 IP 地址之间进行转换。如果您的主机曾经与另一个主机成功地通信，那么在 ARP 表中很可能有相应的条目。可以使用 `arp` 显示特定主机的条目（如果有的话）。

``` shell
# arp infinity02 
Address                  HWtype  HWaddress           Flags Mask            Iface
infinity02               ether   90:e2:ba:00:36:00   C                     enp8s0f0
#
```

还可以使用 `-a` 选项显示整个表。`-n` 选项指定不通过主机名而通过 IP 地址解析。

``` shell
# arp -na 
? (192.168.1.172) at fa:16:3e:c7:f5:ab [ether] on bond0
? (10.124.241.13) at 00:1b:21:94:d9:1c [ether] on enp8s0f0
? (192.168.1.14) at f4:8e:38:02:6a:c4 [ether] on bond0
? (192.168.1.12) at 28:d2:44:77:5d:4f [ether] on bond0
? (192.168.1.162) at 00:1b:21:94:d9:1d [ether] on bond0
? (192.168.1.190) at <incomplete> on bond0
? (192.168.1.188) at <incomplete> on bond0
? (192.168.1.186) at fa:16:3e:ce:dc:bb [ether] on bond0
? (192.168.1.82) at <incomplete> on bond0
#
```

### TCP建立三次握手，以及端口检测

TCP 使用三阶段握手建立连接。客户机通过发送 SYN 同步数据包发起到一个主机（和特定端口）的连接。成功地收到这个数据包之后，主机发送回 SYN-ACK 确认数据包。如果客户机成功地收到这个确认，就通过发送 ACK 确认完成握手。这个过程要求主机服务器正在监听指定的端口，在客户机和主机之间有双向路由，而且没有防火墙阻挡这种通信流。

可以使用 `netstat` 显示从您的主机到其他主机的现有连接及其当前状态。通过在这个命令中使用 `-a` 选项（显示所有套接字的状态）和 `-n` 选项（显示数字地址，避免查找），可以把输出管道连接到适当的 `grep` 以查找处于特定状态的连接（例如 ESTABLISHED 代表完成握手之后活跃的连接）或者到特定主机或端口的连接。

一下分别显示到特定 IP 地址的所有连接（两个到192.168.1.161的连接，它们都已经完全建立了）、到特定 IP 地址上特定端口的连接（192.168.1.161 上的端口 80）以及到任何主机已经完全建立的连接。

``` shell
# netstat -an | grep 192.168.1.161
tcp        0      0 192.168.1.159:3306      192.168.1.161:57164     ESTABLISHED
tcp        0      0 192.168.1.159:3306      192.168.1.161:52552     ESTABLISHED
tcp        0      0 192.168.1.159:3306      192.168.1.161:47240     ESTABLISHED
tcp        0      0 192.168.1.159:3306      192.168.1.161:53184     ESTABLISHED
tcp        0      0 192.168.1.159:3306      192.168.1.161:47551     ESTABLISHED
tcp        0      0 192.168.1.159:3306      192.168.1.161:46440     ESTABLISHED

# netstat -an | grep 192.168.1.161:80
tcp        0      0 192.168.1.160:59980     192.168.1.161:80        TIME_WAIT  

# netstat -an | grep ESTABLISHED
tcp        0      0 10.124.241.11:36678     10.124.241.12:6907      ESTABLISHED
tcp        0      0 10.124.241.11:6808      10.124.241.12:35250     ESTABLISHED
tcp        0      0 10.124.241.11:6820      10.124.241.13:57076     ESTABLISHED
#
```

## Tcpdump介绍

### Tcpdump的安装

　　在linux下tcpdump的安装十分简单，一般由两种安装方式。一种是以rpm包的形式来进行安装。另外一种是以源程序的形式安装。下面介绍rpm形式安装。

​	这种形式的安装是最简单的安装方法，rpm包是将软件编译后打包成二进制的格式，通过rpm命令可以直接安装，不需要修改任何东西。以超级用户登录，使用命令如下：

``` shell
# rpm -ivh tcpdump-4.9.0-5.el7.x86_64.rpm
或者
# yum install -y tcpdump.x86_64
#
```

### Tcpdump的使用

```shell
tcpdump [-aAbdDefhHIJKlLnNOpqStuUvxX#] [ -B size ] [ -c count ]
		[ -C file_size ] [ -E algo:secret ] [ -F file ] [ -G seconds ]
		[ -i interface ] [ -j tstamptype ] [ -M secret ] [ --number ]
		[ -Q|-P in|out|inout ]
		[ -r file ] [ -s snaplen ] [ --time-stamp-precision precision ]
		[ --immediate-mode ] [ -T type ] [ --version ] [ -V file ]
		[ -w file ] [ -W filecount ] [ -y datalinktype ] [ -z postrotate-command ]
		[ -Z user ] [ expression ]
```

`-a` 		将网络地址和广播地址转变成容易识别的名字 ；
`-A`		**只使用 ascii 打印报文的全部数据，不要和 `-X` 一起使用**
`-c number`	 **截取 number 个报文，然后结束**
`-d` 		将已截获的数据包的代码以人容易理解的格式输出； 
`-dd` 	将已截获的数据包的代码以C程式的格式输出； 
`-ddd` 	将已截获的数据包的代码以十进制格式输出； 
`-e` 		输出数据链路层的头部信息； 
`-f` 		将internet地址以数字形式输出；
`-F` 		从指定的文档中读取过滤规则，忽略命令行中指定的其他过滤规则；
`-i` 		**指定监听的网络接口**； 
`-l` 		将标准输出变为行缓冲方式； 
`-n` 		**不将网络地址转换成易识别的主机名，只以数字形式列出主机地址(如IP地址)，这样能够避免DNS查询**； 
`-nn `		**不要解析域名和端口**
`-S`   	**显示绝对的序列号（sequence number），而不是相对编号**。
`-r` 		从指定的文档中读取数据包(该文档一般通过-w选项产生)； 
`-t` 		不输出时间戳；
`-T` 		**将截获的数据包直接解释为指定类型的报文，现在支持的类型有cnfp、rpc、rtp、snmp、vat和wb；** 
`-v` 		输出较周详的信息，例如IP包中的TTL和服务类型信息； 
`-vv` 	输出详尽的报文信息； 
`-v, -vv, -vvv`  **显示更多的详细信息 **
`-w` 		将截获的数据包直接写入指定的文档中，不对其进行分析和输出；
`-X`		**同时用 hex 和 ascii 显示报文的内容。**
`-XX`     	同 `-X`，但同时显示以太网头部。

### 使用方式

#### 简单使用

1. `tcpdump -nS` 

   **监听所有端口，直接显示 ip 地址**

2. `tcpdump -nnvvS`

   **显示更详细的数据报文，包括 tos, ttl, checksum 等**

3. `tcpdump -nnvvXS`

   **显示数据报的全部数据信息，用 hex 和 ascii 两列对比输出**

抓ICMP的请求包和回复包查看所有数据

``` shell
# tcpdump -nnnvvvXSs 0 -c 2 icmp -i ens785f0
tcpdump: listening on ens785f0, link-type EN10MB (Ethernet), capture size 262144 bytes
18:27:49.126951 IP (tos 0x0, ttl 64, id 38785, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.124.11 > 10.10.124.12: ICMP echo request, id 33622, seq 1, length 64
	0x0000:  4500 0054 9781 4000 4001 96fc 0a0a 7c0b  E..T..@.@.....|.
	0x0010:  0a0a 7c0c 0800 a7b5 8356 0001 a5d5 975a  ..|......V.....Z
	0x0020:  0000 0000 cfef 0100 0000 0000 1011 1213  ................
	0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223  .............!"#
	0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233  $%&'()*+,-./0123
	0x0050:  3435 3637                                4567
18:27:49.127084 IP (tos 0x0, ttl 64, id 14152, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.124.12 > 10.10.124.11: ICMP echo reply, id 33622, seq 1, length 64
	0x0000:  4500 0054 3748 0000 4001 3736 0a0a 7c0c  E..T7H..@.76..|.
	0x0010:  0a0a 7c0b 0000 afb5 8356 0001 a5d5 975a  ..|......V.....Z
	0x0020:  0000 0000 cfef 0100 0000 0000 1011 1213  ................
	0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223  .............!"#
	0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233  $%&'()*+,-./0123
	0x0050:  3435 3637                                4567
2 packets captured
2 packets received by filter
0 packets dropped by kernel
#
```

#### 过滤器

机器上的网络报文数量异常的多，为了能直接分析具体问题有关的数据包（比如访问某个网站的数据，或者 icmp 超时的报文等等），而这些数据只占到很小的一部分。把所有的数据截取下来，从里面找到想要的信息无疑是一件很费时费力的工作。而 tcpdump 提供了灵活的语法可以精确地截取关心的数据报，简化分析的工作量。这些选择数据包的语句就是过滤器（filter）！

过滤器也可以简单地分为三类：`type`, `dir` 和 `proto`。

`Type` 让你区分报文的类型，主要由 `host`（主机）, `net`（网络） 和 `port`（端口） 组成。`src` 和 `dst` 也可以用来过滤报文的源地址和目的地址。

**host: 过滤某个主机的数据报文**

```shell
# tcpdump host 1.2.3.4
```

**src, dst: 过滤源地址和目的地址**

```shell
# tcpdump src 2.3.4.5
# tcpdump dst 3.4.5.6
```

**net: 过滤某个网段的数据，CIDR模式**

```shell
# tcpdump net 1.2.3.0/24
```

**proto: 过滤某个协议的数据，支持 tcp, udp 和 icmp。使用的时候可以省略 proto 关键字。**

```shell
# tcpdump icmp
```

**port: 过滤通过某个端口的数据报**

```shell
# tcpdump port 3389
```

**src/dst, port, protocol: 结合三者**

```shell
# tcpdump src port 1025 and tcp
# tcpdump udp and src port 53
```

此外还有指定端口和数据报文范围的过滤器：

**port 范围**

```shell
# tcpdump portrange 21-23
```

**数据报大小，单位是字节**

```shell
# tcpdump less 32
# tcpdump greater 128
# tcpdump > 32
# tcpdump <= 128
```

过于过滤器的更多详细信息，可访问 tcpdump 官方 map page 的 [PCAP-FILTER 部分](http://www.tcpdump.org/manpages/pcap-filter.7.html)。

### 输出到文件

使用 tcpdump 截取数据报文的时候，默认会打印到屏幕的默认输出，你会看到按照顺序和格式，很多的数据一行行快速闪过，根本来不及看清楚所有的内容。不过，tcpdump 提供了把截取的数据保存到文件的功能，以便后面使用其他图形工具（比如 wireshark，Snort）来分析。

`-w` 选项用来把数据报文输出到文件，比如下面的命令就是把所有 80 端口的数据导入到文件

```shell
# tcpdump -w capture_file.pcap port 80
```

`-r` 可以读取文件里的数据报文，显示到屏幕上。

```shell
# tcpdump -nXr capture_file.pcap host web30
```

**NOTE：保存到文件的数据不是屏幕上看到的文件信息，而是包含了额外信息的固定格式 pcap，需要特殊的软件来查看，使用 vim 或者 cat 命令会出现乱码。**

### 强大的过滤器

过滤的真正强大之处在于你可以随意组合它们，而连接它们的逻辑就是常用的 `与/AND/&&`、 `或/OR/||` 和 `非/not/!`。

**源地址是 10.5.2.3，目的端口是 3389 的数据报**

```shell
# tcpdump -nnvS src 10.5.2.3 and dst port 3389
```

**从 192.168 网段到 10 或者 172.16 网段的数据报**

```shell
# tcpdump -nvX src net 192.168.0.0/16 and dat net 10.0.0.0/8 or 172.16.0.0/16
```

**从 Mars 或者 Pluto 发出的数据报，并且目的端口不是 22**

```shell
# tcpdump -vv src mars or pluto and not dat port 22
```

对于比较复杂的过滤器表达式，为了逻辑的清晰，可以使用括号。不过默认情况下，tcpdump 把 `()` 当做特殊的字符，所以必须使用单引号 `'` 来消除歧义：

```shell
# tcpdump -nvv -c 20 'src 10.0.2.4 and (dat port 3389 or 22)'
```

## Tcpdump使用示例

使用 `tcpdump` 监视从特定适配器发送出的数据，这个命令在发送每个数据包时显示其内容。通过使用这个命令的各种选项，可以以描述形式或原始形式显示更多或更少的数据包，可以通过布尔表达式筛选希望看到的数据类型。例如，在监视适配器 `ens785f0` 上的数据包时，可以只显示发送到特定主机的数据。

``` shell
# tcpdump -i ens785f0 dst host hyhive02 | more
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens785f0, link-type EN10MB (Ethernet), capture size 262144 bytes
17:33:04.223786 IP hyhive01.mysql > hyhive02.43146: Flags [P.], seq 924326488:924326599, ack 863908697, win 1432, options [nop,nop,TS
 val 1415643095 ecr 1416348117], length 111
17:33:04.225244 IP hyhive01.mysql > hyhive02.43146: Flags [P.], seq 111:122, ack 68, win 1432, options [nop,nop,TS val 1415643097 ecr
 1416348123], length 11
17:33:04.231028 IP hyhive01.mysql > hyhive02.43146: Flags [P.], seq 122:233, ack 212, win 1432, options [nop,nop,TS val 1415643102 ec
r 1416348125], length 111
17:33:04.231117 IP hyhive01.35406 > hyhive02.smc-https: Flags [.], ack 4292655759, win 7393, options [nop,nop,TS val 1415643103 ecr 1
416348090], length 0
17:33:04.232414 IP hyhive01.mysql > hyhive02.43146: Flags [P.], seq 233:244, ack 279, win 1432, options [nop,nop,TS val 1415643104 ec
r 1416348130], length 11
#
```

可以只显示来自特定主机的数据包

``` shell
# tcpdump -i ens785f0 src host hyhive02 | more
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens785f0, link-type EN10MB (Ethernet), capture size 262144 bytes
17:33:54.303438 IP hyhive02.43146 > hyhive01.mysql: Flags [P.], seq 865302774:865302841, ack 925132653, win 254, options [nop,nop,TS 
val 1416398202 ecr 1415693174], length 67
17:33:54.305047 IP hyhive02.43146 > hyhive01.mysql: Flags [P.], seq 67:211, ack 12, win 254, options [nop,nop,TS val 1416398203 ecr 1
415693175], length 144
17:33:54.310991 IP hyhive02.43146 > hyhive01.mysql: Flags [P.], seq 211:278, ack 123, win 254, options [nop,nop,TS val 1416398209 ecr
 1415693181], length 67
17:33:54.312514 IP hyhive02.43146 > hyhive01.mysql: Flags [P.], seq 278:422, ack 134, win 254, options [nop,nop,TS val 1416398211 ecr
 1415693183], length 144
17:33:54.313959 IP hyhive02.25672 > hyhive01.52588: Flags [P.], seq 3554009917:3554010133, ack 1031241662, win 2672, options [nop,nop
,TS val 1416398212 ecr 1415692793], length 216
17:33:54.314002 IP hyhive02.25672 > hyhive01.52588: Flags [P.], seq 216:1296, ack 1, win 2672, options [nop,nop,TS val 1416398212 ecr
 1415692793], length 1080
17:33:54.317856 IP hyhive02.43146 > hyhive01.mysql: Flags [P.], seq 422:489, ack 245, win 254, options [nop,nop,TS val 1416398216 ecr
 1415693188], length 67
 #
```

可以只显示特定端口收发的数据包

``` shell
# tcpdump -i ens785f0 dst host hyhive02 -vv -c 10
tcpdump: listening on ens785f0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:40:36.958399 IP (tos 0x8, ttl 64, id 44796, offset 0, flags [DF], proto TCP (6), length 63)
    hyhive01.mysql > hyhive02.43146: Flags [P.], cksum 0x0c5c (incorrect -> 0x4ff8), seq 2169601339:2169601350, ack 3017621634, win 1432, options [nop,nop,TS val 1491695830 ecr 1492400856], length 11
14:40:36.964142 IP (tos 0x8, ttl 64, id 44797, offset 0, flags [DF], proto TCP (6), length 163)
    hyhive01.mysql > hyhive02.43146: Flags [P.], cksum 0x0cc0 (incorrect -> 0x9ca5), seq 11:122, ack 145, win 1432, options [nop,nop,TS val 1491695836 ecr 1492400858], length 111
14:40:36.965861 IP (tos 0x8, ttl 64, id 44798, offset 0, flags [DF], proto TCP (6), length 63)
    hyhive01.mysql > hyhive02.43146: Flags [P.], cksum 0x0c5c (incorrect -> 0x4e9c), seq 122:133, ack 212, win 1432, options [nop,nop,TS val 1491695837 ecr 1492400864], length 11
14:40:36.971527 IP (tos 0x8, ttl 64, id 44799, offset 0, flags [DF], proto TCP (6), length 163)
    hyhive01.mysql > hyhive02.43146: Flags [P.], cksum 0x0cc0 (incorrect -> 0x9b49), seq 133:244, ack 356, win 1432, options [nop,nop,TS val 1491695843 ecr 1492400866], length 111
14:40:36.973246 IP (tos 0x8, ttl 64, id 44800, offset 0, flags [DF], proto TCP (6), length 63)
    hyhive01.mysql > hyhive02.43146: Flags [P.], cksum 0x0c5c (incorrect -> 0x4d40), seq 244:255, ack 423, win 1432, options [nop,nop,TS val 1491695845 ecr 1492400871], length 11
# 
```

按 Control-C 停止跟踪。`tcpdump` 命令的特性比这些简单的示例丰富得多，所以建议熟悉它的手册页。

从这三个示例的输出可以看出，这个命令显示以下信息：

- 时间戳
- 源主机.源端口
- 目标主机.目标端口
- 数据包标志
- 其他数据包信息

可以使用这个命令检查应该发送到目标主机的通信流是否离开了选定的主机，以及是否有返回的通信流。如果没有出现入站通信流，可能是主机没有响应，或者在您的主机和目标主机之间没有有效的路由。如果特定的服务（TCP 端口）没有响应或者防火墙阻挡您发送的数据包类型，通常会在数据包标志字段中看到 `R`，这表示连接已经复位。关于 TCP 数据包的准确布局和格式的更多信息，请参考 RFC 793: Transmission Control Protocol（见 [参考资料](https://www.ibm.com/developerworks/cn/aix/library/au-aixnetworkproblem1/index.html#artrelatedtopics) 中的链接）。

根据问题的性质，有时候最好运行 `tcpdump` 一段时间，同时使用 `-w` 选项把数据包信息捕捉到文件中。捕捉到足够的数据之后，按 Control-C 停止跟踪。可以使用 `-r`选项处理这个文件，读取捕捉的数据包数据。然后可以使用各种选项和布尔参数分析数据。

``` shell
# tcpdump -i ens785f0 dst host hyhive02 -vv -w test1.pcap
tcpdump: listening on ens785f0, link-type EN10MB (Ethernet), capture size 262144 bytes
^C2799 packets captured
2824 packets received by filter
13 packets dropped by kernel
#
```

## 理解 tcpdump 的输出

截取数据只是第一步，第二步就是理解这些数据，下面就解释一下 tcpdump 命令输出各部分的意义。

```
21:27:06.995846 IP (tos 0x0, ttl 64, id 45646, offset 0, flags [DF], proto TCP (6), length 64)
    192.168.1.106.56166 > 124.192.132.54.80: Flags [S], cksum 0xa730 (correct), seq 992042666, win 65535, options [mss 1460,nop,wscale 4,nop,nop,TS val 663433143 ecr 0,sackOK,eol], length 0

21:27:07.030487 IP (tos 0x0, ttl 51, id 0, offset 0, flags [DF], proto TCP (6), length 44)
    124.192.132.54.80 > 192.168.1.106.56166: Flags [S.], cksum 0xedc0 (correct), seq 2147006684, ack 992042667, win 14600, options [mss 1440], length 0

21:27:07.030527 IP (tos 0x0, ttl 64, id 59119, offset 0, flags [DF], proto TCP (6), length 40)
    192.168.1.106.56166 > 124.192.132.54.80: Flags [.], cksum 0x3e72 (correct), ack 2147006685, win 65535, length 0
```

最基本也是最重要的信息就是数据报的源地址/端口和目的地址/端口，上面的例子第一条数据报中，源地址 ip 是 `192.168.1.106`，源端口是 `56166`，目的地址是 `124.192.132.54`，目的端口是 `80`。 `>` 符号代表数据的方向。

此外，上面的三条数据还是 tcp 协议的三次握手过程，第一条就是 `SYN` 报文，这个可以通过 `Flags [S]` 看出。下面是常见的 TCP 报文的 Flags:

- `[S]`： SYN（开始连接）
- `[.]`: 没有 Flag
- `[P]`: PSH（推送数据）
- `[F]`: FIN （结束连接）
- `[R]`: RST（重置连接）

而第二条数据的 `[S.]` 表示 `SYN-ACK`，就是 `SYN` 报文的应答报文。

## 参考文档

本文主要参考了下面两篇文章，算是翻译和二次创作。

- [A tcpdump Tutorial and Primer](https://danielmiessler.com/study/tcpdump/)
- [A Quick and Practical Reference for tcpdump](http://bencane.com/2014/10/13/quick-and-practical-reference-for-tcpdump/)
- [RFC 793: Transmission Control Protocol](https://www.ibm.com/developerworks/cn/aix/library/au-aixnetworkproblem1/index.html#artrelatedtopics)
- [UNIX 网络性能分析](https://www.ibm.com/developerworks/cn/aix/library/au-networkperfanalysis/)





# 网络丢包排除补充

在开始之前，先用一张图解释 linux 系统接收网络报文的过程。

1. 首先网络报文通过物理网线发送到网卡
2. 网络驱动程序会把网络中的报文读出来放到 ring buffer 中，这个过程使用 DMA（Direct Memory Access），不需要 CPU 参与
3. 内核从 ring buffer 中读取报文进行处理，执行 IP 和 TCP/UDP 层的逻辑，最后把报文放到应用程序的 socket buffer 中
4. 应用程序从 socket buffer 中读取报文进行处理

![](http://oydlbqndl.bkt.clouddn.com/006tKfTcgy1fnf8b0c64xj31hc0u0goa.jpg)

在接收 UDP 报文的过程中，图中任何一个过程都可能会主动或者被动地把报文丢弃，因此丢包可能发生在网卡和驱动，也可能发生在系统和应用。

之所以没有分析发送数据流程，一是因为发送流程和接收类似，只是方向相反；另外发送流程报文丢失的概率比接收小，只有在应用程序发送的报文速率大于内核和网卡处理速率时才会发生。

NOTE：文中出现的 `RX`（receive） 表示接收报文，`TX`（transmit） 表示发送报文。

## 确认有 UDP 丢包发生

要查看网卡是否有丢包，可以使用 `ethtool -S eth0` 查看，在输出中查找 `bad` 或者 `drop`对应的字段是否有数据，在正常情况下，这些字段对应的数字应该都是 0。如果看到对应的数字在不断增长，就说明网卡有丢包。

``` SHELL
# ethtool -S ens785f0 | egrep 'bad|drop'
     rx_dropped: 0
     tx_dropped: 0
     fcoe_bad_fccrc: 0
     rx_fcoe_dropped: 0
#
```

另外一个查看网卡丢包数据的命令是 `ifconfig`，它的输出中会有 `RX`(receive 接收报文)和 `TX`（transmit 发送报文）的统计数据：

``` shell
# ifconfig ens785f0
ens785f0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.10.124.11  netmask 255.255.255.0  broadcast 10.10.124.255
        inet6 fe80::21b:21ff:fea7:c6a8  prefixlen 64  scopeid 0x20<link>
        ether 00:1b:21:a7:c6:a8  txqueuelen 1000  (Ethernet)
        RX packets 2894729231  bytes 934132986909 (869.9 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3101312636  bytes 1235029705263 (1.1 TiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
#
```

此外，linux 系统也提供了各个网络协议的丢包信息，可以使用 `netstat -s` 命令查看，加上 `--udp` 可以只看 UDP 相关的报文数据：

``` shell
# netstat -s -u
IcmpMsg:
    InType0: 413786
    InType3: 1103
    InType8: 961726
    OutType0: 961726
    OutType3: 207
    OutType8: 413796
Udp:
    48605321 packets received
    139 packets to unknown port received.
    0 packet receive errors
    64345839 packets sent
    0 receive buffer errors
    0 send buffer errors
UdpLite:
IpExt:
    InMcastPkts: 1
    OutMcastPkts: 5776590
    InBcastPkts: 12579233
    OutBcastPkts: 4468
    InOctets: 2018612234664
    OutOctets: 2284128062353
    InMcastOctets: 28
    OutMcastOctets: 231063600
    InBcastOctets: 4091010269
    OutBcastOctets: 1039598
    InNoECTPkts: 3963723378
    InECT0Pkts: 1449
#
```

对于上面的输出，关注下面的信息来查看 UDP 丢包的情况：

- `packet receive errors` 不为空，并且在一直增长说明系统有 UDP 丢包
- `packets to unknown port received` 表示系统接收到的 UDP 报文所在的目标端口没有应用在监听，一般是服务没有启动导致的，并不会造成严重的问题
- `receive buffer errors` 表示因为 UDP 的接收缓存太小导致丢包的数量

**NOTE**： 并不是丢包数量不为零就有问题，对于 UDP 来说，如果有少量的丢包很可能是预期的行为，比如丢包率（丢包数量/接收报文数量）在万分之一甚至更低。

同理，参数为`-t`为TCP相关数据

``` shell
# netstat -s -t 
IcmpMsg:
    InType0: 413782
    InType3: 1103
    InType8: 961721
    OutType0: 961721
    OutType3: 207
    OutType8: 413792
Tcp:
    16567340 active connections openings
    20752499 passive connection openings
    400223 failed connection attempts
    275668 connection resets received
    678 connections established
    3582847102 segments received
    4076984013 segments send out
    1976433 segments retransmited
    0 bad segments received.
    835289 resets sent
UdpLite:
TcpExt:
    192553 invalid SYN cookies received
    81 ICMP packets dropped because they were out-of-window
    16090536 TCP sockets finished time wait in fast timer
    40 packets rejects in established connections because of timestamp
    93945938 delayed acks sent
    6520 delayed acks further delayed because of locked socket
    Quick ack mode was activated 22528 times
    30630846 packets directly queued to recvmsg prequeue.
    74866576 bytes directly in process context from backlog
    12851721593 bytes directly received in process context from prequeue
    2345801160 packet headers predicted
    28479385 packets header predicted and directly queued to user
    155053378 acknowledgments not containing data payload received
    2247255940 predicted acknowledgments
    821 times recovered from packet loss by selective acknowledgements
    Detected reordering 4 times using FACK
    Detected reordering 815 times using SACK
    Detected reordering 519 times using time stamp
    73 congestion windows fully recovered without slow start
    247 congestion windows partially recovered using Hoe heuristic
    136 congestion windows recovered without slow start by DSACK
    59 congestion windows recovered without slow start after partial ack
    TCPLostRetransmit: 21
    6 timeouts after SACK recovery
    11 timeouts in loss state
    32287 fast retransmits
    789 forward retransmits
    72 retransmits in slow start
    71 other TCP timeouts
    TCPLossProbes: 1947510
    TCPLossProbeRecovery: 1934752
    21 SACK retransmits failed
    22549 DSACKs sent for old packets
    13 DSACKs sent for out of order packets
    1944242 DSACKs received
    5 DSACKs for out of order packets received
    199260 connections reset due to unexpected data
    7417 connections reset due to early user close
    18 connections aborted due to timeout
    TCPSACKDiscard: 1
    TCPDSACKIgnoredOld: 30
    TCPDSACKIgnoredNoUndo: 1931732
    TCPSpuriousRTOs: 33
    TCPSackShifted: 4648
    TCPSackMerged: 2756
    TCPSackShiftFallback: 166974
    TCPDeferAcceptDrop: 2082586
    IPReversePathFilter: 12578
    TCPRetransFail: 2
    TCPRcvCoalesce: 68983264
    TCPOFOQueue: 96126
    TCPOFOMerge: 13
    TCPChallengeACK: 6
    TCPSpuriousRtxHostQueues: 11
    TCPAutoCorking: 11926427
    TCPFromZeroWindowAdv: 19461
    TCPToZeroWindowAdv: 19461
    TCPWantZeroWindowAdv: 2190239
    TCPSynRetrans: 18
    TCPOrigDataSent: 3314823161
    TCPHystartTrainDetect: 54214
    TCPHystartTrainCwnd: 1418025
    TCPHystartDelayDetect: 24
    TCPHystartDelayCwnd: 1293
    TCPACKSkippedSynRecv: 8
    TCPACKSkippedPAWS: 2
    TCPACKSkippedSeq: 284
IpExt:
    InMcastPkts: 1
    OutMcastPkts: 5776578
    InBcastPkts: 12579213
    OutBcastPkts: 4468
    InOctets: 2018610840631
    OutOctets: 2284126585331
    InMcastOctets: 28
    OutMcastOctets: 231063120
    InBcastOctets: 4091001105
    OutBcastOctets: 1039598
    InNoECTPkts: 3963717324
    InECT0Pkts: 1449
```

## 网卡或者驱动丢包

判断网卡或者驱动丢包，则是通过 `ethtool -S eth0` 中有无 `rx_***_errors` 

``` shell
# ethtool -S ens785f0 | grep rx_ | grep errors
     rx_errors: 0
     rx_over_errors: 0
     rx_crc_errors: 0
     rx_frame_errors: 0
     rx_fifo_errors: 0
     rx_missed_errors: 0
     rx_long_length_errors: 0
     rx_short_length_errors: 0
     rx_csum_offload_errors: 0
#
```

`netstat -i` 也会提供每个网卡的接发报文以及丢包的情况，正常情况下输出中 error 或者 drop 应该为 0。

``` shell
# netstat -i 
Kernel Interface table
Iface      MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
br-ex     1500  3825278      0      0 0             8      0      0      0 BMRU
eno1      1500  3220078      0      0 0        468449      0      0      0 BMU
eno2      1500 50642352      0     20 0       8232878      0      0      0 BMRU
ens785f0  1500 2895287635      0      0 0      3101890350      0      0      0 BMRU
lo       65536 1041392979      0      0 0      1041392979      0      0      0 LRU
#
```

如果硬件或者驱动没有问题，一般网卡丢包是因为设置的缓存区（ring buffer）太小，可以使用 `ethtool` 命令查看和设置网卡的 ring buffer。`ethtool -g` 可以查看某个网卡的 ring buffer

``` shell
# ethtool -g ens785f0
Ring parameters for ens785f0:
Pre-set maximums:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		4096
Current hardware settings:
RX:		512
RX Mini:	0
RX Jumbo:	0
TX:		512
#
```

Pre-set 表示网卡最大的 ring buffer 值，可以使用 `ethtool -G ens785f0 rx 1024` 设置它的值。

``` shell
# ethtool -G ens785f0 rx 1024
# ethtool -g ens785f0
Ring parameters for ens785f0:
Pre-set maximums:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		4096
Current hardware settings:
RX:		1024
RX Mini:	0
RX Jumbo:	0
TX:		512
#
```

ethtool 工具非常强大，有兴趣可以深入研究下去

例如测试网卡，网卡测试命令： `ethtool -t p5p2 online` 

详细命令为`ethtool -t|--test devname [offline|online|external_lb]`，其中online表示不需要重启网卡的测试，offline表示要重启网卡

``` shell
# ethtool -t ens785f0 online 
The test result is PASS
The test extra info:
Register test  (offline)	 0
Eeprom test    (offline)	 0
Interrupt test (offline)	 0
Loopback test  (offline)	 0
Link test   (on/offline)	 0
```

定位网卡位置：`ethtool -p DEVNAME `  查看相应设备名称对应的设备位置

``` shell
# ethtool -p eno1 
^C
#
```

回车后与eno1 相对应的网卡接口旁边的指示灯就会闪烁，这样就能很快确定 eno1 网口的位置。（按下Ctrl+C 结束命令，停止闪烁）

资料：

[网卡参数查询及设置工具ethtool](http://blog.csdn.net/wuzhimang/article/details/54581019)

[10G(82599EB) 网卡测试优化(ethtool)](http://jaseywang.me/2013/11/02/10g82599eb-%E7%BD%91%E5%8D%A1%E6%B5%8B%E8%AF%95%E4%BC%98%E5%8C%96ethtool/)

[ethtool命令](http://man.linuxde.net/ethtool)

## Linux 系统丢包

linux 系统丢包的原因很多，常见的有：UDP 报文错误、防火墙、UDP buffer size 不足、系统负载过高等，这里对这些丢包原因进行分析。

### UDP报文错误

如果在传输过程中UDP 报文被修改，会导致 checksum 错误，或者长度错误，linux 在接收到 UDP 报文时会对此进行校验，一旦发明错误会把报文丢弃。

如果希望 UDP 报文 checksum 即使有错也要发送给应用程序，可以在通过 socket 参数禁用 UDP checksum 检查（需要有setsockopt）：

``` shell
int disable = 1;
setsockopt(sock_fd, SOL_SOCKET, SO_NO_CHECK, (void*)&disable, sizeof(disable)
```

### 防火墙

如果系统防火墙丢包，表现的行为一般是所有的 UDP 报文都无法正常接收，当然不排除防火墙只 drop 一部分报文的可能性。

如果遇到丢包比率非常大的情况，请先检查防火墙规则，保证防火墙没有主动 drop UDP 报文。

### UDP buffer size 不足

linux 系统在接收报文之后，会把报文保存到缓存区中。因为缓存区的大小是有限的，如果出现 UDP 报文过大（超过缓存区大小或者 MTU 大小）、接收到报文的速率太快，都可能导致 linux 因为缓存满而直接丢包的情况。

在系统层面，linux 设置了` receive buffer `可以配置的最大值，可以在下面的文件中查看，一般是 linux 在启动的时候会根据内存大小设置一个初始值。

- /proc/sys/net/core/rmem_max：允许设置的 receive buffer 最大值
- /proc/sys/net/core/rmem_default：默认使用的 receive buffer 值
- /proc/sys/net/core/wmem_max：允许设置的 send buffer 最大值
- /proc/sys/net/core/wmem_dafault：默认使用的 send buffer 最大值

但是这些初始值并不是为了应对大流量的 UDP 报文，如果应用程序接收和发送 UDP 报文非常多，需要讲这个值调大。可以使用 `sysctl` 命令让它立即生效：

```shell
# sysctl -w net.core.rmem_max=26214400 # 设置为 25M
```

也可以修改 `/etc/sysctl.conf` 中对应的参数在下次启动时让参数保持生效。

如果报文报文过大，可以在发送方对数据进行分割，保证每个报文的大小在 MTU 内。

另外一个可以配置的参数是 `netdev_max_backlog`，它表示 linux 内核从网卡驱动中读取报文后可以缓存的报文数量，默认是 1000，可以调大这个值，比如设置成 2000：

```shel
# sysctl -w net.core.netdev_max_backlog=2000
```

### 系统负载过高

系统 CPU、memory、IO 负载过高都有可能导致网络丢包，比如 CPU 如果负载过高，系统没有时间进行报文的 checksum 计算、复制内存等操作，从而导致网卡或者 socket buffer 出丢包；memory 负载过高，会应用程序处理过慢，无法及时处理报文；IO 负载过高，CPU 都用来响应 IO wait，没有时间处理缓存中的 UDP 报文。

linux 系统本身就是相互关联的系统，任何一个组件出现问题都有可能影响到其他组件的正常运行。对于系统负载过高，要么是应用程序有问题，要么是系统不足。对于前者需要及时发现，debug 和修复；对于后者，也要及时发现并扩容。

## 应用丢包

上面提到系统的 UDP buffer size，调节的 sysctl 参数只是系统允许的最大值，每个应用程序在创建 socket 时需要设置自己 socket buffer size 的值。

linux 系统会把接受到的报文放到 socket 的 buffer 中，应用程序从 buffer 中不断地读取报文。所以这里有两个和应用有关的因素会影响是否会丢包：socket buffer size 大小以及应用程序读取报文的速度。

对于第一个问题，可以在应用程序初始化 socket 的时候设置 socket receive buffer 的大小，比如下面的代码把 socket buffer 设置为 20MB：

```
uint64_t receive_buf_size = 20*1024*1024;  //20 MB
setsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &receive_buf_size, sizeof(receive_buf_size));
```

如果不是自己编写和维护的程序，修改应用代码是件不好甚至不太可能的事情。很多应用程序会提供配置参数来调节这个值，请参考对应的官方文档；如果没有可用的配置参数，只能给程序的开发者提 issue 了。

很明显，增加应用的 receive buffer 会减少丢包的可能性，但同时会导致应用使用更多的内存，所以需要谨慎使用。

另外一个因素是应用读取 buffer 中报文的速度，对于应用程序来说，处理报文应该采取异步的方式

## 包丢在什么地方

想要详细了解 linux 系统在执行哪个函数时丢包的话，可以使用 `dropwatch` 工具，它监听系统丢包信息，并打印出丢包发生的函数地址：

```shell
# dropwatch -l kas
Initalizing kallsyms db
dropwatch> start
Enabling monitoring...
Kernel monitoring activated.
Issue Ctrl-C to stop monitoring
1 drops at skb_queue_purge+18 (0xffffffff8155e028)
1 drops at tcp_rcv_state_process+1b0 (0xffffffff815cb280)
2 drops at unix_stream_connect+2ca (0xffffffff81620aca)
4 drops at unix_stream_connect+2ca (0xffffffff81620aca)
6 drops at unix_stream_connect+2ca (0xffffffff81620aca)
4 drops at unix_release_sock+196 (0xffffffff8161fc16)
4 drops at unix_stream_connect+2ca (0xffffffff81620aca)
1 drops at unix_stream_connect+2ca (0xffffffff81620aca)
2 drops at unix_release_sock+196 (0xffffffff8161fc16)
2 drops at unix_stream_connect+2ca (0xffffffff81620aca)
1 drops at unix_dgram_sendmsg+4d0 (0xffffffff81621150)
^CGot a stop message
dropwatch> Terminating dropwatch...
Shutting down ...
#
```

通过这些信息，找到对应的内核代码处，就能知道内核在哪个步骤中把报文丢弃，以及大致的丢包原因。

此外，还可以使用 linux perf 工具监听 `kfree_skb`（把网络报文丢弃时会调用该函数） 事件的发生：

```shell
# perf record -g -a -e skb:kfree_skb
# perf script
```

关于 perf 命令的使用和解读，网上有很多文章可以参考。

## 总结

- UDP 本身就是无连接不可靠的协议，适用于报文偶尔丢失也不影响程序状态的场景，比如视频、音频、游戏、监控等。对报文可靠性要求比较高的应用不要使用 UDP，推荐直接使用 TCP。当然，也可以在应用层做重试、去重保证可靠性
- 如果发现服务器丢包，首先通过监控查看系统负载是否过高，先想办法把负载降低再看丢包问题是否消失
- 如果系统负载过高，UDP 丢包是没有有效解决方案的。如果是应用异常导致 CPU、memory、IO 过高，请及时定位异常应用并修复；如果是资源不够，监控应该能及时发现并快速扩容
- 对于系统大量接收或者发送 UDP 报文的，可以通过调节系统和程序的 socket buffer size 来降低丢包的概率
- 应用程序在处理 UDP 报文时，要采用异步方式，在两次接收报文之间不要有太多的处理逻辑

# 参考文献

[理解 Linux 网络栈（1）：Linux 网络协议栈简单总结](http://www.openstack.cn/?p=4756)

[linux tcp調優](https://hk.saowen.com/a/46b112fad8536dc310c3522d7e709a3058c0d2ecd61f94c0a88a21989f29b3dd)

[linux下TCP/IP及内核参数优化调优](https://www.sudops.com/linux-kernel-tcp-ip-sysctl-optimize.html)

[linux TCP 和 socket 参数设置](http://blog.csdn.net/bigtree_3721/article/details/51284995)

[TCP 的那些事儿（上）](http://blog.csdn.net/bigtree_3721/article/details/51320422)

[修改Linux内核参数，减少TCP连接中的TIME-WAIT](http://blog.51cto.com/zhangziqiang/500204)