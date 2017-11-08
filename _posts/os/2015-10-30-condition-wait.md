---
layout: post
title: 条件变量的虚假唤醒
category: 操作系统
tags: 条件变量
---

### 一、使用信号量与锁配合也可以实现多线程操作，使用条件变量有什么优势？
这个问题感觉也没明白，看了stack overflow上的一些讨论，感觉条件变量能够实现的，信号量都能实现。

1. 使用条件变量可以一次唤醒所有等待者，而这个信号量没有的功能，感觉是最大区别。
2. 信号量是有一个值（状态的），而条件变量是没有的，没有地方记录唤醒（发送信号）过多少次，也没有地方记录唤醒线程（wait返回）过多少次。从实现上来说一个信号量可以是用mutex + counter + condition variable实现的，而一个condition感觉没特别的地方，只有有个“原子操作”的普通event。因为信号量有一个状态，如果想精准的同步，那么信号量可能会有特殊的地方。
3. 其他说的说法就有些争议。
有的说linux下信号量多在内核空间，而条件变量多在用户空间，而条件变量多在用户空间实现。信号量在内核空间的话会会有过多的用户空间/内核空间切换，消耗性能。
下面一段引用的话：

> 在Posix.1基本原理一文声称，有了互斥锁和条件变量还提供信号量的原因是：“本标准提供信号量的而主要目的是提供一种进程间同步的方式；这些进程可能共享也可能不共享内存区。互斥锁和条件变量是作为线程间的同步机制说明的；这些线程总是共享(某个)内存区。这两者都是已广泛使用了多年的同步方式。每组原语都特别适合于特定的问题”。尽管信号量的意图在于进程间同步，互斥锁和条件变量的意图在于线程间同步，但是信号量也可用于线程间，互斥锁和条件变量也可用于进程间。应当根据实际的情况进行决定。信号量最有用的场景是用以指明可用资源的数量。
> 由于起源不同，导致了两种理念，一中理念力挺条件变量(condition variable)，觉得信号量没有什么用（例如POSIX
> Thread模型中没有信号量的概念，虽然也提出了Posix
> Semaphore，但是为什么一开始不把它放在一起呢？）；另一理念恰好相反（例如window刚开始没有条件变量的概念，只有信号量的概念）。
> 进化到后来，目前的linux和window都同时具备了这二者。

### 二、虚拟唤醒问题

条件变量中存在的问题：虚假唤醒
Linux中帮助中提到的：

> 在多核处理器下，pthread_cond_signal可能会激活多于一个线程（阻塞在条件变量上的线程）。 On a
> multi-processor, it may be impossible for an implementation of
> pthread_cond_signal() to avoid the unblocking of more than one thread
> blocked on a condition variable.  
> 结果是，当一个线程调用pthread_cond_signal()后，多个调用pthread_cond_wait()或pthread_cond_timedwait()的线程返回。这种效应成为”虚假唤醒”(spurious
> wakeup) 
> 
> The effect is that more than one thread can return from its call to
> pthread_cond_wait() or pthread_cond_timedwait() as a result of one
> call to pthread_cond_signal(). This effect is called "spurious
> wakeup". Note that the situation is self-correcting in that the number
> of threads that are so awakened is finite; for example, the next
> thread to call pthread_cond_wait() after the sequence of events above
> blocks.
> 
> 虽然虚假唤醒在pthread_cond_wait函数中可以解决，为了发生概率很低的情况而降低边缘条件（fringe
> condition）效率是不值得的，纠正这个问题会降低对所有基于它的所有更高级的同步操作的并发度。所以pthread_cond_wait的实现上没有去解决它。
> 
> While this problem could be resolved, the loss of efficiency for a
> fringe condition that occurs only rarely is unacceptable, especially
> given that one has to check the predicate associated with a condition
> variable anyway. Correcting this problem would unnecessarily reduce
> the degree of concurrency in this basic building block for all
> higher-level synchronization operations.

所以通常的标准解决办法是这样的：
将条件的判断从if 改为while,举例子说明：
下面为线程处理函数：

```
static void *thread_func(void *arg)
{
    while (1) {
    pthread_mutex_lock(&mtx);           //这个mutex主要是用来保证pthread_cond_wait的并发性
    while (msg_list.empty())   {     //pthread_cond_wait里的线程可能会被意外唤醒（虚假唤醒），如果这个时候，则不是我们想要的情况。这个时候，应该让线程继续进入pthread_cond_wait
        pthread_cond_wait(&cond, &mtx);
    }
		msg = msg_list.pop();
        pthread_mutex_unlock(&mtx);             //临界区数据操作完毕，释放互斥锁
		// handle msg
    }
    return 0;
}
```

