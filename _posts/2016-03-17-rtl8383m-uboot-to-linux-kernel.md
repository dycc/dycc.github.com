---
title: 'RTL8382M从uboot跳转到Linux的过程'
layout: post
guid: urn:uuid:2016-03-17-rtl8383m-uboot-to-linux-kernel
tags:
    - Linux
---

uboot autostart的时候会调用`“rtk network on; bootm 0xb42a0000”`

在执行`bootm`的时候`static int bootm_start(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])`的参数中

`cmdtp`为`bootm`的结构，`flag=0，argc = 2，argv[0] = bootm，argv[1] = 0xb42a0000`

## 1. bootm_start

详解`bootm_start(cmdtp, flag, argc, argv);`

`getenv_yesno`获取环境变量`verify`的值，如果为字符n，则返回0，否则返回1

`boot_get_kernel - simple_strtoul`，获取`bootm`后面的地址，`0xb42a0000=0x142a0000`，flash中linux kernel地址`img_addr`

到此以后首先解释下image heard结构

	typedef struct image_header {
		uint32_t	ih_magic;	/* Image Header Magic Number	*/
		uint32_t	ih_hcrc;	/* Image Header CRC Checksum	*/
		uint32_t	ih_time;	/* Image Creation Timestamp	*/
		uint32_t	ih_size;	/* Image Data Size		*/
		uint32_t	ih_load;	/* Data	 Load  Address		*/
		uint32_t	ih_ep;		/* Entry Point Address		*/
		uint32_t	ih_dcrc;	/* Image Data CRC Checksum	*/
		uint8_t		ih_os;		/* Operating System		*/
		uint8_t		ih_arch;	/* CPU architecture		*/
		uint8_t		ih_type;	/* Image Type			*/
		uint8_t		ih_comp;	/* Compression Type		*/
		uint8_t		ih_name[IH_NMLEN];	/* Image Name		*/
	} image_header_t;

此结构为`0x40byte`

首先我们通过`uboot`启动的`kernel vmlinux`是经过gzip压缩的文件，`vmlinux.bix`，是uImage文件格式，因为是通过`mkimage`工具生成的

而我们加载的`uimage`文件`vmlinux.bix`是由gzip压缩后加上`image_header_t`形成的

此结构的目的是为了方便uboot加载kernel

`boot_get_kernel  - genimg_get_format`，检查`image_header_t.ih_magic`是否合法，配置内核是我们设定此值为`0x12345000`，但是在此没有进行检查，因此判断为`IMAGE_FORMAT_LEGACY`

	printf("## Booting kernel from Legacy Image at %08lx ...\n",img_addr);

	boot_get_kernel - image_get_kernel
	image_check_magic，检查magic
	image_check_hcrc，检查heard crc
	image_print_contents，打印image_header_t结构字段
	image_check_dcrc，检查校验data crc
	image_check_target_arch，检查cpu

`boot_get_kernel -  image_get_type`,获取kernel类型，`IH_TYPE_KERNEL`或者`IH_TYPE_KERNEL_NOLOAD`

`*os_data = image_get_data(hdr);`，获取压缩过的kernel的首地址，也就是`flash load`地址加上0x40的头部

`*os_len = image_get_data_size(hdr);`获取去掉heard之后的kernel数据长度，为2164592byte，加上头部为2,164,656 字节

`boot_get_kernel - memmove`，copy heard结构到bootm_headers_t

`images->legacy_hdr_os = hdr;`报存flash加载地址到bootm_headers_t

`images->legacy_hdr_valid = 1;`设定flash加载地址有效

返回img_addr加载地址

	os_hdr = img_addr
	cmdtp, flag, argc, argv,&images, &images.os.image_start, &images.os.image_len

最后两个参数表示压缩内核的起始点和长度

还是进入IMAGE_FORMAT_LEGACY分支

	images.os.type = image_get_type(os_hdr);
	images.os.comp = image_get_comp(os_hdr);
	images.os.os = image_get_os(os_hdr);
	images.os.end = image_get_image_end(os_hdr);
	images.os.load = image_get_load(os_hdr);

`images.legacy_hdr_valid`刚才设定为1

`images.ep = image_get_ep(&images.legacy_hdr_os_copy);`获取entry point，也就是kernel_entry的地址8020c000

`boot_get_ramdisk`，没有虚拟磁盘

`images.os.start = (ulong)os_hdr;`加载地址0x80000000

## 2. bootm_load_os

	uint8_t comp = os.comp;压缩格式
	ulong load = os.load;加载地址0x80000000
	ulong blob_start = os.start;镜像起始地址（flash）0xb42a0000
	ulong blob_end = os.end;镜像结束地址（flash）
	ulong image_start = os.image_start;镜像起始地址（flash）0xb42a0000
	ulong image_len = os.image_len;镜像结束地址（flash）
	__maybe_unused uint unc_len = CONFIG_SYS_BOOTM_LEN;镜像最长长度4M
	
`gunzip((void *)load, unc_len,(uchar *)image_start, &image_len)；`将flash中的压缩kernel解压到0x80000000，镜像最长4M，不是指内存中的

`flush_cache`刷新缓存

返回`BOOTM_ERR_OVERLAP`

## 3. lmb_reserve

不知道什么鬼

## 4. boot_fn = boot_os[images.os.os];

获取`linuxkernel`的启动函数

	do_bootm_linux

## 5. boot_fn(0, argc, argv, &images);

执行`do_bootm_linux`

## 6. do_bootm_linux

这是内核启动前最后的一个函数，该函数主要完成启动参数的初始化，并将板子设定为满足内核启动的环境

	theKernel = (void (*)(int, char **, char **, int *))images->ep;

获取入口点

`linux_params_init`，格式化启动参数`bootargs`，初始化linux环境变量存储区

256个参数，每个参数256byte，一个环境变量空间

	sprintf (env_buf, "%lu", (ulong)(gd->ram_size >> 20));
	linux_env_set ("memsize", env_buf);

	sprintf (env_buf, "0x%08X", (uint) UNCACHED_SDRAM (images->rd_start));
	linux_env_set ("initrd_start", env_buf);

	sprintf (env_buf, "0x%X", (uint) (images->rd_end - images->rd_start));
	linux_env_set ("initrd_size", env_buf);

	sprintf (env_buf, "0x%08X", (uint) (gd->bd->bi_flashstart));
	linux_env_set ("flash_start", env_buf);

	sprintf (env_buf, "0x%X", (uint) (gd->bd->bi_flashsize));
	linux_env_set ("flash_size", env_buf);

	cp = getenv("ethaddr");
	if (cp != NULL) {
		linux_env_set("ethaddr", cp);
	}

	cp = getenv("eth1addr");
	if (cp != NULL) {
		linux_env_set("eth1addr", cp);
	}//设定IP地址

	printf ("\nStarting kernel ...\n\n");

`theKernel (linux_argc, linux_argv, linux_env, 0);`

从入口点启动linux kernel，入口点在DRAM开头

uboot解压并传递启动参数的工作到此结束