# FIO

 FIO是测试IOPS的非常好的工具，用来对硬件进行压力测试和验证，支持13种不同的I/O引擎，

包括:sync,mmap, libaio, posixaio, SG v3, splice, null, network, syslet, guasi, solarisaio 等等。 

> fio 官网地址：http://freshmeat.net/projects/fio/ 

## FIO安装 

``` shell 
wget http://brick.kernel.dk/snaps/fio-2.0.7.tar.gz 

yum install libaio-devel 

tar -zxvf fio-2.0.7.tar.gz 

cd fio-2.0.7 

make 

make install 
```

## 随机读测试： 

随机读： 

``` shell
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=16k -size=200G -numjobs=10 -runtime=1000 -group_reporting -name=mytest 
```

说明： 
`filename=/dev/sdb1` 测试文件名称，通常选择需要测试的盘的data目录。 
`direct=1` 测试过程绕过机器自带的buffer。使测试结果更真实。 
`rw=randwrite` 测试随机写的I/O 
`rw=randrw `测试随机写和读的I/O 
`bs=16k `单次io的块文件大小为16k 
`bsrange=512-2048` 同上，提定数据块的大小范围 
`size=5g `本次的测试文件大小为5g，以每次4k的io进行测试。 
`numjobs=30` 本次的测试线程为30. 
`runtime=1000` 测试时间为1000秒，如果不写则一直将5g文件分4k每次写完为止。 
`ioengine=psync` io引擎使用pync方式 
`rwmixwrite=30` 在混合读写的模式下，写占30% 
`group_reporting `关于显示结果的，汇总每个进程的信息。 
此外 
`lockmem=1g` 只使用1g内存进行测试。 
`zero_buffers `用0初始化系统buffer。 
`nrfiles=8 `每个进程生成文件的数量。 
顺序读： 

``` shell
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest 
```

随机写： 

``` shell
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest 
```

顺序写： 

``` shell
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest 
```

混合随机读写： 

``` shell
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=100 -group_reporting -name=mytest -ioscheduler=noop 
```

# IOZONE

## IOZone概述

 iozone是一个文件系统的benchmark工具，可以测试不同的操作系统中文件系统的读写性能。可以测试 Read,write, re-read,re-write, read backwards, read strided, fread, fwrite, randomread, pread, mmap, aio_read, aio_write 等等不同的模式下的硬盘的性能。

*测试的时候请注意，设置的测试文件的大小一定要大过你的内存(最佳为内存的两倍大小)，不然Linux会给你的读写的内容进行缓存，会使数值非常不真实。*

## IOZone特征

（1）   使用ANSI ‘C’编写

（2）   POSIX 异步I/O支持.

（3）   Mmap() 文件I/O支持.

（4）   普通文件I/O支持

（5）   单一流测试Single stream measurement.

（6）   多流测试Multiple stream measurement.

（7）   支持POSIX 线程.多进程测试.结果生成采用直观的Excel表格形式.

（8）   I/O 延迟数据Latency data for plots.

（9）   兼容64位系统.兼容大文件.

（10）  吞吐量测试中使用Stonewalling来避免不同步的问题.

（11）  可以配置处理器缓存大小.可选择是否使用fsync, O_SYNC进行测试.

（12）  可以针对NFS进行测试.

## IOZone安装

### 下载安装程序

首先从官方下载最新的iozone源码包，然后编译适合自己target的执行文件。　　    

``` shell
tar iozone3_347.tar
#cd iozone3_347/src/current　　
#make linux (直接执行make有帮助选项) 
```

## IOZone几种测试的定义

**Write**：测试向一个新文件写入的性能。当一个新文件被写入时，不仅仅是那些文件中的数据需要被存储，还包括那些用于定位数据存储在存储介质的具体位置的额外信息。这些额外信息被称作“元数据”。它包括目录信息，所分配的空间和一些与该文件有关但又并非该文件所含数据的其他数据。拜这些额外信息所赐，Write的性能Re-write：测试向一个已存在的文件写入的性能。当一个已存在的文件被写入时，所需工作量较少，因为此时元数据已经存在。Re-write的性能通常比Write的性能高。

