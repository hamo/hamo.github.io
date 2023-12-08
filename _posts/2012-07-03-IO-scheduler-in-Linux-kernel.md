---
title: "Linux内核中的IO调度器简介"
date: 2012-07-03T13:45+08:00
categories:
  - linux
  - kernel
tags:
  - linux
  - kernel
  - IO
---

从2.6系列开始，Linux内核引入了全新的IO调度子系统。与2.4系列内核只有一个惟一的，通用的I/O调度器相比，最新的Linux内核提供了CFQ(默认), deadline和noop三种IO调度器。（anticipatory调度器从2.6.33开始被移除，[commit 492af6350a5ccf087e4964104a276ed358811458](http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=492af6350a5ccf087e4964104a276ed358811458)，被CFQ所取代）。在这篇文章里，我会首先介绍三种IO调度器各自的特点和应用场景，之后会介绍Linux内核提供的为每一个块设备指定IO调度器和调整IO调度器参数的接口。Ok, let's go!

### CFQ(Complete Fair Queuing) ###
CFQ实现了一种QoS的IO调度算法。该算法为每一个进程分配一个时间窗口，在该时间窗口内，允许进程发射IO请求。通过时间窗口在不同进程间的移动，保证了对于所有进程而言都有公平的发射IO请求的权利。同时，与CFS(Complete Fair Schedule)相同，该算法也实现了进程的优先级控制，可以保证高优先级进程可以获得更长的时间窗口。主要代码位于
    block/cfq-iosched.c
CFQ调度算法适用于系统中存在多任务I/O请求的情况，通过在多进程中轮换，保证了系统I/O请求整体的低延迟。但是，对于只有少数进程存在大量密集的I/O请求的情况，则会出现明显的I/O性能下降。

CFQ调度器主要提供如下三个优化参数：

1. slice_idle

如果一个进程在自己的时间窗口里，经过slice_idle时间都没有发射I/O请求，则调度选择下一个程序。通过该机制，可以有效利用I/O请求的局部性原理，提高系统的I/O吞吐量。

2. quantum

该参数控制在一个时间窗口内可以发射的I/O请求的最大数目。

3. low_latency

对于I/O请求延时非常重要的任务，将该参数设置为1可以降低I/O请求的延时。

### NOOP ###
顾名思义，NOOP调度器并不完成任何复杂的工作，只是将上层发来的I/O请求直接发送至下层，实现了一个FIFO的I/O请求队列。主要代码位于
    block/noop-iosched.c
其应用环境主要有以下两种：一是物理设备包含自己的I/O调度程序，比如SCSI的TCQ；二是寻道时间可以忽略不计的设备，比如SSD等。

### DEADLINE ###
DEADLINE调度算法主要针对I/O请求的延时而设计，每个I/O请求都被附加一个最后执行期限。该算法维护两类队列，一是按照扇区排序的读写请求队列；二是按照过期时间排序的读写请求队列。如果当前没有I/O请求过期，则会按照扇区顺序执行I/O请求；如果发现过期的I/O请求，则会处理按照过期时间排序的队列，直到所有过期请求都被发射为止。在处理请求时，该算法会优先考虑读请求。主要代码位于
    block/deadline-iosched.c
当系统中存在的I/O请求进程比较少时，与CFQ算法相比，DEADLINE算法可以提供较高的I/O吞吐率，特别是对于使用了自带I/O请求队列的设备，如SCSI的TCQ的时候。

DEADLINE调度算法提供如下三个参数：

1. writes_starved

该参数控制当读写队列均不为空时，发射多少个读请求后，允许发射写请求。

2. read_expire

参数控制读请求的过期时间，单位毫秒。

3. write_expire

参数控制写请求的过期时间，单位毫秒。

### 为块I/O设备设置I/O调度算法的方法 ###
Linux内核允许用户为每个单独的块I/O设备设置不同的I/O调度算法。这样，根据块设备的不同以及读写该设备的应用不同，可以最大限度的提升系统的I/O吞吐率。设置接口通过sysfs导出。如果当前系统中没有挂载sysfs的话，使用如下命令可以将sysfs挂载至/sys目录下。

``` console
root@trinity:~# mount -t sysfs sysfs /sys
```
我们以sda为例，介绍修改设备调度算法的方式。
``` console
root@trinity:~# cat /sys/block/sda/queue/scheduler 
noop deadline [cfq] 
root@trinity:~# echo "deadline" > /sys/block/sda/queue/scheduler 
root@trinity:~# cat /sys/block/sda/queue/scheduler 
noop [deadline] cfq 
```
当然，这里的修改是临时的。系统重启后设备的调度算法会重置为默认的CFQ。如果希望保证每次系统启动都使用自定的调度算法，可以在
    /etc/fstab
文件中为对应的设备传递如下参数：
    elevator=deadline
便会将对应设备的调度算法设置为deadline，其他算法类似。
