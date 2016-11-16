---
title: 'Linux中断对I2C总线的影响'
layout: post
guid: urn:uuid:2016-10-15-linux-interrupt-and-i2c
tags:
    - Linux
    - I2C
---

此处讨论的内核版本是linux-2.6.19 for mips-32。

### 1. i2c driver

PoE IC通过i2c与主IC（RTL8382M）通信，为标准i2c过程，代码如下：

	int32 IP80xI2cInit(void)
	{
		osal_printf("IP808_i2c_init...\n");
		_sda_output();
		_scl_output();
		return SYS_ERR_OK;
	}

	void _i2c_wait(void)
	{
		IP80X_DELAY_US(50);
	}

	void i2c_start(void)
	{
		_sda_high();
		_scl_high();
		_i2c_wait();
		_sda_low();
		_i2c_wait();
		_scl_low();
	}

	void i2c_stop(void)
	{
		_i2c_wait();
		_scl_low();
		_sda_low();
		_i2c_wait();
		_scl_high();
		_i2c_wait();
		_sda_high();
	}

	int32 _ack_get(void)
	{
		_scl_low();
		_sda_high();
		_sda_input();
		_i2c_wait();
		_scl_high();
		_i2c_wait();
		if(_sda_read())
		{
			_sda_output();
			i2c_stop();
			osal_printf("_ack_get err...\n");
			return SYS_ERR_FAILED;
		}
		_i2c_wait();
		_scl_low();
		_sda_output();
		return SYS_ERR_OK;
	}

	void _ack_set(void)
	{
		_scl_low();
		_sda_low();
		_i2c_wait(); 
		_scl_high();
		_i2c_wait(); 
		_scl_low();
	}

	void _ack_no_set(void)
	{
		_scl_low();
		_sda_high();
		_i2c_wait(); 
		_scl_high();
		_i2c_wait(); 
		_scl_low();
	}

	void i2c_write(uint8 value)
	{
		uint8 i=9;
		while(--i)
		{
			_i2c_wait();
			if((value & 0x80) != 0x80)
				_sda_low();
			else
				_sda_high();
			_i2c_wait();
			_scl_high();
			_i2c_wait();
			_scl_low();
			value <<= 1;
		}
		_sda_high();
	}

	uint8 i2c_read(void)
	{
		uint8 value=0;
		uint8 i=9;
		_sda_input();
		while(--i)
		{
			value <<= 1;
			_i2c_wait();
			_scl_high();
			_i2c_wait();
			if (_sda_read() == 1)
				value |= 0x01;
			_i2c_wait();
			_scl_low();
		}
		_sda_output();
		return value;
	}

	int32 ip80x_write(uint8 slave_addr, uint8 reg_addr, uint8 value)
	{
		int32 ret;
		i2c_start();							// start I2C
		i2c_write(slave_addr << 1);				// slave address
		ret = _ack_get();						// get ack
		if(SYS_ERR_FAILED == ret)
			return ret;
		i2c_write(reg_addr);					// register address
		ret = _ack_get();						// get ack
		if(SYS_ERR_FAILED == ret)
			return ret;
		i2c_write(value);						// send data
		ret = _ack_get();						// get ack
		if(SYS_ERR_FAILED == ret)
			return ret;
		i2c_stop();								// stop I2C
		return SYS_ERR_OK;
	}

	int32 ip80x_read(uint8 slave_addr, uint8 reg_addr, uint8 *value)
	{
		uint32 ret;
		i2c_start();							// start I2C
		i2c_write(slave_addr << 1);				// slave address
		ret = _ack_get();						// get ack
		if(SYS_ERR_FAILED == ret)
			return ret;
		i2c_write(reg_addr);					// register address
		ret = _ack_get();						// get ack
		if(SYS_ERR_FAILED == ret)
			return ret;
		i2c_stop();								// stop I2C
		i2c_start();
		i2c_write((slave_addr << 1) | 0x01);	// slave address
		ret = _ack_get();						// get ack
		if(SYS_ERR_FAILED == ret)
			return ret;
		*value = i2c_read();					// read data
		_ack_no_set();							// get ack
		i2c_stop();								// stop I2C
		return SYS_ERR_OK;
	}
	
