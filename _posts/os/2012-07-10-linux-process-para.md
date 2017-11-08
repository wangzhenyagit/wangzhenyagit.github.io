---
layout: post
title: Linux下进程具体属性
category: 操作系统
tags: linux
---

## 进程数据结构（进程描述符）
直接查看下源码(这里是0.11版本的内核)中的文件/include/linux/sched.h，Linux的进程控制块为一个由结构task_struct所定义的数据结构，这个结构就在上面的sched.h中。这个文件中有一行代码：
```
extern struct task_struct *task[NR_TASKS];
```
为记录指向各PCB的指针，指针数组定义于/kernel/sched.c中，原定义为
```
struct task_struct * task[NR_TASKS] = {&(init_task.task), };
```
NR_TASKS为最多可以同时运行的进程的数目，在sched.h中定义（#define NR_TASKS 64）。
task_struct结构的内容如下（为了写注释有改动，像unsigned long start_code,end_code,end_data,brk,start_stack;会展开成多行写）：
```
struct task_struct {
/* these are hardcoded - don't touch */
        long state;     /* -1 unrunnable, 0 runnable（就绪）, >0 stopped */
        long counter;   //任务运行时间片
        long priority;  
        long signal;    //信号。每个比特代表一种信号。
        struct sigaction sigaction[32];
        long blocked;   /* bitmap of masked signals 进程信号屏蔽码*/
/* various fields */
        int exit_code;  //退出码，父进程会取得
         unsigned long start_code;  //代码段地址
        unsigned long end_code;    //代码长度（字节数）
        unsigned long end_data;    //代码长度+数据长度（字节数）
        unsigned long brk;    //总长度
        unsigned long start_stack; //堆栈段地址
        long pid,father,pgrp,session,leader; //进程、父进程、进程组、会话、会话首进程号
        unsigned short uid,euid,suid;  //用户id、有效用户id、保存的用户id
        unsigned short gid,egid,sgid;  //组id、有效组id，保存的组id
        long alarm;                    //闹钟定时
        long utime,stime,cutime,cstime,start_time;  //用户态（系统态）运行时间、子进程用户态（系统太）
        unsigned short used_math;      //是否使用了协处理器
/* file system info */
        int tty;                //进程使用tty的子设备号，－1标识没有使用
        unsigned short umask;   //文件创建屏蔽字
        struct m_inode * pwd;   //当前工作目录i节点结构
        struct m_inode * root;  //根节点i节点结构
        struct m_inode * executable;  //执行文件i节点结构
        unsigned long close_on_exec;  //执行时关闭文件句柄位图标志
        struct file * filp[NR_OPEN];  //进程使用的文件表结构
/* ldt for this task 0 - zero 1 - cs 2 - ds&ss */
        struct desc_struct ldt[3];    //本任务的局部表描述符。0－空，1－代码段cs，2－数据段和堆栈段ds&ss
/* tss for this task */
        struct tss_struct tss;        //进程的任务状态段信息结构
};
```
 
## 进程部分属性说明
1. 进程ID和父进程ID
2. 实际用户ID、实际组ID、有效用户ID、有效组ID、保存的设置用户ID、保存的设置组ID、附加组、文件模式创建屏蔽字
3. 进程组ID、会话组ID、控制终端
4. tms_utime、tms_stime、tms_cutime、tms_ustime
5. 信号屏蔽字、信号集、闹钟
6. 文件锁（记录锁）
7. 环境 
8. 运行状态

### 进程ID
每个进程都是有ID的，线程也有，但是系统中的进程ID号是有限的，而且可能同时运行的进程的数目也是有限制的，所以进程ID号也是一种资源。僵尸是进程在进程结束后父进程没有及时的回收终止子进程的有关信息情况下产生的，这些僵尸进程会占用进程ID。
 
