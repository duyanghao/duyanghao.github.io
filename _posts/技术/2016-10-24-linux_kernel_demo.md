---
layout: post
title: 麻雀虽小，五脏俱全——Linux demo内核分析
category: 技术
tags: Linux
keywords:
description:
---

内核是一个运维开发人员进阶的必要知识，许多高层应用都利用了内核的一些特性，包括最近几年特别流行的Docker技术，要理解这些高层应用技术，则必须熟悉内核。这里给出一个我在学习内核的过程中遇到的一个很经典的内核demo例子，作为内核的入门，个人认为是非常有用的！

直接通过git将[代码](https://github.com/duyanghao/DemoOs)拉下来，下面是该内核代码在Linux 0.12环境下编译后在Bochs下的模拟执行图： 

![](/public/img/linux/linux_demo.png)

这个Demo OS主要实现的功能就如大家看到的一样，有两个任务（不能说是进程，因为并没有PCB结构），分别打印字符A和B（利用系统调用中断0x80）。由定时中断（每10ms发生一次）切换两个任务，这样每个任务大约可以打印10个字符，轮流交替（每打印一个字符则循环等待到1ms时间）。

虽说这个OS小，但是X86 CPU体系结构的基本特性都应用到了。其中包括GDT、LDT、IDT、BIOS、描述符（中断门和陷阱门）、选择符、内核态、用户态、实模式、保护模式以及中断等。

下面根据源码详细说明原理：

![](/public/img/linux/linux_github_demo.png)

项目中主要是三个文件，Makefile、boot.s、head.s。其中boot.s为as86汇编编写的引导启动程序，head.s为GUN as汇编语言编制的内核程序。

boot.s：

```
!   boot.s
!
! It then loads the system at 0x10000, using BIOS interrupts. Thereafter
! it disables all interrupts, changes to protected mode, and calls the 

BOOTSEG = 0x07c0
SYSSEG  = 0x1000            ! system loaded at 0x10000 (65536).
SYSLEN  = 17                ! sectors occupied.

entry start
start:
    jmpi    go,#BOOTSEG
go: mov ax,cs
    mov ds,ax
    mov ss,ax
    mov sp,#0x400       ! arbitrary value >>512

! ok, we've written the message, now
load_system:
    mov dx,#0x0000
    mov cx,#0x0002
    mov ax,#SYSSEG
    mov es,ax
    xor bx,bx
    mov ax,#0x200+SYSLEN
    int     0x13
    jnc ok_load
die:    jmp die

! now we want to move to protected mode ...
ok_load:
    cli         ! no interrupts allowed !
    mov ax, #SYSSEG
    mov ds, ax
    xor ax, ax
    mov es, ax
    mov cx, #0x2000
    sub si,si
    sub di,di
    rep
    movw
    mov ax, #BOOTSEG
    mov ds, ax
    lidt    idt_48      ! load idt with 0,0
    lgdt    gdt_48      ! load gdt with whatever appropriate

! absolute address 0x00000, in 32-bit protected mode.
    mov ax,#0x0001  ! protected mode (PE) bit
    lmsw    ax      ! This is it!
    jmpi    0,8     ! jmp offset 0 of segment 8 (cs)

gdt:    .word   0,0,0,0     ! dummy

    .word   0x07FF      ! 8Mb - limit=2047 (2048*4096=8Mb)
    .word   0x0000      ! base address=0x00000
    .word   0x9A00      ! code read/exec
    .word   0x00C0      ! granularity=4096, 386

    .word   0x07FF      ! 8Mb - limit=2047 (2048*4096=8Mb)
    .word   0x0000      ! base address=0x00000
    .word   0x9200      ! data read/write
    .word   0x00C0      ! granularity=4096, 386

idt_48: .word   0       ! idt limit=0
    .word   0,0     ! idt base=0L
gdt_48: .word   0x7ff       ! gdt limit=2048, 256 GDT entries
    .word   0x7c00+gdt,0    ! gdt base = 07xxx
.org 510
    .word   0xAA55
```

boot.s编译出的代码共512B，刚好一个扇区长度，将被存放于软盘映像文件的第一个扇区。PC加电启动后，ROM BIOS中的程序会把启动盘上第一个扇区加载到物理内存0x7c00（31KB）位置开始处，并把执行权转移到0x7c00处开始运行boot程序代码。


