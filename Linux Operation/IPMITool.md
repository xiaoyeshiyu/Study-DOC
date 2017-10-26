# IPMITool和其中常用命令

## IPMI解读

IPMI（IntelligentPlatform Management Interface）即智能平台管理接口是使硬件管理具备“智能化”的新一代通用接口标准。用户可以利用 IPMI 监视服务器的物理特征，如温度、电压、电扇工作状态、电源供应以及机箱入侵等。ipmi 最大的优势在于它是独立于 CPU BIOS 和 OS 的，所以用户无论在开机还是关机的状态下，只要接通电源就可以实现对服务器的监控。ipmi 是一种规范的标准，其中最重要的物理部件就是BMC(Baseboard Management Controller)，一种嵌入式管理微控制器，它相当于整个平台管理的“大脑”，通过它 ipmi 可以监控各个传感器的数据并记录各种事件的日志。 

ipmitool是一种可用在 Linux/Unix 系统下的基于命令行方式的 ipmi 平台管理工具。它支持 ipmi 1.5 和ipmi 2.0 规范（最新的规范为 ipmi 2.0）。利用它可以实现获取传感器的信息、显示系统日志内容、网络远程开关机等功能。其主要功能包括读取和显示传感器数据（SDR），显示System Evernt Log（SEL）的内容，显示打印Field Replaceable Unit（FRU）信息，读取和设置BMC模块的LAN配置，远程控制服务器主机的电源。 

ipmitool支持IPMI-over-LAN和系统Kernel中的设备驱动（openIPMI in Linux, BMC in Solaris, openIPMI in FreeBSD) 接口。即可以本地运行，也可以在远程机器上运行，实现对服务器主机的管理和控制。

## 系统管理命令

### 管理类

1. 查看设备信息

``` shell
[root@hyhive ~]# ipmitool chassis status 
System Power         : on
Power Overload       : false
Power Interlock      : inactive
Main Power Fault     : false
Power Control Fault  : false
Power Restore Policy : previous
Last Power Event     : 
Chassis Intrusion    : inactive
Front-Panel Lockout  : inactive
Drive Fault          : false
Cooling/Fan Fault    : false
```

2. 查看BMC的信息

   ``` shell
   [root@hyhive ~]# ipmitool mc info 
   Device ID                 : 32
   Device Revision           : 1
   Firmware Revision         : 3.45
   IPMI Version              : 2.0
   Manufacturer ID           : 10876
   Manufacturer Name         : Supermicro
   Product ID                : 2097 (0x0831)
   Product Name              : Unknown (0x831)
   Device Available          : yes
   Provides Device SDRs      : no
   Additional Device Support :
       Sensor Device
       SDR Repository Device
       SEL Device
       FRU Inventory Device
       IPMB Event Receiver
       IPMB Event Generator
       Chassis Device
   ```

3. 查看IPMI用户

``` shell
[root@hyhive ~]# ipmitool user list 
ID  Name	     Callin  Link Auth	IPMI Msg   Channel Priv Limit
2   ADMIN            true    false      false      Unknown (0x00)
```

3. 增加用户

``` shell
[root@hyhive ~]# ipmitool user set name 3 test1
[root@hyhive ~]# ipmitool user list 
ID  Name	     Callin  Link Auth	IPMI Msg   Channel Priv Limit
2   ADMIN            true    false      false      Unknown (0x00)
3   test1            true    false      false      Unknown (0x00)
[root@hyhive ~]# ipmitool user set password 3 test1
[root@hyhive ~]# ipmitool user priv 3 20 
[root@hyhive ~]# ipmitool user list 
ID  Name	     Callin  Link Auth	IPMI Msg   Channel Priv Limit
2   ADMIN            true    false      false      Unknown (0x00)
3   test1            true    false      false      ADMINISTRATOR
[root@hyhive ~]# ipmitool -U test1 -P test1 user list 
ID  Name	     Callin  Link Auth	IPMI Msg   Channel Priv Limit
2   ADMIN            true    false      false      Unknown (0x00)
3   test1            true    false      false      ADMINISTRATOR
```

