---
title: 'Linux内核线程与临界区内的线程切换错误'
layout: post
guid: urn:uuid:2016-09-12-kernel-thread-and-lock
tags:
    - 内核分析
---

<iframe style="float:right" frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="http://music.163.com/outchain/player?type=2&id=480916&auto=1&height=66"></iframe>

此处讨论的内核版本是linux-2.6.19 for mips-32。

此内核为分时系统，调度时间为1s。

### 1. 内核线程 exp1

	uint32 index = 0;

	int test_thread_A(void *data)
	{
		while(1)
		{
			printk("A: %u\n", index);
			index++;
		}
		
		return 1;
	}

	int test_thread_B(void *data)
	{
		while(1)
		{
			printk("B: %u\n", index);
			index++;
		}
		
		return 1;
	}

	void Create_My_Test_Thread(void)
	{
		struct task_struct *test_task_A, *test_task_B;

		test_task_A = kthread_create(test_thread_A, NULL, "test_task");

		if(IS_ERR(test_task_A))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_A = NULL;
		  return;
		}

		test_task_B = kthread_create(test_thread_B, NULL, "test_task");

		if(IS_ERR(test_task_B))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_B = NULL;
		  return;
		}
		
		wake_up_process(test_task_A);
		wake_up_process(test_task_B);
		
		return;
	}

结果：

	......
	A:0
	A:1
	......
	B:1578
	B:1579
	......
	A:3679
	A:3680
	......
	
两个内核线程定时被调度，一切正常。
	
### 2. 内核线程 exp2

	uint32 index = 0;

	int test_thread_A(void *data)
	{
		uint32 idx;
		
		while(1)
		{
			for(idx = 0; idx < 5; idx++)
			{
				printk("A-%u: %u\n", idx, index);
				index++;
				sleep(1);
			}
		}
		
		return 1;
	}

	int test_thread_B(void *data)
	{
		uint32 idx;
		
		while(1)
		{
			for(idx = 0; idx < 5; idx++)
			{
				printk("B-%u: %u\n", idx, index);
				index++;
				sleep(1);
			}
		}
		
		return 1;
	}

	void Create_My_Test_Thread(void)
	{
		struct task_struct *test_task_A,*test_task_B;

		test_task_A = kthread_create(test_thread_A, NULL, "test_task");

		if(IS_ERR(test_task_A))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_A = NULL;
		  return;
		}

		test_task_B = kthread_create(test_thread_B, NULL, "test_task");

		if(IS_ERR(test_task_B))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_B = NULL;
		  return;
		}
		
		wake_up_process(test_task_A);
		wake_up_process(test_task_B);
		
		return;
	}

结果：

	......
	A-0: 3343
	B-0: 3344
	......
	A-2: 3345
	B-2: 3346
	......
	A-4: 3347
	B-4: 3348
	......
	
现在加入sleep函数,由于sleep会导致内核线程睡眠，因此线程会主动schedule。

### 3. 内核线程 exp3

现在我希望在每个线程执行for循环的时候不允许被打断，因此我需要给这段加锁，以防止线程被切换。

我选择的是`spinlock`，锁被占用时忙等，并不会切换，因此在临界区的代码不会被打断。

也就是说临界区代码是`原子`操作。

	#include <linux/spinlock.h>
	spinlock_t spinlock = SPIN_LOCK_UNLOCKED;
	spin_lock_init(&spinlock);

	uint32 index = 0;

	int test_thread_A(void *data)
	{
		uint32 idx;
		
		while(1)
		{
			spin_lock(spinlock);
			
			for(idx = 0; idx < 5; idx++)
			{
				printk("A-%u: %u\n", idx, index);
				index++;
				osal_time_mdelay(1000);
			}

			spin_unlock(spinlock);
		}
		
		return 1;
	}

	int test_thread_B(void *data)
	{
		uint32 idx;
		
		while(1)
		{
			spin_lock(spinlock);
			
			for(idx = 0; idx < 5; idx++)
			{
				printk("B-%u: %u\n", idx, index);
				index++;
				osal_time_mdelay(1000);
			}

			spin_unlock(spinlock);
		}
		
		return 1;
	}

	void Create_My_Test_Thread(void)
	{
		struct task_struct *test_task_A,*test_task_B;

		test_task_A = kthread_create(test_thread_A, NULL, "test_task");

		if(IS_ERR(test_task_A))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_A = NULL;
		  return;
		}

		test_task_B = kthread_create(test_thread_B, NULL, "test_task");

		if(IS_ERR(test_task_B))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_B = NULL;
		  return;
		}
		
		wake_up_process(test_task_A);
		wake_up_process(test_task_B);
		
		return;
	}
	
