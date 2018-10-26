---
title: 'RTL8382M Linux源码阅读思路 - head.s（1）'
layout: post
guid: urn:uuid:2016-03-19-rtl8383m-source-code-head-s-1
tags:
    - 内核分析
---

从`kernel_entry`开始，看`kernel_entry_setup`，`cpu init`

设定`C0_ErrCtl`寄存器，开始找datasheet有没有关于`mips CP0`寄存器的资料

从8382M上找到了关于内部非扩展GPIO的使用方法，通过寄存器设定，但是关于CP0寄存器的资料未找到

继续看`kernel_entry_setup`，既然没找到，跳过，只知道这段是设定CP0寄存器的bit28，关掉某个功能

继续看`setup_c0_status_pri`，搜索一下，就在heard.s 上面

此函数里有一个指令`setup_c0_status`，是专门用来设定CP0状态寄存器$12的

此指令的注释：暂时禁止中断，标志着内核模式和设置`ST0_KX`因此使用64位地址时，CPU不吐火。（注意，这句话是针对64bit cpu，和我无关）CPU的状态寄存器的初始化完全是`per_cpu_trap_init`后进行。

`setup_c0_status 0 0`的意思是关全局中断

`ARC64_TWIDDLE_PC`忽略，未定义`CONFIG_ARC64`和`CONFIG_MAPPED_KERNEL`

忽略被`CONFIG_MIPS_MT_SMTC`包含的

c文件中`__attribute__((nomips16))`即表示以mips32的方式去编译

所以在s文件中`.set nomips16`表示以mips32的方式去编译

忽略`CONFIG_KERNEL_DECRY`

heard.s 第522行，t0，t1存储bss段首尾

清空bss段

	LONG_S		a0, fw_arg0		# firmware arguments
	LONG_S		a1, fw_arg1
	LONG_S		a2, fw_arg2
	LONG_S		a3, fw_arg3
	
uboot中从uboot启动kernel时`theKernel (linux_argc, linux_argv, linux_env, 0);`

因此，

	a0 = linux_argc
	a1 = linux_argv
	a2 = linux_env
	a3 = 0

`MTC0        zero, CP0_CONTEXT    # clear context register`

清空TLB寄存器

	PTR_LA		$28, init_thread_union
	PTR_LI		sp, _THREAD_SIZE - 32
	PTR_ADDU	sp, $28
	set_saved_sp	sp, t0, t1
	PTR_SUBU	sp, 4 * SZREG		# init stack pointer

初始化内核线程栈

	vmlinux.lds.S (linux-2.6.x\arch\mips\kernel)    4048    2016/3/17

关于`init_thread_union`以及`sp`指针的初始化及内存分布图见下一篇[《RTL8382M Linux源码阅读思路 - head.s（2）》](/2016/03/19/rtl8383m-source-code-head-s-2.html) 