**Re-write**：测试向一个已存在的文件写入的性能。当一个已存在的文件被写入时，所需工作量较少，因为此时元数据已经存在。Re-write的性能通常比Write的性能高。

**Read**：测试读一个已存在的文件的性能。

**Re-Read**：测试读一个最近读过的文件的性能。Re-Read性能会高些，因为操作系统通常会缓存最近读过的文件数据。这个缓存可以被用于读以提高性能。

**Random Read**：测试读一个文件中的随机偏移量的性能。许多因素可能影响这种情况下的系统性能，例如：操作系统缓存的大小，磁盘数量，寻道延迟和其他。

**Random Write**：测试写一个文件中的随机偏移量的性能。同样，许多因素可能影响这种情况下的系统性能，例如：操作系统缓存的大小，磁盘数量，寻道延迟和其他。

**Random Mix**：测试读写一个文件中的随机偏移量的性能。同样，许多因素可能影响这种情况下的系统性能，例如：操作系统缓存的大小，磁盘数量，寻道延迟和其他。这个测试只有在吞吐量测试模式下才能进行。每个线程/进程运行读或写测试。这种分布式读/写测试是基于round robin 模式的。最好使用多于一个线程/进程执行此测试。

**Backwards Read**：测试使用倒序读一个文件的性能。这种读文件方法可能看起来很可笑，事实上，有些应用确实这么干。MSC Nastran是一个使用倒序读文件的应用程序的一个例子。它所读的文件都十分大（大小从G级别到T级别）。尽管许多操作系统使用一些特殊实现来优化顺序读文件的速度，很少有操作系统注意到并增强倒序读文件的性能。

**Record Rewrite**：测试写与覆盖写一个文件中的特定块的性能。这个块可能会发生一些很有趣的事。如果这个块足够小（比CPU数据缓存小），测出来的性能将会非常高。如果比CPU数据缓存大而比TLB小，测出来的是另一个阶段的性能。如果比此二者都大，但比操作系统缓存小，得到的性能又是一个阶段。若大到超过操作系统缓存，又是另一番结果。

**Strided Read**：测试跳跃读一个文件的性能。举例如下：在0偏移量处读4Kbytes，然后间隔200Kbytes,读4Kbytes，再间隔200Kbytes，如此反复。此时的模式是读4Kbytes，间隔200Kbytes并重复这个模式。这又是一个典型的应用行为，文件中使用了数据结构并且访问这个数据结构的特定区域的应用程序常常这样做。许多操作系统并没注意到这种行为或者针对这种类型的访问做一些优化。同样，这种访问行为也可能导致一些有趣的性能异常。一个例子是在一个数据片化的文件系统里，应用程序的跳跃导致某一个特定的磁盘成为性能瓶颈。

**Fwrite**：测试调用库函数fwrite()来写文件的性能。这是一个执行缓存与阻塞写操作的库例程。缓存在用户空间之内。如果一个应用程序想要写很小的传输块，fwrite()函数中的缓存与阻塞I/O功能能通过减少实际操作系统调用并在操作系统调用时增加传输块的大小来增强应用程序的性能。

这个测试是写一个新文件，所以元数据的写入也是要的。

**Frewrite**：测试调用库函数fwrite()来写文件的性能。这是一个执行缓存与阻塞写操作的库例程。缓存在用户空间之内。如果一个应用程序想要写很小的传输块，fwrite()函数中的缓存与阻塞I/O功能能通过减少实际操作系统调用并在操作系统调用时增加传输块的大小来增强应用程序的性能。

这个测试是写入一个已存在的文件，由于无元数据操作，测试的性能会高些。

**Fread**：测试调用库函数fread()来读文件的性能。这是一个执行缓存与阻塞读操作的库例程。缓存在用户空间之内。如果一个应用程序想要读很小的传输块，fwrite()函数中的缓存与阻塞I/O功能能通过减少实际操作系统调用并在操作系统调用时增加传输块的大小来增强应用程序的性能。

**Freread**：这个测试与上面的fread 类似，除了在这个测试中被读文件是最近才刚被读过。这将导致更高的性能，因为操作系统缓存了文件数据。通常会比Re-write的性能低。