### 实际用户ID、实际组ID、有效用户ID、有效组ID、保存的设置用户ID、保存的设置组ID、附加组ID
这里分三个问题来讲：
1. 权限问题一直是很复杂的。先简单介绍下几个概念：
1）实际用户ID和实际组ID
这两个ID标识我们究竟是谁，也就是以什么用户登录的，在一个登录会话间这些值并不改变。
2）有效用户ID和有效组ID
这两个ID决定了我们文件的访问权限，下面会具体说明这个ID的作用
3）保存的设置用户ID和保存的设置组ID
其实为有效用户ID和有效组ID的副本
4）文件的访问权限
文件的访问权限分为三类，所有者，组，其他，每类都有读、写、可执行三项。
可以用 setuid() setgid() 来设置实际用户（组）和有效用户（组）ID
可以用 chmod()改变文件的权限，chown改变文件的所有者和所有组

2. 进程每次打开、创建或者删除一个文件的时候，内核就进行文件访问权限测试，测试分四步：
1）若进程的有效用户ID是0（root），则允许访问。
2）若进程的有效用户ID等于文件的所有者ID（也就是进程拥有此文件），：那么若所有者适当的访问权限被设置，则允许访问，否则拒绝访问。
3）若进程的有效组或进程的附加组ID之一等于文件的组ID，那么若所有者适当的访问权限被设置，则允许访问，否则拒绝访问。
4）若其他用户适当的访问权限被设置，则允许访问，否则拒绝
按照顺序执行四步，如果进程拥有此文件（2），则按照用户访问权限批注或拒绝该进程对文件的访问，不查看组访问权限。下面也类似，也就是说并不是一定四步都执行完，可能从第二步开始就break了。

3. 关于文件模式创建屏蔽字(umask)
每个进程创建一个新的文件或者目录的时候，都需要一个文件模式创建屏蔽字，这个屏蔽字是默认的创建的文件或者目录的权限。系统的大多数用户并不处理，通常在登录的时候由shell的启动文件设置一次，然后从不改变。可用umask命令直接查看，也可以直接用umask 777直接更改。注意，这里是屏蔽(u意义),如果设置成777，那么一个目录的权限为d---------。
一般而言，在设计程序时候，我们总是试图使用最小特权（least privilege）模型。依照此模型，我们的程序应该具有完成给定任务所需的最小特权，这减少了安全性受到损害的可能性。
 
## 进程组ID、会话组ID、控制终端
在网上搜了下，好多也都是apue上的概念，我理解，进程组，会话组的概念出现是为了linux方便管理的，主要是为了作业控制的。最常见的就是在终端上发送一个中断信号（crtl+c），这个中断信号会发送到控制终端的前台进程组。简单说，进程组是一个或者多个进程的集合，而会话（session）是一个或者多个进程组的集合。理解这两个概念有以下几点：
- 进程组中的进程通常与同一作业相关联，可以接收来自同一终端的各种信号，每个进程组有一个唯一的进程组ID；每个进程组可以有一个组长进程，组长进程的ID等于进程组的ID；
- 会话是一个或者多个进程组的集合。通常，shell的管道将几个进程变成一组的（这也是为什么说进程组中的进程与同一作业相关）。
- 一个会话可以有一个控制终端。这通常是登录到其上的终端设备或伪终端设备。
- 一个会话中的进程组可以被分成一个前台进程组（foreground process group）以及一个或几个后台进程组（background process group）。
- 如果一个会话有一个控制终端，则它有一个前台进程组，会话中的其他进程组为后台进程组。
- 无论何时键入终端的中断键（delete 或者ctrl + c），就会将中断信号发送给前台进程组的所有进程。
- 无论何时键入终端退出键（ctrl + \），就会将退出信号发送给前台进程组的所有进程。
- 建立与终端连接的会话首进程被称为控制进程。
对于这个问题，还是举例简单的例子来说明下，还是在red hat上搞的，用的为bourne-again shell。
ps中的TPGID：ID of the foreground process group on the tty (terminal)，that the process is connected to, or -1 if the process is not connected to a tty.
运行以下命令

>ps -o pid,ppid,pgrp,session,tpgid,comm

