https://0xax.gitbooks.io/linux-insides/

分析的是64位，不是32位

```
Virtual memory map with 4 level page tables:

0000000000000000 - 00007fffffffffff (=47 bits) user space, different per mm
hole caused by [48:63] sign extension
ffff800000000000 - ffff87ffffffffff (=43 bits) guard hole, reserved for hypervisor
ffff880000000000 - ffffc7ffffffffff (=64 TB) direct mapping of all phys. memory
ffffc80000000000 - ffffc8ffffffffff (=40 bits) hole
ffffc90000000000 - ffffe8ffffffffff (=45 bits) vmalloc/ioremap space
ffffe90000000000 - ffffe9ffffffffff (=40 bits) hole
ffffea0000000000 - ffffeaffffffffff (=40 bits) virtual memory map (1TB)
... unused hole ...
ffffec0000000000 - fffffbffffffffff (=44 bits) kasan shadow memory (16TB)
... unused hole ...
ffffff0000000000 - ffffff7fffffffff (=39 bits) %esp fixup stacks
... unused hole ...
ffffffef00000000 - fffffffeffffffff (=64 GB) EFI region mapping space
... unused hole ...
ffffffff80000000 - ffffffff9fffffff (=512 MB)  kernel text mapping, from phys 0
ffffffffa0000000 - ffffffffff5fffff (=1526 MB) module mapping space
ffffffffff600000 - ffffffffffdfffff (=8 MB) vsyscalls
ffffffffffe00000 - ffffffffffffffff (=2 MB) unused hole
```

系统版本

```shell
lthn2759@g83-gs28-19007:~$ lsb_release -d
Description:	Debian GNU/Linux 9.4 (stretch)
lthn2759@g83-gs28-19007:~$ hostnamectl
   Static hostname: g83-gs28-19007.i.nease.net
         Icon name: computer-vm
           Chassis: vm
        Machine ID: ec2c92068aa97b3b969227b8f2203152
           Boot ID: d1d8edf9372c401dba57e0dc7cccd29c
    Virtualization: kvm
  Operating System: Debian GNU/Linux 9 (stretch)
            Kernel: Linux 4.9.0-6-amd64
      Architecture: x86-64
```

https://www.kernel.org/

longterm:	4.9.154	2019-01-31

```
lthn2759@g83-gs28-19007:~$ lspci
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma]
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:01.3 Non-VGA unclassified device: Intel Corporation 82371AB/EB/MB PIIX4 ACPI (rev 08)
00:03.0 VGA compatible controller: Device 1d0f:1111
00:04.0 Non-Volatile memory controller: Device 1d0f:8061
00:05.0 Ethernet controller: Device 1d0f:ec20
00:1f.0 Non-Volatile memory controller: Device 1d0f:8061
```

