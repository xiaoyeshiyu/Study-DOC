# 使用smartctl进行SMART测试

所有现代硬盘都可通过`SMART`属性监视其当前状态。这些值提供有关硬盘各种参数的信息，并可提供有关磁盘剩余寿命或任何可能的错误的信息。此外，可以执行各种`SMART`测试，以确定磁盘上的任何硬件问题。本文介绍如何使用**smartctl（Smartmontools）**对Linux进行此类测试。

## 安装Smartmontools

Smartmontools可以使用yum直接安装：

``` shell
# yum install -y smartmontools
```

要确保硬盘支持SMART并启用，请使用以下命令（本例中为硬盘/ dev / sda）：

``` shell
# smartctl -i /dev/sda
smartctl 6.4 2015-06-04 r4109 [x86_64-linux-3.10.0-514.el7.x86_64] (local build)
Copyright (C) 2002-15, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     HGST HUS726030ALE610
Serial Number:    N8GH1H5Y
LU WWN Device Id: 5 000cca 244c6d728
Firmware Version: APBDT7JN
User Capacity:    3,000,592,982,016 bytes [3.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    7200 rpm
Form Factor:      3.5 inches
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ACS-2, ATA8-ACS T13/1699-D revision 4
SATA Version is:  SATA 3.1, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Wed Oct 11 07:09:17 2017 CST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
```

上面的参数中，`SMART support is: Available - device has SMART capability.`显示磁盘SMART是支持的，`SMART support is: Enabled`是可用的

## 可用测试

`SMART`根据规格类型和`SCSI`设备提供两种不同的测试。这些测试中的每一个都可以以两种模式进行：

- 前景模式
- 背景模式

在**背景模式**下，测试的优先级低，这意味着正常指令继续由硬盘处理。如果硬盘驱动器正忙，则测试暂停，然后以较低的加载速度继续运行，因此**不会中断操作**。
在**前景模式下，**所有命令将在测试期间以“检查条件”状态进行应答。因此，仅当不使用硬盘时，才建议使用此模式。原则上，**背景模式**是首选模式。

### ATA / SCSI测试

#### 短期测试

短期测试的目标是快速识别有缺陷的硬盘驱动器。因此，短测试的最大运行时间为2分钟。测试通过将磁盘划分成三个不同的段来检查磁盘。测试以下几个方面：

- **电气特性**：控制器测试自己的电子元件，由于这是每个制造商特有的，因此无法准确解释正在测试的内容。例如，可以想到测试内部RAM，读/写电路或头电子元件。
- **机械特性**：要测试的伺服系统和定位机制的确切顺序也是每个制造商特有的。
- **读/校验**：它将读取磁盘的某个区域并验证某些数据，读取的区域的大小和位置也是每个制造商特有的。

#### 长期测试

长期测试被设计为生产中的最终测试，与**短暂测试**相同，有两个差异。第一个：**没有时间限制，**并且在读/校验中检查整个磁盘，而不仅仅是一个部分。例如，长期测试可以用于确认短测试的结果。

### ATA指定测试

此处列出的所有测试仅适用于ATA硬盘驱动器。

#### 运输测试

可以执行该测试以确定在几分钟内传输硬盘时的损坏。

#### 选择测试

在选定的测试期间，检查指定的逻辑块范围。要扫描的逻辑块以以下格式指定：

```shell
# smartctl -t select,10-20 /dev/sda  ## LBA 10 to LBA 20（incl.）
# smartctl -t select,10+11 /dev/sda  ## LBA 10 to LBA 20（incl.）
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-358.el6.x86_64] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Selective self-test routine immediately in off-line mode".
SPAN         STARTING_LBA           ENDING_LBA
   0                   10                   20
Drive command "Execute SMART Selective self-test routine immediately in off-line mode" successful.
Testing has begun.
```

也可以有多个范围（最多5个）进行扫描：

```shell
# smartctl -t select,0-10 -t select,10-20 /dev/sda
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-358.el6.x86_64] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Selective self-test routine immediately in off-line mode".
SPAN         STARTING_LBA           ENDING_LBA
   0                    0                   10
   1                   10                   20
Drive command "Execute SMART Selective self-test routine immediately in off-line mode" successful.
Testing has begun.
```

## smartctl的测试程序

在执行测试之前，使用以下命令显示各种测试的持续时间的近似值：