如果不存在虚假唤醒的情况，那么下面代码：

```
    while (msg_list.empty())   {
        pthread_cond_wait(&cond, &mtx);
    }
```

可以为

```
    if (msg_list.empty())   {
        pthread_cond_wait(&cond, &mtx);
    }
```

但是存在虚假唤醒的时候，如果用if，而不用while，那么但被虚假唤醒的时候，不会再次while判断，而是继续下面执行msg = msg_list.pop();这其实是逻辑上有问题的。因为下面的代码已经假定了msg_list不是空的。写成如下也是可以的：

```
    if (msg_list.empty())   {
        pthread_cond_wait(&cond, &mtx);
    }
	if(msg_list.empty())
		continue;
```

pthread_cond_wait中的while()不仅仅在等待条件变量前检查条件变量，实际上在等待条件变量后也检查条件变量。

这样对condition进行多做一次判断，即可避免“虚假唤醒”.
这就是为什么在pthread_cond_wait()前要加一个while循环来判断条件是否为假的原因。
有意思的是这个问题也存在几乎所有地方，包括: linux 条件等待的描述, POSIX Threads的描述, window API(condition variable), java等等。

在linux的帮助中对条件变量的描述是：

> 添加while检查的做法被认为是增加了程序的健壮性，在IEEE Std 1003.1-2001中认为spurious wakeup是允许的。
> 
> An added benefit of allowing spurious wakeups is that applications are
> forced to code a predicate-testing-loop around the condition wait.
> This also makes the application tolerate superfluous condition
> broadcasts or signals on the same condition variable that may be coded
> in some other part of the application. The resulting applications are
> thus more robust. Therefore, IEEE Std 1003.1-2001 explicitly documents
> that spurious wakeups may occur.

在POSIX Threads中:
David R. Butenhof 认为多核系统中 条件竞争（race condition）导致了虚假唤醒的发生，并且认为完全消除虚假唤醒本质上会降低了条件变量的操作性能。

> “…, but on some multiprocessor systems, making condition wakeup
> completely predictable might substantially slow all condition variable
> operations. The race conditions that cause spurious wakeups should be
> considered rare”

在window的条件变量中:

MSDN帮助中描述为，spurious wakeups问题依然存在，条件需要重复check。

> Condition variables are subject to spurious wakeups (those not
> associated with an explicit wake) and stolen wakeups (another thread
> manages to run before the woken thread). Therefore, you should recheck
> a predicate (typically in a while loop) after a sleep operation
> returns.

在Java中，对等待的写法如下：

```
synchronized (obj) {  
    while (<condition does not hold>)  
        obj.wait(); 
     ... // Perform action appropriate to condition  

 }
```

Effective java 曾经提到Item 50: Never invoke wait outside a loop. 
显然，虚假唤醒是个问题,但它也是在JLS的第三版的JDK5的修订中才得以澄清。在JDK 5的Javadoc进行更新
 

> A thread can also wake up without being notified, interrupted, or
> timing out, a so-called spurious wakeup. While this will rarely occur
> in practice, applications must guard against it by testing for the
> condition that should have caused the thread to be awakened, and
> continuing to wait if the condition is not satisfied. In other words,
> waits should always occur in loops.   Apparently, the spurious wakeup
> is an issue (I doubt that it is a well known issue) that intermediate
> to expert developers know it can happen but it just has been clarified
> in JLS third edition which has been revised as part of JDK 5
> development. The javadoc of wait method in JDK 5 has also been updated

### 三、pthread_cond_timedwait有几个原子操作？
[帮助文档](http://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_cond_wait.html)中说了了一个:

> These functions atomically release mutex and cause the calling thread
> to block on the condition variable cond;

这个必须是原子操作，因为条件变量与信号量不同，如果不是原子操作，在释放锁后，等待信号前，如果正好发生了信号，这次信号就永久丢失了，没有信号量中的“状态”表示“曾经有一个信号发生过”。

帮助文档中还有一句“Upon successful return, the mutex has been locked and is owned by the calling thread. ”。返回的时候是不是原子操作影响不大，但是返回的时候必须是获取到锁的。即使发生了上面说的虚拟唤醒，那么他们也是一个个获取到锁之后返回的。因为如果是获取不到锁也返回，那么对条件变量的判断就是不安全的了。

### 参考：
[http://blog.csdn.net/fengge8ylf/article/details/6896380](http://blog.csdn.net/fengge8ylf/article/details/6896380)