```
lthn2759@g83-gs28-19007:~$ /sbin/ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 10.71.35.82  netmask 255.255.255.128  broadcast 10.71.35.127
        ether 06:4d:de:e8:d6:96  txqueuelen 1000  (Ethernet)
        RX packets 4554575  bytes 8310434088 (7.7 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5500403  bytes 542066800 (516.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1  (Local Loopback)
        RX packets 30065  bytes 4088714 (3.8 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 30065  bytes 4088714 (3.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

```
lthn2759@g83-gs28-19007:~$ /sbin/ethtool -i eth0
driver: ena
version: 1.5.2g
firmware-version: 
expansion-rom-version: 
bus-info: 0000:00:05.0
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: no
```

https://github.com/amzn/amzn-drivers

Elastic Network Adapter (ENA)

```
pumpkin@Pumpkins-MacBook-Air amzn-drivers$ git tag
ena_linux_1.1.2
ena_linux_1.1.3
ena_linux_1.2.0
ena_linux_1.3.0
ena_linux_1.4.0
ena_linux_1.5.0
ena_linux_1.5.1
ena_linux_1.5.2
ena_linux_1.5.3
ena_linux_1.6.0
ena_linux_2.0.2
pumpkin@Pumpkins-MacBook-Air amzn-drivers$ git checkout ena_linux_1.5.3
Note: checking out 'ena_linux_1.5.3'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 857b4b8 linux/ena: update ENA linux driver to version 1.5.3
```

top然后按1

```
top - 16:45:39 up 5 days, 37 min,  1 user,  load average: 14.87, 16.30, 17.11
Tasks: 499 total,   3 running, 496 sleeping,   0 stopped,   0 zombie
%Cpu0  : 23.1 us,  6.3 sy,  0.0 ni, 55.6 id,  0.0 wa,  0.0 hi, 15.0 si,  0.0 st
%Cpu1  : 25.6 us,  8.0 sy,  0.0 ni, 51.5 id,  0.0 wa,  0.0 hi, 15.0 si,  0.0 st
%Cpu2  : 27.8 us,  7.6 sy,  0.0 ni, 51.9 id,  0.0 wa,  0.0 hi, 12.7 si,  0.0 st
%Cpu3  : 26.9 us,  6.7 sy,  0.0 ni, 50.8 id,  0.0 wa,  0.0 hi, 15.5 si,  0.0 st
%Cpu4  : 25.3 us,  6.7 sy,  0.0 ni, 51.7 id,  0.0 wa,  0.0 hi, 16.3 si,  0.0 st
%Cpu5  : 25.2 us,  7.1 sy,  0.0 ni, 50.0 id,  0.0 wa,  0.0 hi, 17.7 si,  0.0 st
%Cpu6  : 29.9 us,  5.2 sy,  0.0 ni, 49.8 id,  0.0 wa,  0.0 hi, 15.1 si,  0.0 st
%Cpu7  : 24.3 us,  8.3 sy,  0.0 ni, 50.2 id,  0.0 wa,  0.0 hi, 17.3 si,  0.0 st
%Cpu8  : 25.5 us,  9.4 sy,  0.0 ni, 64.8 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu9  : 31.6 us,  8.6 sy,  0.0 ni, 59.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu10 : 23.5 us,  9.2 sy,  0.0 ni, 67.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu11 : 24.7 us,  8.6 sy,  0.0 ni, 66.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu12 : 24.1 us,  9.2 sy,  0.0 ni, 66.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu13 : 28.1 us,  8.0 sy,  0.0 ni, 63.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu14 : 25.1 us,  8.1 sy,  0.0 ni, 66.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu15 : 25.0 us,  9.5 sy,  0.0 ni, 65.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu16 : 24.1 us,  7.6 sy,  0.0 ni, 68.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu17 : 20.2 us,  8.5 sy,  0.0 ni, 71.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu18 : 23.4 us,  8.1 sy,  0.0 ni, 68.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu19 : 25.3 us,  7.4 sy,  0.0 ni, 67.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu20 : 21.3 us,  8.0 sy,  0.0 ni, 70.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu21 : 23.1 us,  7.1 sy,  0.0 ni, 69.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu22 : 24.7 us,  7.7 sy,  0.0 ni, 67.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu23 : 22.2 us,  7.5 sy,  0.0 ni, 70.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu24 : 23.6 us,  7.4 sy,  0.0 ni, 69.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu25 : 22.1 us,  7.5 sy,  0.0 ni, 70.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu26 : 21.9 us,  7.6 sy,  0.0 ni, 70.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu27 : 19.2 us,  7.3 sy,  0.0 ni, 73.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu28 : 20.7 us,  7.7 sy,  0.0 ni, 71.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu29 : 22.1 us,  5.4 sy,  0.0 ni, 72.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu30 : 23.6 us,  6.7 sy,  0.0 ni, 69.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu31 : 21.2 us,  7.1 sy,  0.0 ni, 71.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu32 : 23.0 us,  7.0 sy,  0.0 ni, 70.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu33 : 21.8 us,  7.6 sy,  0.0 ni, 70.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu34 : 20.6 us,  9.3 sy,  0.0 ni, 70.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu35 : 19.6 us,  7.8 sy,  0.0 ni, 72.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu36 : 19.0 us,  7.8 sy,  0.0 ni, 73.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu37 : 23.2 us,  4.5 sy,  0.0 ni, 72.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu38 : 21.5 us,  6.7 sy,  0.0 ni, 71.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu39 : 23.2 us,  5.4 sy,  0.0 ni, 71.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu40 : 25.2 us,  7.0 sy,  0.0 ni, 67.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

去除调试打印代码`dev_dbg`

```c
// linux/device.h

#define dev_dbg(dev, format, ...)		     \
do {						     \
	dynamic_dev_dbg(dev, format, ##__VA_ARGS__); \
} while (0)
```