输出：
```
 PID  PPID  PGRP  SESS TPGID COMMAND
 3395  3393  3395  3395  3469 bash
 3469  3395  3469  3395  3469 ps
```
上面说的tpgid为终端的前台进程组，其实也可以理解为一个会话的前台进程组，因为一个会话可以有一个终端，如果有的话，应该是一一对应的关系。可以看出在linux上（其他的操作系统可能不一样，linux是支持作业的，如果不支持作业可能不一样），bash为后台进程组的，而ps进程为前台进程组（tpgid可以看出），且这个会话（终端）的前台进程组只有一个进程为ps（前台进程组id为ps所在进程组id，且ps为这个组的组长）。还可以看出ps进程的父进程为3495即为bash，那么可知bash在运行一个进程的时候都是用fork命令启动一个子进程，在用exec命令来替换进程的。
>ps -o pid,ppid,pgrp,session,tpgid,comm | cat

输出：
```
 PID  PPID  PGRP  SESS TPGID COMMAND
 3395  3393  3395  3395  3501 bash
 3501  3395  3501  3395  3501 ps
 3502  3395  3501  3395  3501 cat
```
可以看出，用管道连接在一起的两个进程ps和cat在一个进程组里面（3501），且为前台进程组，而且这两个进程的父进程都为3395（bash）。

>ps -o pid,ppid,pgrp,session,tpgid,comm | cat &

先输出：
[1] 3508
在输出：
```
  PID  PPID  PGRP  SESS TPGID COMMAND
 3395  3393  3395  3395  3395 bash
 3507  3395  3507  3395  3395 ps
 3508  3395  3507  3395  3395 cat
```
先出的[1] 3508，为作业号号，然后输出的为进程的输出结果，在按一下空格会出现[1]+  Done ps -o pid,ppid,pgrp,session,tpgid,comm | cat为作业1完成的提示。这里的ps和cat为一个后台进程组（3507），tpgid指示的前台进程组为3395，为用户登录的bash。
 
## tms_utime、tms_stime、tms_cutime、tms_cstime
注意：在fork后子进程中，这几个时间参数被置为0。
任何一进程都可以调用times函数来获得他自己，以及已经终止的子进程的墙上时钟时间、用户CPU时间和系统CPU时间。这几个概念在前面的博客中（http://blog.csdn.net/ysu108/article/details/7476810）提到过，不再重复。这里说明下这个times函数，用man 2 times可以得到这个函数的说明：times() stores the current process times in the struct tms that buf points to.  The struct tms is as defined in <sys/times.h>:
```      
	   struct tms {
              clock_t tms_utime;  /* user time */
              clock_t tms_stime;  /* system time */
              clock_t tms_cutime; /* user time of children */
              clock_t tms_cstime; /* system time of children */
       };
```
>The  tms_utime field contains the CPU time spent executing instructions of the calling process.  The tms_stime field contains the CPU time spent in the system while executing tasks on behalf of the calling process.  The tms_cutime field contains the sum of  the  tms_utime  and  tms_cutime values  for all waited-for terminated children.  The tms_cstime field contains the sum of the tms_stime and tms_cstime values for all waited-for terminated children.All times reported are in clock ticks.
>RETURN VALUE times()  returns  the number of clock ticks that have elapsed since an arbitrary point in the past.

需要注意的有：这个结构没有包含墙上时钟时间的任何测量值，函数返回墙上时钟时间作为其函数值，此值为相对过去的某一时刻测量的，所以不能用其绝对值，而必须使用其相对值。有系统命令time,可以直接查看进程运行的时间。
 
