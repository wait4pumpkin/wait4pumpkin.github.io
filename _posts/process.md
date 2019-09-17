# 进程

## 僵尸进程

子进程退出（发SIGCHILD，父进程默认忽略），父进程没有wait/waitpid。父进程也退出后，init进程回收。

## 孤儿进程

父进程退出，子进程被init进程接管

## fork两次

fork两次是为了使worker进程是孤儿进程，那父进程就不需要wait，也不会出现僵尸进程。具体流程，A进程fork生成B进程，B进程fork生成worker进程，B进程退出，worker进程成为孤儿进程。A进程持续运行，不需要管worker进程什么时候退出（不需要wait），但要负责进程B的wait。

## 进程状态

### 三态

+ 运行：时间片用完，转为就绪；等待资源，转为等待。
+ 等待：资源满足，转为就绪
+ 就绪：CPU空闲转为运行

### 五态

加上新建和终止两个状态

### 七态

考虑进程可能挂起，从主存换出到二级存储器（不能直接执行，要先换入）

+ 挂起就绪
+ 挂起等待

## PS进程状态

+ D: 不可中断睡眠，一般由IO引起，同步IO在做读或写操作时，cpu不能做其它事情，只能等待，这时进程处于这种状态
+ R(unable): 进程处于运行或就绪状态
+ I(dle)：空闲状态
+ S(leeping): 可中断睡眠
+ T(raced): 跟踪状态，已停止的，进程收到SIGSTOP、SIGSTP、SIGTIN、SIGTOU信号后停止运行
+ Z(ombie): 僵尸进程
+ <: 高优先级（not nice to other users）
+ N: 低优先级（nice to other users）
+ l: 多线程
+ +: 在前台进程组
+ s: Session Leader

## nice

取值范围是-20至19，一共40个级别。值越小，表示进程优先级越高。

nice值可以影响优先级（ps里的NI），但nice并不是优先级priority（PRI）。

nice是静态优先级（设置后不改变），priority是动态优先级（O(1)调度器下会变化）。

## 优先级

内核进程优先级范围0 - 139（MAX_PRIO：140），实时进程0 - 99，修改nice值得出的优先级范围是100 - 139。

## 调度

进程分为两类（根据优先级区分）
+ 实时进程
+ 非实时进程

`chrt`可以用来设置一个进程的调度策略

```shell
pumpkin@Departures:$ chrt --help
Show or change the real-time scheduling attributes of a process.

Set policy:
 chrt [options] <priority> <command> [<arg>...]
 chrt [options] --pid <priority> <pid>

Get policy:
 chrt [options] -p <pid>

Policy options:
 -b, --batch          set policy to SCHED_BATCH
 -d, --deadline       set policy to SCHED_DEADLINE
 -f, --fifo           set policy to SCHED_FIFO
 -i, --idle           set policy to SCHED_IDLE
 -o, --other          set policy to SCHED_OTHER
 -r, --rr             set policy to SCHED_RR (default)
```

实时进程可以用的调度策略
+ SCHED_FIFO
+ SCHED_RR
+ SCHED_DEADLINE

对于实时进程，优先级高的必定先于优先级低的执行。策略只有在优先级相同时才生效，FIFO是先进先出执行，RR是按时间片轮转（100ms），DEADLINE（EDF，Earliest DeadlineFirst）是要求在限时内必须执行限定配额。

非实时进程
+ SCHED_BATCH
+ SCHED_OTHER
+ SCHED_IDLE

### O(1)调度

2.6开始引入的，2.6.23之后内核将调度算法替换成了CFS。优先级的体现，就是优先级高的时间片更大。

O(1)算法还会把进程区分为CPU消耗型（消耗完时间片才让出）和IO消耗型（一般主动让出CPU）。CPU会根据这个情况，动态调整priority，范围是±5，来确保IO消耗型的进程响应更快。

经典时间片分配，调度时只关心R状态的进程，使用两个队列进行管理，时间片未必耗尽的进程队列，和时间片耗尽的进程队列。根据需要调度的进程数和CPU核心数，可以估计出系统的繁忙程度。

```shell
# 绝对值，分别是1min、5min、15min的平均值
pumpkin@Departures:~$ uptime
 11:14:27 up 9 days,  1:50,  1 user,  load average: 2.76, 2.76, 2.86
```

父进程fork的时候会分一半时间片给子进程，为了避免不停fork占用过多时间片。CPU tick触发时，会检查当前执行进程时间片是否耗尽或者是否有更高优先级的进程在等待，若满足则暂停执行当前进程，换最高优先级的进程执行。

### CFS完全公平调度（Completely Fair Scheduler）

所谓公平，就是在一段比较小的时间内每个进程都获得相同的执行时间。这个比较小的时间就是任意一个R状态进程被调度的最大延时时间，也可以叫做调度周期（sched_latency_n）。

调度器记录了每个进程占用的CPU时间（vruntime），并以vruntime为key构造红黑树。每次被调度的进程，就是树的最左节点（vruntime最少）。sched_min_granularity_ns用来控制每个进程最少占用的CPU时间，避免频繁切换。

调度的时机
+ 当前进程的状态转换（进程终止退出或者进程休眠）；
+ 当前进程主动放弃CPU（状态变为sleep、调用sched_yield()）；
+ 当前进程的vruntime时间大于每个进程的理想占用时间（简单计算，sched_latency_ns／进程数，通过时钟中断）；
+ 当进程从中断、异常或系统调用返回。

对于优先级的支持，对于同样流逝的真实时间，高优先级的进程vruntime增长得慢。不同nice值的cpu消耗时间比例在内核中是按照，每差一级cpu占用时间差10%左右，这个原则来设定的。如果有两个nice值为0的进程同时占用cpu，那么它们应该每人占50%的cpu；如果将其中一个进程的nice值调整为1的话，那么此时应保证优先级高的进程比低的多占用10%的cpu，就是nice值为0的占55%，nice值为1的占45%。就是说，相邻的两个nice值之间的cpu占用时间比例的差别应该大约为1.25。

新进程的vruntime按当前队列中的min_vruntime设置（若设置为0会抢占太多CPU）。如果设置了sched_child_runs_first，则会保证子进程的vruntime比父进程小，使子进程先于父进程调度。

IO消耗型的程序，因为多数时间在休眠，需要进行补偿，否则vruntime比较小被持续调度。补偿方式为，如果进程是从sleep状态被唤醒的，而且GENTLE_FAIR_SLEEPERS属性的值为true，则vruntime被设置为sched_latency_ns的一半和当前进程的vruntime值中比较大的那个。

```shell
pumpkin@Departures:~$ cat /proc/sys/kernel/sched_latency_ns
24000000
```

SCHED_BATCH与SCHED_OTHER的区别是，显式标记为CPU消耗型的程序，调用sched_yield主动让出CPU不会记录vruntime，从而可以继续被优先调度。

SCHED_IDLE则是优先级很低，比nice值19还要低，只有CPU空闲才进行调度。

针对多核，在每个核心维护一个队列（避免锁的消耗），然后定期在核心之间进行均衡（idle_balance，/proc/sched_debug），迁移到另一核心是vruntime要进行调整（保持与min_vruntime的相对值）。

## Reference

+ [linux进程状态查询——ps](https://blog.51cto.com/desert/335979)
+ [理解Linux进程](https://www.kancloud.cn/kancloud/understanding-linux-processes/52200)
+ [深入 Linux 的进程优先级](https://linux.cn/article-7325-1.html)