去除异常处理代码
```c
// ena_netdev.c

rc = pci_enable_device_mem(pdev);
if (rc) {
	dev_err(&pdev->dev, "pci_enable_device_mem() failed!\n");
	return rc;
}
```

化简为

```c
pci_enable_device_mem(pdev);
```

linux版本确定，去除无效的宏分支

```c
#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 8, 0)
// ...
#else
// ...
#endif


```c
static struct pci_driver ena_pci_driver = {
	.name		= DRV_MODULE_NAME,
	.id_table	= ena_pci_tbl,
	.probe		= ena_probe,
	.remove		= ena_remove,
#ifdef CONFIG_PM
	.suspend    = ena_suspend,
	.resume     = ena_resume,
#endif
#ifdef HAVE_SRIOV_CONFIGURE
	.sriov_configure = ena_sriov_configure,
#endif /* HAVE_SRIOV_CONFIGURE */
};

static int __init ena_init(void)
{
	ena_wq = create_singlethread_workqueue(DRV_MODULE_NAME);
	return pci_register_driver(&ena_pci_driver);
}

static void __exit ena_cleanup(void)
{
	pci_unregister_driver(&ena_pci_driver);
	if (ena_wq) {
		destroy_workqueue(ena_wq);
		ena_wq = NULL;
	}
}

module_init(ena_init);
module_exit(ena_cleanup);
```

`suspend`和`resume`是电源管理相关的挂起和恢复函数，不考虑
sriov后面再考略

`struct ena_adapter`是驱动的私有数据，跟内核定义无关

IO中断，tx和rx共用

```c
static irqreturn_t ena_intr_msix_io(int irq, void *data)
{
	struct ena_napi *ena_napi = data;

	ena_napi->tx_ring->first_interrupt = true;
	ena_napi->rx_ring->first_interrupt = true;

	atomic_set(&ena_napi->unmask_interrupt, 1);
	napi_schedule_irqoff(&ena_napi->napi);

	return IRQ_HANDLED;
}
```

```c
// include/linux/netdevice.h

/**
 *	napi_schedule_irqoff - schedule NAPI poll
 *	@n: NAPI context
 *
 * Variant of napi_schedule(), assuming hard irqs are masked.
 */
static inline void napi_schedule_irqoff(struct napi_struct *n)
{
	if (napi_schedule_prep(n))
		__napi_schedule_irqoff(n);
}
```

```c
// net/core/dev.c

/**
 * __napi_schedule_irqoff - schedule for receive
 * @n: entry to schedule
 *
 * Variant of __napi_schedule() assuming hard irqs are masked
 */
void __napi_schedule_irqoff(struct napi_struct *n)
{
	____napi_schedule(this_cpu_ptr(&softnet_data), n);
}

/* Called with irq disabled */
static inline void ____napi_schedule(struct softnet_data *sd,
				     struct napi_struct *napi)
{
	list_add_tail(&napi->poll_list, &sd->poll_list);
	__raise_softirq_irqoff(NET_RX_SOFTIRQ);
}
```

```c
// softirq.c

void __raise_softirq_irqoff(unsigned int nr)
{
	trace_softirq_raise(nr);
	or_softirq_pending(1UL << nr);
}
```

```c
// include/trace/events/irq.h

/**
 * softirq_raise - called immediately when a softirq is raised
 * @vec_nr:  softirq vector number
 *
 * When used in combination with the softirq_entry tracepoint
 * we can determine the softirq raise to run latency.
 */
DEFINE_EVENT(softirq, softirq_raise,

	TP_PROTO(unsigned int vec_nr),

	TP_ARGS(vec_nr)
);