## 信号屏蔽字、信号集、闹钟
注意：在子进程中未处理的信号集设置为空集、未处理的闹钟被清除
### 信号概念
信号提供了一种处理异步事件的方法，产生信号的事件对于进程而言是随机的出现的，进程不能简单地测试一个变量来判断是否产生了一个信号，而是必须告诉内核“在此信号出现时请执行下列操作”，可以要求内核在某个信号出现时候按照下列三种方式之一来处理：1.忽略此信号，但是有两个信号不能忽略SIGKILL和SIGSTOP，这种信号不能忽略的原因是他们向超级用户提供了是进程终止或者停止的可靠方法。2.捕捉信号，为了做到这一点要通知内核在某种信号发生时候调用一个用户函数。3.执行默认动作，大多数的信号默认动作是终止该进程。
每个进程都有一个信号屏蔽字（signal mask），它规定了当前要阻塞递送到该进程的信号集。对于每种可能的信号，该屏蔽子都有一位与之对应。对于某种信号，若其对应位已设置，则它当前是被阻塞的。进程可以调用sigprocmask来检测和更改当前信号屏蔽字。信号数量可能会超过整型包含的二进制位数（这种用每一位来表示一个特征的用法在linux中相当普遍），因此POSIX定义了一个新数据类型sigset_t，用于保存一个信号集合。
### 信号与进程
- 当一个程序启动的时候，所有信号状态都是系统默认或忽略，通常所有信号都被设置为她们的默认动作，除非调用exec函数，确切的讲，exec函数将原先设置为要捕捉信号都改为他们的默认动作，其他信号的状态则不改变，对于一个进程原先要捕捉的信号，当执行一个程序后，自然不能在捕捉它了，因为信号捕捉函数的地址很可能在所执行的新程序中已无意义。一个具体的例子就是当shell在后台执行一个进程的时候，shell自动将进程对中断和退出信号的处理方式设置为忽略，于是当用户按中断键的时候不会影响到后台进程。
- fork的时候,前面其实都是铺垫，当一个进程调用fork的时候，其子进程继承父进程的信号处理方式，因为子进程在开始时复制了父进程的存储映像，所以信号捕捉函数的地址在子进程中是有意义的。
- alarm,使用alarm函数可以设置一个计时器，在将来某个指定的时间该计时器会超时。当计时器超时时，产生SIGALRM信号。如果不忽略或不捕捉此信号，则其默认动作是终止调用该alarm函数的进程。每个进程只能有一个闹钟时钟。
 
## 文件锁（记录锁）
记录锁（record locking）的功能是：当一个进程正在读或修改文件的某个部分时，它可以阻止其他进程修改同一文件区。对于UNIX系统而言，“记录”这个词是一种误用，因为UNIX系统内核根本没有使用文件记录这种概念。更适合的术语是字节范围锁（byte-range locking），因为它锁定的只是文件中的一个区域（也可能是整个文件）。
记录锁不仅可以方式多个进程多于同一个文件做修改，还可以创建单一的进程（类似单例模式），不能启动两个同样的进程。当进程创建的时候先去锁定一个目标文件（一般文件名为进程的名字），当第二个相同进程创建的时候也要去锁定同样的文件，因为第一个进程已经创建，锁定失败，进程退出。
 
## 环境
这个部分应该是进程运行的时候的程序映像中高地址部分的内容，和命令行参数一起。有HOME（用户工作的目录）、PATH、SHELL、USER、LOGNAME。
 
## 运行状态
Linux 的进程有五种状态，同样在sched.h头文件中定义
在文件的前面定义了进程的运行状态

#define TASK_RUNNING            0 可运行状态，相当于进程三种状态的执行和就绪状态

#define TASK_INTERRUPTIBLE      1 中断等待状态。处于这种状态唤醒的原因可能是信号或定时中断，或者I\O就绪

#define TASK_UNINTERRUPTIBLE    2 不可中断等待状态，主要是等待I\O

#define TASK_ZOMBIE             3 僵死状态，进程已经结束已经释放除了PCB以外的部分系统资源，等待父进程wait()读取结束状态

#define TASK_STOPPED            4 进程已经停止

注意：这是0.11的内核，在1.0的内核以上就多了一种状态，在1.0内核的sched.h中有定义

#define TASK_SWAPPING           5 交换状态，进程的页面也可以从内存转换到外存，以空出内存空间 

## 参考：
apue
linux 内核0.11完全注解
