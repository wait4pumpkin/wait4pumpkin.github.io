# Chandy-Lamport Snapshot

## 快照

Snapshot除了记录每个节点的本地状态之外，还记录了节点之间的信道状态。

## 用途

+ Checkpoint: 灾难恢复
+ 垃圾回收
+ 死锁检测
+ 调试

## 算法

任意一个节点开始执行Snapshot
+ 记录本地状态
+ 发送Marker消息到所有输出信道

当从某个信道接收到Marker消息时：
+ 如果是第一次接收到，则记录本地状态，并广播Marker消息到所有输出信道；开始记录所有输入信道的消息
+ 停止记录该信道的消息

停止条件：所有节点接收到所有信道的Marker消息

## Reference

+ [Distributed Snapshots](https://www.cs.princeton.edu/courses/archive/fall18/cos418/docs/p4-snapshots.pdf)
