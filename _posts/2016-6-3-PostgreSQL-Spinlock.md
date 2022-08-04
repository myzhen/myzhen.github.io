---
layout: post
title: PostgreSQL的SpinLock源码分析
categories: lock
description: the spinlock of PostgreSQL
keywords: postgresql lock
---

# PostgreSQL中自旋锁的源码分析

在PostgreSQL中，有两种实现SpinLock的方式：

- **硬件无关的实现方式（Hardware-independent implementation）**
  - spin.c
  - spin.h
- **硬件相关的实现方式（Hardware-dependent implementation）**
  - s_lock.c
  - s_lock.h

如果我们编译时，本地机器的硬件并不支持TAS指令，而我们在configure时又没有指定--disable-spinlock选项，则在编译时会报错：`PostgreSQL does not have native spinlock support on this platform.  To continue the compilation, rerun configure using --disable-spinlocks.  However, performance will be poor.  Please report this to pgsql-bugs@lists.postgresql.org.`:

```c
/* s_lock.h */
#ifdef HAVE_SPINLOCKS	/* !HAVE_SPINLOCKS */

/* Blow up if we didn't have any way to do spinlocks */
#ifndef HAS_TEST_AND_SET
#error PostgreSQL does not have native spinlock support on this platform.  To continue the compilation, rerun configure using --disable-spinlocks.  However, performance will be poor.  Please report this to pgsql-bugs@lists.postgresql.org.
#endif

#else	/* !HAVE_SPINLOCKS */

......
    
#endif	/* HAVE_SPINLOCKS */
```

## 信号量实现的SpinLock（硬件无关方式）

​	在PostgreSQL中，信号量的数据结构被定义为`PGSemaphore`。其实PG是对信号量进行了一个自己的封装，因为不同的OS架构中对信号量的内核层接口不同，为了跨平台运行，信号量的底层实现依OS框架不同而调用不同的操作系统接口来处理，详见`src/include/storage/pg_sema.h`。

​	当configure时指定`--disable-spinlock`时，说明不支持TAS指令，才会使用信号量模拟spinlock，大部分情况下，我们都不会使用到这种spinlock的实现方式。内部使用128个信号量来模拟spinlock，也就是最多支持128种spinlock。这需要在共享内存中开辟一块空间来保存这128个信号量。如果机器支持**TAS**指令，那么`NUM_EMULATION_SEMAPHORES`的值为0，不会在共享内存中开启空间来保存信号量了。

下面是实现代码`spin.c`，因为代码很少，直接贴整个文件看：