// arch/x86/include/asm/hardirq.h
#define or_softirq_pending(x)	this_cpu_or(irq_stat.__softirq_pending, (x))
```


# 附录

## Per-CPU Variable

Per-CPU Variable是内核的一个feature，顾名思义，就是每个CPU（核心）都有一个变量的副本，修改当前CPU的变量值，并不会影响其他CPU的值。也就是说Per-CPU Variable是不会存在并行读写，读写不需要显式加锁。这样的实现显示是为了性能考虑，每个CPU都有各自的独立cache-line，而不需要在变量更新的时候刷新其他CPU的cache。在前面已经看到Per-CPU Variable的一个使用例子，当前CPU触发的硬中断只会修改当前核心的软中断标记。

Per-CPU Varibale的使用说明，可以查看Linux的文档this_cpu_ops.txt。

以下是一个简单的使用例子，定义一个Per-CPU的int变量x，并且用指定的方法去读取当前执行CPU的变量值：

```c
DEFINE_PER_CPU(int, x);
int z = this_cpu_read(x);
```

这里分三部分来分析Per-CPU的实现，分别是变量的声明/定义，读写操作，和初始化。

### 声明/定义

因为声明（declare）和定义（define）的实现时类似的，所以这里只分析声明的调用。如上述使用例子所描述的，声明的入口是`DECLARE_PER_CPU`:

```c
// include/linux/percpu-defs.h

#define DECLARE_PER_CPU(type, name)					\
	DECLARE_PER_CPU_SECTION(type, name, "")

#define DECLARE_PER_CPU_SECTION(type, name, sec)			\
	extern __PCPU_ATTRS(sec) __typeof__(type) name

#define __PCPU_ATTRS(sec)						\
	__percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))	\
	PER_CPU_ATTRIBUTES
```

```c
// include/asm-generic/percpu.h

#define PER_CPU_BASE_SECTION ".data..percpu"
```

宏展开后，可以看出Per-CPU变量的特别之处，就是放置在一个特别的段内（.data..percpu）。当然这里还有一个特别的修饰词`__percpu`：

```c
// include/linux/compiler.h

#ifdef __CHECKER__
# define __percpu	__attribute__((noderef, address_space(3)))
#else /* __CHECKER__ */
# define __percpu
#endif /* __CHECKER__ */
```

`__percpu`是Linux的静态代码检查工具Sparse所使用的，gcc是会直接忽略因此对运行时没有任何影响。这个检查是为了避免把Per-CPU变量当成是普通变量读写的代码错误。关于Sparse的更详细说明可以查看Linux文档dev/tools/sparse.rst。

### 读/写

读写操作其实是类似的，这里是关心读操作的实现。当然对于写操作来说，代码书写的时候要考虑更多。假如操作Per-CPU变量时，没有关闭内核抢占，两次操作的变量值可能是不同CPU上的副本，例如

```c
this_cpu_inc(x); // 原子
// 抢占，当前进程可能被在其他CPU执行，前后两次修改的x值是不同CPU上的副本
this_cpu_inc(x); // 原子
```

对于RMW（Read Modify Write）操作，在没有关闭中断的情况下，也有可能出现脏读，例如

```c
int z = this_cpu_read(x); // x = 1, z = 1
// 中断发生，对x进行自减，x = 0
this_cpu_write(x, z + 1);  // ERROR：x = 2
```

这两个例子只是为了说明Per-CPU变量虽然不存在并行读写，但依然要考虑逻辑上的并发问题。这跟用户层中执行多线程程序一样，即使运行环境是单核CPU，也依然需要同步原语来避免抢占导致的并发问题。

再回到读操作`this_cpu_read`的实现上

```c
// include/linux/percpu-defs.h

#define this_cpu_read(pcp)		__pcpu_size_call_return(this_cpu_read_, pcp)

#define __pcpu_size_call_return(stem, variable)				\
({									\
	typeof(variable) pscr_ret__;					\
	__verify_pcpu_ptr(&(variable));					\
	switch(sizeof(variable)) {					\
	case 1: pscr_ret__ = stem##1(variable); break;			\
	case 2: pscr_ret__ = stem##2(variable); break;			\
	case 4: pscr_ret__ = stem##4(variable); break;			\
	case 8: pscr_ret__ = stem##8(variable); break;			\
	default:							\
		__bad_size_call_parameter(); break;			\
	}								\
	pscr_ret__;							\
})
```

假设变量类型是int

```c
// arch/x86/include/asm/percpu.h

#define this_cpu_read_4(pcp)		percpu_from_op("mov", pcp)

#define percpu_from_op(op, var)				\
({							\
	typeof(var) pfo_ret__;				\
	switch (sizeof(var)) {				\
	case 4:						\
		asm volatile(op "l "__percpu_arg(1)",%0"\
		    : "=r" (pfo_ret__)			\
		    : "m" (var));			\
		break;					\
	default: __bad_percpu_size();			\
	}						\
	pfo_ret__;					\
})