结果：

	......
	A-0: 210
	BUG: scheduling while atomic: test_task/0x00000001/78
	Call Trace:
	[<80009038>] dump_stack+0x8/0x34
	[<801fd238>] schedule+0x6c/0xc08
	[<801fefc0>] schedule_timeout+0xa8/0xe0
	[<801fe830>] interruptible_sleep_on_timeout+0xa8/0x15c
	[<c00ce524>] osal_time_sleep+0x50/0x9c [rtcore]
	[<c00b4334>] test_thread_A+0x64/0xa8 [custom_poe]
	[<800434bc>] kthread+0x1c4/0x210
	[<800042b8>] kernel_thread_helper+0x10/0x18

	B-0: 211
	BUG: scheduling while atomic: test_task/0x00000001/79
	Call Trace:
	[<80009038>] dump_stack+0x8/0x34
	[<801fd238>] schedule+0x6c/0xc08
	[<801fefc0>] schedule_timeout+0xa8/0xe0
	[<801fe830>] interruptible_sleep_on_timeout+0xa8/0x15c
	[<c00ce524>] osal_time_sleep+0x50/0x9c [rtcore]
	[<c00b43dc>] test_thread_B+0x64/0xa8 [custom_poe]
	[<800434bc>] kthread+0x1c4/0x210
	[<800042b8>] kernel_thread_helper+0x10/0x18

	A-1: 212
	BUG: scheduling while atomic: test_task/0x00000001/78
	Call Trace:
	[<80009038>] dump_stack+0x8/0x34
	[<801fd238>] schedule+0x6c/0xc08
	[<801fefc0>] schedule_timeout+0xa8/0xe0
	[<801fe830>] interruptible_sleep_on_timeout+0xa8/0x15c
	[<c00ce524>] osal_time_sleep+0x50/0x9c [rtcore]
	[<c00b4334>] test_thread_A+0x64/0xa8 [custom_poe]
	[<800434bc>] kthread+0x1c4/0x210
	[<800042b8>] kernel_thread_helper+0x10/0x18

	B-1: 213
	BUG: scheduling while atomic: test_task/0x00000001/79
	Call Trace:
	[<80009038>] dump_stack+0x8/0x34
	[<801fd238>] schedule+0x6c/0xc08
	[<801fefc0>] schedule_timeout+0xa8/0xe0
	[<801fe830>] interruptible_sleep_on_timeout+0xa8/0x15c
	[<c00ce524>] osal_time_sleep+0x50/0x9c [rtcore]
	[<c00b43dc>] test_thread_B+0x64/0xa8 [custom_poe]
	[<800434bc>] kthread+0x1c4/0x210
	[<800042b8>] kernel_thread_helper+0x10/0x18

	A-2: 214
	BUG: scheduling while atomic: test_task/0x00000001/78
	Call Trace:
	[<80009038>] dump_stack+0x8/0x34
	[<801fd238>] schedule+0x6c/0xc08
	[<801fefc0>] schedule_timeout+0xa8/0xe0
	[<801fe830>] interruptible_sleep_on_timeout+0xa8/0x15c
	[<c00ce524>] osal_time_sleep+0x50/0x9c [rtcore]
	[<c00b4334>] test_thread_A+0x64/0xa8 [custom_poe]
	[<800434bc>] kthread+0x1c4/0x210
	[<800042b8>] kernel_thread_helper+0x10/0x18

	B-2: 215
	BUG: scheduling while atomic: test_task/0x00000001/79
	Call Trace:
	[<80009038>] dump_stack+0x8/0x34
	[<801fd238>] schedule+0x6c/0xc08
	[<801fefc0>] schedule_timeout+0xa8/0xe0
	[<801fe830>] interruptible_sleep_on_timeout+0xa8/0x15c
	[<c00ce524>] osal_time_sleep+0x50/0x9c [rtcore]
	[<c00b43dc>] test_thread_B+0x64/0xa8 [custom_poe]
	[<800434bc>] kthread+0x1c4/0x210
	[<800042b8>] kernel_thread_helper+0x10/0x18
	......