```c
/*-------------------------------------------------------------------------
 *
 * spin.c
 *	   Hardware-independent implementation of spinlocks.
 *
 * 对于拥有test-and-set (TAS)指令的机器，在s_lock.h/.c中定义了自旋锁实现。
 * 这个文件只包含一个使用信号量PGSemaphores来模拟自旋锁的实现，用来替代TAS操作。
 * 除非以不涉及内核调用的方式（eg：sem_init）实现信号量，否则这太慢了，不会很有用。
 * For machines that have test-and-set (TAS) instructions, s_lock.h/.c
 * define the spinlock implementation.  This file contains only a stub
 * implementation for spinlocks using PGSemaphores.  Unless semaphores
 * are implemented in a way that doesn't involve a kernel call, this
 * is too slow to be very useful :-(
 *
 * 从注释中可知，spin.c是使用信号量来模拟的自旋锁，如果运行PG机器不支持TAS指令集才会
 * 使用信号量实现的自旋锁，但是信号量实现的自旋锁性能差，只要我目前看到PG的信号量都涉及
 * 到了操作系统内核的调用（PGSemaphore的代码我还没看，也许我是错的），但是查了一下确
 * 存在一种硬件支持的信号量实现，信号量的硬实现叫做“HSEM”（Hareware Semaphore）模块，
 * 被集成到STM32单片机的CPU中，也许上面注释中说的：不涉及内核调用方式实现的信号量就是HSEM吧。
 *
 * Portions Copyright (c) 1996-2021, PostgreSQL Global Development Group
 * Portions Copyright (c) 1994, Regents of the University of California
 *
 *
 * IDENTIFICATION
 *	  src/backend/storage/lmgr/spin.c
 *
 *-------------------------------------------------------------------------
 */
#include "postgres.h"

#include "storage/pg_sema.h"
#include "storage/shmem.h"
#include "storage/spin.h"

/* 注意：
 * HAVE_SPINLOCKS PG在configure时定义的宏，用来指示数据库是否禁用本地自旋锁，也就是是否
 * 禁用TAS指令实现的自旋锁！所以，也就是说PG可以在configure时通过指定--disable-spinlock选项
 * 来关闭TAS方式的自旋锁，但是这会导致数据库 性能的下降，默认是打开的，所以大多数情况下，
 * 这个宏都是被定义的。
 * 详见：configure脚本，configure后定义在src/include/pg_config.h中。
 */
#ifndef HAVE_SPINLOCKS

/*
 * No TAS, so spinlocks are implemented as PGSemaphores.
 */
/*
 * NUM_EMULATION_SEMAPHORES：
 *	如果机器即不支持本地的自旋锁也不支持原子操作（硬件支持），那么自旋锁和原子操作都需要通过信号量
 *	来模拟实现，这个宏表示模拟这两个操作一共需要的信号量。
 * NUM_SPINLOCK_SEMAPHORES：
 *	通过信号量模拟spinlock时需要的信号量数，默认值是128，所以数据库系统最大同时可用的SpinLock的
 *	数量为128个，128这个默认值的选取是参考OS信号量的默认值也是126（/proc/sys/kernel/sem），
 *	减少NUM_SPINLOCK_SEMAPHORES的值可以减少OS资源的消耗，增加NUM_SPINLOCK_SEMAPHORES可以
 *	提升性能，但是TAS是最快的。
 *	TODO TEST:
 *	如果，我们在编译前加大NUM_SPINLOCK_SEMAPHORES的值，同时修改OS的信号量大小，是否会提升性能？
 *	也许，稍后可以跑个TPCC测试一下提升效果是否明显，前提是不支持TAS，否则也用不上信号量来
 *	模拟spinlock。
 */
#ifndef HAVE_ATOMICS
#define NUM_EMULATION_SEMAPHORES (NUM_SPINLOCK_SEMAPHORES + NUM_ATOMICS_SEMAPHORES)
#else
#define NUM_EMULATION_SEMAPHORES (NUM_SPINLOCK_SEMAPHORES)
#endif							/* DISABLE_ATOMICS */

/* backend本地指针，指向共享内存中保存的模拟spinlock的信号量数组 */
PGSemaphore *SpinlockSemaArray;

#else							/* !HAVE_SPINLOCKS */

#define NUM_EMULATION_SEMAPHORES 0

#endif							/* HAVE_SPINLOCKS */

/*
 * Report the amount of shared memory needed to store semaphores for spinlock
 * support.
 * 被CreateSharedMemoryAndSemaphores调用，报告在共享内存中需要开辟多大空间来存储信号量。
 */
Size
SpinlockSemaSize(void)
{
	return NUM_EMULATION_SEMAPHORES * sizeof(PGSemaphore);
}

/*
 * Report number of semaphores needed to support spinlocks.
 * 返回模拟自旋锁和原子操作需要的信号量总数。
 */
int
SpinlockSemas(void)
{
	return NUM_EMULATION_SEMAPHORES;
}

#ifndef HAVE_SPINLOCKS

/*
 * Initialize spinlock emulation.
 *
 * This must be called after PGReserveSemaphores().
 * 也是被CreateSharedMemoryAndSemaphores调用的子程序，用来初始化共享内存中的信号量数组。
 * 注意，只有在configure时指定了--disable-spinlock时，即禁用了TAS方式的自旋锁时，
 * CreateSharedMemoryAndSemaphores才会调用此子例程，来初始化共享内存的信号量数组。
 * 通过信号量模拟spinlock要现有PG的信号量，所有调用顺序必须位于PGReserveSemaphores()之后。
 */
void
SpinlockSemaInit(void)
{
	PGSemaphore *spinsemas;
	int			nsemas = SpinlockSemas();
	int			i;

   	/* 
   	 * 一般情况下，我们访问共享内存是需要首先要获取SpinLock来保护共享内存的并发访问，但是此时
   	 * 共享内存锁（spinlock）显然还不存在，所以使用了不带锁的ShmemAllocUnlocked（）函数
   	 * 来分配共享内存的空间，大部分情况下我们使用的都是ShmemAlloc（）来分配共享内存空间的。
   	 * 在ShmemAlloc中会先执行SpinLockAcquire(ShmemLock);来保护共享内存。
   	 */
	/*
	 * We must use ShmemAllocUnlocked(), since the spinlock protecting
	 * ShmemAlloc() obviously can't be ready yet.
	 */
	spinsemas = (PGSemaphore *) ShmemAllocUnlocked(SpinlockSemaSize());
	for (i = 0; i < nsemas; ++i)
		spinsemas[i] = PGSemaphoreCreate();
	SpinlockSemaArray = spinsemas;	//显然，spinsemas是一个共享内存地址空间，本地保存一份。
}

/*
 * s_lock.h hardware-spinlock emulation using semaphores
 *
 * We map all spinlocks onto NUM_EMULATION_SEMAPHORES semaphores.  It's okay to
 * map multiple spinlocks onto one semaphore because no process should ever
 * hold more than one at a time.  We just need enough semaphores so that we
 * aren't adding too much extra contention from that.
 *
 * There is one exception to the restriction of only holding one spinlock at a
 * time, which is that it's ok if emulated atomic operations are nested inside
 * spinlocks. To avoid the danger of spinlocks and atomic using the same sema,
 * we make sure "normal" spinlocks and atomics backed by spinlocks use
 * distinct semaphores (see the nested argument to s_init_lock_sema).
 *
 * slock_t is just an int for this implementation; it holds the spinlock
 * number from 1..NUM_EMULATION_SEMAPHORES.  We intentionally ensure that 0
 * is not a valid value, so that testing with this code can help find
 * failures to initialize spinlocks.
 */

/*
 * 下面这些函数都是spinlock的底层实现接口，可以将信号量理解是一个二进制信号量，即有效值只有0和1。
 * 信号量为1表示有资源可用，即可以获得锁，这时获取这个锁然后
 * 信号量递减为0表示没有资源可用，其他并发进程发现是0就会盲等，释放锁时递增为1，锁变为可用状态。
 */

static inline void
s_check_valid(int lockndx)
{
    /*
     * 如果TAS被禁用，使用信号量模拟自旋锁时，都是使用slock_t *lock来定义一个SpinLock的。
     * slock_t就是一个int值，然后初始化这个Spinlock后变量lock的值其实就是模拟spinlock
     * 信号量在共享内存数组的下标来表示的。此函数检查是否是一个有效的spinlock号，即是否是一
     * 个有效的共享内存信号量数组下标。
     */
	if (unlikely(lockndx <= 0 || lockndx > NUM_EMULATION_SEMAPHORES))
		elog(ERROR, "invalid spinlock number: %d", lockndx);
}

/* 
 * 初始化一个spinlock，返回此spinlock使用的信号量在共享内存的下标。
 * #define SpinLockInit(lock)	S_INIT_LOCK(lock)
 * #define S_INIT_LOCK(lock)	s_init_lock_sema(lock, false)
 * SpinLockInit是上层接口，我们在访问共享内存等需要spinlock的时候是调用它来初始化一个
 * 自旋锁的，在下层会根据spinlock的不同实现方式调用对应的初始化例程，同理，请求和释放spinlock
 * 也一样。比如 S_INIT_LOCK 的另一种实现方式，如下：
 * #define S_INIT_LOCK(lock) \
	do { \
		volatile slock_t *lock_ = (lock); \
		lock_->sema[0] = -1; \
		lock_->sema[1] = -1; \
		lock_->sema[2] = -1; \
		lock_->sema[3] = -1; \
	} while (0)
 */
void
s_init_lock_sema(volatile slock_t *lock, bool nested)
{
	static uint32 counter = 0;
	uint32		offset;
	uint32		sema_total;
	uint32		idx;

	if (nested)
	{
		/*
		 * To allow nesting atomics inside spinlocked sections, use a
		 * different spinlock. See comment above.
		 */
		offset = 1 + NUM_SPINLOCK_SEMAPHORES;
		sema_total = NUM_ATOMICS_SEMAPHORES;
	}
	else
	{
		offset = 1;
		sema_total = NUM_SPINLOCK_SEMAPHORES;
	}

    // 本次spinlock初始化使用的信号量在共享内存中的下标值，即spinlock的锁标识值。
	idx = (counter++ % sema_total) + offset;

	/* double check we did things correctly */
	s_check_valid(idx);

    // 成功，返回spinlock的锁标识
	*lock = idx;
}

/* 对应上层接口 SpinLockRelease */
void
s_unlock_sema(volatile slock_t *lock)
{
	int			lockndx = *lock;

	s_check_valid(lockndx);
	// 解锁信号量（递增计数）
	PGSemaphoreUnlock(SpinlockSemaArray[lockndx - 1]);
}

/* 对应上层接口 SpinLockFree，信号量事项的spinlock当前不会使用这个接口，暂时用不上 */
bool
s_lock_free_sema(volatile slock_t *lock)
{
	/* We don't currently use S_LOCK_FREE anyway */
	elog(ERROR, "spin.c does not support S_LOCK_FREE()");
	return false;
}

int
tas_sema(volatile slock_t *lock)
{
	int			lockndx = *lock;

	s_check_valid(lockndx);

    // 锁定信号量（递减计数）
	/* Note that TAS macros return 0 if *success* */
	return !PGSemaphoreTryLock(SpinlockSemaArray[lockndx - 1]);
}

#endif							/* !HAVE_SPINLOCKS */

```