#define __percpu_arg(x)		__percpu_prefix "%" #x
#define __percpu_seg		gs
```

```c
// include/linux/stringify.h

#define __stringify_1(x...)	#x
#define __stringify(x...)	__stringify_1(x)
```

一堆宏展开之后，其实`this_cpu_read`关键操作就是一行汇编`mov ax, gs:[var]`，所以Per-CPU变量所保存的就是以gs寄存器（x86下是fs）为基地址的偏移量。

### 初始化

总结一下前面声明/定义，和读/写两节。Per-CPU变量是定义在一个特殊的段内，既然每个CPU都有变量的独立副本，那就必定有一个地方把这个段复制了多份，并分配给各个CPU。另一方面，Per-CPU变量的读写依赖于gs寄存器，初始化的时候需要把这个特殊段的地址赋值到gs寄存器。这两步就是Per-CPU变量初始化的关键步骤，我们从具体代码中印证一下这个想法。

```c
// arch/x86/kernel/setup-percpu.c

void __init setup_per_cpu_areas(void)
{
	atom_size = PMD_SIZE;
	rc = pcpu_embed_first_chunk(PERCPU_FIRST_CHUNK_RESERVE,
						dyn_size, atom_size,
						pcpu_cpu_distance,
						pcpu_fc_alloc, pcpu_fc_free);

	/* alrighty, percpu areas up and running */
	delta = (unsigned long)pcpu_base_addr - (unsigned long)__per_cpu_start;
	for_each_possible_cpu(cpu) {
		per_cpu_offset(cpu) = delta + pcpu_unit_offsets[cpu];
		per_cpu(this_cpu_off, cpu) = per_cpu_offset(cpu);
		per_cpu(cpu_number, cpu) = cpu;
	}
}
```

删减后代码关键是`pcpu_embed_first_chunk`会根据CPU的数量和Per-CPU段的大小分配内存，同时把每个段的基地址记录在`pcpu_unit_offsets`。




```c
// mm/percpu.c

const char * const pcpu_fc_names[PCPU_FC_NR] __initconst = {
	[PCPU_FC_AUTO]	= "auto",
	[PCPU_FC_EMBED]	= "embed",
	[PCPU_FC_PAGE]	= "page",
};

enum pcpu_fc pcpu_chosen_fc __initdata = PCPU_FC_AUTO;

static int __init percpu_alloc_setup(char *str)
{
	if (!str)
		return -EINVAL;

	if (0)
		/* nada */;
#ifdef CONFIG_NEED_PER_CPU_EMBED_FIRST_CHUNK
	else if (!strcmp(str, "embed"))
		pcpu_chosen_fc = PCPU_FC_EMBED;
#endif
#ifdef CONFIG_NEED_PER_CPU_PAGE_FIRST_CHUNK
	else if (!strcmp(str, "page"))
		pcpu_chosen_fc = PCPU_FC_PAGE;
#endif
	else
		pr_warn("unknown allocator %s specified\n", str);

	return 0;
}
early_param("percpu_alloc", percpu_alloc_setup);
```



http://david-grs.github.io/tls_performance_overhead_cost_linux/

https://stackoverflow.com/questions/16978959/how-are-percpu-pointers-implemented-in-the-linux-kernel

https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html


## ftrace

trace/ftrace.txt

https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/


## 中断

```c
// arch/x86/kernel/head64.c

asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data)
{
	// ...
	for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
		set_intr_gate(i, early_idt_handler_array[i]);
	load_idt((const struct desc_ptr *)&idt_descr);
	// ...
}

// arch/x86/include/asm/desc.h
#define load_idt(dtr)				native_load_idt(dtr)

static inline void native_load_idt(const struct desc_ptr *dtr)
{
	asm volatile("lidt %0"::"m" (*dtr));
}

extern struct desc_ptr idt_descr;
extern gate_desc idt_table[];

// arch/x86/include/desc-defs.h
struct desc_ptr {
	unsigned short size;
	unsigned long address;
} __attribute__((packed)) ;

// arch/x86/kernel/cpu/common.c
struct desc_ptr idt_descr __ro_after_init = {
	.size = NR_VECTORS * 16 - 1,
	.address = (unsigned long) idt_table,
};