##测试参数

下面是一些在IOZone进行测试时的相关选项：

Usage: 

``` shell
iozone -sfilesize_Kb [-f [path]filename]

-i test -p -A -Z -M -h

-lmin_number_procs -v [-x]

-d microseconds -V pattern

-T -B-G -H depth [-U mount_point]

-S cache_size-K [-g max_filesize_Kb]

-n min_filesize_Kb -Q -c [-b filename]

-J milliseconds -Y filename [-W]

-ymin_recordsize_Kb [-+m filename]

-+u  -+ppercent_read -+t 
```

下面会介绍一些比较常用的选项参数，其他的一些选项参数可以在IOZone的主页（<http://www.iozone.org/>）上进行下载IOZone的使用手册进行了解。

-h 显示帮助。

-a 用来使用全自动模式。生成包括所有测试操作的报告，使用的块大小从4k到16M，文件大小从64k到512M。

-A 这种版本的自动模式提供更加全面的测试但是消耗更多时间。参数–a在文件不小于32MB时将自动停止使用低于64K的块大小测试。这节省了许多时间。而参数–A则告诉Iozone你不介意等待，即使在文件非常大时也希望进行小块 的测试。

注意： 不推荐在Iozone3.61版中使用这个参数。使用–az –i 0 –i 1替代。

-R 产生Excel到标准输出，-b 指定输出到指定文件上. 比如 -Rb ttt.xls

-i N 用来选择测试项, 比如Read/Write/Random 比较常用的是0 1 2,可以指定成-i 0 -i 1-i 2。 

> 0=write/rewrite
>
> 1=read/re-read
>
> 2=random-read/write
>
> 3=Read-backwards 
>
> 4=Re-write-record 
>
> 5=stride-read 
>
> 6=fwrite/re-fwrite 
>
> 7=fread/Re-fread 
>
> 8=random mix 
>
> 9=pwrite/Re-pwrite 
>
> 10=pread/Re-pread 
>
> 11=pwritev/Re-pwritev
>
> 12=preadv/Re-preadv 

*-r block size* 指定一次写入/读出的块大小 

*-s file size* 指定测试文件的大小 

*-f filename* 指定测试文件的名字,完成后会自动删除(这个文件必须指定你要测试的那个硬盘中) 

批量测试项: 

*-g -n* 指定测试文件大小范围,最大测试文件为4G,可以这样写 -g 4G 

*-y -q* 指定测试块的大小范围 

## 测试实例

下面是测试中的一个测试实例：

``` shell
./iozone -a -g 16G -i 0 -i 1 -i 2 -f /opt/iozone -Rb iotestresult-8KB.xls
```

*进行全面测试.最大测试文件为16G，从64K开始。测试write/rewrite，read/re-read，random-read/write。本测试是将9个盘做RAID5之后挂载到/opt/iozone，测试地方在/opt/iozone。输出文件到iotestresult-8KB.xls中。*

### 结果分析：

使用这条测试命令测试磁盘后，下面列举出了产生的Execl文件中的一段表，并进行了简单的解释。

| Writer Report |      |      |       |       |       |
| ------------- | ---- | ---- | ----- | ----- | ----- |
|               | 4    | 8    | 16    | 32    | 64    |
| 32768         | 5300 | 8166 | 12726 | 16702 | 24441 |
| 65536         | 5456 | 8285 | 9630  | 16101 | 18679 |
| 131072        | 5539 | 6968 | 9453  | 13086 | 14136 |
| 262144        | 5088 | 7092 | 9634  | 11602 | 14776 |
| 524288        | 5427 | 9356 | 10502 | 13056 | 13865 |
| 1048576       | 6061 | 9625 | 11528 | 12632 | 13466 |

在Execl文件中的这段表，它说明了这个表单是关于*write*的测试结果，左侧一列是测试文件大小*（Kbytes)*，最上边一行是记录大小，中间数据是测试的传输速度。举例说明，比如表中的*“5300”*，意思是测试文件大小为*32M*，以记录大小为*4K*来进行传输，它的传输速度为*5300Kbytes/s*。

 