`spin.h`中定义要使用自旋锁spinlock需要的数据结构和操作接口：

- **SpinLock 数据结构，用于定义一个spinlock锁变量**

  - slock_t *lock;	// 定义个一个自旋锁变量 lock.

- **SpinLock 操作接口（宏定义）**

  - void SpinLockInit(volatile **slock_t *lock**)
    - 在使用一个`slock_t *lock`定义的自旋锁前，先初始化lock表示的spinlock锁（lock是共享内存中信号量数组的下标值）。
  - void SpinLockAcquire(volatile **slock_t *lock**)
    - 获取自旋锁（即加锁请求），必要时等待（如果现在无法加锁）。如果无法在“合理”时间内获取锁，则超时并中止——通常约为 1 分钟。所以，spinlock是支持timeout的，不会一直盲等。
  - void SpinLockRelease(volatile **slock_t *lock)**
    - 释放锁
  - bool SpinLockFree(**slock_t *lock**)
    - 测试锁是否空闲（是否可加锁），如果空闲则返回true，如果已锁定则返回 false，此操作不会更改锁的状态。

  **可以看到，所有spinlock的操作接口的参数都是 slock_t 类型的变量**，其实这就是一个int值。另外注意，上面的四个spinlock操作接口是数据库内部操作自旋锁所使用的接口，**不管是信号量还是TAS方式实现的自旋锁，在内核代码中都使用上面四个接口。**

  另外，如spin.h的注释中说的，自旋锁要记住一个规则：自旋锁不能持有太长时间，不要超过几个指令，同时不要在持有自旋锁的临界区中执行CHECK_FOR_INTERRUPTS() ，也就是说自旋锁支持有时不允许接收中断。