4. 重设密码

``` shell
[root@hyhive ~]# ipmitool user set password 2 ADMIN
## 2指的是user ID，ADMIN指的是之后的密码
```



### 远程电源控制类

1. 查看服务器当前开电状态

``` shell
[root@hyhive ~]# ipmitool power status 
Chassis Power is on
```

2. 服务器的开机，关机，reset和power cycle

``` shell
[root@hyhive ~]# ipmitool power 
chassis power Commands: status, on, off, cycle, reset, diag, soft
```

power cycle 和power reset的区别在于前者从掉电到上电有１秒钟的间隔，而后者是很快上电

### 读取系统状态类

1. 显示系统所有传感器列表

``` shel
[root@hyhive ~]# ipmitool sensor list 
CPU1 Temp        | 44.000     | degrees C  | ok    | 0.000     | 0.000     | 0.000     | 98.000    | 103.000   | 103.000   
CPU2 Temp        | 44.000     | degrees C  | ok    | 0.000     | 0.000     | 0.000     | 98.000    | 103.000   | 103.000   
PCH Temp         | 49.000     | degrees C  | ok    | 0.000     | 5.000     | 16.000    | 90.000    | 95.000    | 100.000   
System Temp      | 34.000     | degrees C  | ok    | -10.000   | -5.000    | 0.000     | 80.000    | 85.000    | 90.000    
Peripheral Temp  | 49.000     | degrees C  | ok    | -10.000   | -5.000    | 0.000     | 80.000    | 85.000    | 90.000    
Vcpu1VRM Temp    | 45.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 95.000    | 100.000   | 105.000   
Vcpu2VRM Temp    | 46.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 95.000    | 100.000   | 105.000   
VmemABVRM Temp   | 36.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 95.000    | 100.000   | 105.000   
VmemCDVRM Temp   | 39.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 95.000    | 100.000   | 105.000   
VmemEFVRM Temp   | 47.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 95.000    | 100.000   | 105.000   
VmemGHVRM Temp   | 40.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 95.000    | 100.000   | 105.000   
P1-DIMMA1 Temp   | 36.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
P1-DIMMB1 Temp   | 36.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
P1-DIMMC1 Temp   | 41.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
P1-DIMMD1 Temp   | 39.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
P2-DIMME1 Temp   | 43.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
P2-DIMMF1 Temp   | 40.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
P2-DIMMG1 Temp   | 36.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
P2-DIMMH1 Temp   | 37.000     | degrees C  | ok    | -5.000    | 0.000     | 5.000     | 80.000    | 85.000    | 90.000    
FAN1             | na         |            | na    | na        | na        | na        | na        | na        | na        
FAN2             | 3100.000   | RPM        | ok    | 300.000   | 500.000   | 700.000   | 25300.000 | 25400.000 | 25500.000 
FAN3             | 3000.000   | RPM        | ok    | 300.000   | 500.000   | 700.000   | 25300.000 | 25400.000 | 25500.000 
FAN4             | 3100.000   | RPM        | ok    | 300.000   | 500.000   | 700.000   | 25300.000 | 25400.000 | 25500.000 
FAN5             | 3200.000   | RPM        | ok    | 300.000   | 500.000   | 700.000   | 25300.000 | 25400.000 | 25500.000 
FAN6             | 3100.000   | RPM        | ok    | 300.000   | 500.000   | 700.000   | 25300.000 | 25400.000 | 25500.000 
FANA             | na         |            | na    | na        | na        | na        | na        | na        | na        
FANB             | na         |            | na    | na        | na        | na        | na        | na        | na        
12V              | 12.189     | Volts      | ok    | 10.173    | 10.299    | 10.740    | 12.945    | 13.260    | 13.386    
5VCC             | 5.052      | Volts      | ok    | 4.246     | 4.298     | 4.480     | 5.390     | 5.546     | 5.598     
3.3VCC           | 3.401      | Volts      | ok    | 2.789     | 2.823     | 2.959     | 3.554     | 3.656     | 3.690     
VBAT             | 2.898      | Volts      | ok    | 2.326     | 2.430     | 2.508     | 3.678     | 3.782     | 3.886     
Vcpu1            | 1.818      | Volts      | ok    | 1.242     | 1.260     | 1.395     | 1.899     | 2.088     | 2.106     
Vcpu2            | 1.818      | Volts      | ok    | 1.242     | 1.260     | 1.395     | 1.899     | 2.088     | 2.106     
VDIMMAB          | 1.209      | Volts      | ok    | 0.948     | 0.975     | 1.047     | 1.344     | 1.425     | 1.443     
VDIMMCD          | 1.209      | Volts      | ok    | 0.948     | 0.975     | 1.047     | 1.344     | 1.425     | 1.443     
VDIMMEF          | 1.218      | Volts      | ok    | 0.948     | 0.975     | 1.047     | 1.344     | 1.425     | 1.443     
VDIMMGH          | 1.209      | Volts      | ok    | 0.948     | 0.975     | 1.047     | 1.344     | 1.425     | 1.443     
5VSB             | 5.026      | Volts      | ok    | 4.246     | 4.298     | 4.480     | 5.390     | 5.546     | 5.598     
3.3VSB           | 3.248      | Volts      | ok    | 2.789     | 2.823     | 2.959     | 3.554     | 3.656     | 3.690     
1.5V PCH         | 1.518      | Volts      | ok    | 1.320     | 1.347     | 1.401     | 1.644     | 1.671     | 1.698     
1.2V BMC         | 1.227      | Volts      | ok    | 1.020     | 1.047     | 1.092     | 1.344     | 1.371     | 1.398     
1.05V PCH        | 1.068      | Volts      | ok    | 0.870     | 0.897     | 0.942     | 1.194     | 1.221     | 1.248     
Chassis Intru    | 0x0        | discrete   | 0x0000| na        | na        | na        | na        | na        | na        
PS2 Status       | 0x1        | discrete   | 0x0100| na        | na        | na        | na        | na        | na        
PS1 Status       | 0x1        | discrete   | 0x0100| na        | na        | na        | na        | na        | na   
```

