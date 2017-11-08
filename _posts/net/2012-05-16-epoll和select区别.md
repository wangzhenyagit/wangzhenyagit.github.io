---
layout: post
title: epoll和select区别
category: 网络编程
tags: epoll select 
---

先说下本文框架，先是问题引出，然后概括两个机制的区别和联系，最后介绍每个接口的用法
# 问题引出 联系区别
问题的引出，当需要读两个以上的I/O的时候，如果使用阻塞式的I/O，那么可能长时间的阻塞在一个描述符上面，另外的描述符虽然有数据但是不能读出来，这样实时性不能满足要求，大概的解决方案有以下几种：
1. 使用多进程或者多线程，但是这种方法会造成程序的复杂，而且对与进程与线程的创建维护也需要很多的开销。（Apache服务器是用的子进程的方式，优点可以隔离用户）
2. 用一个进程，但是使用非阻塞的I/O读取数据，当一个I/O不可读的时候立刻返回，检查下一个是否可读，这种形式的循环为轮询（polling），这种方法比较浪费CPU时间，因为大多数时间是不可读，但是仍花费时间不断反复执行read系统调用。
3. 异步I/O（asynchronous I/O），当一个描述符准备好的时候用一个信号告诉进程，但是由于信号个数有限，多个描述符时不适用。
4. 一种较好的方式为I/O多路转接（I/O multiplexing）（貌似也翻译多路复用），先构造一张有关描述符的列表（epoll中为队列），然后调用一个函数，直到这些描述符中的一个准备好时才返回，返回时告诉进程哪些I/O就绪。select和epoll这两个机制都是多路I/O机制的解决方案，select为POSIX标准中的，而epoll为Linux所特有的。
区别（epoll相对select优点）主要有三：

1. select的句柄数目受限，在linux/posix_types.h头文件有这样的声明：#define __FD_SETSIZE    1024  表示select最多同时监听1024个fd。而epoll没有，它的限制是最大的打开文件句柄数目。
2. epoll的最大好处是不会随着FD的数目增长而降低效率，在selec中采用轮询处理，其中的数据结构类似一个数组的数据结构，而epoll是维护一个队列，直接看队列是不是空就可以了。epoll只会对"活跃"的socket进行操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。那么，只有"活跃"的socket才会主动的去调用 callback函数（把这个句柄加入队列），其他idle状态句柄则不会，在这点上，epoll实现了一个"伪"AIO。但是如果绝大部分的I/O都是“活跃的”，每个I/O端口使用率很高的话，epoll效率不一定比select高（可能是要维护队列复杂）。
3. 使用mmap加速内核与用户空间的消息传递。无论是select,poll还是epoll都需要内核把FD消息通知给用户空间，如何避免不必要的内存拷贝就很重要，在这点上，epoll是通过内核于用户空间mmap同一块内存实现的。

# 接口
## select
```int select(int maxfdp1, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict exceptfds, struct timeval *restrict tvptr);
struct timeval{
  long tv_sec;
  long tv_usec;
}
```
有三种情况：tvptr == NULL 永远等待；tvptr->tv_sec == 0 && tvptr->tv_usec == 0 完全不等待；不等于0的时候为等待的时间。select的三个指针都可以为空，这时候select提供了一种比sleep更精确的定时器。注意select的第一个参数maxfdp1并不是描述符的个数，而是最大的描述符加1，一是起限制作用，防止出错，二来可以给内核轮询的时候提供一个上届，提高效率。select返回－1表示出错，0表示超时，返回正值是所有的已经准备好的描述符个数（同一个描述符如果读和写都准备好，对结果影响是+2）。
```
int FD_ISSET(int fd, fd_set *fdset);  
```
fd在描述符集合中非0，否则返回0
```
int FD_CLR(int fd, fd_set *fd_set); int FD_SET(int fd, fd_set *fdset) ;int FD_ZERO(fd_set *fdset);
```
用一段linux 中man里的话
>FD_ZERO()  clears  a set.FD_SET() and  FD_CLR() respectively add and remove a given file descriptor from a set.  FD_ISSET() tests to see if a file descriptor is part of the set; this is useful after select() returns.

这几个函数与描述符的0和1没关系，只是添加删除检测描述符是否在set中。

## epoll
```
int epoll_create(int size);
```

创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
EPOLL_CTL_ADD：注册新的fd到epfd中；
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；
第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
```
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
```

events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。

EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
### 关于epoll工作模式ET，LT
LT(level triggered)是缺省的工作方式，并且同时支持block和no-block socket.在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表．
ET (edge-triggered)是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了，但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)
```
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)```

等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。
# 参考：
APUE(I/O多路转接)  
linux man epoll select  
[http://blog.chinaunix.net/uid-22663647-id-1771846.html ](http://blog.chinaunix.net/uid-22663647-id-1771846.html )  
[http://www.cnblogs.com/OnlyXP/archive/2007/08/10/851222.html](http://www.cnblogs.com/OnlyXP/archive/2007/08/10/851222.html)
