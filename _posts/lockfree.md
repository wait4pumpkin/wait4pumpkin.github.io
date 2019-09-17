# Introduction

## Wait-freedom

Each thread moves forward regardless of external factors.

在同步算法里面的最强保证，无论其他线程是什么执行状态，都保证每一个线程都在有限步内完成操作。可能包含的原语，如`atomic_fetch_add`、`atomic_exchange`，但一般不包含`atomic_compare_exchange`（因为通常会带有循环，重复执行直到成功）。

## Lock-freedom

A system as a whole moves forward regardless of anything.

比wait-freedowm更弱的保证，保证整个系统可以move forward，但不保证单个线程（有线程可能会饿死）。

## Obstruction-freedom

A thread makes forward progress only if it does not encounter contention from other threads.

比lock-freedowm更弱的保证，只保证线程间在没有竞争的情况下单个线程会move forward，活锁就是其中一种可能情况。

## Termination-safety

以上三种均保证，在单个线程终止时，整个系统依然可以move forward。

## Blocking Algorithms

最弱的保证，整个系统可能不会move forward，例如死锁。

## Practical Implications

+ lock free算法make forward progress的必定是正在运行的线程，而mutex-based则可能是non-running的线程（例如死锁在休眠），所以实际上整个系统没有move progress。

+ lock-based不能用的场景，如信号处理，可以用lock free

+ 某些系统，线程异常退出时，mutex-based算法可能没有释放锁

## Performance/scalability

lock free其实只是forward progress的保证，跟performance是正交的


# First Things First

影响性能的关键并不是多少条（atomic）指令，而是有多少共享的写入操作，也就是每条指令所需要的cache line transfer，只才是影响scalability的最重要因素。

+ 写共享必定是线程越多越慢

+ 如果没有写共享，系统是linearly scales，即使是用atomic（atomic会比普通变量稍慢，因为指令加了lock，但依然是scalable）。

+ load必定是scalable，即使是同时读，atomic的load在x86下和普通变量是相同的（mov）。


# Producer-Consumer Queues

