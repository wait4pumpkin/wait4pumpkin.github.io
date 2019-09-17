## Namespaces

Linux提供的，用于分离进程树、网络接口、挂载点以及进程间通信等资源的方法。Docker通过Linux 的Namespaces对不同的容器实现了隔离。

Linux的命名空间机制提供了以下七种不同的命名空间，通过这七个选项我们能在创建新的进程时设置新进程应该在哪些资源上与宿主机器进行隔离。

+ CLONE_NEWCGROUP
+ CLONE_NEWIPC
+ CLONE_NEWNET
+ CLONE_NEWNS
+ CLONE_NEWPID
+ CLONE_NEWUSER
+ CLONE_NEWUTS

### 进程

### 网络

Host
Container
None
Bridge

## Control 

## UDP监听在0.0.0.0问题



## Reference

+ [Docker 核心技术与实现原理](https://draveness.me/docker)