### 2. 问题引出

以上代码在c51的机器上面运行正常，但是移植到linux-2.6.19 for mips-32平台就出现了问题。

具体表现为随机打印'_ack_get err...'错误，通讯周期越长越频繁。

### 3. 解决思路

1.既然只有部分通信失败，所以driver多半没有问题，这里面最有可能影响的就只有wait时间了，目前的值为50us，可能由于时间不合适导致。

2.由于i2c通信过程是不可重入的，所以可能是读写代码没有加入临界区保护，因为此机器带有web，PoE内核线程polling的时候如果同时刷新页面i2c总线就会出现竞争，如果不加保护肯定会出现上一次通信被下一次打断。

3.linux interrupt会导致通信被中断，尝试关中断，但是硬中断在linux里面好像是不能关闭的，可以参考c51，因为主ic集成了中断控制器，可以考虑直接操作寄存器禁止所有硬中断。

4.又linux interrupt时间不知道多久，必需用示波器测量i2c失败过程波形图分析，而且不确定中断是否会导致i2c失败，因为i2c并没有严格限制最长时间。

5.参考PoE IC，是否其针对i2c又特殊规定。

### 4. 解决过程

首先使用osal_sem_mutex_take函数和osal_sem_mutex_give函数给i2c读写代码加锁，需要说明的是，这两个函数相当于spinlock，竞争会导致忙等，同时关闭软中断，但是不能禁止硬中断，代码如下：

	osal_mutex_t poe_sem;
	
	poe_sem = osal_sem_mutex_create();
	
	int32 ip80x_reg_get(uint8 slave_addr, uint8 page, uint8 reg_addr, uint8 *value)
	{
		int32 ret;

		osal_sem_mutex_take(poe_sem,OSAL_SEM_WAIT_FOREVER);

		ret = IP80xPageSet(page);
		if(SYS_ERR_FAILED == ret)
		{
			osal_sem_mutex_give(poe_sem);
			return ret;
		}

		ret = ip80x_read(slave_addr,reg_addr,value);

		osal_sem_mutex_give(poe_sem);
		return ret;
	}

	int32 ip80x_reg_set(uint8 slave_addr, uint8 page, uint8 reg_addr, uint8 value)
	{
		int32 ret;

		osal_sem_mutex_take(poe_sem,OSAL_SEM_WAIT_FOREVER);

		ret = IP80xPageSet(page);
		if(SYS_ERR_FAILED == ret)
		{
			osal_sem_mutex_give(poe_sem);
			return ret;
		}

		ret = ip80x_write(slave_addr,reg_addr,value);

		osal_sem_mutex_give(poe_sem);
		return ret;
	}
	
测试证明加锁无效，所以很有可能是系统某个地方引起的通讯中断。

下面必需使用示波器测量i2c波形分析。

![图1](/../media/files/2016/11/1.jpg "图1")
<center>图1</center>
![图2](/../media/files/2016/11/2.png "图2")
<center>图2</center>
![图3](/../media/files/2016/11/3.png "图3")
<center>图3</center>
![图4](/../media/files/2016/11/4.png "图4")
<center>图4</center>

分别测量clock和data input线，蓝色表示clock，黄色表示从PoE IC到host的data input。黄色线low表示PoE IC给出ack确认信号，蓝色一个上升沿表示host给PoE IC一个bit数据，长一点的高表示start信号。

图1是正常波形，可以看到读出一个byte和写入4个byte的波形。