// arch/x86/kernel/irqinit.c
void __init native_init_IRQ(void)
{
	// ...
#ifndef CONFIG_X86_LOCAL_APIC
#define first_system_vector NR_VECTORS
#endif
	for_each_clear_bit_from(i, used_vectors, first_system_vector) {
		/* IA32_SYSCALL_VECTOR could be used in trap_init already. */
		set_intr_gate(i, irq_entries_start +
				8 * (i - FIRST_EXTERNAL_VECTOR));
	}
	// ...
}
```

```c
// arch/x86/entry/entry_64.S
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
    pushq	$(~vector+0x80)			/* Note: always in signed byte range */
    vector=vector+1
    jmp	common_interrupt
    .align	8
    .endr
END(irq_entries_start)

common_interrupt:
	ASM_CLAC
	addq	$-0x80, (%rsp)			/* Adjust vector to [-256, -1] range */
	interrupt do_IRQ
```

```c
// arch/x86/kernel/irq.c

/*
 * do_IRQ handles all normal device IRQ's (the special
 * SMP cross-CPU interrupts have their own specific
 * handlers).
 */
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
	struct pt_regs *old_regs = set_irq_regs(regs);
	struct irq_desc * desc;
	/* high bit used in ret_from_ code  */
	unsigned vector = ~regs->orig_ax;

	/*
	 * NB: Unlike exception entries, IRQ entries do not reliably
	 * handle context tracking in the low-level entry code.  This is
	 * because syscall entries execute briefly with IRQs on before
	 * updating context tracking state, so we can take an IRQ from
	 * kernel mode with CONTEXT_USER.  The low-level entry code only
	 * updates the context if we came from user mode, so we won't
	 * switch to CONTEXT_KERNEL.  We'll fix that once the syscall
	 * code is cleaned up enough that we can cleanly defer enabling
	 * IRQs.
	 */

	entering_irq();

	/* entering_irq() tells RCU that we're not quiescent.  Check it. */
	RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");

	desc = __this_cpu_read(vector_irq[vector]);

	if (!handle_irq(desc, regs)) {
		ack_APIC_irq();

		if (desc != VECTOR_RETRIGGERED) {
			pr_emerg_ratelimited("%s: %d.%d No irq handler for vector\n",
					     __func__, smp_processor_id(),
					     vector);
		} else {
			__this_cpu_write(vector_irq[vector], VECTOR_UNUSED);
		}
	}

	exiting_irq();

	set_irq_regs(old_regs);
	return 1;
}
```

## softirq

ksoftirq

Documentation/kernel-per-CPU-kthreads.txt

### 软中断触发

```c
// arch/x86/include/asm/apic.h
static inline void exiting_irq(void)
{
	irq_exit();
}
```

```c
// kernel/softirq.c
/*
 * Exit an interrupt context. Process softirqs if needed and possible:
 */
void irq_exit(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
	local_irq_disable();
#else
	WARN_ON_ONCE(!irqs_disabled());
#endif

	account_irq_exit_time(current);
	preempt_count_sub(HARDIRQ_OFFSET);
	if (!in_interrupt() && local_softirq_pending())
		invoke_softirq();

	tick_irq_exit();
	rcu_irq_exit();
	trace_hardirq_exit(); /* must be last! */
}

static inline void invoke_softirq(void)
{
	if (ksoftirqd_running(local_softirq_pending()))
		return;

	if (!force_irqthreads) {
#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
		/*
		 * We can safely execute softirq on the current stack if
		 * it is the irq stack, because it should be near empty
		 * at this stage.
		 */
		__do_softirq();
#else
		/*
		 * Otherwise, irq_exit() is called on the task stack that can
		 * be potentially deep already. So call softirq in its own stack
		 * to prevent from any overrun.
		 */
		do_softirq_own_stack();
#endif
	} else {
		wakeup_softirqd();
	}
}

/*
 * If ksoftirqd is scheduled, we do not want to process pending softirqs
 * right now. Let ksoftirqd handle this at its own rate, to get fairness,
 * unless we're doing some of the synchronous softirqs.
 */
