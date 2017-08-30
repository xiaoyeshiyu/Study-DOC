# 实例使用crash分析Kdump转储kernel崩溃内核

## Kdump

Kdump 是一种基于 kexec 的 Linux 内核崩溃捕获机制，将 kernel 崩溃前的内存镜像保存，程序员通过分析该文件找出 kernel 崩溃的原因，从而进行系统改进。

### Kdump 的基本概念

#### 什么是 kexec ？

Kexec 是实现kdump 机制的关键，它包括 2 个组成部分：

> 一是内核空间的系统调用 kexec_load，负责在生产内核（production kernel 或 first kernel）启动时将捕获内核（capture kernel 或 sencond kernel）加载到指定地址。
>
> 二是用户空间的工具 kexec-tools，他将捕获内核的地址传递给生产内核，从而在系统崩溃的时候能够找到捕获内核的地址并运行。没有 kexec 就没有 kdump。 

先有 kexec 实现了在一个内核中可以启动另一个内核，才让 kdump 有了用武之地。kexec 原来的目的是为了节省 kernel 开发人员重启系统的时间，谁能想到这个“偷懒”的技术却孕育了最成功的内存转存机制呢？

#### 什么是 kdump ？

Kdump 的概念出现在 2005 左右，是迄今为止最可靠的内核转存机制，已经被主要的 linux™ 厂商选用。kdump 是一种先进的基于 kexec 的内核崩溃转储机制。当系统崩溃时，kdump 使用 kexec 启动到第二个内核。第二个内核通常叫做捕获内核，以很小内存启动以捕获转储镜像。第一个内核保留了内存的一部分给第二内核启动用。由于 kdump 利用 kexec 启动捕获内核，绕过了 BIOS，所以第一个内核的内存得以保留。这是内核崩溃转储的本质。

kdump 需要两个不同目的的内核，生产内核和捕获内核。生产内核是捕获内核服务的对像。捕获内核会在生产内核崩溃时启动起来，与相应的 ramdisk 一起组建一个微环境，用以对生产内核下的内存进行收集和转存。

#### 如何使用 kdump

构建系统和 dump-capture 内核，此操作有 2 种方式可选：

>  1）构建一个单独的自定义转储捕获内核以捕获内核转储；
>
>  2） 或者将系统内核本身作为转储捕获内核，这就不需要构建一个单独的转储捕获内核。

 

方法（2）只能用于可支持可重定位内核的体系结构上；目前 i386，x86_64，ppc64 和 ia64 体系结构支持可重定位内核。构建一个可重定位内核使得不需要构建第二个内核就可以捕获转储。但是可能有时想构建一个自定义转储捕获内核以满足特定要求。

#### 如何访问捕获内存

在内核崩溃之前所有关于核心映像的必要信息都用 ELF 格式编码并存储在保留的内存区域中。ELF 头所在的物理地址被作为命令行参数（fcorehdr=）传递给新启动的转储内核。 

在 i386 体系结构上，启动的时候需要使用物理内存开始的 640K，而不管操作系统内核转载在何处。因此，这个 640K 的区域在重新启动第二个内核的时候由 kexec 备份。

在第二个内核中，“前一个系统的内存”可以通过两种方式访问：

> 1.通过/dev/oldmem 这个设备接口。

一个“捕捉”设备可以使用“raw”（裸的）方式 “读”这个设备文件并写出到文件。这是关于内存的 “裸”的数据转储，同时这些分析 / 捕捉工具应该足够“智能”从而可以知道从哪里可以得到正确的信息。ELF 文件头（通过命令行参数传递过来的 elfcorehdr）可能会有帮助。

> 2.通过/proc/vmcore。

这个方式是将转储输出为一个 ELF 格式的文件，并且可以使用一些文件拷贝命令（比如 cp，scp 等）将信息读出来。同时，gdb 可以在得到的转储文件上做一些调试（有限的）。这种方式保证了内存中的页面都以正确的途径被保存 ( 注意内存开始的 640K 被重新映射了 )。

#### kdump 的优势

**高可靠性** 

崩溃转储数据可从一个新启动内核的上下文中获取，而不是从已经崩溃内核的上下文。

**多版本支持**

LKCD(LinuxKernel Crash Dump)，netdump，diskdump 已被纳入 LDPs(Linux Documen-tation Project) 内核。SUSE和 RedHat 都对 kdump 有技术支持。

### Kdump实现流程

#### RHEL6.2执行流程