```c
/*-------------------------------------------------------------------------
 *
 * spin.h
 *	   Hardware-independent implementation of spinlocks.
 *
 *
 *	The hardware-independent interface to spinlocks is defined by the
 *	typedef "slock_t" and these macros:
 *
 *	void SpinLockInit(volatile slock_t *lock)
 *		Initialize a spinlock (to the unlocked state).
 *
 *	void SpinLockAcquire(volatile slock_t *lock)
 *		Acquire a spinlock, waiting if necessary.
 *		Time out and abort() if unable to acquire the lock in a
 *		"reasonable" amount of time --- typically ~ 1 minute.
 *
 *	void SpinLockRelease(volatile slock_t *lock)
 *		Unlock a previously acquired lock.
 *
 *	bool SpinLockFree(slock_t *lock)
 *		Tests if the lock is free. Returns true if free, false if locked.
 *		This does *not* change the state of the lock.
 *
 *	Callers must beware that the macro argument may be evaluated multiple
 *	times!
 *
 * 调用代码中的加载和存储操作保证不会针对这些操作重新排序，因为它们包含编译器屏障。 
 * （在 PostgreSQL 9.5 之前，调用者需要使用 volatile 限定符来访问受自旋锁保护的数据。）
 *
 *	Load and store operations in calling code are guaranteed not to be
 *	reordered with respect to these operations, because they include a
 *	compiler barrier.  (Before PostgreSQL 9.5, callers needed to use a
 *	volatile qualifier to access data protected by spinlocks.)
 *
 * 请记住，自旋锁不能保持多于几条指令的编码规则。特别是，我们假设在持有自旋锁时不可能
 * 发出 CHECK_FOR_INTERRUPTS()，因此没有必要在这些宏中执行 HOLD/RESUME_INTERRUPTS()。
 *
 *	Keep in mind the coding rule that spinlocks must not be held for more
 *	than a few instructions.  In particular, we assume it is not possible
 *	for a CHECK_FOR_INTERRUPTS() to occur while holding a spinlock, and so
 *	it is not necessary to do HOLD/RESUME_INTERRUPTS() in these macros.
 *
 * 这些宏是根据 s_lock.h 提供的与硬件相关的宏来实现的。此头文件当前没有添加任何额外的功能，
 * 但过去已经存在，并且可能有一天会再次出现。
 *	These macros are implemented in terms of hardware-dependent macros
 *	supplied by s_lock.h.  There is not currently any extra functionality
 *	added by this header, but there has been in the past and may someday
 *	be again.
 *
 *
 * Portions Copyright (c) 1996-2021, PostgreSQL Global Development Group
 * Portions Copyright (c) 1994, Regents of the University of California
 *
 * src/include/storage/spin.h
 *
 *-------------------------------------------------------------------------
 */
#ifndef SPIN_H
#define SPIN_H

#include "storage/s_lock.h"
#ifndef HAVE_SPINLOCKS
#include "storage/pg_sema.h"
#endif

/* 
 * spinlock的初始化、加锁、释放锁通用接口。
 * 信号量和TAS无论哪种实现方式都使用下面这四个接口。
 */
#define SpinLockInit(lock)	S_INIT_LOCK(lock)
#define SpinLockAcquire(lock) S_LOCK(lock)
#define SpinLockRelease(lock) S_UNLOCK(lock)
#define SpinLockFree(lock)	S_LOCK_FREE(lock)

// 共享内存在创建时，用下面接口来获取信息从而为自旋锁的信号量申请空间和初始化、
extern int	SpinlockSemas(void);
extern Size SpinlockSemaSize(void);

#ifndef HAVE_SPINLOCKS
extern void SpinlockSemaInit(void);
extern PGSemaphore *SpinlockSemaArray;
#endif

#endif							/* SPIN_H */
```

