---
title: "ARM体系架构下的同步操作（二）"
date: 2012-06-29T12:31+08:00
categories: ARM
categories:
  - ARM
tags:
  - ARM
---
在[上一篇文章](/arm/2012/06/28/synchronization-on-ARM-one/ "ARM体系架构下的同步操作（一）")中，我们介绍了ARM体系架构下，为了实现对内存地址同步访问而引入的
    LDREX
    STREX
两条指令。在这篇文章里，首先会以Linux Kernel中ARM架构的原子相加操作为例，介绍这两条指令的使用方法；之后，会介绍GCC提供的一些内置函数，这些同步函数使用这两条指令完成同步操作。
### Linux Kernel中的atomic_add函数 ###
如下是Linux Kernel中使用的atomic_add函数的定义，它实现原子的给v指向的atomic_t增加i的功能。
``` c atomic_add
static inline void atomic_add(int i, atomic_t *v)
{
        unsigned long tmp;
        int result;

        __asm__ __volatile__("@ atomic_add\n"
"1:     ldrex   %0, [%3]\n"
"       add     %0, %0, %4\n"
"       strex   %1, %0, [%3]\n"
"       teq     %1, #0\n"
"       bne     1b"
        : "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)
        : "r" (&v->counter), "Ir" (i)
        : "cc");
}
```
在第7行，使用LDREX指令将v->counter所指向的内存地址的值装入寄存器中，并初始化exclusive monitor。
在第8行，将该寄存器中的值与i相加。
在第9，10，11行，使用STREX指令尝试将修改后的值存入原来的地址，如果STREX写入%1寄存器的值为0，则认为原子更新成功，函数返回；如果%1寄存器的值不为0，则认为exclusive monitor拒绝了本次对内存地址的访问，则跳转回第7行重新进行以上所述的过程，直到成功将修改后的值写入内存为止。该过程可能多次反复进行，但可以保证，在最后一次的读-修改-写回的过程中，没有其他代码访问该内存地址。
### GCC内置的原子操作函数 ###
看了上面的GCC内联汇编，是不是有点晕？在用户态下，GCC为我们提供了一系列内置函数，这些函数可以让我们既享受原子操作的好处，又免于编写复杂的内联汇编指令。这一系列的函数均以__sync开头，分为如下几类：
``` c 
type __sync_fetch_and_add (type *ptr, type value, ...)
type __sync_fetch_and_sub (type *ptr, type value, ...)
type __sync_fetch_and_or (type *ptr, type value, ...)
type __sync_fetch_and_and (type *ptr, type value, ...)
type __sync_fetch_and_xor (type *ptr, type value, ...)
type __sync_fetch_and_nand (type *ptr, type value, ...)
```
这一系列函数完成对ptr所指向的内存地址的对应操作，并返回操作之前的值。
``` c
type __sync_add_and_fetch (type *ptr, type value, ...)
type __sync_sub_and_fetch (type *ptr, type value, ...)
type __sync_or_and_fetch (type *ptr, type value, ...)
type __sync_and_and_fetch (type *ptr, type value, ...)
type __sync_xor_and_fetch (type *ptr, type value, ...)
type __sync_nand_and_fetch (type *ptr, type value, ...)
```
这一系列函数完成对ptr所指向的内存地址的对应操作，并返回操作之后的值。
``` c
bool __sync_bool_compare_and_swap (type *ptr, type oldval, type newval, ...)
type __sync_val_compare_and_swap (type *ptr, type oldval, type newval, ...)
```
这两个函数完成对变量的原子比较和交换。即如果ptr所指向的内存地址存放的值与oldval相同的话，则将其用newval的值替换。
返回bool类型的函数返回比较的结果，相同为true，不同为false；返回type的函数返回的是ptr指向地址交换前存放的值。