2. 读取系统传感器SDR信息

``` shell
[root@hyhive ~]# ipmitool sdr info 
SDR Version                         : 0x51
Record Count                        : 52
Free Space                          : 1634 bytes
Most recent Addition                : 
Most recent Erase                   : 
SDR overflow                        : no
SDR Repository Update Support       : non-modal
Delete SDR supported                : yes
Partial Add SDR supported           : yes
Reserve SDR repository supported    : yes
SDR Repository Alloc info supported : yes
```

3. 读取系统所有SDR列表

``` shel
[root@hyhive ~]# ipmitool sdr list 
CPU1 Temp        | 44 degrees C      | ok
CPU2 Temp        | 45 degrees C      | ok
PCH Temp         | 49 degrees C      | ok
System Temp      | 34 degrees C      | ok
Peripheral Temp  | 49 degrees C      | ok
Vcpu1VRM Temp    | 46 degrees C      | ok
Vcpu2VRM Temp    | 47 degrees C      | ok
VmemABVRM Temp   | 36 degrees C      | ok
VmemCDVRM Temp   | 39 degrees C      | ok
VmemEFVRM Temp   | 48 degrees C      | ok
VmemGHVRM Temp   | 41 degrees C      | ok
P1-DIMMA1 Temp   | 36 degrees C      | ok
P1-DIMMB1 Temp   | 36 degrees C      | ok
P1-DIMMC1 Temp   | 41 degrees C      | ok
P1-DIMMD1 Temp   | 39 degrees C      | ok
P2-DIMME1 Temp   | 43 degrees C      | ok
P2-DIMMF1 Temp   | 41 degrees C      | ok
P2-DIMMG1 Temp   | 37 degrees C      | ok
P2-DIMMH1 Temp   | 37 degrees C      | ok
FAN1             | no reading        | ns
FAN2             | 3100 RPM          | ok
FAN3             | 3000 RPM          | ok
FAN4             | 3100 RPM          | ok
FAN5             | 3200 RPM          | ok
FAN6             | 3100 RPM          | ok
FANA             | no reading        | ns
FANB             | no reading        | ns
12V              | 12.19 Volts       | ok
5VCC             | 5.05 Volts        | ok
3.3VCC           | 3.40 Volts        | ok
VBAT             | 2.90 Volts        | ok
Vcpu1            | 1.82 Volts        | ok
Vcpu2            | 1.82 Volts        | ok
VDIMMAB          | 1.21 Volts        | ok
VDIMMCD          | 1.21 Volts        | ok
VDIMMEF          | 1.22 Volts        | ok
VDIMMGH          | 1.21 Volts        | ok
5VSB             | 5.03 Volts        | ok
3.3VSB           | 3.25 Volts        | ok
1.5V PCH         | 1.52 Volts        | ok
1.2V BMC         | 1.23 Volts        | ok
1.05V PCH        | 1.07 Volts        | ok
Chassis Intru    | 0x00              | ok
PS2 Status       | 0x01              | ok
PS1 Status       | 0x01              | ok
```

