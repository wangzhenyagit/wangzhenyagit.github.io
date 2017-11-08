---
layout: post
title: 用Debug Diagnostic诊断内存泄露问题
category: 我的技术
tags: Debug Diagnostic
---

调试一个MFC的程序，发现解决内存泄露问题，用Debug Diagnostic Tool1.2 
在定位的时候，需要注意多跑几次，多诊断几次，线程很多的话，每次的结果可能不一样，给的Call stack sample也可能不一样，不行就多跑一会。

即使多跑几次，堆栈信息也可能不足以一次定位到问题。给了一些分配内存大小的信息“Top 10 allocation sizes by allocation count”和“Top 10 allocation sizes by total size”，一开始也搞不清给这东西有啥用，后来才知道，很多情况下，即使有符号和堆栈信息，也不是一下子能定位到出问题的地方，通过这两个信息，很多时候对分析有很大帮助，如图：分析说明，在“CProtocolScanner!CThreadGroup::WorkProc”工作线程中，有个分配43k内存的地方是有地方的，这个43k的大小是自己后来改的，一开始是40k，这个40k是程序中的一个临时buffer的大小，我找到所有使用这个大小分配内存的地方，并且都修改了下，改成了41k，42k，43k等，在分析一次，就找到那个有问题的43k内存大小的地方了。扩展下，如果分析到了有问题的地方每次分配内存的大小，还可以通过查找对应结构体或者类的大小，来判断是哪个对象有泄露问题。问题定位最后如下图：
 
<img src = "http://img.blog.csdn.net/20170414150550491?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXN1MTA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" />

参考：
http://www.cnblogs.com/zhcncn/archive/2013/01/11/2856000.html 