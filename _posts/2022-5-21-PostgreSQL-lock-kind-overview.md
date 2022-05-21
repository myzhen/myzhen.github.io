---
layout: post
title: PostgreSQL中的4种锁类型
categories: lock
description: Four lock types in PostgreSQL
keywords: postgresql lock
---

# PostgreSQL中的4种锁类型

​	PostgreSQL实现并发控制的基本方法是使用锁来控制`临界区`的互斥访问。后台进程对磁盘文件进行访问操作时，首先要获得锁，如果成功获得目标锁，则进入临界区执行磁盘的读写访问，访问完成后退出临界区并释放锁。否则，无法进入临界区进行，进程会睡眠，直到被别的后台进程唤醒后尝试重新获得锁。

​	PostgreSQL中包含 `spinlock`、`lwlock`、`Regular lock`、`SIReadLock predicate lock`四种类型，像`spinlock`和`lwlock`只用内核代码中，控制内核代码的临界区访问，所以也称之为“系统锁”；这种锁内核开发人员在编写代码时会经常的接触到，而数据库使用人员DBA接触的是`Regular lock`，主要用在SQL中，由用户发出SQL语句来驱动锁请求，所以也被称为“事务锁”，而谓词锁是后面数据库版本中添加的一种锁，用于实现完全可串行化事务。

## spinlock

​	自旋锁（spin lock）主要用于封锁时间`非常短`的情况，如果一个锁被持有超过几十条指令或者跨越类别的内核调用，甚至是对一个非零子程序的调用，请不要使用自旋锁。自旋锁主要用作轻量级锁（lwlock）的基础设施，也就是说轻量级锁lwlock是基于spinlock实现的。

​	Spinlock是最底层的锁，使用硬件提供的原子操作原语`test-and-set`（tas如果可用）实现的。等待进程繁忙循环，直到他们可以得到锁（即我们常说的盲等状态，但是不会一直等下去，有如下所述的超时）。没有提供死锁检测、等待队列、错误时自动释放或任何其他细节。如果在一分钟左右后无法获得锁，则会出现超时（与预期的锁保持时间相比（自旋锁保持时间都非常短），这一分钟大约就是永远无法获得锁，所以这肯定是一个错误条件）。

​	Spinlock分为与机器相关的实现方法(定义在s_lock.c中)和与机器不相关的实现方法(定义在spin.c中)。spinlock的特点是：封锁时间很短，没有等待队列和死锁检测机制，事务结束时不能自动释放spinlock。最为最底层的一种锁，一般不直接使用spinlock，而是利用它来实现其他锁（LWLock）。毫无疑问，依赖硬件的spinlock机制肯定比不依赖硬件的spinlock机制速度快。因为不依赖硬件的spinlock机制需要使用PG的信号量来模拟spinlock。

​	如果机器有TAS(test-and-set)指令集，那么PpostgreSQL会使用s_lock.c和s_lock.h中定义的spinlock实现机制。如果机器没有TAS指令集，那么不依赖硬件的spinlock的实现定义在spin.c中，需要用到PostgreSQL定义的信号量PGSemaphore。

- **最底层的锁，可用其来实现LWLock的一种实现方式，当然LWLock还有其他实现机制，与CPU架构有关**
- **硬件相关和硬件无关两种实现方式**
- **封锁的持续时间很短，没有死锁检测、等待队列、elog()错误时的自动释放等机制，有超时机制。**
- **实现机制TAS（test-and-set）**

> Spinlock示例，这是LWLock的底层接口pg_atomic_compare_exchange_u32_impl，pg_atomic_compare_exchange_u32_impl有多个实现版本，不同硬件平台有不同的实现方式，这个例子是使用spinlock来实现的方式，用于演示spinlock的使用，调用SpinLockAcquire和SpinLockRelease获取和释放spinlock：

```c
bool
pg_atomic_compare_exchange_u32_impl(volatile pg_atomic_uint32 *ptr,
									uint32 *expected, uint32 newval)
{
	bool		ret;

/*
 * Do atomic op under a spinlock. It might look like we could just skip
 * the cmpxchg if the lock isn't available, but that'd just emulate a
 * 'weak' compare and swap. I.e. one that allows spurious failures. Since
 * several algorithms rely on a strong variant and that is efficiently
 * implementable on most major architectures let's emulate it here as
 * well.
 */
SpinLockAcquire((slock_t *) &ptr->sema);

/* perform compare/exchange logic */
ret = ptr->value == *expected;
*expected = ptr->value;
if (ret)
	ptr->value = newval;

/* and release lock */
SpinLockRelease((slock_t *) &ptr->sema);

return ret;

}
```