## TAS指令实现的SpinLock（硬件相关方式）

​	目前，主流的多处理器架构都在硬件水平上提供了对并发同步的支持，因此，我们大部分情况都是使用硬件同步指令TAS实现的spinlock。一个TAS(Test-and-Set)指令包括两个子步骤，**把给定的内存地址设置为1，然后返回之前的旧值。**这两个子步骤在硬件上实现为一个**原子操作**，执行期间不会被其他处理器打断。

​	**值得注意的是，TAS指令是在1个比特位(bit)上实现的，这限制了变量非0即1，不会有其他值，并且TAS总是把变量设置为1。**可见，TAS生而为自旋锁，下面是使用TAS实现自旋锁的伪代码示例：

```c
lock = 0	//shared state
while(test_and_set(lock) == 0)	//test_and_set返回的是内存地址的旧值，如果是0说明可以获得锁。
{
    //test_and_set使用执行的都是将内存地址设置为1，并返回内存地址的旧值，所以只要调用了tas内存地址
    //的值都会是1，返回1，则表示不能获得锁，继续尝试。获取锁的线程，在操作完临界区后会将内存地址设为0
	//try lock and exec tas again。
    //do nothing
}
//临界区代码
lock = 0	//release
```

当第一个线程执行这段代码时，TAS指令会立即把lock设置为1，并返回0 ，线程退出while循环进入临界区（开始操作临界区代码）。如果另一个线程尝试进入临界区，TAS会把lock设置为1，但是也会返回1(由第一个线程的TAS指令设置为1了已经)，此时第二个线程会一直while循环(忙等待)，直到第一个线程退出临界区代码，执行了lock=0，即释放了锁。这种通过while-loop等待获取锁的实现称为自旋锁(spin lock)。

