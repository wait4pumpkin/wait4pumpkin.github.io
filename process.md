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

## 调度

## Reference

+ [linux进程状态查询——ps](https://blog.51cto.com/desert/335979)
+ [理解Linux进程](https://www.kancloud.cn/kancloud/understanding-linux-processes/52200)
+ [深入 Linux 的进程优先级](https://linux.cn/article-7325-1.html)