4. 显示系统可替代器件列表

``` shell
[root@hyhive ~]# ipmitool fru list 
FRU Device Description : Builtin FRU Device (ID 0)
 Board Mfg Date        : Mon Jan  1 08:00:00 1996
 Board Mfg             : Supermicro
 Board Serial          :           
 Product Serial        :    
```

### 系统日志类

1. 显示所有系统事件日志

``` shell
[root@hyhive ~]# ipmitool sel elist 
   1 | 07/04/2017 | 10:22:47 | OS Boot | Installation started () | Asserted
   2 | 07/04/2017 | 10:27:15 | OS Boot | Installation completed () | Asserted
   3 | 07/05/2017 | 15:37:08 | Unknown #0xff |  | Asserted
   4 | 07/05/2017 | 15:38:37 | Unknown #0xff |  | Asserted
   5 | 07/05/2017 | 15:54:47 | OS Boot | Installation started () | Asserted
   6 | 07/05/2017 | 15:58:44 | OS Boot | Installation completed () | Asserted
   7 | 07/10/2017 | 11:14:00 | Power Supply PS1 Status | Failure detected () | Asserted
   8 | 07/10/2017 | 11:14:37 | Power Supply PS1 Status | Failure detected () | Deasserted
```

2. 显示当前BMC时间

``` shell
[root@hyhive ~]# ipmitool sel time get 
08/31/2017 19:18:21
```

### 网络接口相关命令

1. 显示channel 1的相关配置

``` shell
[root@hyhive ~]# ipmitool lan print 1 
Set in Progress         : Set Complete
Auth Type Support       : NONE MD2 MD5 PASSWORD 
Auth Type Enable        : Callback : MD2 MD5 PASSWORD 
                        : User     : MD2 MD5 PASSWORD 
                        : Operator : MD2 MD5 PASSWORD 
                        : Admin    : MD2 MD5 PASSWORD 
                        : OEM      : MD2 MD5 PASSWORD 
IP Address Source       : DHCP Address
IP Address              : 192.168.10.239
Subnet Mask             : 255.255.255.0
MAC Address             : ac:1f:6b:15:dc:96
SNMP Community String   : public
IP Header               : TTL=0x00 Flags=0x00 Precedence=0x00 TOS=0x00
BMC ARP Control         : ARP Responses Enabled, Gratuitous ARP Disabled
Default Gateway IP      : 192.168.10.1
Default Gateway MAC     : 00:00:00:00:00:00
Backup Gateway IP       : 0.0.0.0
Backup Gateway MAC      : 00:00:00:00:00:00
802.1q VLAN ID          : Disabled
802.1q VLAN Priority    : 0
RMCP+ Cipher Suites     : 1,2,3,6,7,8,11,12
Cipher Suite Priv Max   : XaaaXXaaaXXaaXX
                        :     X=Cipher Suite Unused
                        :     c=CALLBACK
                        :     u=USER
                        :     o=OPERATOR
                        :     a=ADMIN
                        :     O=OEM
```

