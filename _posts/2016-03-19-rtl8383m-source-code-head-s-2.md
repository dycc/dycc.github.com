---
title: 'RTL8382M Linux源码阅读思路 - head.s（2）'
layout: post
guid: urn:uuid:2016-03-19-rtl8383m-source-code-head-s-2
tags:
    - 内核分析
---

	PTR_LA		$28, init_thread_union
	PTR_LI		sp, _THREAD_SIZE - 32
	PTR_ADDU	sp, $28
	set_saved_sp	sp, t0, t1
	PTR_SUBU	sp, 4 * SZREG		# init stack pointer
	j		start_kernel
	
第1行：将`init_thread_union`结构地址赋值给`$28`，`$28 = $gp`在mips中表示全局数据区指针寄存器

`init_thread_union`结构体实例定义在：

	/*
	 * Initial thread structure.
	 *
	 * We need to make sure that this is 8192-byte aligned due to the
	 * way process stacks are handled. This is done by making sure
	 * the linker maps this in the .text segment right after head.S,
	 * and making head.S ensure the proper alignment.
	 *
	 * The things we do for performance..
	 */
	union thread_union init_thread_union
		__attribute__((__section__(".data.init_task"),
					   __aligned__(THREAD_SIZE))) =
			{ INIT_THREAD_INFO(init_task) };
	/*
	 * Initial task structure.
	 *
	 * All other task structs will be allocated on slabs in fork.c
	 */
	struct task_struct init_task = INIT_TASK(init_task);
	
以上代码翻译成人话意思是：定义一个`thread_union`类型的共用体`init_thread_union`并赋值，具体解释如下：

首先看此共用体类型的声明：

	union thread_union {
		struct thread_info thread_info;
		unsigned long stack[THREAD_SIZE/sizeof(long)];
	};
	
继续thread_info结构体声明：

	/*
	 * low level task data that entry.S needs immediate access to
	 * - this struct should fit entirely inside of one cache line
	 * - this struct shares the supervisor stack pages
	 * - if the contents of this structure are changed, the assembly constants
	 *   must also be changed
	 */
	struct thread_info {
		struct task_struct	*task;		/* main task structure */
		struct exec_domain	*exec_domain;	/* execution domain */
		unsigned long		flags;		/* low level flags */
		unsigned long		tp_value;	/* thread pointer */
		__u32			cpu;		/* current CPU */
		int			preempt_count;	/* 0 => preemptable, <0 => BUG */
		mm_segment_t		addr_limit;	/* thread address space:
							   0-0xBFFFFFFF for user-thead
							   0-0xFFFFFFFF for kernel-thread
							*/
		struct restart_block	restart_block;
		struct pt_regs		*regs;
	};
	
具体每个字段什么意思我也不知道@@

继续……

`thread_union`共用体变量类型实际上是一个共用的内存结构，根据除`thread_info`的另一个数组stack可以猜出来，此共用结构可以同时作为`thread_info`和栈数组使用（好像是废话……）

上面解释完了`thread_union`共用体类型的字段，因此`init_thread_union`就是一个`thread_union`的实例化

`__attribute__((__section__(".data.init_task"),`表示此共用体结构必须被编译在内核空间的`.data.init_task`段，具体在哪里有`vmlinux.lds`定义

`__aligned__(THREAD_SIZE))) =`表示此结构对齐到`THREAD_SIZE`，就是`0x2000`，`8k byte`
因此，thread_info和栈数组共用8k

`{ INIT_THREAD_INFO(init_task) };`是此实例在`.data.init_task`的赋值，在被定义的时候赋值，赋值的地址是`.data.init_task`
查看`system.map`得知`801d2000 D init_thread_union`

`struct task_struct init_task = INIT_TASK(init_task);`表示在`init_thread_union`后面8k开外定义一个`init_task`结构实例并初始化赋值

翻译过来就是说，前一个定义在内核数据段的开头定义了个一个线程结构与内核堆栈的共用体共8k，这8k后面紧接着定义个一个进程（任务）结构

因此，第1行的意思就是将内核数据段开头的内核堆栈区的底部（也就是开头）指针赋值给了`$28`全局（数据段）指针

第2行：将内核数据段开头8k刚好是一个`init_thread_union`结构的，也可以说是内核堆栈区-32byte的大小，赋值给$sp，也就是mips保留的（内核）堆栈区大小

第3行：将`$sp`堆栈指针（之前保存堆栈区大小）加上`$gp`内核数据段开头地址，因此，`$sp`刚好指向`init_thread_union`结构尾巴，也就是内核堆栈区顶部

第4行：保存这两个指针

第5行：`PTR_SUBU    sp, 4 * SZREG        # init stack pointer`

将`$sp`内核堆栈区减`4*4个byte`，原因不清楚，猜测可能是为了保存`start_kernel`的参数

第7行：`j        start_kernel`

j指令将跳转到`start_kernel`地址