```shell
# smartctl -c /dev/sda
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-358.el6.x86_64] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF READ SMART DATA SECTION ===
General SMART Values:
Offline data collection status:  (0x02)	Offline data collection activity
					was completed without error.
					Auto Offline Data Collection: Disabled.
Self-test execution status:      (   0)	The previous self-test routine completed
					without error or no self-test has ever 
					been run.
Total time to complete Offline 
data collection: 		(    0) seconds.
Offline data collection
capabilities: 			 (0x7d) SMART execute Offline immediate.
					No Auto Offline data collection support.
					Abort Offline collection upon new
					command.
					Offline surface scan supported.
					Self-test supported.
					Conveyance Self-test supported.
					Selective Self-test supported.
SMART capabilities:            (0x0003)	Saves SMART data before entering
					power-saving mode.
					Supports SMART auto save timer.
Error logging capability:        (0x01)	Error logging supported.
					General Purpose Logging supported.
Short self-test routine 
recommended polling time: 	 (   1) minutes.
Extended self-test routine
recommended polling time: 	 (  48) minutes.
Conveyance self-test routine
recommended polling time: 	 (   2) minutes.
SCT capabilities: 	       (0x0025)	SCT Status supported.
					SCT Data Table supported.
```

以下命令启动所需的测试（在后台模式下）：

```shell
# smartctl -t <short|long|conveyance|select> /dev/sda
```

也可以执行“离线”测试。但是，仅进行标准自检（Short Test）。

示例输出：

```shell
# smartctl -t short /dev/sda
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-358.el6.x86_64] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Short self-test routine immediately in off-line mode".
Drive command "Execute SMART Short self-test routine immediately in off-line mode" successful.
Testing has begun.
Please wait 1 minutes for test to complete.
Test will complete after Wed Oct 11 15:40:28 2017

Use smartctl -X to abort test.
```

要在前台模式下执行测试，必须在命令中添加“-C”。

```
# smartctl -t <short|long|conveyance|select> -C /dev/sda
```

## 查看测试结果

通常，测试结果包含在以下命令的输出中：

```shell
# smartctl -a /dev/sda
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-358.el6.x86_64] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF INFORMATION SECTION ===
Device Model:     KINGSTON SV300S37A60G
Serial Number:    50026B724A01E182
LU WWN Device Id: 5 0026b7 24a01e182
Firmware Version: 580ABBF0
User Capacity:    60,022,480,896 bytes [60.0 GB]
Sector Size:      512 bytes logical/physical
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   8
ATA Standard is:  ACS-2 revision 3
Local Time is:    Wed Oct 11 15:41:49 2017 CST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

General SMART Values:
Offline data collection status:  (0x02)	Offline data collection activity
					was completed without error.
					Auto Offline Data Collection: Disabled.
Self-test execution status:      (   0)	The previous self-test routine completed
					without error or no self-test has ever 
					been run.
Total time to complete Offline 
data collection: 		(    0) seconds.
Offline data collection
capabilities: 			 (0x7d) SMART execute Offline immediate.
					No Auto Offline data collection support.
					Abort Offline collection upon new
					command.
					Offline surface scan supported.
					Self-test supported.
					Conveyance Self-test supported.
					Selective Self-test supported.
SMART capabilities:            (0x0003)	Saves SMART data before entering
					power-saving mode.
					Supports SMART auto save timer.
Error logging capability:        (0x01)	Error logging supported.
					General Purpose Logging supported.
Short self-test routine 
recommended polling time: 	 (   1) minutes.
Extended self-test routine
recommended polling time: 	 (  48) minutes.
Conveyance self-test routine
recommended polling time: 	 (   2) minutes.
SCT capabilities: 	       (0x0025)	SCT Status supported.
					SCT Data Table supported.

SMART Attributes Data Structure revision number: 10
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x0032   095   095   050    Old_age   Always       -       8601004262
  5 Reallocated_Sector_Ct   0x0033   099   099   003    Pre-fail  Always       -       0
  9 Power_On_Hours          0x0032   085   085   000    Old_age   Always       -       145066815403100
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       120
171 Unknown_Attribute       0x000a   100   100   000    Old_age   Always       -       0
172 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       0
174 Unknown_Attribute       0x0030   000   000   000    Old_age   Offline      -       97
177 Wear_Leveling_Count     0x0000   000   000   000    Old_age   Offline      -       96
181 Program_Fail_Cnt_Total  0x000a   100   100   000    Old_age   Always       -       0
182 Erase_Fail_Count_Total  0x0032   100   100   000    Old_age   Always       -       0
187 Reported_Uncorrect      0x0012   100   100   000    Old_age   Always       -       0
189 High_Fly_Writes         0x0000   029   041   000    Old_age   Offline      -       77312098333
194 Temperature_Celsius     0x0022   029   041   000    Old_age   Always       -       29 (Min/Max 18/41)
195 Hardware_ECC_Recovered  0x001c   102   102   000    Old_age   Offline      -       8601004262
196 Reallocated_Event_Count 0x0033   099   099   003    Pre-fail  Always       -       0
201 Soft_Read_Error_Rate    0x001c   102   102   000    Old_age   Offline      -       8601004262
204 Soft_ECC_Correction     0x001c   102   102   000    Old_age   Offline      -       8601004262
230 Head_Amplitude          0x0013   100   100   000    Pre-fail  Always       -       100
231 Temperature_Celsius     0x0013   091   091   010    Pre-fail  Always       -       0
233 Media_Wearout_Indicator 0x0032   000   000   000    Old_age   Always       -       9261
234 Unknown_Attribute       0x0032   000   000   000    Old_age   Always       -       14820
241 Total_LBAs_Written      0x0032   000   000   000    Old_age   Always       -       14820
242 Total_LBAs_Read         0x0032   000   000   000    Old_age   Always       -       6033

SMART Error Log not supported
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%     13404         -
# 2  Selective offline   Completed without error       00%     13404         -
# 3  Selective offline   Completed without error       00%     13404         -
# 4  Selective offline   Completed without error       00%     13404         -
# 5  Short offline       Completed without error       00%     13403         -

SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0       10  Not_testing
    2       10       20  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x0):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.
```

