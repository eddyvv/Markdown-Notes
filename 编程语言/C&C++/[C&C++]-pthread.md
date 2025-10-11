# POSIX线程-pthread

## 线程管理

### pthread_create

pthread_create 函数应在进程中创建一个新线程，其属性由 *attr* 指定。如果 attr 为 NULL，则应使用默认属性。如果以后修改了 attr 指定的属性，则线程的属性不会受到影响。成功完成后，pthread_create 应将创建线程的 ID 存储在线程引用的位置。

```c
#include <pthread.h>
int pthread_create(pthread_t *restrict thread,
       const pthread_attr_t *restrict attr,
       void *(*start_routine)(void*), void *restrict arg);
```

创建线程将会执行 start_routine，并将 arg 作为其唯一参数。如果 start_routine 返回，则效果应类似于隐式调用 [pthread_exit()，](https://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_exit.html) 使用 start_routine 的返回值作为退出状态。请注意，最初调用 main() 的线程与此不同。当它从 main() 返回时，效果应该是隐式调用 [exit()](https://pubs.opengroup.org/onlinepubs/9699919799/functions/exit.html) 使用 main() 的返回值作为退出状态。

参数：

* pthread_t *：向调用者传递子线程线程号。
* const pthread_attr_t *：负责控制线程的各种属性，这也是线程在创建的时候，最为复杂的一个参数，一般可以为NULL。

* void (start_routine) (void *)：负责指定子线程需要调用的函数，这个参数需要的是一个函数指针。
* void *：负责指定，子线程所运行的函数的参数值。

返回值：

* 0：函数执行成功；
* EAGAIN：系统缺乏创建另一个线程所需的资源，否则将超出系统对进程 {PTHREAD_THREADS_MAX} 中线程总数施加的限制。
* EPERM：调用方没有适当的权限来设置所需的调度参数或调度策略。

如果成功，pthread_create() 函数应返回零;否则，应返回错误编号以指示错误。

### pthread_exit

pthread_exit用于线程结束，多线程编程中，线程结束执行的方式有 3 种，分别是：

1. 线程将指定函数体中的代码执行完后自行结束；
2. 线程执行过程中，被同一进程中的其它线程（包括主线程）强制终止；
3. 线程执行过程中，遇到 pthread_exit() 函数结束执行。

```c
#include <pthread.h>
void pthread_exit(void *value_ptr);
```

参数：

* value_ptr：retval 是 void* 类型的指针，可以指向任何类型的数据，它指向的数据将作为线程退出时的返回值。如果线程不需要返回任何数据，将 retval 参数置为 NULL 即可。

<font color=red>注意</font>：

1. 默认属性的线程执行结束后并不会立即释放占用的资源，直到整个进程执行结束，所有线程的资源以及整个进程占用的资源才会被操作系统回收。

2. pthread_exit的作用是只退出当前子线程，记住是只。即使你放在主线程，它也会只退出主线程，其它线程有运行的仍会继续运行。

### pthread_join

pthread_join()函数，以阻塞的方式等待thread指定的线程结束。当函数返回时，被等待线程的资源被收回。如果线程已经结束，那么该函数会立即返回。并且thread指定的线程必须是可结合（joinable[^1]）的。

```c
#include <pthread.h>
int pthread_join(pthread_t thread, void **retval);
```

参数：

* pthread_t：线程标识符，即线程ID，标识唯一线程
* void **：用户定义的指针，用来存储被等待线程的返回值

返回值：

* 0：成功；
* EDEADLK：检测到死锁；

当线程A调用线程B并在A中调用 pthread_join() 时，A线程会处于阻塞状态，直到B线程结束后，A线程才会继续执行下去。当 pthread_join() 函数返回后，被调用线程才算真正意义上的结束，它的内存空间也会被释放（如果被调用线程是非分离的）。这里有三点需要注意：

1. 被释放的内存空间仅仅是系统空间，你必须手动清除程序分配的空间，比如 malloc() 分配的空间。

2. 一个线程只能被一个线程所连接。

3. 被连接的线程必须是非分离的，否则连接会出错。

所以可以看出pthread_join()有两种作用：

1. 用于等待其他线程结束：当调用 pthread_join() 时，当前线程会处于阻塞状态，直到被调用的线程结束后，当前线程才会重新开始执行。
2. 对线程的资源进行回收：如果一个线程是非分离的（joinable[^1]，默认情况下创建的线程都是非分离）并且没有对该线程使用 pthread_join() 的话，该线程结束后并不会释放其内存空间，这会导致该线程变成了“僵尸线程”。

### pthread_detach

pthread_detach用于将此线程设置为分离状态，创建一个线程的默认状态是可连接的，如果一个线程运行结束但没有被连接，则它的状态类似于进程中的僵尸进程（Zombie Process），即还有一部分资源没有被回收（退出状态码），因此创建线程者应该调用pthread_join来等待线程运行结束，并可以获取线程的退出代码，回收其资源（类似于wait,waitpid）

```c
#include <pthread.h>
int pthread_detach(pthread_t tid)；
```

参数：

* pthread_t：线程标识符，即线程ID，标识唯一线程。

返回值：

* 0：成功

pthread_detach 的调用是非阻塞的，可立即返回。

### pthread_equal

此函数用于比较线程 ID t1 和 t2。

```c
#include <pthread.h>
int pthread_equal(pthread_t t1, pthread_t t2);
```

如果 t1 和 t2 相等，则 pthread_equal() 函数应返回非零值；否则，应返回零。

### pthread_self

获取当前线程ID。

```c
#include <pthread.h>
pthread_t pthread_self(void);
```

### pthread_setschedparam

在 POSIX 线程库（pthread）中，pthread_setschedparam 函数用于设置线程的调度策略和优先级。POSIX的调度策略见[POSIX 标准调度策略](#SCHED)

```c
#include <pthread.h>
int pthread_setschedparam(pthread_t thread, int policy,
       const struct sched_param *param); 
```

参数：

* thread：指定目标线程ID。
* policy：
* param：

返回值：

* 0：成功；
* ESRCH：线程 thread 不存在。



### pthread_getschedparam

pthread_getschedparam函数用于获取现有线程的调度策略和优先级。

```c
#include <pthread.h>
int pthread_getschedparam(pthread_t thread, int *restrict policy,
       struct sched_param *restrict param);

```

参数：

* thread：指定目标线程ID。
* policy：指向将存储 schedpolicy 属性值的位置。
* param： 指向将存储 schedparam 属性值的位置。

返回值：

* 0：成功；
* ESRCH：线程 thread 不存在。

### pthread_setschedprio

设置线程优先级

```c
#include <pthread.h>
int pthread_setschedprio(pthread_t thread, int prio);
```

### pthread_cancel

pthread_cancel用于向线程提出取消请求，pthread_cancel调用并不等待线程终止，它只提出请求。线程在取消请求(pthread_cancel)发出后会继续运行，直到到达某个[取消点(CancellationPoint)](#Cancelation-point)。取消点是线程检查是否被取消并按照请求进行动作的一个位置。

```c
#include <pthread.h>
int pthread_cancel(pthread_t thread);
```

如果线程处于无限循环中，且循环体内没有执行至取消点的必然路径，则线程无法由外部其他线程的取消请求而终止。因此在这样的循环体的必经路径上应该加入pthread_testcancel()调用。

### pthread_testcancel

pthread_testcancel()的目的很简单，就是产生一个取消点。线程如果已有处于挂起状态的取消请求，那么只要调用该函数，线程就会随之终止。

```c
#include <pthread.h>
void pthread_testcancel(void);
```

当线程执行的代码未包含取消点时，可以周期性地调用 pthread_testcancel()，以确保对其他线程向其发送的取消请求做出及时响应。

### pthread_cleanup_push

### pthread_cleanup_pop

函数 pthread_cleanup_push()和 pthread_cleanup_pop()分别负责向调用线程的清理函数栈添加和移除清理函数。

```c
#include <pthread.h>
void pthread_cleanup_pop(int execute);
void pthread_cleanup_push(void (*routine)(void*), void *arg);
```

一旦有处于挂起状态的取消请求，线程在执行到取消点时如果只是草草收场，这会将共享变量以及 Pthreads 对象（例如互斥量）置于一种不一致状态，可能导致进程中其他线程产生错误结果、死锁，甚至造成程序崩溃。为规避这一问题，线程可以设置一个或多个清理函数，当线程遭取消时会自动运行这些函数，在线程终止之前可执行诸如修改全局变量，解锁互斥量等动作。

每个线程都可以拥有一个清理函数栈。当线程遭取消时，会沿该栈自顶向下依次执行清
理函数，首先会执行最近设置的函数，接着是次新的函数，以此类推。当执行完所有清理函数后，线程终止。

注意：

1. 如果pthread_cleanup_pop参数 execute 非零，那么无论如何都会执行清理函数。

2. 为便于编码，若线程因调用 pthread_exit()而终止，则也会自动执行尚未从清理函数栈中弹出（pop）的清理函数。线程正常返回（return）时不会执行清理函数。

### pthread_once

确保某个初始化操作在多线程环境中只执行一次，即使多个线程同时尝试执行该操作。其核心设计目标是提供线程安全的、高效的一次性初始化机制。

```c
#include <pthread.h>
int pthread_once(pthread_once_t *once_control,
       void (*init_routine)(void));
pthread_once_t once_control = PTHREAD_ONCE_INIT;
```

利用参数 once_control 的状态，函数 pthread_once() 可以确保无论有多少线程对pthread_once()调用了多少次，也只会执行一次由 init 指向的调用者定义函数。

参数 once_control 必须是一指针，指向初始化为 PTHREAD_ONCE_INIT 的静态变量。 







## 常见问题

### exit、return、pthread_exit，pthread_cancel各自退出效果和pthread_join，pthread_detach的作用

1. return：返回到调用者那里去。注意，在主线程退出时效果与exit,exit一样。即return有可能会退出进程(放在主线程时)，有可能会退出线程，也有可能什么也不退出。
2. pthread_exit()：只退出当前子线程。注意：在主线程退出时，其它线程不会结束。同样可以执行。所以这个只字非常重要。并且，与return一样，pthread_exit退出的线程也需要调用pthread_join去回收子线程的资源(8k左右)，否则服务器长时间运行会浪费资源导致无法再创建新线程。
3. exit，_exit： 将进程退出，无论哪个子线程调用整个程序都将结束。
4. pthread_join()：阻塞等待回收子线程。
5. pthread_cancel()：可以杀死子线程，但必须需要一个契机，这个契机就是系统调用。一般方法是调用pthread_testcancel提供契机处理。并且join再回收该被杀死的子线程的返回值为pthread.h中的宏#define PTHREAD_CANCELED ((void *) -1)。
6. pthread_detach：
   1. 注意，所有线程的错误号返回都只能使用strerror这个函数判断，不能使用perror，因为perror是调用进程的全局错误号，不适合单独线程的错误分析，所以只能使用strerror。
   2. 线程可以被置为detach状态，这样的线程一旦终止就立刻回收它占用的所有资源，而不保留终止状态。不能对一个已经处于detach状态的线程调用pthread_join，这样的调用将返回EINVAL错误。也就是说，如果已经对一个线程调用了pthread_detach就不能再调用pthread_join了。

### 线程的分离状态

线程属性值中有一个分离状态，即在任何一个时间点上，线程是可结合的（joinable）， 或者是分离的（detached）。一个可结合的线程能够被其他线程收回其资源和杀死；在被其他线程回收之前， 它的存储器资源（如栈）是不释放的。相反，一个分离的线程是不能被其他线程回收或杀死的， 它的存储器资源在它终止时由系统自动释放。

总而言之：线程的分离状态决定一个线程以什么样的方式来终结自己。

进程中的线程可以调用pthread_join()函数来等待某个线程的终止，获取该线程的终止状态，并收回线程所占的资源， 如果对线程的返回状态不感兴趣，可以将pthread_join函数的rval_ptr参数设置为NULL。

如果一个线程是可结合的，意味着这条线程在退出时不会自动释放自身资源，而会成为僵尸线程， 同时意味着该线程的退出值可以被其他线程获取。因此，如果不需要某个线程的退出值的话， 那么最好将线程设置为分离状态，以保证该线程不会成为僵尸线程。

### <span id="SCHED">POSIX 标准调度策略</span>

| 策略宏      | 数值 | 描述                      | 优先级范围            |
| ----------- | ---- | ------------------------- | --------------------- |
| SCHED_FIFO  | 1    | 先进先出实时调度策略      | 1（最低）~ 99（最高） |
| SCHED_RR    | 2    | 时间片轮转实时调度策略    | 1（最低）~ 99（最高） |
| SCHED_OTHER | 0    | 默认非实时调度策略（CFS） | 0（唯一有效值）       |

#### SCHED_FIFO（先进先出）

它是一种实时的先进先出调用策略，且只能在超级用户（root）下运行。这种调用策略仅仅被使用于优先级大于0的线程。它意味着，使用SCHED_FIFO的可运行线程将一直抢占使用SCHED_OTHER的运行线程J。此外SCHED_FIFO是一个非分时的简单调度策略，当一个线程变成可运行状态，它将被追加到对应优先级队列的尾部((POSIX 1003.1)。当所有高优先级的线程终止或者阻塞时，它将被运行。对于相同优先级别的线程，按照简单的先进先运行的规则运行。我们考虑一种很坏的情况，如果有若干相同优先级的线程等待执行，然而最早执行的线程无终止或者阻塞动作，那么其他线程是无法执行的，除非当前线程调用如pthread_yield之类的函数，所以在使用SCHED_FIFO的时候要小心处理相同级别线程的动作。

**特性**：

* 线程一旦获得 CPU，将一直运行直到主动放弃（如调用 sched_yield()）或被更高优先级线程抢占。
* 同一优先级的线程按 FIFO 队列顺序执行。
* 高优先级线程可能导致低优先级线程饥饿（长时间无法执行）。

#### SCHED_RR（时间片轮转）

鉴于SCHED_FIFO调度策略的一些缺点，SCHED_RR对SCHED_FIFO做出了一些增强功能。从实质上看，它还是SCHED_FIFO调用策略。它使用最大运行时间来限制当前进程的运行，当运行时间大于等于最大运行时间的时候，当前线程将被切换并放置于相同优先级队列的最后。这样做的好处是其他具有相同级别的线程能在“自私“线程下执行。SCHED_RR需要特权权限。

**特性**：

* 与 SCHED_FIFO 类似，但每个线程执行一个时间片后，会被放入队列尾部，允许同优先级线程轮流执行。
* 时间片长度由操作系统决定（如 Linux 中可通过 sched_rr_get_interval() 查询）。
* 仍会抢占低优先级线程，但同一优先级内不会导致饥饿。

#### SCHED_OTHER（默认策略）

它是默认的线程分时调度策略，所有的线程的优先级别都是0，线程的调度是通过分时来完成的。简单地说，如果系统使用这种调度策略，程序将无法设置线程的优先级。请注意，这种调度策略也是抢占式的，当高优先级的线程准备运行的时候，当前线程将被抢占并进入等待队列。这种调度策略仅仅决定线程在可运行线程队列中的具有相同优先级的线程的运行次序。

**特性**：

* 非实时策略，基于系统负载动态调整线程优先级。
* 在 Linux 中实现为完全公平调度器（CFS），使用虚拟运行时间（vruntime）确保公平性。
* 线程优先级只能通过 nice 值（-20~19）间接调整，且 nice 值仅影响调度权重，不直接映射到优先级数值。

### <span id="Cancelation-point">什么是取消点（Cancelation-point）</span>

根据POSIX标准，pthread_join()、pthread_testcancel()、pthread_cond_wait()、pthread_cond_timedwait()、sem_wait()、sigwait()等函数以及read()、write()等会引起阻塞的系统调用都是Cancelation-point，而其他pthread函数都不会引起Cancelation动作。但是pthread_cancel的手册页声称，由于LinuxThread库与C库结合得不好，因而目前C库函数都不是Cancelation-point；但CANCEL信号会使线程从阻塞的系统调用中退出，并置EINTR错误码，因此可以在需要作为Cancelation-point的系统调用前后调用pthread_testcancel()，从而达到POSIX标准所要求的目标。





[^1]:指的是线程可以被其他线程等待其结束的状态，即线程可以被回收或结合。

# 参考

[多线程pthread_create的参数](https://www.jianshu.com/p/d16d0e1efc74)

[C语言多线程教程（pthread）（线程创建pthread_t，指定线程run方法pthread_create()函数，加mutex锁，解锁，伪共享 false sharing【假共享】）](https://blog.csdn.net/Dontla/article/details/120748957)

[pthread_exit()函数：终止线程](https://c.biancheng.net/view/8608.html)

[07LinuxC线程学习之pthread_exit函数和总结exit、return、pthread_exit，pthread_cancel各自退出效果和join，detach的作用](https://blog.csdn.net/weixin_44517656/article/details/112253724)

[pthread_join函数介绍和使用实例](https://blog.csdn.net/qq_37858386/article/details/78185064)

[pthread_join(3) — Linux manual page](https://man7.org/linux/man-pages/man3/pthread_join.3.html)

[线程的分离状态](https://doc.embedfire.com/linux/rk356x/linux_base/zh/latest/system_programing/thread/thread.html#id7)

[用pthread_setschedparam设置调度策略](https://blog.csdn.net/weixin_41554427/article/details/148931529)

[POSIX线程优先级设置](https://blog.csdn.net/Liangren_/article/details/115175951)

[posix多线程--线程取消](https://www.cnblogs.com/shijingjing07/p/5402881.html)

[线程取消 (pthread_cancel)](https://www.cnblogs.com/renxinyuan/p/3877556.html)

[pthread_cancel用法及常见问题](https://blog.csdn.net/u012377333/article/details/40513153)

[Posix线程编程指南（1）](https://www.kerneltravel.net/blog/2020/posix_thread_guide1/)

[《Linux-UNIX系统编程手册》](https://book.douban.com/subject/25809330/)

[《POSIX多线程程序设计》](https://book.douban.com/subject/1236825/)

[《UNIX环境高级编程（第3版）》](https://book.douban.com/subject/25900403/)

[posix多线程有感--线程高级编程（pthread_key_t）](https://blog.csdn.net/ctthuangcheng/article/details/8894972)