图2为接图1之前的波形，正常应该是最左第一个ack表示host已经发出去了一个byte的数据，由于从后面data线有一个byte数据出现可以确定，前面27个clock上升沿加上后面有data的9个clock上升沿必定是一个完整的读周期，所以正常情况data应该每隔8个clock上升沿有一个low的ack回复，但是这里没见到。异常出现在AB测量点，clock会出现10ms左右的低电平，AB点前到前一个ack有三个clock上升沿加上AB点后的五个上升沿刚好为一个byte写入，但是第九个周期却没有data ack，因此问题应该出现在这里。

图3可以看到这样的中断还有很多，似乎随机出现。

图4为半个上升沿的时间，测量为700us。

从波形图可以看出来，这些中断应该对i2c通信有影响，但是为什么会这样呢，正常来说10ms不应该影响正常通信才对，因为这很可能是系统中断，而之前用Microsemi的就不会出现这种情况，所以会不会是IC本身的限制，因此需要仔细查看PoE IC的datasheet看看是否有特殊规定。

![图5](/../media/files/2016/11/5.png "图5")
<center>图5</center>

很明显了，clock处于低电平不能超过7mini-seconds，也就是7ms，否则就终止通信，所以也就是系统因为某些原因导致通信中断10ms，超过IC的规定，因此出现错误。

从以上信息来看，系统导致通信中断，基本上就是硬件中断的时间太长，看datasheet可知RTL8382M集成中断控制器，并且有相应寄存器可以enable/disable，类似于c51的EA寄存器。接下来就禁止所有中断测试一下，代码如下：

	int32 ip80x_reg_get(uint8 slave_addr, uint8 page, uint8 reg_addr, uint8 *value)
	{
		int32 ret,inter;

		osal_sem_mutex_take(poe_sem,OSAL_SEM_WAIT_FOREVER);
		inter = REG32(0xB8003000);
		REG32(0xB8003000) = 0;

		ret = IP80xPageSet(page);
		if(SYS_ERR_FAILED == ret)
		{
			REG32(0xB8003000) = inter;
			osal_sem_mutex_give(poe_sem);
			return ret;
		}

		ret = ip80x_read(slave_addr,reg_addr,value);

		REG32(0xB8003000) = inter;
		osal_sem_mutex_give(poe_sem);
		return ret;
	}

	int32 ip80x_reg_set(uint8 slave_addr, uint8 page, uint8 reg_addr, uint8 value)
	{
		int32 ret,inter;

		osal_sem_mutex_take(poe_sem,OSAL_SEM_WAIT_FOREVER);
		inter = REG32(0xB8003000);
		REG32(0xB8003000) = 0;

		ret = IP80xPageSet(page);
		if(SYS_ERR_FAILED == ret)
		{
			REG32(0xB8003000) = inter;
			osal_sem_mutex_give(poe_sem);
			return ret;
		}

		ret = ip80x_write(slave_addr,reg_addr,value);

		REG32(0xB8003000) = inter;
		osal_sem_mutex_give(poe_sem);
		return ret;
	}
	
REG32(0xB8003000) = 0;为禁止所有硬中断，结果证明确实为某一个外设的硬中断导致的，继续测试证明，具体为time/counter0和nic。

中断时间过长的源头就是中断处理函数内容太多，跟踪代码得知：

	static struct irqaction timer_irqaction = {
		.handler = timer_interrupt,
		.flags = IRQF_DISABLED | IRQF_TIMER,
		.name = "timer",
	};

此为TC0 irq的irqaction，`timer_interrupt`为中断处理函数。

	osal_isr_register(RTK_DEV_NIC, _maple_nic_isr_handler, NULL);
	
`_maple_nic_isr_handler`为RTK_DEV_NIC irq的中断处理函数。

到此，`timer_interrupt`和`_maple_nic_isr_handler`执行时间过长为导致i2c错误的原因，接下来就是简化中断执行的时间，尽量在最短的时间执行中断，避免对其他线程造成影响。