![图 1. RHEL6.2 执行流程](https://cl.ly/0s3t3Z463O0r/Image%202017-08-30%20at%2010.37.38%20AM.png)

### 配置 kdump

#### 安装软件包和实用程序

Kdump 用到的各种工具都在 kexec-tools 中。kernel-debuginfo 则是用来分析 vmcore 文件。从rhel5 开始，kexec-tools 已被默认安装在发行版。而 novell 也在 sles10 发行版中把 kdump 集成进来。所以如果使用的是 rhel5 和 sles10 之后的发行版，那就省去了安装 kexec-tools 的步骤。而如果需要调试 kdump 生成的 vmcore 文件，则需要手动安装 kernel-debuginfo 包。

* 检查安装包操作（注意安装包的版本需要和内核版本号一致）：

  ``` shell
  [root@localhost ~]# rpm -qa | grep kexec
  kexec-tools-2.0.7-50.el7.x86_64
  [root@localhost ~]# uname -a 
  Linux localhost.localdomain 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
  [root@localhost ~]# rpm -qa 'kernel*debuginfo*'
  kernel-debuginfo-3.10.0-514.el7.x86_64
  kernel-debuginfo-common-x86_64-3.10.0-514.el7.x86_64
  ```

*debuginfo包可在http://debuginfo.centos.org/处下载* 

#### 修改内核引导参数，为启动捕获内核预留内存（在kexec-tools安装好之后，参数默认即可）

通过下面的方法来配置 kdump 使用的内存大小。添加启动参数"crashkernel=Y@X"，这里，Y是为 kdump 捕捉内核保留的内存，X 是保留部分内存的开始位置。

> 对于 i386 和 x86_64, 编辑 /etc/grub.conf, 在内核行的最后添加"crashkernel=128M"。
>
> 对于 ppc64，在 /etc/yaboot.conf 最后添加"crashkernel=128M"。
>
> 在 ia64, 编辑 /etc/elilo.conf，添加"crashkernel=256M"到内核行。 

#### kdump 配置文件

kdump 的配置文件是 /etc/kdump.conf（RHEL6.2）；/etc/sysconfig/kdump(SLES11 sp2)。每个文件头部都有选项说明，可以根据使用需求设置相应的选项。 

#### 启动 kdump 服务

在设置了预留内存后，需要重启机器，否则 kdump 是不可使用的。启动 kdump 服务：

``` shell
[root@localhost ~]#systemctl restart kdump
```

### 测试配置是否有效

可以通过 kexec 加载内核镜像，让系统准备好去捕获一个崩溃时产生的 vmcore。可以通过 sysrq 强制系统崩溃。

``` shell
[root@localhost~]# echo c > /proc/sysrq-trigger 
```

这造成内核崩溃，如配置有效，系统将重启进入 kdump 内核，当系统进程进入到启动 kdump 服务的点时，vmcore 将会拷贝到你在 kdump 配置文件中设置的位置。

*RHEL 的缺省目录是 : /var/crash；SLES 的缺省目录是 : /var/log/dump。* 

然后系统重启进入到正常的内核。一旦回复到正常的内核，就可以在上述的目录下发现 vmcore 文件，即内存转储文件。 

可以使用之前安装的 kernel-debuginfo 中的 crash 工具来进行分析（crash 的更多详细用法将在本系列后面的文章中有介绍）。

``` shell
[root@localhost~]# crash /usr/lib/debug/lib/modules/3.10.0-514.el7.x86_64/vmlinux/var/crash/127.0.0.1-2017-08-21-02\:31\:15/vmcore
```

## crash

### 什么是 crash

如前文所述，当 linux 系统内核发生崩溃的时候，可以通过 kdump 等方式收集内核崩溃之前的内存，生成一个转储文件 vmcore。内核开发者通过分析该 vmcore 文件就可以诊断出内核崩溃的原因，从而进行操作系统的代码改进。那么 crash 就是一个被广泛使用的内核崩溃转储文件分析工具，掌握 crash 的使用技巧，对于定位问题有着十分重要的作用。

### 使用 crash 的先决条件

由于 crash 用于调试内核崩溃的转储文件，因此使用 crash 需要依赖如下条件：

>  1.kernel 映像文件 vmlinux 在编译的时候必须指定了 -g 参数，即带有调试信息。
>
>  2.需要有一个内存崩溃转储文件（例如 vmcore），或者可以通过 /dev/mem 或 /dev/crash 访问的实时系统内存。如果 crash 命令行没有指定转储文件，则 crash 默认使用实时系统内存，这时需要 root 权限。
>
>  3.crash 支持的平台处理器包括：x86, x86_64, ia64, ppc64, arm, s390, s390x ( 也有部分crash 版本支持 Alpha 和 32-bit PowerPC，但是对于这两种平台的支持不保证长期维护)。
>
>  4.crash 支持 2.2.5-15（含）以后的 Linux 内核版本。随着 Linux 内核的更新，crash 也在不断升级以适应新的内核。

### crash 安装指南

要想使用 crash调试内核转储文件，需要安装 crash 工具和内核调试信息包。不同的发行版安装包名称略有差异，这里仅列出 RHEL 和 SLES 发行版对应的安装包名称如下：

**表 1. crash 工具和内核调试包** 

| 系统版本      | crash工具名称 | 内核调试信息包                                  |
| --------- | --------- | ---------------------------------------- |
| RHEL6.2   | crash     | kernel-debuginfo-common   kernel-debuginfo |
| SLES11SP2 | crash     | kernel-default-debuginfo    kernel-ppc64-debuginfo |

以 RHEL为例，安装 crash 及内核调试信息包的步骤如下：

``` shell
[root@localhost ~]# rpm -ivh crash-7.1.5-2.el7.x86_64.rpm 
[root@localhost ~]# uname -a 
Linux localhost.localdomain 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
[root@localhost ~]# rpm -ivh kernel-debuginfo-common-x86_64-3.10.0-514.el7.x86_64.rpm 
[root@localhost ~]# rpm -ivh kernel-debuginfo-3.10.0-514.el7.x86_64.rpm 
```

### 启动 crash

#### 启动参数说明

使用 crash 调试转储文件，需要在命令行输入两个参数：debug kernel 和 dump file，其中 dump file 是内核转储文件的名称，debug kernel 是由内核调试信息包安装的，不同的发行版名称略有不同，以 RHEL为例：

``` shell
[root@localhost~]# crash /usr/lib/debug/lib/modules/3.10.0-514.el7.x86_64/vmlinux/var/crash/127.0.0.1-2017-08-21-02\:31\:15/vmcore
```

使用 crash -h 或 man crash 可以查看 crash 支持的一系列选项，这里仅以常用的选项为例说明如下：

> -h：打印帮助信息
>
> -d：设置调试级别
>
> -S：使用/boot/System.map 作为默认的映射文件
>
> -s：不显示版本、初始调试信息等，直接进入命令行
>
> -ifile：启动之后自动运行 file 中的命令，再接受用户输入

#### crash 报告分析

crash 命令启动后，会产生一个转储文件的分析报告摘要，如下图所示。

##### 清单 1. crash 报告

``` shell
[root@hyhive ~]# crash /usr/lib/debug/lib/modules/3.10.0-514.el7.x86_64/vmlinux /var/crash/127.0.0.1-2017-07-21-17\:07\:17/vmcore

crash 7.1.5-2.el7
Copyright (C) 2002-2016  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.
 
GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

      KERNEL: /usr/lib/debug/lib/modules/3.10.0-514.el7.x86_64/vmlinux
    DUMPFILE: /var/crash/127.0.0.1-2017-07-21-17:07:17/vmcore  [PARTIAL DUMP]
        CPUS: 56
        DATE: Fri Jul 21 17:07:00 2017
      UPTIME: 8 days, 03:43:48
LOAD AVERAGE: 1.98, 1.73, 1.73
       TASKS: 4444
    NODENAME: hyhive
     RELEASE: 3.10.0-514.el7.x86_64
     VERSION: #1 SMP Tue Nov 22 16:42:41 UTC 2016
     MACHINE: x86_64  (2600 Mhz)
      MEMORY: 255.9 GB
       PANIC: "BUG: unable to handle kernel paging request at ffff8800fc14dfb0"
         PID: 21793
     COMMAND: "sh"
        TASK: ffff883f97751f60  [THREAD_INFO: ffff8830e1b50000]
         CPU: 49
       STATE: TASK_RUNNING (PANIC)

crash> 
```

> KERNEL:系统崩溃时运行的 kernel 文件
>
> DUMPFILE:内核转储文件
>
> CPUS: 所在机器的 CPU 数量
>
> DATE: 系统崩溃的时间
>
> TASKS:系统崩溃时内存中的任务数
>
> NODENAME:崩溃的系统主机名
>
> RELEASE:和 VERSION: 内核版本号
>
> MACHINE:CPU 架构
>
> MEMORY:崩溃主机的物理内存
>
> PANIC:崩溃类型，常见的崩溃类型包括：
>
> 1. SysRq(System Request)：通过魔法组合键导致的系统崩溃，通常是测试使用。通过 echo c > /proc/sysrq-trigger，就可以触发系统崩溃。
> 2. oops：可以看成是内核级的 Segmentation Fault。应用程序如果进行了非法内存访问或执行了非法指令，会得到 Segfault 信号，一般行为是 coredump，应用程序也可以自己截获 Segfault 信号，自行处理。如果内核自己犯了这样的错误，则会弹出 oops 信息。

#### crash内置命令简介

crash 命令行启动后，可以通过一些内置命令来打印系统崩溃前的信息。

##### bt -backtrace

bt 命令用于查看系统崩溃前的堆栈等信息，这是系统调试中非常常用和好用的一个命令。

``` shell
crash> bt
PID: 21793  TASK: ffff883f97751f60  CPU: 49  COMMAND: "sh"
 #0 [ffff8830e1b538e8] machine_kexec at ffffffff81059cdb
 #1 [ffff8830e1b53948] __crash_kexec at ffffffff81105182
 #2 [ffff8830e1b53a18] crash_kexec at ffffffff81105270
 #3 [ffff8830e1b53a30] oops_end at ffffffff8168ee88
 #4 [ffff8830e1b53a58] no_context at ffffffff8167ea93
 #5 [ffff8830e1b53aa8] __bad_area_nosemaphore at ffffffff8167eb29
 #6 [ffff8830e1b53af0] bad_area_nosemaphore at ffffffff8167ec93
 #7 [ffff8830e1b53b00] __do_page_fault at ffffffff81691c1e
 #8 [ffff8830e1b53b60] do_page_fault at ffffffff81691dc5
 #9 [ffff8830e1b53b90] page_fault at ffffffff8168e088
    [exception RIP: handle_mm_fault+319]
    RIP: ffffffff811b084f  RSP: ffff8830e1b53c40  RFLAGS: 00010282
    RAX: 00000000fc14d000  RBX: 00007ffefed4c150  RCX: 00003ffffffff000
    RDX: ffff8800fc14dfb0  RSI: 000000000000000b  RDI: 00000000fc14d000
    RBP: ffff8830e1b53cc8   R8: ffff883f43635440   R9: 000000000000001d
    R10: 000000000000008f  R11: 0000000000002941  R12: ffff883f43635440
    R13: ffff8800fc14dfb0  R14: 0000000000000029  R15: ffff883ff7059f40
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#10 [ffff8830e1b53cd0] __do_page_fault at ffffffff81691a94
#11 [ffff8830e1b53d30] do_page_fault at ffffffff81691dc5
#12 [ffff8830e1b53d60] page_fault at ffffffff8168e088
    [exception RIP: __put_user_4+32]
    RIP: ffffffff81326b30  RSP: ffff8830e1b53e10  RFLAGS: 00050297
    RAX: 0000000000000300  RBX: 00007fffffffeffd  RCX: 00007ffefed4c150
    RDX: ffff883ff73d3a80  RSI: 0000000000000300  RDI: ffff881185f95248
    RBP: ffff8830e1b53e80   R8: ffff883f99e5f044   R9: 000000000000001d
    R10: 000000000000008f  R11: 0000000000002941  R12: ffff883f8934edd0
    R13: ffff8830e1b53ef8  R14: 0000000000005522  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#13 [ffff8830e1b53e10] wait_consider_task at ffffffff8108b102
#14 [ffff8830e1b53e88] do_wait at ffffffff8108b4c0
#15 [ffff8830e1b53ef0] sys_wait4 at ffffffff8108c6b0
#16 [ffff8830e1b53f80] system_call_fastpath at ffffffff816965c9
    RIP: 00007f4210f9f27c  RSP: 00007ffefed4bf30  RFLAGS: 00010246
    RAX: 000000000000003d  RBX: ffffffff816965c9  RCX: 0000ca6491781260
    RDX: 0000000000000000  RSI: 00007ffefed4c150  RDI: ffffffffffffffff
    RBP: 0000000000e2e340   R8: 0000000000e2e340   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000246  R12: 0000000000e2da90
    R13: 0000000000000001  R14: 0000000000000000  R15: 0000000000000000
    ORIG_RAX: 000000000000003d  CS: 0033  SS: 002b
```

如上输出中，以“# 数字”开头的行为调用堆栈，即系统崩溃前内核依次调用的一系列函数，通过这个可以迅速推断内核在何处崩溃。

##### log -dump system message buffer

log 命令可以打印系统消息缓冲区，从而可能找到系统崩溃的线索。

``` shell
crash> log
[618639.865172] kvm [20669]: vcpu0 unhandled rdmsr: 0x611
[618639.865362] kvm [20669]: vcpu0 unhandled rdmsr: 0x639
[618639.865533] kvm [20669]: vcpu0 unhandled rdmsr: 0x641
[618639.865704] kvm [20669]: vcpu0 unhandled rdmsr: 0x619
[618639.868951] kvm [20669]: vcpu0 unhandled rdmsr: 0x60d
[618698.976409] kvm: zapping shadow pages for mmio generation wraparound
[618699.458405] kvm_get_msr_common: 6 callbacks suppressed
[618699.458409] kvm [22064]: vcpu0 unhandled rdmsr: 0x1c9
[618699.458582] kvm [22064]: vcpu0 unhandled rdmsr: 0x1a6
[618699.458750] kvm [22064]: vcpu0 unhandled rdmsr: 0x1a7
[618699.458918] kvm [22064]: vcpu0 unhandled rdmsr: 0x3f6
[618699.525572] kvm [22064]: vcpu0 unhandled rdmsr: 0x606
[618700.003113] kvm [22064]: vcpu0 unhandled rdmsr: 0x611
[618700.003291] kvm [22064]: vcpu0 unhandled rdmsr: 0x639
[618700.003463] kvm [22064]: vcpu0 unhandled rdmsr: 0x641
[618700.003626] kvm [22064]: vcpu0 unhandled rdmsr: 0x619
[618700.005819] kvm [22064]: vcpu0 unhandled rdmsr: 0x60d
[618759.027146] kvm: zapping shadow pages for mmio generation wraparound
[618759.400453] kvm_get_msr_common: 6 callbacks suppressed
```

##### ps -display process status information

ps 命令用于显示进程的状态，（如图）带 > 标识代表是活跃的进程。

``` shell
crash> ps
   PID    PPID  CPU       TASK        ST  %MEM     VSZ    RSS  COMM
>     0      0   0  ffffffff819c1460  RU   0.0       0      0  [swapper/0]
>     0      0   1  ffff880173acaf10  RU   0.0       0      0  [swapper/1]
>     0      0   2  ffff880173acbec0  RU   0.0       0      0  [swapper/2]
>     0      0   3  ffff880173acce70  RU   0.0       0      0  [swapper/3]
>     0      0   4  ffff880173acde20  RU   0.0       0      0  [swapper/4]
>     0      0   5  ffff880173acedd0  RU   0.0       0      0  [swapper/5]
>     0      0   6  ffff880173af0000  RU   0.0       0      0  [swapper/6]
>     0      0   7  ffff880173af0fb0  RU   0.0       0      0  [swapper/7]
>     0      0   8  ffff880173af1f60  RU   0.0       0      0  [swapper/8]
>     0      0   9  ffff880173af2f10  RU   0.0       0      0  [swapper/9]
>     0      0  10  ffff880173af3ec0  RU   0.0       0      0  [swapper/10]
>     0      0  11  ffff880173af4e70  RU   0.0       0      0  [swapper/11]
>     0      0  12  ffff880173af5e20  RU   0.0       0      0  [swapper/12]
>     0      0  13  ffff880173af6dd0  RU   0.0       0      0  [swapper/13]
>     0      0  14  ffff8820f3db0000  RU   0.0       0      0  [swapper/14]
>     0      0  15  ffff8820f3db0fb0  RU   0.0       0      0  [swapper/15]
>     0      0  16  ffff8820f3db1f60  RU   0.0       0      0  [swapper/16]
>     0      0  17  ffff8820f3db2f10  RU   0.0       0      0  [swapper/17]
>     0      0  18  ffff8820f3db3ec0  RU   0.0       0      0  [swapper/18]
>     0      0  19  ffff8820f3db4e70  RU   0.0       0      0  [swapper/19]
>     0      0  20  ffff8820f3db5e20  RU   0.0       0      0  [swapper/20]
>     0      0  21  ffff8820f3db6dd0  RU   0.0       0      0  [swapper/21]
>     0      0  22  ffff8820f3dd8000  RU   0.0       0      0  [swapper/22]
```

##### dis -disassembling instruction

dis 命令用于对给定地址的内容进行反汇编。

``` shell
crash> dis -l ffffffff8168e088
/usr/src/debug/kernel-3.10.0-514.el7/linux-3.10.0-514.el7.x86_64/arch/x86/kernel/entry_64.S: 1316
0xffffffff8168e088 <page_fault+40>:     jmpq   0xffffffff8168e2c0 <error_exit>
```

##### struct– view data struct 

struct 命令用于查看数据结构的定义原型。

``` shell
crash> struct -o vm_struct
struct vm_struct {
   [0] struct vm_struct *next;
   [8] void *addr;
  [16] unsigned long size;
  [24] unsigned long flags;
  [32] struct page **pages;
  [40] unsigned int nr_pages;
  [48] phys_addr_t phys_addr;
  [56] const void *caller;
}
SIZE: 64
```

## 案例实测

如前文所述，当 linux 系统内核发生崩溃的时候，可以通过 kdump 等方式收集内核崩溃之前的内存，生成一个转储文件 vmcore。内核开发者通过分析该 vmcore 文件就可以诊断出内核崩溃的原因，从而进行操作系统的代码改进。那么 crash 就是一个被广泛使用的内核崩溃转储文件分析工具，掌握 crash 的使用技巧，对于定位问题有着十分重要的作用。

这里采用笔者在实际测试工作中发现的 CentOS 系统下的系统崩溃问题作为案例来进行讲解。该系统已经配置了 kdump 启用，因此在系统发生崩溃之后，在 /var/crash/ 当天日期 / 目录下面生成一个 vmcore 文件，下面我们来对这个文件进行分析。

1. 首先启动crash 

``` shell
[root@hyhive ~]# crash /usr/lib/debug/lib/modules/3.10.0-514.el7.x86_64/vmlinux /var/crash/127.0.0.1-2017-07-21-17\:07\:17/vmcore

crash 7.1.5-2.el7
Copyright (C) 2002-2016  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.
 
GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

      KERNEL: /usr/lib/debug/lib/modules/3.10.0-514.el7.x86_64/vmlinux
    DUMPFILE: /var/crash/127.0.0.1-2017-07-21-17:07:17/vmcore  [PARTIAL DUMP]
        CPUS: 56
        DATE: Fri Jul 21 17:07:00 2017
      UPTIME: 8 days, 03:43:48
LOAD AVERAGE: 1.98, 1.73, 1.73
       TASKS: 4444
    NODENAME: hyhive
     RELEASE: 3.10.0-514.el7.x86_64
     VERSION: #1 SMP Tue Nov 22 16:42:41 UTC 2016
     MACHINE: x86_64  (2600 Mhz)
      MEMORY: 255.9 GB
       PANIC: "BUG: unable to handle kernel paging request at ffff8800fc14dfb0"
         PID: 21793
     COMMAND: "sh"
        TASK: ffff883f97751f60  [THREAD_INFO: ffff8830e1b50000]
         CPU: 49
       STATE: TASK_RUNNING (PANIC)

crash> 
```

可以看到内核版本是3.10.0-514.el7.x86_64，这是CentOS 7.3的开发版本

2. 用bt命令看一下堆栈

``` shell
crash> bt
PID: 21793  TASK: ffff883f97751f60  CPU: 49  COMMAND: "sh"
 #0 [ffff8830e1b538e8] machine_kexec at ffffffff81059cdb
 #1 [ffff8830e1b53948] __crash_kexec at ffffffff81105182
 #2 [ffff8830e1b53a18] crash_kexec at ffffffff81105270
 #3 [ffff8830e1b53a30] oops_end at ffffffff8168ee88
 #4 [ffff8830e1b53a58] no_context at ffffffff8167ea93
 #5 [ffff8830e1b53aa8] __bad_area_nosemaphore at ffffffff8167eb29
 #6 [ffff8830e1b53af0] bad_area_nosemaphore at ffffffff8167ec93
 #7 [ffff8830e1b53b00] __do_page_fault at ffffffff81691c1e
 #8 [ffff8830e1b53b60] do_page_fault at ffffffff81691dc5
 #9 [ffff8830e1b53b90] page_fault at ffffffff8168e088
    [exception RIP: handle_mm_fault+319]
    RIP: ffffffff811b084f  RSP: ffff8830e1b53c40  RFLAGS: 00010282
    RAX: 00000000fc14d000  RBX: 00007ffefed4c150  RCX: 00003ffffffff000
    RDX: ffff8800fc14dfb0  RSI: 000000000000000b  RDI: 00000000fc14d000
    RBP: ffff8830e1b53cc8   R8: ffff883f43635440   R9: 000000000000001d
    R10: 000000000000008f  R11: 0000000000002941  R12: ffff883f43635440
    R13: ffff8800fc14dfb0  R14: 0000000000000029  R15: ffff883ff7059f40
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#10 [ffff8830e1b53cd0] __do_page_fault at ffffffff81691a94
#11 [ffff8830e1b53d30] do_page_fault at ffffffff81691dc5
#12 [ffff8830e1b53d60] page_fault at ffffffff8168e088
    [exception RIP: __put_user_4+32]
    RIP: ffffffff81326b30  RSP: ffff8830e1b53e10  RFLAGS: 00050297
    RAX: 0000000000000300  RBX: 00007fffffffeffd  RCX: 00007ffefed4c150
    RDX: ffff883ff73d3a80  RSI: 0000000000000300  RDI: ffff881185f95248
    RBP: ffff8830e1b53e80   R8: ffff883f99e5f044   R9: 000000000000001d
    R10: 000000000000008f  R11: 0000000000002941  R12: ffff883f8934edd0
    R13: ffff8830e1b53ef8  R14: 0000000000005522  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#13 [ffff8830e1b53e10] wait_consider_task at ffffffff8108b102
#14 [ffff8830e1b53e88] do_wait at ffffffff8108b4c0
#15 [ffff8830e1b53ef0] sys_wait4 at ffffffff8108c6b0
#16 [ffff8830e1b53f80] system_call_fastpath at ffffffff816965c9
    RIP: 00007f4210f9f27c  RSP: 00007ffefed4bf30  RFLAGS: 00010246
    RAX: 000000000000003d  RBX: ffffffff816965c9  RCX: 0000ca6491781260
    RDX: 0000000000000000  RSI: 00007ffefed4c150  RDI: ffffffffffffffff
    RBP: 0000000000e2e340   R8: 0000000000e2e340   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000246  R12: 0000000000e2da90
    R13: 0000000000000001  R14: 0000000000000000  R15: 0000000000000000
    ORIG_RAX: 000000000000003d  CS: 0033  SS: 002b
crash> 
```

可以看到“ #9 [ffff8830e1b53b90] page_fault at ffffffff8168e088”后面出现“ [exception RIP: handle_mm_fault+319]”字样，此时kernel出现问题，exception RIP对应的#9，#0到#8只是系统再执行到引起panic的代码后的例行操作，本身对于解panic没有帮助。

RIP是当前执行的指令的地址。

到这里可以看到的是：*系统在执行__do_page_fault这个函数调用handle_mm_fault()时发生panic，具体地址是handle_mm_fault+319* 

3. 用dis命令对内核函数handle_mm_fault进行反编绘，（下图截取部分）

``` shell
crash> dis handle_mm_fault
0xffffffff811b0710 <handle_mm_fault>:   nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff811b0715 <handle_mm_fault+5>: push   %rbp
0xffffffff811b0716 <handle_mm_fault+6>: mov    %rsp,%rbp
0xffffffff811b0719 <handle_mm_fault+9>: push   %r15
0xffffffff811b071b <handle_mm_fault+11>:        mov    %rdi,%r15
0xffffffff811b071e <handle_mm_fault+14>:        push   %r14
0xffffffff811b0720 <handle_mm_fault+16>:        mov    %ecx,%r14d
0xffffffff811b0723 <handle_mm_fault+19>:        push   %r13
0xffffffff811b0725 <handle_mm_fault+21>:        push   %r12
0xffffffff811b0727 <handle_mm_fault+23>:        mov    %rsi,%r12
0xffffffff811b072a <handle_mm_fault+26>:        push   %rbx
0xffffffff811b072b <handle_mm_fault+27>:        mov    %rdx,%rbx
0xffffffff811b072e <handle_mm_fault+30>:        sub    $0x60,%rsp
0xffffffff811b0732 <handle_mm_fault+34>:        mov    %gs:0x28,%rax
0xffffffff811b073b <handle_mm_fault+43>:        mov    %rax,-0x30(%rbp)
0xffffffff811b073f <handle_mm_fault+47>:        xor    %eax,%eax
0xffffffff811b0741 <handle_mm_fault+49>:        mov    %gs:0xcdc0,%rax
0xffffffff811b074a <handle_mm_fault+58>:        movq   $0x0,(%rax)
0xffffffff811b0751 <handle_mm_fault+65>:        mov    0x938c9d(%rip),%esi        # 0xffffffff81ae93f4 <mem_cgroup_subsys+84>
0xffffffff811b0757 <handle_mm_fault+71>:        incq   %gs:0x11b78
0xffffffff811b0760 <handle_mm_fault+80>:        test   %esi,%esi
0xffffffff811b0762 <handle_mm_fault+82>:        jne    0xffffffff811b076e <handle_mm_fault+94>
0xffffffff811b0764 <handle_mm_fault+84>:        mov    $0xb,%esi
0xffffffff811b0769 <handle_mm_fault+89>:        callq  0xffffffff811eea80 <__mem_cgroup_count_vm_event>
0xffffffff811b076e <handle_mm_fault+94>:        mov    %gs:0xcdc0,%rax
0xffffffff811b0777 <handle_mm_fault+103>:       mov    0x478(%rax),%edx
0xffffffff811b077d <handle_mm_fault+109>:       lea    0x1(%rdx),%ecx
0xffffffff811b0780 <handle_mm_fault+112>:       cmp    $0x40,%edx
0xffffffff811b0783 <handle_mm_fault+115>:       mov    %ecx,0x478(%rax)
0xffffffff811b0789 <handle_mm_fault+121>:       jg     0xffffffff811b14b7 <handle_mm_fault+3495>
0xffffffff811b078f <handle_mm_fault+127>:       mov    %r14d,%eax
0xffffffff811b0792 <handle_mm_fault+130>:       and    $0x80,%eax
0xffffffff811b0797 <handle_mm_fault+135>:       mov    %eax,-0x3c(%rbp)
0xffffffff811b079a <handle_mm_fault+138>:       jne    0xffffffff811b0af0 <handle_mm_fault+992>
0xffffffff811b07a0 <handle_mm_fault+144>:       testb  $0x40,0x52(%r12)
0xffffffff811b07a6 <handle_mm_fault+150>:       jne    0xffffffff811b149e <handle_mm_fault+3470>
0xffffffff811b07ac <handle_mm_fault+156>:       mov    %rbx,%r13
0xffffffff811b07af <handle_mm_fault+159>:       shr    $0x24,%r13
0xffffffff811b07b3 <handle_mm_fault+163>:       and    $0xff8,%r13d
0xffffffff811b07ba <handle_mm_fault+170>:       add    0x58(%r15),%r13
0xffffffff811b07be <handle_mm_fault+174>:       mov    0x0(%r13),%rdi
0xffffffff811b07c2 <handle_mm_fault+178>:       test   %rdi,%rdi
0xffffffff811b07c5 <handle_mm_fault+181>:       je     0xffffffff811b14c8 <handle_mm_fault+3512>
0xffffffff811b07cb <handle_mm_fault+187>:       mov    %rdi,%rax
0xffffffff811b07ce <handle_mm_fault+190>:       nopl   0x0(%rax)
0xffffffff811b07d2 <handle_mm_fault+194>:       mov    %rbx,%rdx
0xffffffff811b07d5 <handle_mm_fault+197>:       movabs $0xffff880000000000,%rdi
0xffffffff811b07df <handle_mm_fault+207>:       movabs $0x3ffffffff000,%rcx
0xffffffff811b07e9 <handle_mm_fault+217>:       shr    $0x1b,%rdx
0xffffffff811b07ed <handle_mm_fault+221>:       and    %rcx,%rax
0xffffffff811b07f0 <handle_mm_fault+224>:       and    $0xff8,%edx
0xffffffff811b07f6 <handle_mm_fault+230>:       add    %rdi,%rdx
0xffffffff811b07f9 <handle_mm_fault+233>:       add    %rax,%rdx
0xffffffff811b07fc <handle_mm_fault+236>:       mov    %rdx,%r13
0xffffffff811b07ff <handle_mm_fault+239>:       je     0xffffffff811b0ae0 <handle_mm_fault+976>
0xffffffff811b0805 <handle_mm_fault+245>:       mov    (%rdx),%rdi
0xffffffff811b0808 <handle_mm_fault+248>:       test   $0xffffffffffffff9f,%rdi
0xffffffff811b080f <handle_mm_fault+255>:       je     0xffffffff811b14e7 <handle_mm_fault+3543>
0xffffffff811b0815 <handle_mm_fault+261>:       mov    %rdi,%rax
0xffffffff811b0818 <handle_mm_fault+264>:       nopl   0x0(%rax)
0xffffffff811b081c <handle_mm_fault+268>:       mov    %rbx,%rdx
0xffffffff811b081f <handle_mm_fault+271>:       movabs $0xffff880000000000,%r13
0xffffffff811b0829 <handle_mm_fault+281>:       movabs $0x3ffffffff000,%rcx
0xffffffff811b0833 <handle_mm_fault+291>:       shr    $0x12,%rdx
0xffffffff811b0837 <handle_mm_fault+295>:       and    %rcx,%rax
0xffffffff811b083a <handle_mm_fault+298>:       and    $0xff8,%edx
0xffffffff811b0840 <handle_mm_fault+304>:       add    %r13,%rdx
0xffffffff811b0843 <handle_mm_fault+307>:       add    %rax,%rdx
0xffffffff811b0846 <handle_mm_fault+310>:       mov    %rdx,%r13
0xffffffff811b084f <handle_mm_fault+319>:       mov    (%rdx),%rdi
```

可以看到+319出现*mov    %rdx,%r13* ，意味着%rdx赋值给%rdi，但是在+310可以看到%rdx已经赋值给%r13，所以可能是从别的地方转到这一行，%r13可能已经失效。

4. 对应

``` shell
    [exception RIP: handle_mm_fault+319]
    RIP: ffffffff811b084f  RSP: ffff8830e1b53c40  RFLAGS: 00010282
    RAX: 00000000fc14d000  RBX: 00007ffefed4c150  RCX: 00003ffffffff000
    RDX: ffff8800fc14dfb0  RSI: 000000000000000b  RDI: 00000000fc14d000
    RBP: ffff8830e1b53cc8   R8: ffff883f43635440   R9: 000000000000001d
    R10: 000000000000008f  R11: 0000000000002941  R12: ffff883f43635440
    R13: ffff8800fc14dfb0  R14: 0000000000000029  R15: ffff883ff7059f40
	ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
```

可以看到**R13: ffff8800fc14dfb0** ，查看R13里面内容是什么

``` shell
crash> kmem ffff8800fc14dfb0
kmem: WARNING: cannot find mem_map page for address: ffff8800fc14dfb0
```

或者用dis进行反编译

``` shell
crash> dis -l ffff8800fc14dfb0
dis: WARNING: ffff8800fc14dfb0: no associated kernel symbol found
   0xffff8800fc14dfb0:  dis: seek error: kernel virtual address: ffff8800fc14dfb0  type: "gdb_readmem_callback"
Cannot access memory at address 0xffff8800fc14dfb0
```

从结果看出寄存器的值在内存空间中无法查询到。具体原因还需要深究，笔者能力有限，先写到这里，日后有思路再书下文。









参考资料：

> https://www.ibm.com/developerworks/cn/linux/l-cn-kdump1/index.html 深入探索 Kdump，第 1 部分：带你走进 Kdump 的世界
>
> http://www.360doc.com/content/11/0721/17/6580811_135034260.shtml Linxu却页中断处理
>
> http://www.cnblogs.com/cherishui/p/3881428.html Kernel Panic常见原因以及解决方法
>
> http://blog.csdn.net/fishermandong/article/details/12112381 Kernel Panic (Kdump) 解析实例之一