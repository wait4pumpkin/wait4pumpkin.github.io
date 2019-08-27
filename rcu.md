### RCU与seqlock区别

两者都是适用于读多写少的场合，在没有写时，读都是非常快，并且写操作不会被读操作阻塞。区别在于seqlock在读写冲突时，读是阻塞的（虽然不是休眠等待，但是在不断重试，循环内做的操作都是会被丢弃的），而RCU依然可以进行读操作（可能读到非最新版本）。

seqlock简单实现，写写冲突加锁解决。每次开始写++seq，写完再++。则奇数seq就是正在写，读要重试。同时要求seq是原子量（按顺序一致性Sequential consistency，事实上可以放松）。

```c
static inline void write_seqlock(seqlock_t *sl)
{
	spin_lock(&sl->lock);
	write_seqcount_begin(&sl->seqcount);
}

static inline void write_sequnlock(seqlock_t *sl)
{
	write_seqcount_end(&sl->seqcount);
	spin_unlock(&sl->lock);
}

static inline void raw_write_seqcount_begin(seqcount_t *s)
{
	s->sequence++;
	smp_wmb();
}

static inline void raw_write_seqcount_end(seqcount_t *s)
{
	smp_wmb();
	s->sequence++;
}

static inline unsigned read_seqbegin(const seqlock_t *sl)
{
	return read_seqcount_begin(&sl->seqcount);
}

static inline unsigned read_seqretry(const seqlock_t *sl, unsigned start)
{
	return read_seqcount_retry(&sl->seqcount, start);
}

static inline unsigned raw_read_seqcount_begin(const seqcount_t *s)
{
	unsigned ret = __read_seqcount_begin(s);
	smp_rmb();
	return ret;
}

static inline unsigned __read_seqcount_begin(const seqcount_t *s)
{
	unsigned ret;

repeat:
	ret = READ_ONCE(s->sequence);
	if (unlikely(ret & 1)) {
		cpu_relax();
		goto repeat;
	}
	return ret;
}

static inline int __read_seqcount_retry(const seqcount_t *s, unsigned start)
{
	return unlikely(s->sequence != start);
}

// 使用例子
u64 get_jiffies_64(void)
{
	unsigned int seq;
	u64 ret;

	do {
		seq = read_seqbegin(&jiffies_lock);
		ret = jiffies_64;
	} while (read_seqretry(&jiffies_lock, seq));
	return ret;
}
```

### 内存模型

x86只有StoreLoad可能会乱序，常用的release-acquire事实上并没有额外指令

```c++
// Thread A
x.store(1, std::memory_order_release);  // StoreStore

// Thread B
auto v = x.load(std::memory_order_accquire);  // LoadLoad
```

## Reference

+ [What is RCU, Fundamentally?](https://lwn.net/Articles/262464)
+ [seqlock（顺序锁）](https://www.cnblogs.com/linuxbug/p/4840139.html)
+ [Synchronization primitives in the Linux kernel. Part 6.](https://0xax.gitbooks.io/linux-insides/SyncPrim/linux-sync-6.html)