## LWLock

​	这些锁通常用于对`共享内存中的数据结构`的访问封锁，避免同一份共享内存数据被不同进程并发修改带来的数据不一致问题，即共享内存同一时间只允许一个进程进行改写。LWLocks 支持独占和共享锁模式（用于对共享对象的读/写和只读访问）。没有提供死锁检测，但 LWLock 管理器会在 `elog() 期间自动释放持有的 LWLock`，因此在持有 LWLock 时使用elog()发出一个错误是安全的。当没有锁竞争时，获取或释放 LWLock 非常快（几十条指令）。当进程为了获得 LWLock而必须等待时，它会阻塞 SysV 信号量，以免消耗 CPU 时间。 等待进程将按到达顺序被授予锁。没有超时。

- **主要用于封锁共享内存中的数据结构**
- **没有死锁检测机制、没有超时机制、有等待队列、能自动释放锁。**
- **申请和释放接口为：LWLockAcquire和LWLockRelease，不举例了，内核代码中很多。**
- **LWLock使用原子操作CAS（compare-and-exchange）比较并交换实现封锁。**

## 常规锁

​	也被称为事务锁，或重量级的锁。常规锁管理器支持多种锁模式，具有表驱动语义，具有完整的死锁检测和事务结束自动释放功能。对于所有用户驱动的锁请求都应该使用常规锁。

- **重锁，有死锁检测、能自动释放锁**

- **对数据对象加锁，而LWLock是对共享内存的数据结构加锁，spinlock加锁的临界区更小，是对共享内存中数据结构的结构体成员加锁。**

- **加锁颗粒度很多，表级锁、页级、元组级等。**

## 谓词锁

用于辅助解决一种SSI的冲突情况，还不了解，暂不做笔记。

*TODO*

## NOTE

获取自旋锁或轻量级锁会导致查询取消和 die() 中断被推迟，直到所有此类锁都被释放。但是，常规锁不存在这样的限制。另请注意，我们可以在等待常规锁时接受查询的取消和 die() 中断，但在等待自旋锁或 LW 锁时我们不会接受它们。因此，当等待时间可能超过几秒钟时，使用 LW 锁并不是一个好主意。

## TAS和CAS

这里简单说一下这两个原子操作指令，什么是原子操作呢？

> 原子操作(atomic operation)是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch （切换到另一个线程）。保证访问共享资源是唯一的。

**一般原子操作需要CPU等硬件支持**，使用硬件同步原语来实现原子操作。常用的原语有`CAS（compare-and-swap）`、`TAS（test-and-set）`。通过查阅资料也可以看到，TAS指令的特点就是自旋锁，CAS更适合实现轻量级锁。

原来**TAS**和**CAS**是需要硬件支持这种指令集的，至此我们清楚了，为什么spinlock和LWLock都有多重实现方式了。

1. ***spinlock为什么有两种实现方式？***

   - TAS指令实现方式，也被叫做硬件相关的spinlock实现方式。

   - 信号量模拟TAS操作，也被叫做硬件无关的spinlock实现方式。

     有两种方式的原因，很显然，硬件先关的TAS实现的spinlock性能自然是最好的，但是不支持TAS指令集的机器上我们总不能没有自旋锁啊，显然要用信号量来实现spinlock喽。

2. ***LWLock有几种实现方式？***

   先说结论，我认为与spinlock一样，也是两种实现方式。

   - 方式1：如果机器硬件支持CAS指令，则直接使用CAS指令来实现LWLock。

   - 方式2：如果机器硬件不支持CAS指令，则使用spinlock进行加锁（`spinlock本身也有硬件相关和硬件无关情况`），然后C代码模拟CAS操作，最后释放spinlock锁的流程来实现LWLock。

     对于LWLock，显然也是硬件支持CAS指令的方式性能是最佳的，因为依赖spinlock的实现方式可能机器硬件也不支持TAS指令，那么性能自然更差了，即使硬件支持TAS指令，但是模拟比较并交换的代码运行起来自然也会比硬件直接支持的比较并交换指令要差了。



> **参考：**

> 源码src/backend/storage/lmgr/README
>
> 《数据库事务处理的艺术 - 李海翔》
>
> 《PostgreSQL技术内幕-数据处理深度探索 - 张树杰》