---
layout: post
title: linux操作系统及内核
category: 操作系统
tags: linux
---

知识重新编码~
# 操作系统概述
在Richard Stevens的unix环境高级编程中这样定义“它控制计算机硬件资源，提供程序运行环境。一般而言我们称这种软件为内核（kernel），它相对较小，位于环境的中心”。总结下就是控制硬件，提供环境。程序员主要关心的是所谓的环境，主要说下提供什么环境。

操作系统都会想它们运行的程序提供各种服务，执行新的程序，打开文件，读文件，分配存储空间，获得当前时间等（一般通过系统调用）。
广义上，操作系统还有内核外的系统调用，基于系统调用的shell（也是一种特殊的应用程序，为其他应用程序提供接口）和库函数（对系统调用的封装），和基于shell，系统调用，库函数（这三个东西基本组成了我们常用的环境）的应用软件。有图如下：

<img src="http://img.my.csdn.net/uploads/201205/03/1336009173_3544.jpg" height="25%" width="25%" />
 
在使用linux的man帮助的时候可以指定是查询系统命令还是系统调用使用man时可以指定不同的section来浏览，各个section 如下：
1 - commands
2 - system calls
3 - library calls
其实还有其他的section 不常见就没有列出来，可以man 1 chmod 也可以 man 2 chmod 得到的帮助内容是不同的。
为了增加unix可移植性，IEEE定义了POSIX的标准，后来这个标准不只限于unix操作系统。POSIX标准只是定义了一套接口，并没有规定接口的实现（类似于概要设计），（各个操作系统对接口的实现可能有所不同），也没有详细的区分系统调用和库函数，所有的例程都叫做函数。需要说明的是，并不是每个操作系统都严格遵守POSIX标准，POSIX标准现在是一个很大的协议族（类似于TCP/IP），标准很多。
 
# Linux 是什么内核是什么
Linus Torvalds1991年的一片文章上写道

>LINUX is a free unix-like kernel for 386-AT computers, coming with full source code. It is meant for hackers/computer science students to use, learn and enjoy. It is written mostly inC, but parts of it are in gnu-format assembler, and the boot-sequence is in intel 086 assembly language. TheC-code is relatively ANSI, with a few GNU enhancements (mostly__asm__ andinline)

－其实，linux只是一个主要用c写的内核。
从不同的角度来看，内核担任的角色不同。从纯技术角度来看，内核只是软件和硬件的一个中间层，它把从软件发来的请求发送给硬件，完成寻址等操作，还充当了底层驱动。
从应用程序角度来看，内核是对硬件的一个高层次的抽象，应用程序与硬件没有联系，只与内核有联系，内核是应用程序知道的最底层。
从多个并发的进程的角度来看，内核是一个资源管理器，它完成对进程的切换，调度，共享计算机资源（CPU，内存，磁盘，网络等）。
还可以把内核看成一个库，通过系统调用向内核发送各种请求。
 
# 内核有什么
这个问题是淘宝面试的时候问我的问题，当时不知道从何下手，简单的总结下。有什么，最简单的就是直接看看内核源代码文件夹下有什么，一般内核文件在linux的目录/usr/src/kernels的文件夹下，我安装的操作系统是redhet的，当时没有安装上内核源文件，而且即使是安装上了也是2.6版本的，也不便于学习，所以下载了一个0.11版本的在http://www.oldlinux.org/index_cn.html上面，1.0版本及以上的可在http://www.kernel.org/pub/linux/kernel/上下载到。
简单看下1.0版本有什么文件主要的：

<img src="http://img.my.csdn.net/uploads/201205/20/1337502195_1875.jpg" />

drivers：驱动代码
fs：文件系统的代码
include ：包含文件，这个文件利用其他模块重建内核
init：初始化代码，内核工作的起点  //这里面有内核初始化程序main.c，是内核完成所有初始化工作并进入正常运行的关键
ipc：进程间通信的相关代码
kernel：主内核的代码 //最重要的是进程调度函数schedule()、sleep_on()函数和有关的系统调用程序
mm：内存管理的代码
net：网络管理的代码
0.11版本的.c文件代码有8578行，而1.0版本里面的.c文件代码大概有14w行，其中drives文件夹下就有7w行，2.6版本的有几百万行，估计那是任何大婶也读不完的～
上面简单的说明了源代码的目录结构，如果从系统的结构来看，linux操作系统可以分成五个比较核心的模块，进程调度模块，内存管理模块，文件系统模块，进程间通信模块和网络接口模块。其中的内存管理模块用于确保所有的进程能够安全地共享机器主要内存区，同时内存管理模块还支持虚拟内存的管理方式，使得Linux支持进程使用比实际内存空间多的内存容量。文件系统模块用于支持对外部设备的驱动和存储，虚拟文件系统模块通过对向所有的外部存储设备提供一个通用的文件接口，隐藏了各种硬件设备的不同细节，提高兼容性。下面是操作系统各个模块间的简单关系，虚线和虚框表示0.11上还为实现。

<img src="http://img.my.csdn.net/uploads/201205/21/1337566172_8285.jpg" />

从图中可以看出，所有的模块都与进程调度模块存在依赖关系，因为他们都需要依靠进程调度程序来挂起（暂停）或重新运行它们的进程。还可以根据源代码的结构将内核结构划分成如下的形式：

<img src="http://img.my.csdn.net/uploads/201205/21/1337566172_7469.jpg" />