`s_lock.h`中定义个一些宏，用来实现不同平台的spinlock相关操作，但是，注意：此文件中的任何宏都不打算被直接调用。而是通过 spin.h 中与硬件无关的宏调用它们，前面也说了spinlock调用接口是被定义在spin.h中的，如下的加粗部分：

#define **SpinLockInit(lock)**			S_INIT_LOCK(lock)
#define **SpinLockAcquire(lock)** 	S_LOCK(lock)
#define **SpinLockRelease(lock)** 	S_UNLOCK(lock)
#define **SpinLockFree(lock)**			S_LOCK_FREE(lock)

而`s_lock.h`中定义宏都是上述调用接口比如`SpinLockAcquire`宏的实现部分，也是一个宏：

#define SpinLockInit(lock)			**S_INIT_LOCK(lock)**
#define SpinLockAcquire(lock) 	**S_LOCK(lock)**
#define SpinLockRelease(lock) 	 **S_UNLOCK(lock)**
#define SpinLockFree(lock)			**S_LOCK_FREE(lock)**

在`s_lock.h`中，每个受支持的平台都提供以下与硬件相关的宏，但也依据平台不同，spinlock锁的数据结构**slock_t**和**S_INIT_LOCK、S_LOCK、S_UNLOCK、S_LOCK_FREE、TAS**接口实现不同。

- Spinlock数据结构slock_t，根据平台不同，slock_t也有不同的定义，比如

  - typedef unsigned char slock_t;	/* AMD Opteron, Intel EM64T */
  - typedef int slock_t;                       /* ARM and ARM64 */
  - ....等

- SpinLock操作宏，根据平台不同，宏的定义不同，宏名字都是一份，

  - void S_INIT_LOCK(slock_t *lock)
    - 初始化一个spinlock，且为未加锁状态。
  - int S_LOCK(slock_t *lock)
    - 获取自旋锁（即加锁请求），必要时等待（如果现在无法加锁）。如果无法在“合理”时间内获取锁，则超时并中止——通常约为 1 分钟。所以，spinlock是支持timeout的，不会一直盲等。
  - void S_UNLOCK(slock_t *lock)
    - 释放锁
  - bool S_LOCK_FREE(slock_t *lock)
    - 测试锁是否空闲（是否可加锁），如果空闲则返回true，如果已锁定则返回 false，此操作不会更改锁的状态。
  - void SPIN_DELAY(void)
    - 在自旋锁等待循环内发出延迟操作。

  通常，S_LOCK() 加锁操作是根据更低级别的宏 TAS() 和 TAS_SPIN() 来实现的：

  - int TAS(slock_t *lock)
    - 原子的test-and-set指令。尝试获取锁，但是不会等待，如果成功则返回 0，如果无法获取锁则返回非零。
  - int TAS_SPIN(slock_t *lock)
    - 与 TAS() 类似，但此版本用于等待先前发现争用的锁。默认情况下，这与 TAS() 相同，但在某些体系结构上，最好使用解锁指令轮询一个被竞争的锁，并仅在它看起来空闲时重试原子的test-and-set指令。

  `TAS()` 和 `TAS_SPIN()` 不是 API 的一部分，不应直接调用。

注意：在某些平台上，即使锁未锁定，TAS() 和/或 TAS_SPIN() 有时也可能会报告获取锁失败。例如，在Alpha TAS() 上，如果被中断，将“失败”。因此，即使你确定锁是空闲的，也必须始终使用重试循环。

这些宏的职责是确保编译器不会在实际锁获取之前或在锁释放之后重新排序对共享内存的访问。 在 PostgreSQL 9.5 之前，这是调用者的责任，这意味着调用者必须使用 volatile 限定的指针来引用自旋锁本身和自旋锁临界区中访问的共享数据。这在符号上很尴尬，很容易忘记（因此容易出错），并且阻止了一些有用的编译器优化。由于这些原因，我们现在要求宏本身可以防止编译器重新排序，以便调用者不需要采取特殊的预防措施。

在内存排序较弱的平台上，TAS()、TAS_SPIN() 和 S_UNLOCK() 宏必须进一步包含硬件级别的内存栅栏指令，以防止在硬件级别进行类似的重新排序。TAS() 和 TAS_SPIN() 必须保证在获得锁之前不会执行宏之后发出的加载和存储。相反，S_UNLOCK() 必须保证在释放锁之前执行宏之前发出的加载和存储。