如果只显示测试结果，也可以使用以下命令：

```shell
# smartctl -l selftest /dev/sda
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-358.el6.x86_64] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF READ SMART DATA SECTION ===
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%     13404         -
# 2  Selective offline   Completed without error       00%     13404         -
# 3  Selective offline   Completed without error       00%     13404         -
# 4  Selective offline   Completed without error       00%     13404         -
# 5  Short offline       Completed without error       00%     13403         -
```

显示磁盘全部的健康状态

```shell
# smartctl -H /dev/sda
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-358.el6.x86_64] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED   ##说明自检通过，磁盘健康
```

查看磁盘错误日志

```shell
# smartctl -l error  /dev/sdb

Sample Output

smartctl 6.2 2013-07-26 r3841 [x86_64-linux-3.13.0-32-generic] (local build)
Copyright (C) 2002-13, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF READ SMART DATA SECTION ===
SMART Error Log Version: 1
ATA Error Count: 1
	CR = Command Register [HEX]
	FR = Features Register [HEX]
	SC = Sector Count Register [HEX]
	SN = Sector Number Register [HEX]
	CL = Cylinder Low Register [HEX]
	CH = Cylinder High Register [HEX]
	DH = Device/Head Register [HEX]
	DC = Device Command Register [HEX]
	ER = Error register [HEX]
	ST = Status register [HEX]
Powered_Up_Time is measured from power on, and printed as
DDd+hh:mm:SS.sss where DD=days, hh=hours, mm=minutes,
SS=sec, and sss=millisec. It "wraps" after 49.710 days.

Error 1 occurred at disk power-on lifetime: 10042 hours (418 days + 10 hours)
  When the command that caused the error occurred, the device was active or idle.

  After command completion occurred, registers were:
  ER ST SC SN CL CH DH
  -- -- -- -- -- -- --
  84 51 00 00 00 00 a0

  Commands leading to the command that caused the error were:
  CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
  -- -- -- -- -- -- -- --  ----------------  --------------------
  ec 00 00 00 00 00 a0 08  35d+16:09:47.709  IDENTIFY DEVICE
  ec 00 00 00 00 00 a0 08  35d+16:09:42.437  IDENTIFY DEVICE
  ec 00 00 00 00 00 a0 08  35d+16:09:42.159  IDENTIFY DEVICE
  ef 03 46 00 00 00 a0 08  35d+16:09:42.159  SET FEATURES [Set transfer mode]
  ec 00 00 00 00 00 a0 08  35d+16:09:42.159  IDENTIFY DEVICE
```

### 处理过程

首先通过`smartctl -H /dev/sda`检查磁盘健康状态，然后`smartctl -a /dev/sda`查看磁盘详细情况，再对磁盘进行短期测试`smartctl -t short /dev/sda`，最后查看磁盘测试结果`smartctl -l selftest /dev/sda`，基本磁盘健康状态就可以定位出来，最后检查磁盘错误日志`smartctl -l error  /dev/sdb`

