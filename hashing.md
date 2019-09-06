# Rehash

## 渐进式Rehash

要rehash的时候创建一个新的表，每次操作同时修改两个哈希表，并且rehash部分的key。考虑一直没有请求的情况，可以做一个定时触发源。如果没有rehash完空间又用完了，可以考虑忽略这次的rehash，或者直接STW（Stop The World）把rehash做完，又或者增加每次rehash的key数量加速。

## 前缀分层

根据hash的前缀，转发请求到不同的哈希表，每个哈希表独立生长。假设hash分布平均，那每个哈希表不会太大，rehash消耗也不大。

## 二次方增长

如果按二次方增长，rehash的时候要么就还在原来的bucket，要么就在两倍的位置，可以直接按bit确定，不需要计算取余。

## 多阶hash列表

分为多阶哈希表（每阶大小是质数），发现哈希冲突时往高阶溢出（高阶更小）。每一阶事实上也是独立，需要扩容的时候可以考虑增加新一阶哈希表，或者对某一阶的哈希表rehash。

## LSM

写的时候永远是顺序的。读的时候要做多版本检测，并且要做Compaction。

# 一致性hash

## Rendezvous hashing

选择bucket时，使用哈希函数h(key, bucket)计算每个bucket的值，选择最大的。缺点是bucket多时耗时长。

## Jump hash

目标
+ 平衡：对象均匀地分布在各个桶
+ 单调：桶的数量变化时，尽可能少的对象移动

假设`ch(key, num_buckets)`是`num_buckets`时的hash函数
+ 当`num_buckets`为1时，ch返回必定为0
+ 当`num_buckets`为2时，有一半ch应该返回0，有一半返回1
+ 当`num_buckets`从`n`变为`n+1`时，`ch(k, n+1)`应该有`n/(n+1)`保持与`ch(k, n)`相同，`1/(n+1)`进行了移动

因此有以下伪代码

```c
int ch(int key, int num_buckets) {
    random.seed(key) ;
    int b = 0; // This will track ch(key, j +1) .
    for (int j = 1; j < num_buckets; j++) {
        if (random.next() < 1.0 / (j+1)) b = j;
    }
    return b;
}
```

以上的时间复杂度是log(num_buckets)，`j`越大时跳转的概率越大，因此要实现Jump，降低复杂度。

假设上一次的跳变结果是`b`，`b`之后下一次跳变的结果是`j`，那就是在`b + 1`到`j - 1`之间都没有任何跳转

对于在区间`(b, j)`内的任意整数`i`，j是下一个结果的概率记为:

`P(j >= i) = P(ch(k, i) == ch(k, b + 1))`

展开，每次都不跳转

`P(j >= i) = P(ch(k, b + 1) == ch(k, b + 2)) * P(ch(k, b + 2) == ch(k, b + 3)) * P(ch(k, b + 3) == ch(k, b + 4)) * ... * P(ch(k, i - 1) == ch(k, i))`

单次不变的概率为

`P(ch(k, i) == ch(k, i + 1)) = i / (i + 1)`

因此

`P(j >= i) = (b + 1) / (b + 2) * (b + 2) / (b + 3) * ... * (i - 1) / i = (b + 1) / i`

假设当随机数`r < (b + 1) / i`时，有`j >= i`，则`i < (b + 1) / r`，`(b + 1) / r`是`i`的上界，所以`j = floor((b + 1) / r)`，实现了跳的效果，复杂度是O(log(num_buckets))


```c
int ch(int key, int num_buckets) {
    random.seed(key) ;
    int b = -1; //  bucket number before the previous jump
    int j = 0; // bucket number before the current jump
    while(j < num_buckets){
        b = j;
        double r = random.next(); // 0 < r < 1.0
        j = floor((b + 1) / r);
    }
    return b;
}
```

假如使用线性同余法生成随机数，则可以得出以下代码

`rand(n + 1) = (a * rand(n) + b) % N`

```c
int32_t JumpConsistentHash(uint64_t key, int32_t num_buckets) {
    int64_t b = -1, j = 0;
    while (j < num_buckets) {
        b = j;
        key = key * 2862933555777941757ULL + 1;
        j = (b + 1) * (double(1LL << 31) / double((key >> 33) + 1));
    }
    return b;
}
```

# Reference

+ [从HashMap的rehash到分布式KV数据库](https://zhuanlan.zhihu.com/p/50827247)
+ [nikhilgarg28/rendezvous](https://github.com/nikhilgarg28/rendezvous)
+ [【翻译/介绍】jump Consistent hash:零内存消耗，均匀，快速，简洁，来自Google的一致性哈希算法](https://blog.helong.info/blog/2015/03/13/jump_consistent_hash/)