在大多数受支持的平台上，TAS() 使用以汇编语言编写的 tas() 函数来执行硬件原子TAS指令 也可以使用等效的操作系统提供的互斥例程。

如果没有系统特定的 TAS() 可用（即，未定义 HAVE_SPINLOCKS），那么我们将求助于使用 SysV 信号量的仿真spinlock（参见 spin.c）。由于每次锁定或解锁的内核调用成本，此仿真将比正确的 TAS() 实现慢得多。 一份旧的报告是，在使用 SysV 信号量代码时，Postgres 大约 40% 的时间都花在了 semop(2) 上。

TAS操作使用的是内嵌汇编代码，下面x86_64为例的代码片段：

```C
#ifdef __x86_64__		/* AMD Opteron, Intel EM64T */
#define HAS_TEST_AND_SET

typedef unsigned char slock_t;	// spinlock变量的数据结构

#define TAS(lock) tas(lock)		//tas()是内嵌汇编实现

/*
 * On Intel EM64T, it's a win to use a non-locking test before the xchg proper,
 * but only when spinning.
 *
 * See also Implementing Scalable Atomic Locks for Multi-Core Intel(tm) EM64T
 * and IA32, by Michael Chynoweth and Mary R. Lee. As of this writing, it is
 * available at:
 * http://software.intel.com/en-us/articles/implementing-scalable-atomic-locks-for-multi-core-intel-em64t-and-ia32-architectures
 */
#define TAS_SPIN(lock)    (*(lock) ? 1 : TAS(lock))

static __inline__ int
tas(volatile slock_t *lock)
{
	register slock_t _res = 1;

	__asm__ __volatile__(
		"	lock			\n"
		"	xchgb	%0,%1	\n"
:		"+q"(_res), "+m"(*lock)
:		/* no inputs */
:		"memory", "cc");
	return (int) _res;
}

#define SPIN_DELAY() spin_delay()

static __inline__ void
spin_delay(void)
{
	/*
	 * Adding a PAUSE in the spin delay loop is demonstrably a no-op on
	 * Opteron, but it may be of some use on EM64T, so we keep it.
	 */
	__asm__ __volatile__(
		" rep; nop			\n");
}

#endif	 /* __x86_64__ */
```

至此，spinlock的代码基本也就分析完了，嗯... 我主要是对PG内部的spinlock有一定的基本了解，在平时的开发过程中一般也不会涉及到修改这部分的内容。通过分析，我们主要的目的是知道如何使用spinlock，以及一些注意事项，**比如不要持有spinlock太久，不要在spinlock持有时发出中断等**。

## Spinlock在PostgreSQL中的使用示例

spinlock在PostgreSQL中的作用主要是保护临界区（共享内存）的并发访问，并且是短暂的一个锁定。所以一般用于保护多个进程并发访问（读和写）位于共享内存中的全局变量的某些属性（结构体的成员），比如在共享内存中有一个checkpoint操作的全局数据结构CheckpointerShmem，如果我们想通过spinlock来保护这个结构体的某些成员变量的话，**首先必须在这个结构体中定义一个slock_t类型的成员变量**，这就是spinlock的锁变量，一个标记位。然后在访问要保护的结构体成员时必须先通过slock_t成员获取spinlock，再去访问这些结构体成员的值，完成后立即释放spinlock。

在CheckpointerShmemStruct中只有一个`slock_t`类型的锁变量`ckpt_lck`，理论是我们可以定义多个slock_t的锁变量，不同的锁变量保护不同的结构体成员。

```c
typedef struct
{
	pid_t		checkpointer_pid;	/* PID (0 if not started) */

    // 定义spinlock锁
	slock_t		ckpt_lck;		/* protects all the ckpt_* fields */

	int			ckpt_started;	/* advances when checkpoint starts */
	int			ckpt_done;		/* advances when checkpoint done */
	int			ckpt_failed;	/* advances when checkpoint fails */

	int			ckpt_flags;		/* checkpoint flags, as defined in xlog.h */

	ConditionVariable start_cv; /* signaled when ckpt_started advances */
	ConditionVariable done_cv;	/* signaled when ckpt_done advances */

	uint32		num_backend_writes; /* counts user backend buffer writes */
	uint32		num_backend_fsync;	/* counts user backend fsync calls */

	int			num_requests;	/* current # of requests */
	int			max_requests;	/* allocated array size */
	CheckpointerRequest requests[FLEXIBLE_ARRAY_MEMBER];
} CheckpointerShmemStruct;

static CheckpointerShmemStruct *CheckpointerShmem;
```