#define SOFTIRQ_NOW_MASK ((1 << HI_SOFTIRQ) | (1 << TASKLET_SOFTIRQ))
static bool ksoftirqd_running(unsigned long pending)
{
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

	if (pending & SOFTIRQ_NOW_MASK)
		return false;
	return tsk && (tsk->state == TASK_RUNNING);
}

/*
 * we cannot loop indefinitely here to avoid userspace starvation,
 * but we also don't want to introduce a worst case 1/HZ latency
 * to the pending events, so lets the scheduler to balance
 * the softirq load for us.
 */
static void wakeup_softirqd(void)
{
	/* Interrupts are disabled: no need to stop preemption */
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

	if (tsk && tsk->state != TASK_RUNNING)
		wake_up_process(tsk);
}
```

```c
// kernel/softirq.c

static struct smp_hotplug_thread softirq_threads = {
	.store			= &ksoftirqd,
	.thread_should_run	= ksoftirqd_should_run,
	.thread_fn		= run_ksoftirqd,
	.thread_comm		= "ksoftirqd/%u",
};

static __init int spawn_ksoftirqd(void)
{
	cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
				  takeover_tasklets);
	BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

	return 0;
}
early_initcall(spawn_ksoftirqd);

static void run_ksoftirqd(unsigned int cpu)
{
	local_irq_disable();
	if (local_softirq_pending()) {
		/*
		 * We can safely run softirq on inline stack, as we are not deep
		 * in the task stack here.
		 */
		__do_softirq();
		local_irq_enable();
		cond_resched_rcu_qs();
		return;
	}
	local_irq_enable();
}
```

### 软中断注册

```c
// net/core/dev.c

static int __init net_dev_init(void)
{
	// ...
	open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	open_softirq(NET_RX_SOFTIRQ, net_rx_action);
	// ...
}

static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```


### 软中断处理

```c
// kernel/softirq.c

asmlinkage __visible void __softirq_entry __do_softirq(void)
{
	unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
	unsigned long old_flags = current->flags;
	int max_restart = MAX_SOFTIRQ_RESTART;
	struct softirq_action *h;
	bool in_hardirq;
	__u32 pending;
	int softirq_bit;

	/*
	 * Mask out PF_MEMALLOC s current task context is borrowed for the
	 * softirq. A softirq handled such as network RX might set PF_MEMALLOC
	 * again if the socket is related to swap
	 */
	current->flags &= ~PF_MEMALLOC;

	pending = local_softirq_pending();
	account_irq_enter_time(current);

	__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
	in_hardirq = lockdep_softirq_start();

restart:
	/* Reset the pending bitmask before enabling irqs */
	set_softirq_pending(0);

	local_irq_enable();

	h = softirq_vec;

	while ((softirq_bit = ffs(pending))) {
		unsigned int vec_nr;
		int prev_count;

		h += softirq_bit - 1;

		vec_nr = h - softirq_vec;
		prev_count = preempt_count();

		kstat_incr_softirqs_this_cpu(vec_nr);

		trace_softirq_entry(vec_nr);
		h->action(h);
		trace_softirq_exit(vec_nr);
		if (unlikely(prev_count != preempt_count())) {
			pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
			       vec_nr, softirq_to_name[vec_nr], h->action,
			       prev_count, preempt_count());
			preempt_count_set(prev_count);
		}
		h++;
		pending >>= softirq_bit;
	}

	rcu_bh_qs();
	local_irq_disable();

	pending = local_softirq_pending();
	if (pending) {
		if (time_before(jiffies, end) && !need_resched() &&
		    --max_restart)
			goto restart;

		wakeup_softirqd();
	}

	lockdep_softirq_end(in_hardirq);
	account_irq_exit_time(current);
	__local_bh_enable(SOFTIRQ_OFFSET);
	WARN_ON_ONCE(in_interrupt());
	tsk_restore_flags(current, old_flags, PF_MEMALLOC);
}
```

## NAPI

### 注册

```c
void netif_napi_add(struct net_device *dev, struct napi_struct *napi,
		    int (*poll)(struct napi_struct *, int), int weight)
{
	INIT_LIST_HEAD(&napi->poll_list);
	hrtimer_init(&napi->timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_PINNED);
	napi->timer.function = napi_watchdog;
	napi->gro_count = 0;
	napi->gro_list = NULL;
	napi->skb = NULL;
	napi->poll = poll;
	if (weight > NAPI_POLL_WEIGHT)
		pr_err_once("netif_napi_add() called with weight %d on device %s\n",
			    weight, dev->name);
	napi->weight = weight;
	list_add(&napi->dev_list, &dev->napi_list);
	napi->dev = dev;
#ifdef CONFIG_NETPOLL
	spin_lock_init(&napi->poll_lock);
	napi->poll_owner = -1;
#endif
	set_bit(NAPI_STATE_SCHED, &napi->state);
	napi_hash_add(napi);
}
EXPORT_SYMBOL(netif_napi_add);
```

### 处理

```c
int netdev_budget __read_mostly = 300;