以下转载至[BUG: scheduling while atomic 分析](http://blog.csdn.net/cfy_phonex/article/details/12090943)

> _linux内核打印"BUG: scheduling while atomic"和"bad: scheduling from the idle thread"错误的时候，通常是在`中断处理函数`中调用了可以休眠的函数，如semaphore,mutex,sleep之类的可休眠的函数。_

> _而linux内核要求在中断处理的时候，不允许系统调度，不允许抢占，要等到中断处理完成才能做其他事情。因此，要充分考虑中断处理的时间，一定不能太久。_

> _另外一个能产生此问题的是在idle进程里面，做了不该做的事情。现在Linux用于很多手持式设备，为了降低功耗，通常的作法是在idle进程里面降低CPU或RAM的频率、关闭一些设备等等。要保证这些动作的原子性才能确保不发生"bad: scheduling from the idle thread"这样的错误！_

> _禁止内核抢占是指内核不会主动的抢占你的process，但是现在是你在自己的程序中主动call schedule()，kernel并不能阻止你这么作。_

所谓`中断处理函数`其实就相当于此时的`临界区`，都不允许线程切换，不管是内核操作还是自愿，只要有schedule内核就会报错，sleep函数恰好主动call schedule()，导致错误。

### 4. 内核线程 exp4

	#include <linux/spinlock.h>
	spinlock_t spinlock = SPIN_LOCK_UNLOCKED;
	spin_lock_init(&spinlock);

	uint32 index = 0;

	int test_thread_A(void *data)
	{
		uint32 idx;
		
		while(1)
		{
			spin_lock(spinlock);
			
			for(idx = 0; idx < 5; idx++)
			{
				printk("A-%u: %u\n", idx, index);
				index++;
				osal_time_mdelay(1000);
			}

			spin_unlock(spinlock);
		}
		
		return 1;
	}

	int test_thread_B(void *data)
	{
		uint32 idx;
		
		while(1)
		{
			spin_lock(spinlock);
			
			for(idx = 0; idx < 5; idx++)
			{
				printk("B-%u: %u\n", idx, index);
				index++;
				osal_time_mdelay(1000);
			}

			spin_unlock(spinlock);
		}
		
		return 1;
	}

	void Create_My_Test_Thread(void)
	{
		struct task_struct *test_task_A,*test_task_B;

		test_task_A = kthread_create(test_thread_A, NULL, "test_task");

		if(IS_ERR(test_task_A))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_A = NULL;
		  return;
		}

		test_task_B = kthread_create(test_thread_B, NULL, "test_task");

		if(IS_ERR(test_task_B))
		{ 
		  printk("Unable to start kernel thread.\n");
		  test_task_B = NULL;
		  return;
		}
		
		wake_up_process(test_task_A);
		wake_up_process(test_task_B);
		
		return;
	}

结果
	
	......
	A-0: 3980
	A-1: 3981
	A-2: 3982
	A-3: 3983
	A-4: 3984
	B-0: 3985
	B-1: 3986
	B-2: 3987
	B-3: 3988
	B-4: 3989
	A-0: 3990
	A-1: 3991
	A-2: 3992
	A-3: 3993
	A-4: 3994
	......
	
临界区没有内核或自愿的线程切换，执行正确。