从注释中可以看到`ckpt_lck`这个spinlock锁用来保护所有对`ckpt_`开头的成员变量，那么，所有对共享内存中的全局变量CheckpointerShmem的以ckpt_开头的成员变量访问（读和写）都需要先使用`ckpt_lck`获取自旋锁，然后再去访问成员的值，最后释放锁。

加锁和释放锁的代码片段如下：

```c
void
RequestCheckpoint(int flags)
{
	int			ntries;
	int			old_failed,
				old_started;
   // ...... 其他代码，略
    
   // 修改共享内存数据前，先加锁
	SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    
	// 虽然是读操作，也必须要获取锁，因为不加锁的话，共享内存中的数据可能正在被其他进程修改。
	old_failed = CheckpointerShmem->ckpt_failed;
	old_started = CheckpointerShmem->ckpt_started;
	CheckpointerShmem->ckpt_flags |= (flags | CHECKPOINT_REQUESTED);
    
	// 修改完，释放锁
	SpinLockRelease(&CheckpointerShmem->ckpt_lck);

	if (flags & CHECKPOINT_WAIT)
	{
		int			new_started,
					new_failed;

		/* Wait for a new checkpoint to start. */
		ConditionVariablePrepareToSleep(&CheckpointerShmem->start_cv);
		for (;;)
		{
            // 加锁
			SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
            // 读取临界区数据
			new_started = CheckpointerShmem->ckpt_started;
            // 释放锁
			SpinLockRelease(&CheckpointerShmem->ckpt_lck);

			if (new_started != old_started)
				break;

			ConditionVariableSleep(&CheckpointerShmem->start_cv,
								   WAIT_EVENT_CHECKPOINT_START);
		}
		ConditionVariableCancelSleep();

		/*
		 * We are waiting for ckpt_done >= new_started, in a modulo sense.
		 */
		ConditionVariablePrepareToSleep(&CheckpointerShmem->done_cv);
		for (;;)
		{
			int			new_done;
			// 加锁
			SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
            // 读取数据
			new_done = CheckpointerShmem->ckpt_done;
			new_failed = CheckpointerShmem->ckpt_failed;
            // 释放锁
			SpinLockRelease(&CheckpointerShmem->ckpt_lck);

			if (new_done - new_started >= 0)
				break;

			ConditionVariableSleep(&CheckpointerShmem->done_cv,
								   WAIT_EVENT_CHECKPOINT_DONE);
		}

        ......
}

```

**知识点1：要保护的结构体中必须定义一个或多个 `slock_t` 类型的锁变量。**

**知识点2：注意，是访问结构体的成员，包括读和写都需要加锁和解锁，否则这边的读可能被其他正在写的进程影响，导致数据不一致的异常。因此即使是读共享内存中要保护的数据也要加`spinlock`。**

**知识点3：本地进程在访问（读和写）位于临界区（共享内存）中的数据时刻，进行加锁和解锁请求。**

**知识点4：SpinLock没有死锁检测机制；没有等待队列所以会有锁的竞争；持有spinlock锁期间执行elog()错误不会自动释放，所以即使spinloc有超时机制也不要在锁定期间执行elog()，spinlock有超时机制；不要在锁定期间执行CHECK_FOR_INTERRUPTS()。**



> **参考：**

> [硬件对同步的支持-TAS和CAS指令](https://copyfuture.com/blogs-details/20200626092652301jhqtdjd9vg8tp55)
>
> [【知乎】PostgreSQL中的SpinLock](https://zhuanlan.zhihu.com/p/72543802)
>
> [【CSDN】PostgreSQL中的锁--spinLock、LWLock、Lock](https://blog.csdn.net/qq_43687755/article/details/106478057)
>
> http://c.biancheng.net/view/1232.html