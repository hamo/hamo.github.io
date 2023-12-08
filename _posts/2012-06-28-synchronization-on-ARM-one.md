---
title: "ARM体系架构下的同步操作（一）"
date: 2012-06-28T18:23+08:00
categories:
  - ARM
tags:
  - ARM
---

处理器在访问共享资源时，必须对临界区进行同步，即保证同一时间内，只有一个对临界区的访问者。当共享资源为一内存地址时，原子操作是对该类型共享资源同步访问的最佳方式。随着应用的日益复杂和SMP的广泛使用，处理器都开始提供硬件同步原语以支持原子地更新内存地址。

CISC处理器比如IA32，可以提供单独的多种原子指令完成复杂的原子操作，由处理器保证读-修改-写回过程的原子性。而RISC则不同，由于除Load和Store的所有操作都必须在寄存器中完成，如何保证从装载内存地址到寄存器，到修改寄存器中的值，再到将寄存器中的值写回内存中可以原子性的完成，便成为了处理器设计的关键。

从ARMv6架构开始，ARM处理器提供了Exclusive accesses同步原语，包含两条指令：
    LDREX
    STREX
LDREX和STREX指令，将对一个内存地址的原子操作拆分成两个步骤，同处理器内置的记录exclusive accesses的exclusive monitors一起，完成对内存的原子操作。

### LDREX ###
LDREX与LDR指令类似，完成将内存中的数据加载进寄存器的操作。与LDR指令不同的是，该指令也会同时初始化exclusive monitor来记录对该地址的同步访问。例如
``` gas
LDREX R1, [R0]
```
会将R0寄存器中内存地址的数据，加载进R1中并更新exclusive monitor。

### STREX ###
该指令的格式为：
``` gas
STREX Rd, Rm, [Rn]
```
STREX会根据exclusive monitor的指示决定是否将寄存器中的值写回内存中。如果exclusive monitor许可这次写入，则STREX会将寄存器Rm的值写回Rn所存储的内存地址中，并将Rd寄存器设置为0表示操作成功。如果exclusive monitor禁止这次写入，则STREX指令会将Rd寄存器的值设置为1表示操作失败并放弃这次写入。应用程序可以根据Rd中的值来判断写回是否成功。

在下篇文章中，我会以Linux Kernel中如何编写原子操作代码具体介绍LDREX和STREX的使用方法，并介绍gcc提供的ARM架构下的关于这两条指令的C语言扩展。