static __latent_entropy void net_rx_action(struct softirq_action *h)
{
	struct softnet_data *sd = this_cpu_ptr(&softnet_data);
	unsigned long time_limit = jiffies + 2;
	int budget = netdev_budget;
	LIST_HEAD(list);
	LIST_HEAD(repoll);

	local_irq_disable();
	list_splice_init(&sd->poll_list, &list);
	local_irq_enable();

	for (;;) {
		struct napi_struct *n;

		if (list_empty(&list)) {
			if (!sd_has_rps_ipi_waiting(sd) && list_empty(&repoll))
				return;
			break;
		}

		n = list_first_entry(&list, struct napi_struct, poll_list);
		budget -= napi_poll(n, &repoll);

		/* If softirq window is exhausted then punt.
		 * Allow this to run for 2 jiffies since which will allow
		 * an average latency of 1.5/HZ.
		 */
		if (unlikely(budget <= 0 ||
			     time_after_eq(jiffies, time_limit))) {
			sd->time_squeeze++;
			break;
		}
	}

	__kfree_skb_flush();
	local_irq_disable();

	list_splice_tail_init(&sd->poll_list, &list);
	list_splice_tail(&repoll, &list);
	list_splice(&list, &sd->poll_list);
	if (!list_empty(&sd->poll_list))
		__raise_softirq_irqoff(NET_RX_SOFTIRQ);

	net_rps_action_and_irq_enable(sd);
}
```

```c
static int napi_poll(struct napi_struct *n, struct list_head *repoll)
{
	void *have;
	int work, weight;

	list_del_init(&n->poll_list);

	have = netpoll_poll_lock(n);

	weight = n->weight;

	/* This NAPI_STATE_SCHED test is for avoiding a race
	 * with netpoll's poll_napi().  Only the entity which
	 * obtains the lock and sees NAPI_STATE_SCHED set will
	 * actually make the ->poll() call.  Therefore we avoid
	 * accidentally calling ->poll() when NAPI is not scheduled.
	 */
	work = 0;
	if (test_bit(NAPI_STATE_SCHED, &n->state)) {
		work = n->poll(n, weight);
		trace_napi_poll(n, work, weight);
	}

	WARN_ON_ONCE(work > weight);

	if (likely(work < weight))
		goto out_unlock;

	/* Drivers must not modify the NAPI state if they
	 * consume the entire weight.  In such cases this code
	 * still "owns" the NAPI instance and therefore can
	 * move the instance around on the list at-will.
	 */
	if (unlikely(napi_disable_pending(n))) {
		napi_complete(n);
		goto out_unlock;
	}

	if (n->gro_list) {
		/* flush too old packets
		 * If HZ < 1000, flush all packets.
		 */
		napi_gro_flush(n, HZ >= 1000);
	}

	/* Some drivers may have called napi_schedule
	 * prior to exhausting their budget.
	 */
	if (unlikely(!list_empty(&n->poll_list))) {
		pr_warn_once("%s: Budget exhausted after napi rescheduled\n",
			     n->dev ? n->dev->name : "backlog");
		goto out_unlock;
	}

	list_add_tail(&n->poll_list, repoll);

out_unlock:
	netpoll_poll_unlock(have);

	return work;
}
```

### Busy Poll

https://blog.csdn.net/dog250/article/details/46798501


## sysctl

Documents/sysctl/net.txt

## RSS、RPS、RFS、XFS

Scaling in the Linux Networking Stack

Ducuments/networking/scaling.txt

+ RSS: Receive Side Scaling
+ RPS: Receive Packet Steering
+ RFS: Receive Flow Steering
+ Accelerated Receive Flow Steering
+ XPS: Transmit Packet Steering
