---
layout: post
title: 多线程中condition wait
category: 杂七杂八
tags: 多线程 condition wait
---
一般在多线程编程中都会用到condition_wait，在linux下也就是pthread_cond_wait这个系统调用，大多高性能的多线程程序也离不开这基本的东西。在看线程调度代码的时候突然想起这个方法，一直用ace，很操作系统的东西都忘得差不多了。下面一段是一年前对这个系统调用的理解:

## 条件变量与互斥量一起使用从可以允许线程以无竞争的方式等待特定的条件发生。为什么必须一起使用呢？
1）假如当某个资源满足了一定的条件时它要发送信号告诉需要它的线程，假如现在资源为空，条件不满足，那么线程就要等待，那么在线程条件检查时刻t1，到线程放到表上的时刻t2，在t1到t2这段时间内，可能资源来了，然而CPU调度到另外的工作进程上了，这样条件就不能改变，这样线程就错过了条件变化，虽然等线程放到队列上后，资源变化可以通知线程，但是会有时间延迟。

2）传递给pthread_cond_wait(pthread_cont_t *restrict cond, pthread_mutex_t *restrict 
mutex)中的mutex必须是已经锁住的，如果这个互斥锁不是锁住的，那么假如满足条件，线程一对资源访问的同时线程二也可能访问，就造成了资源的不一致。所以这个过程还是挺复杂的，当需要等待时，把一个资源的互斥量和等待条件一同传给这个wait函数，这个wait函数把线程放到这个条件的等待队列上，然后对互斥量解锁（注意这两步是个原子操作，具体原因如1）。解锁一来可以让条件发生（如果死守着资源，那么资源也就不会变了），二来可以让其他线程同时访问资源，同样可以检查条件。”

以前只理解了在放入等待队列前的一些操作，pthread_cond_wait这个方法在做线程池调用的时候非常常见，用这个接口可以减少多线程的时候的锁竞争，在没有资源的时候进入sleep状态，这个时候是不耗费cpu的，当有资源的时候发送一个或者唤醒所有的线程，否则在线程中只能定时去检查资源是否存在，这样会定时的产生锁竞争，最重要的是浪费了cpu不必要的调度。发送一个signal可以有资源的时候不至于使得很多线程去竞争这个资源，一般而言一个signal只会唤醒一个队列中的线程，至于唤醒的策略“根据各等待线程优先级的高低确定哪个线程接收到信号开始继续执行。如果各线程优先级相同，则根据等待时间的长短来确定哪个线程获得信号”参考：[http://blog.csdn.net/hudashi/article/details/7709421](http://blog.csdn.net/hudashi/article/details/7709421)

在简单总结下这个方法的操作过程

mutex加锁

pthread_cond_wait把线程放入等待队列

pthread_cond_wait自动给mutex解锁

当有线程把资源放入时候会signal或者broadcast这个时候如果是broadcast，那么可能会唤起很多线程对mutex进行锁竞争，但是必须注意一点，竞争到锁后，对拿到的资源需要进行非空检查，因为别的线程可能拿到锁，然后处理完资源把资源清空了资源为空，再次进入pthread_cond_wait中否则处理资源，处理完后然后手动mutex解锁，上面过程中有一次mutex的手动加锁和解锁，可能有多次的pthread_cond_wait自动加锁解锁，但是都是匹配的参考：[http://blog.csdn.net/hairetz/article/details/4535920](http://blog.csdn.net/hairetz/article/details/4535920)
一段简单线程池代码：
```
#include <pthread.h>
#include <unistd.h>

static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

struct node {
int n_number;
struct node *n_next;
} *head = NULL;

/*[thread_func]*/
static void cleanup_handler(void *arg)
{
    printf("Cleanup handler of second thread./n");
    free(arg);
    (void)pthread_mutex_unlock(&mtx);
}
static void *thread_func(void *arg)
{
    struct node *p = NULL;

    pthread_cleanup_push(cleanup_handler, p);
    while (1) {
    pthread_mutex_lock(&mtx);           //这个mutex主要是用来保证pthread_cond_wait的并发性
    while (head == NULL)   {               //这个while要特别说明一下，单个pthread_cond_wait功能很完善，为何这里要有一个while (head == NULL)呢？因为pthread_cond_wait里的线程可能会被意外唤醒，如果这个时候head != NULL，则不是我们想要的情况。这个时候，应该让线程继续进入pthread_cond_wait
        pthread_cond_wait(&cond, &mtx);         // pthread_cond_wait会先解除之前的pthread_mutex_lock锁定的mtx，然后阻塞在等待对列里休眠，直到再次被唤醒（大多数情况下是等待的条件成立而被唤醒，唤醒后，该进程会先锁定先pthread_mutex_lock(&mtx);，再读取资源
                                                //用这个流程是比较清楚的/*block-->unlock-->wait() return-->lock*/
    }
        p = head;
        head = head->n_next;
        printf("Got %d from front of queue/n", p->n_number);
        free(p);
        pthread_mutex_unlock(&mtx);             //临界区数据操作完毕，释放互斥锁
    }
    pthread_cleanup_pop(0);
    return 0;
}

int main(void)
{
    pthread_t tid;
    int i;
    struct node *p;
    pthread_create(&tid, NULL, thread_func, NULL);   //子线程会一直等待资源，类似生产者和消费者，但是这里的消费者可以是多个消费者，而不仅仅支持普通的单个消费者，这个模型虽然简单，但是很强大
    /*[tx6-main]*/
    for (i = 0; i < 10; i++) {
        p = malloc(sizeof(struct node));
        p->n_number = i;
        pthread_mutex_lock(&mtx);             //需要操作head这个临界资源，先加锁，
        p->n_next = head;
        head = p;
        pthread_cond_signal(&cond);
        pthread_mutex_unlock(&mtx);           //解锁
        sleep(1);
    }
    printf("thread 1 wanna end the line.So cancel thread 2./n");
    pthread_cancel(tid);             //关于pthread_cancel，有一点额外的说明，它是从外部终止子线程，子线程会在最近的取消点，退出线程，而在我们的代码里，最近的取消点肯定就是pthread_cond_wait()了。关于取消点的信息，有兴趣可以google,这里不多说了
    pthread_join(tid, NULL);
    printf("All done -- exiting/n");
    return 0;
}
```
 signal/wait只是通知机制，本身不代表资源情况，当发signal时没有人在wait，那么这个signal就会丢失，后来再wait的就等不到signal了，而在多线程中，发signal和wait的顺序往往不能保证的，所以上述代码是需要待改进的。