2. 设置channel 1 的IP地址

``` shell
[root@hyhive ~]# ipmitool lan set 1 ipaddr 192.168.20.238
Setting LAN IP Address to 192.168.20.238
```

3. 设置channel 1 的netmask 

``` shell
[root@hyhive ~]# ipmitool lan set 1 netmask 255.255.0.0
Setting LAN Subnet Mask to 255.255.0.0
```

4. 设置channel 1的网关

``` shell
[root@hyhive ~]# ipmitool lan set 1 defgw ipaddr 192.168.10.1
Setting LAN Default Gateway IP to 192.168.10.1
```

5. 设置channel 1的ip为自动获取

``` shell
[root@hyhive ~]# ipmitool lan set 1 ipsrc dhcp 
```

6. 设置channel 1 的ip为静态

``` shell
[root@hyhive ~]# ipmitool lan set 1 ipsrc static 
```

7. 显示系统默认channel

``` shell
[root@hyhive ~]# ipmitool channel info 
Channel 0xf info:
  Channel Medium Type   : System Interface
  Channel Protocol Type : KCS
  Session Support       : session-less
  Active Session Count  : 0
  Protocol Vendor ID    : 7154
[root@hyhive ~]# ipmitool channel info 1
Channel 0x1 info:
  Channel Medium Type   : 802.3 LAN
  Channel Protocol Type : IPMB-1.0
  Session Support       : multi-session
  Active Session Count  : 0
  Protocol Vendor ID    : 7154
  Volatile(active) Settings
    Alerting            : enabled
    Per-message Auth    : enabled
    User Level Auth     : enabled
    Access Mode         : always available
  Non-Volatile Settings
    Alerting            : enabled
    Per-message Auth    : enabled
    User Level Auth     : enabled
    Access Mode         : always available
```

### 看门狗相关命令

> watchdog一般是一个硬件模块。在嵌入式操作系统中，常见的应用场景是系统长期运行且无人看守，当出现出现系统死机时，watchdog就会自动帮你重启系统。

1. 读取当前看门狗配置

``` shell
[root@hyhive ~]# ipmitool mc watchdog get 
Watchdog Timer Use:     Reserved (0x00)
Watchdog Timer Is:      Stopped
Watchdog Timer Actions: No action (0x00)
Pre-timeout interval:   0 seconds
Timer Expiration Flags: 0x00
Initial Countdown:      0 sec
Present Countdown:      0 sec
```

2. 关掉看门狗

``` shell
[root@hyhive ~]# ipmitool mc watchdog off 
Watchdog Timer Shutoff successful -- timer stopped
[root@hyhive ~]# ipmitool mc watchdog get 
Watchdog Timer Use:     SMS/OS (0x04)
Watchdog Timer Is:      Stopped
Watchdog Timer Actions: No action (0x00)
Pre-timeout interval:   0 seconds
Timer Expiration Flags: 0x00
Initial Countdown:      300 sec
Present Countdown:      300 sec
```

3. 在最近设置的计数器的基础上重启看门狗

``` shell
[root@hyhive ~]# ipmitool mc watchdog reset
IPMI Watchdog Timer Reset -  countdown restarted!
[root@hyhive ~]# ipmitool mc watchdog get 
Watchdog Timer Use:     SMS/OS (0x44)
Watchdog Timer Is:      Started/Running
Watchdog Timer Actions: No action (0x00)
Pre-timeout interval:   0 seconds
Timer Expiration Flags: 0x00
Initial Countdown:      300 sec
Present Countdown:      298 sec
```

