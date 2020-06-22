# **S5VP210 U-BOOT Porting**

----------

## 一、编译命令

> * make O=输出目录
> * export PATH=/home/nltxl/home4/nltxl/linux/opt/5.5.0/bin:$PATH
>* make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi- tiny210_defconfig
> * make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi-

arm-linux-gnueabihf-

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- tiny210_defconfig




## 二、修改步骤

### 1、arch/arm/Kconfig添加ARCH_S5PV210

```c
10onfig ARCH_S5PV210
       bool "Samsung S5PV2XX"
       select CPU_V7
       select DM
       select DM_SERIAL
       select DM_GPIO
       select DM_I2C
...
source "arch/arm/mach-s5pv2xx/Kconfig"
```

### 2、arch/arm/Makefile添加mach-s5pv2xx

```c
machine-$(CONFIG_ARCH_S5PV210)  += s5pv2xx
```

### 3、添加arch/arm/mach-s5pv2xx, 参考s5pc110_gpio_pin

> **arch\arm\mach-s5pv2xx\reset.S**

```c
#define S5PV210_SWRESET			0xE0102000

ENTRY(reset_cpu)
	ldr	r1, =S5PV210_PRO_ID
```
> **arch\arm\mach-s5pv2xx\clock.c**

```c
#ifndef CONFIG_SYS_CLK_FREQ_C110
#define CONFIG_SYS_CLK_FREQ_C110	24000000
#endif
#ifndef CONFIG_SYS_CLK_FREQ_V210
#define CONFIG_SYS_CLK_FREQ_V210	24000000
#endif

...

/* s5pv210: return pll clock frequency */
static unsigned long s5pv210_get_pll_clk(int pllreg)
{
	struct s5pc110_clock *clk =
		(struct s5pc110_clock *)samsung_get_base_clock();
	unsigned long r, m, p, s, mask, fout;
	unsigned int freq;

	switch (pllreg) {
	case APLL:
		r = readl(&clk->apll_con);
		break;
	case MPLL:
		r = readl(&clk->mpll_con);
		break;
	case EPLL:
		r = readl(&clk->epll_con);
		break;
	case VPLL:
		r = readl(&clk->vpll_con);
		break;
	default:
		printf("Unsupported PLL (%d)\n", pllreg);
		return 0;
	}

	/*
	 * APLL_CON: MIDV [25:16]
	 * MPLL_CON: MIDV [25:16]
	 * EPLL_CON: MIDV [24:16]
	 * VPLL_CON: MIDV [24:16]
	 */
	if (pllreg == APLL || pllreg == MPLL)
		mask = 0x3ff;
	else
		mask = 0x1ff;

	m = (r >> 16) & mask;

	/* PDIV [13:8] */
	p = (r >> 8) & 0x3f;
	/* SDIV [2:0] */
	s = r & 0x7;

	freq = CONFIG_SYS_CLK_FREQ_V210;
	if (pllreg == APLL) {
		if (s < 1)
			s = 1;
		/* FOUT = MDIV * FIN / (PDIV * 2^(SDIV - 1)) */
		fout = m * (freq / (p * (1 << (s - 1))));
	} else
		/* FOUT = MDIV * FIN / (PDIV * 2^SDIV) */
		fout = m * (freq / (p * (1 << s)));

	return fout;
}

/* s5pv210: return ARM clock frequency */
static unsigned long s5pv210_get_arm_clk(void)
{
	struct s5pc110_clock *clk =
		(struct s5pc110_clock *)samsung_get_base_clock();
	unsigned long div;
	unsigned long dout_apll, armclk;
	unsigned int apll_ratio;

	div = readl(&clk->div0);

	/* APLL_RATIO: [2:0] */
	apll_ratio = div & 0x7;

	dout_apll = get_pll_clk(APLL) / (apll_ratio + 1);
	armclk = dout_apll;

	return armclk;
}

...

/* s5pv210: return peripheral clock frequency */
static unsigned long s5pv210_get_pclk(void)
{
	return get_pclk_sys(CLK_P);
}

...

/* s5pc1xx: return uart clock frequency */
static unsigned long s5pc1xx_get_uart_clk(int dev_index)
{
	if (cpu_is_s5pc110())
		return s5pc110_get_pclk();
    else if (cpu_is_s5pv210())
        return s5pv210_get_pclk();
	else
		return s5pc100_get_pclk();
}

/* s5pc1xx: return pwm clock frequency */
static unsigned long s5pc1xx_get_pwm_clk(void)
{
	if (cpu_is_s5pc110())
		return s5pc110_get_pclk();
    else if (cpu_is_s5pv210())
    	return s5pv210_get_pclk();
    else
		return s5pc100_get_pclk();
}

unsigned long get_pll_clk(int pllreg)
{
	if (cpu_is_s5pc110())
		return s5pc110_get_pll_clk(pllreg);
    else if (cpu_is_s5pv210())
        return s5pv210_get_pll_clk(pllreg);
	else
		return s5pc100_get_pll_clk(pllreg);
}

unsigned long get_arm_clk(void)
{
	if (cpu_is_s5pc110())
		return s5pc110_get_arm_clk();
    else if (cpu_is_s5pv210())
        return s5pv210_get_arm_clk();
	else
		return s5pc100_get_arm_clk();
}

...

void s5pv210_clock_init(void)
{
	u32 val = 0;

	struct s5pv210_clock *const clock = (struct s5pv210_clock *)samsung_get_base_clock();

	/* 1.设置PLL锁定值 */
	writel(0xFFFF, &clock->apll_lock);
	writel(0xFFFF, &clock->mpll_lock);
	writel(0xFFFF, &clock->epll_lock);
	writel(0xFFFF, &clock->vpll_lock);

	/* 2.设置PLL的PMS值(使用芯片手册推荐的值)，并使能PLL */
	/*	 	P	  	    M   		  S		     EN	*/
	writel((3  << 8) | (125 << 16) | (1 << 0) | (1 << 31), &clock->apll_con0);	/* FOUT_APLL = 1000MHz */
	writel((12 << 8) | (667 << 16) | (1 << 0) | (1 << 31), &clock->mpll_con); 	/* FOUT_MPLL = 667MHz */
	writel((3  << 8) | (48  << 16) | (2 << 0) | (1 << 31), &clock->epll_con0);	/* FOUT_EPLL = 96MHz */
	writel((6  << 8) | (108 << 16) | (3 << 0) | (1 << 31), &clock->vpll_con);	/* FOUT_VPLL = 54MHz */

	/* 3.等待PLL锁定 */
	while (!(readl(&clock->apll_con0) & (1 << 29)));
	while (!(readl(&clock->mpll_con) & (1 << 29)));
	while (!(readl(&clock->apll_con0) & (1 << 29)));
	while (!(readl(&clock->epll_con0) & (1 << 29)));
	while (!(readl(&clock->vpll_con) & (1 << 29)));

	/*
	** 4.设置系统时钟源，选择PLL为时钟输出 */
	/* MOUT_MSYS = SCLKAPLL = FOUT_APLL = 1000MHz
	** MOUT_DSYS = SCLKMPLL = FOUT_MPLL = 667MHz
	** MOUT_PSYS = SCLKMPLL = FOUT_MPLL = 667MHz
	** ONENAND = HCLK_PSYS
	*/
	writel((1 << 0) | (1 << 4) | (1 << 8) | (1 << 12), &clock->src0);

	/* 4.设置其他模块的时钟源 */

	/* 6.设置系统时钟分频值 */
	val = 	(0 << 0)  |	/* APLL_RATIO = 0, freq(ARMCLK) = MOUT_MSYS / (APLL_RATIO + 1) = 1000MHz */
			(4 << 4)  |	/* A2M_RATIO = 4, freq(A2M) = SCLKAPLL / (A2M_RATIO + 1) = 200MHz */
			(4 << 8)  |	/* HCLK_MSYS_RATIO = 4, freq(HCLK_MSYS) = ARMCLK / (HCLK_MSYS_RATIO + 1) = 200MHz */
			(1 << 12) |	/* PCLK_MSYS_RATIO = 1, freq(PCLK_MSYS) = HCLK_MSYS / (PCLK_MSYS_RATIO + 1) = 100MHz */
			(3 << 16) | /* HCLK_DSYS_RATIO = 3, freq(HCLK_DSYS) = MOUT_DSYS / (HCLK_DSYS_RATIO + 1) = 166MHz */
			(1 << 20) | /* PCLK_DSYS_RATIO = 1, freq(PCLK_DSYS) = HCLK_DSYS / (PCLK_DSYS_RATIO + 1) = 83MHz */
			(4 << 24) |	/* HCLK_PSYS_RATIO = 4, freq(HCLK_PSYS) = MOUT_PSYS / (HCLK_PSYS_RATIO + 1) = 133MHz */
			(1 << 28);	/* PCLK_PSYS_RATIO = 1, freq(PCLK_PSYS) = HCLK_PSYS / (PCLK_PSYS_RATIO + 1) = 66MHz */
	writel(val, &clock->div0);

	/* 7.设置其他模块的时钟分频值 */
}
```

> **arch\arm\mach-s5pv2xx\include\mach\power.h**

```c
#define S5PV210_RST_STAT		0xE010A000
#define S5PV210_SLEEP_WAKEUP		(1 << 3)
#define S5PV210_WAKEUP_STAT		0xE010C200
#define S5PV210_OTHERS			0xE010E000
#define S5PV210_USB_PHY_CON		0xE010E80C
#define S5PV210_INFORM0			0xE010F000
```

> **arm\mach-s5pv2xx\include\mach\gpio.h

```c
enum s5pv210_gpio_pin {
	S5PV210_GPIO_A00,
	...

	S5PV210_GPIO_MAX_PORT
};

...

#define S5PV210_GPIO_NUM_PARTS	1
static struct gpio_info s5pv210_gpio_data[S5PV210_GPIO_NUM_PARTS] = {
	{ S5PV210_GPIO_BASE, S5PV210_GPIO_NUM_PARTS },
};

static inline struct gpio_info *get_gpio_data(void)
{
	if (cpu_is_s5pc100())
		return s5pc100_gpio_data;
	else if (cpu_is_s5pc110())
		return s5pc110_gpio_data;
	else if (cpu_is_s5pv210())
        return s5pv210_gpio_data;
	return NULL;
}

static inline unsigned int get_bank_num(void)
{
	if (cpu_is_s5pc100())
		return S5PC100_GPIO_NUM_PARTS;
	else if (cpu_is_s5pc110())
		return S5PC110_GPIO_NUM_PARTS;
    else if (cpu_is_s5pv210())
        return S5PV210_GPIO_NUM_PARTS;
	return 0;
}

...

static const struct gpio_name_num_table s5pv210_gpio_table[] = {
	GPIO_ENTRY('a', S5PV210_GPIO_A00, S5PV210_GPIO_B0, 0),
	GPIO_ENTRY('b', S5PV210_GPIO_B0, S5PV210_GPIO_C00, 0),
	GPIO_ENTRY('c', S5PV210_GPIO_C00, S5PV210_GPIO_D00, 0),
	GPIO_ENTRY('d', S5PV210_GPIO_D00, S5PV210_GPIO_E00, 0),
	GPIO_ENTRY('e', S5PV210_GPIO_E00, S5PV210_GPIO_F00, 0),
	GPIO_ENTRY('f', S5PV210_GPIO_F00, S5PV210_GPIO_G00, 0),
	GPIO_ENTRY('g', S5PV210_GPIO_G00, S5PV210_GPIO_I0, 0),
	GPIO_ENTRY('i', S5PV210_GPIO_I0, S5PV210_GPIO_J00, 0),
	GPIO_ENTRY('j', S5PV210_GPIO_J00, S5PV210_GPIO_MP010, 0),
	GPIO_ENTRY('h', S5PV210_GPIO_H00, S5PV210_GPIO_MAX_PORT, 0),
	{ 0 }
};
```

> **arch\arm\mach-s5pv2xx\include\mach\cpu.h**

```c
/* S5PV210 */
#define S5PV210_PRO_ID		0xE0000000
#define S5PV210_CLOCK_BASE	0xE0100000
#define S5PV210_GPIO_BASE	0xE0200000
#define S5PV210_PWMTIMER_BASE	0xE2500000
#define S5PV210_WATCHDOG_BASE	0xE2700000
#define S5PV210_UART_BASE	0xE2900000
#define S5PV210_SROMC_BASE	0xE8000000
#define S5PV210_MMC_BASE	0xEB000000
#define S5PV210_DMC0_BASE	0xF0000000
#define S5PV210_DMC1_BASE	0xF1400000
#define S5PV210_VIC0_BASE	0xF2000000
#define S5PV210_VIC1_BASE	0xF2100000
#define S5PV210_VIC2_BASE	0xF2200000
#define S5PV210_VIC3_BASE	0xF2300000
#define S5PV210_OTG_BASE	0xEC000000
#define S5PV210_PHY_BASE	0xEC100000
#define S5PV210_USB_PHY_CONTROL 0xE010E80C

...
#if 0
static inline void s5p_set_cpu_id(void)
{
	s5p_cpu_id = readl(S5PV210_PRO_ID);
	s5p_cpu_rev = s5p_cpu_id & 0x000000FF;
	s5p_cpu_id = 0xC000 | ((s5p_cpu_id & 0x00FFF000) >> 12);
}
#else
static inline void s5p_set_cpu_id(void)  
{  
        int id = 0;  
        s5p_cpu_id = readl(S5PV210_PRO_ID);  
        s5p_cpu_rev = s5p_cpu_id & 0x000000FF;  
        id = (s5p_cpu_id & 0xFFFFF000) >> 12;  
        if (id == 0x43110) {
			id = s5p_cpu_id & 0x0F;
			switch (id){
				case 0x00:
					s5p_cpu_id = 0x56210;
					break;  
  
                case 0x01:
					s5p_cpu_id = 0xc110;  
               		break;  
 
                case 0x02:
					s5p_cpu_id = 0xc111;
					break;  
 
                default :
					break;
						}
			}  
}
#endif
...

IS_SAMSUNG_TYPE(s5pc100, 0xc100)
IS_SAMSUNG_TYPE(s5pc110, 0xc110)
IS_SAMSUNG_TYPE(s5pv210, 0xc000)

#ifdef CONFIG_ARCH_S5PV210
#define SAMSUNG_BASE(device, base)				\
static inline unsigned int samsung_get_base_##device(void)	\
{								\
	return S5PV210_##base;  \
}
#else
#define SAMSUNG_BASE(device, base)				\
static inline unsigned int samsung_get_base_##device(void)	\
{								\
	if (cpu_is_s5pc100())					\
		return S5PC100_##base;				\
	else if (cpu_is_s5pc110())				\
		return S5PC110_##base;				\
	else if (cpu_is_s5pv210())				\
		return S5PV210_##base;				\
	else							\
		return 0;					\
}
#endif

SAMSUNG_BASE(clock, CLOCK_BASE)
SAMSUNG_BASE(gpio, GPIO_BASE)
SAMSUNG_BASE(pro_id, PRO_ID)
SAMSUNG_BASE(mmc, MMC_BASE)
SAMSUNG_BASE(sromc, SROMC_BASE)
SAMSUNG_BASE(timer, PWMTIMER_BASE)
SAMSUNG_BASE(uart, UART_BASE)
SAMSUNG_BASE(watchdog, WATCHDOG_BASE)
#ifdef CONFIG_ARCH_S5PV210
SAMSUNG_BASE(dmc0,DMC0_BASE)
SAMSUNG_BASE(dmc1,DMC1_BASE)
#endif

```

> **arm\mach-s5pv2xx\include\mach\clock.h**

```c
struct s5pv210_clock {
	unsigned int	apll_lock;
	unsigned char	res1[0x04];
	unsigned int	mpll_lock;
	unsigned char	res2[0x04];
	unsigned int	epll_lock;
	unsigned char	res3[0x0C];
	unsigned int	vpll_lock;
	unsigned char	res4[0xdc];
	unsigned int	apll_con0;
	unsigned int	apll_con1;
	unsigned int	mpll_con;
	unsigned char	res5[0x04];
	unsigned int	epll_con0;
	unsigned int	epll_con1;
	unsigned char	res6[0x08];
	unsigned int	vpll_con;
	unsigned char	res7[0xdc];
	unsigned int	src0;
	unsigned int	src1;
	unsigned int	src2;
	unsigned int	src3;
	unsigned int	src4;
	unsigned int	src5;
	unsigned int	src6;
	unsigned char	res8[0x64];
	unsigned int	mask0;
	unsigned int	mask1;
	unsigned char	res9[0x78];
	unsigned int	div0;
	unsigned int	div1;
	unsigned int	div2;
	unsigned int	div3;
	unsigned int	div4;
	unsigned int	div5;
	unsigned int	div6;
	unsigned int	div7;
};
```
> **添加dmc.h**

```c
#ifndef __ASM_ARM_ARCH_DRAM_H_
#define __ASM_ARM_ARCH_DRAM_H_
 
#ifndef __ASSEMBLY__
 
struct s5pv210_dmc0 {
        unsigned int    concontrol;
        unsigned int    memcontrol;
        unsigned int    memconfig0;
        unsigned int    memconfig1;
        unsigned int    directcmd;
        unsigned int    prechconfig;
        unsigned int    phycontrol0;
        unsigned int    phycontrol1;
        unsigned char   res1[0x08];
        unsigned int    pwrdnconfig;
        unsigned char   res2[0x04];
        unsigned int    timingaref;
        unsigned int    timingrow;
        unsigned int    timingdata;
        unsigned int    timingpower;
        unsigned int    phystatus;
        unsigned int    chip0status;
        unsigned int    chip1status;
        unsigned int    arefstatus;
        unsigned int    mrstatus;
        unsigned int    phytest0;
        unsigned int    phytest1;
};
 
struct s5pv210_dmc1 {
        unsigned int    concontrol;                                               
		unsigned int    memcontrol;
        unsigned int    memconfig0;
        unsigned int    memconfig1;
        unsigned int    directcmd;
        unsigned int    prechconfig;
        unsigned int    phycontrol0;
        unsigned int    phycontrol1;
        unsigned char   res1[0x08];
        unsigned int    pwrdnconfig;
        unsigned char   res2[0x04];
        unsigned int    timingaref;
        unsigned int    timingrow;
        unsigned int    timingdata;
        unsigned int    timingpower;
        unsigned int    phystatus;
        unsigned int    chip0status;
        unsigned int    chip1status;
        unsigned int    arefstatus;
        unsigned int    mrstatus;
        unsigned int    phytest0;
        unsigned int    phytest1;
};
#endif
#endif
```





> **arch\arm\mach-s5pv2xx\Kconfig**

```c
if ARCH_S5PV210

choice
	prompt "S5PV210 board select"
	optional

config TARGET_TINY210
	bool "Support tiny210 board"
	select OF_CONTROL
	select SUPPORT_SPL
	select SPL

endchoice

config SYS_SOC
	default "s5pv2xx"

source "board/samsung/tiny210/Kconfig"

endif
```
### 4、修改arch/arm/cpu/armv7/Makefile 添加s5pv2xx

```makefile
ifneq (,$(filter s5pc1xx s5pv2xx exynos,$(SOC)))
```
### 5、修改arch/arm/cpu/armv7/s5p-common/cpu_info.c

```c
unsigned int s5p_cpu_id = 0xC000;
```

### 6、修改arch/arm/dts/Makefile

```c
dtb-$(CONFIG_S5PV210) += s5pv2xx-tiny210.dtb
```

### 7、添加arch/arm/dts/s5pv2xx-tiny210.dts

```c
/*
 * Samsung's Exynos4210-based SMDKV310 board device tree source
 *
 * Copyright (c) 2014 Google, Inc
 *
 * SPDX-License-Identifier:	GPL-2.0+
 */

/dts-v1/;

#include "skeleton.dtsi"
#include "s5pv210-pinctrl.dtsi" //参考s5pc110-pinctrl.dtsi

/ {
	model = "Samsung SMDKV210 based on S5PV210";
	compatible = "samsung,tiny210", "samsung,s5pv210";

	aliases {
		serial0 = "/serial@ec000000";
		console = "/serial@ec000000";
		pinctrl0 = &pinctrl0;
	};

	pinctrl0: pinctrl@e0300000 {
		compatible = "samsung,s5pv210-pinctrl";
		reg = <0xe0200000 0x1000>;
	};

	serial@ec000000 {
		compatible = "samsung,s5pv210-uart";
		reg = <0xec000000 0x100>;
		interrupts = <0 51 0>;
		id = <0>;
	};

};
```

### 8、修改common/spl/Kconfig

```c
config SPL_S5PV_SUPPORT
       bool "Support S5PV"
       help
         Enable support for S5PV (Multimedia Card) within SPL. This enables
         the S5PV protocol implementation and allows any enabled drivers to
         be used within SPL. MMC can be used with or without disk partition
         support depending on the application (SPL_LIBDISK_SUPPORT). Enable
         this option to build the drivers in drivers/S5PV as part of an SPL
         build.
```

### 9、修改common/spl/Makefile

```makefile
obj-$(CONFIG_SPL_S5PV_SUPPORT) += spl_s5p.o
```

### 10、添加common/spl/spl_s5p.c

```c
/*
 * (C) Copyright 2010
 * Texas Instruments, <www.ti.com>
 *
 * Aneesh V <aneesh@ti.com>
 *
 * SPDX-License-Identifier:	GPL-2.0+
 */
#include <common.h>
#include <dm.h>
#include <spl.h>
#include <linux/compiler.h>
#include <errno.h>
#include <asm/u-boot.h>
#include <errno.h>
#include <image.h>

DECLARE_GLOBAL_DATA_PTR;


typedef u32(*CopySDMMCtoMem)(u32 channel, u32 start_block, u16 block_size, u32 *trg, u32 init);

void spl_s5pv_copy_mmc_to_ram(void)
{
    u8  ch;
	u32 SDMMC_BASE;

    SDMMC_BASE = *(volatile u32 *)(S5PV_SDMMC_BASE);
	CopySDMMCtoMem copy_bl2 = (CopySDMMCtoMem) (*(u32 *) (S5PV_COPY_SDMMC_BASE));

	u32 ret;
	if (SDMMC_BASE == 0xEB000000) {
		ch = 0;
	}
	else if (SDMMC_BASE == 0xEB200000) {
		ch = 2;
	}

    ret = copy_bl2(ch, S5PV_BL2_LOADBLOCK, S5PV_BL2_BLOCKSIZE, CONFIG_SYS_TEXT_BASE, 0);

    printf("%s [%d]\n",__func__,ret);

}

int spl_s5pv_load_image(struct spl_image_info *spl_image,
		       struct spl_boot_device *bootdev)
{
	u32 boot_mode;
	int err = 0;

	boot_mode = spl_boot_mode(bootdev->boot_device);
	err = -EINVAL;
	switch (boot_mode) {

	case MMCSD_MODE_RAW:
		debug("spl: s5pv boot mode: raw\n");
        spl_s5pv_copy_mmc_to_ram();
        spl_set_header_raw_uboot(spl_image);
        err = 0;
        break;

#ifdef CONFIG_SPL_LIBCOMMON_SUPPORT
	default:
		puts("spl: s5pv: wrong boot mode\n");
#endif
	}

	return err;
}

SPL_LOAD_IMAGE_METHOD("S5PV", 0, BOOT_DEVICE_S5PV, spl_s5pv_load_image);

```
### 11、修改arch/arm/include/asm/spl.h

```c
        BOOT_DEVICE_S5PV,
        BOOT_DEVICE_NONE
 };

```
### 12、添加board\samsung\tiny210\Kconfig

```makefile
if TARGET_TINY210

config SYS_BOARD
	default "tiny210"

config SYS_VENDOR
	default "samsung"

config SYS_SOC
	default "s5pv2xx"

config SYS_CONFIG_NAME
	default "tiny210"

endif
```
### 13、添加board\samsung\tiny210\Makefile

```c
#
# (C) Copyright 2000, 2001, 2002
# Wolfgang Denk, DENX Software Engineering, wd@denx.de.
#
# (C) Copyright 2008
# Guennadi Liakhovetki, DENX Software Engineering, <lg@denx.de>
#
# SPDX-License-Identifier:	GPL-2.0+
#

ifdef CONFIG_SPL_BUILD
# necessary to create built-in.o
obj- := __dummy__.o

hostprogs-y := tools/mktiny210spl
always := $(hostprogs-y)
endif

obj-y	:= tiny210.o
obj-y	+= lowlevel_init.o

obj-$(CONFIG_SPL_BUILD) += tiny210-spl.o
```

### 14、修改board\samsung\tiny210\lowlevel_init.S

```c
/*
 * Copyright (C) 2009 Samsung Electronics
 * Kyungmin Park <kyungmin.park@samsung.com>
 * Minkyu Kang <mk7.kang@samsung.com>
 *
 * SPDX-License-Identifier:	GPL-2.0+
 */

#include <config.h>
#include <asm/arch/cpu.h>
#include <asm/arch/power.h>


/*
 * Register usages:
 *
 * r5 has zero always
 */

	.globl lowlevel_init
lowlevel_init:
	mov	r9, lr

	/* r5 has always zero */
	mov	r5, #0

	ldr	r8, =S5PV210_GPIO_BASE

	/* Disable Watchdog */
	ldr	r0, =S5PV210_WATCHDOG_BASE		@0xEA200000
	orr	r0, r0, #0x0
	str	r5, [r0]

	/* setting SRAM */
	ldr	r0, =S5PV210_SROMC_BASE
	ldr	r1, =0x9
	str	r1, [r0]

	/* S5PV210 has 3 groups of interrupt sources */
	ldr	r0, =S5PV210_VIC0_BASE			@0xE4000000
	ldr	r1, =S5PV210_VIC1_BASE			@0xE4000000
	ldr	r2, =S5PV210_VIC2_BASE			@0xE4000000

	/* Disable all interrupts (VIC0, VIC1 and VIC2) */
	mvn	r3, #0x0
	str	r3, [r0, #0x14]				@INTENCLEAR
	str	r3, [r1, #0x14]				@INTENCLEAR
	str	r3, [r2, #0x14]				@INTENCLEAR

	/* Set all interrupts as IRQ */
	str	r5, [r0, #0xc]				@INTSELECT
	str	r5, [r1, #0xc]				@INTSELECT
	str	r5, [r2, #0xc]				@INTSELECT

	/* Pending Interrupt Clear */
	str	r5, [r0, #0xf00]			@INTADDRESS
	str	r5, [r1, #0xf00]			@INTADDRESS
	str	r5, [r2, #0xf00]			@INTADDRESS

	#ifdef CONFIG_SPL_BUILD
	/* for CLOCK */
	bl s5pv210_clock_init

	/* for DDR */
	bl s5pv210_ddr_init
	#endif

	/* for UART */
	bl uart_asm_init

	/* for TZPC */
	bl tzpc_asm_init

1:
	mov	lr, r9
	mov	pc, lr

/*
 * uart_asm_init: Initialize UART's pins
 */
uart_asm_init:
	ldr r0, =S5PV210_GPIO_BASE
	ldr	r1, =0x22222222
	str	r1, [r0, #0x0]			@ GPA0_CON
	ldr	r1, =0x00022222
	str	r1, [r0, #0x20]			@ GPA1_CON

	mov	pc, lr

/*
 * tzpc_asm_init: Initialize TZPC
 */
tzpc_asm_init:
	ldr	r0, =0xF1500000
	mov	r1, #0x0
	str	r1, [r0]
	mov	r1, #0xff
	str	r1, [r0, #0x804]
	str	r1, [r0, #0x810]
	str	r1, [r0, #0x81C]

	ldr 	r0, =0xFAD00000
	str	r1, [r0, #0x804]
	str	r1, [r0, #0x810]
	str	r1, [r0, #0x81C]

	ldr	r0, =0xE0600000
	str	r1, [r0, #0x804]
	str	r1, [r0, #0x810]
	str	r1, [r0, #0x81C]
	str	r1, [r0, #0x828]

	ldr	r0, =0xE1C00000
	str	r1, [r0, #0x804]
	str	r1, [r0, #0x810]
	str	r1, [r0, #0x81C]

	mov	pc, lr
```

### 15、board\samsung\tiny210\tiny210.c 参考smdkc100.c

```c
#ifdef CONFIG_SPL_BUILD

void s5pv210_ddr_init(void)
{
    struct s5pv210_dmc0 *const dmc0 = (struct s5pv210_dmc0 *)samsung_get_base_dmc0();
    struct s5pv210_dmc1 *const dmc1 = (struct s5pv210_dmc1 *)samsung_get_base_dmc1();

    /* DMC0 */
    writel(0x00101000, &dmc0->phycontrol0);
    writel(0x00101002, &dmc0->phycontrol0);                 /* DLL on */
    writel(0x00000086, &dmc0->phycontrol1);
    writel(0x00101003, &dmc0->phycontrol0);                 /* DLL start */

    while ((readl(&dmc0->phystatus) & 0x7) != 0x7);         /* wait DLL locked */

    writel(0x0FFF2350, &dmc0->concontrol);                  /* Auto Refresh Counter should be off */
    writel(0x00202430, &dmc0->memcontrol);                  /* Dynamic power down should be off */
    writel(0x20E01323, &dmc0->memconfig0);

    writel(0xFF000000, &dmc0->prechconfig);
    writel(0xFFFF00FF, &dmc0->pwrdnconfig);

    writel(0x00000618, &dmc0->timingaref);                  /* 7.8us * 200MHz = 1560 = 0x618  */
    writel(0x19233309, &dmc0->timingrow);
    writel(0x23240204, &dmc0->timingdata);
    writel(0x09C80232, &dmc0->timingpower);

    writel(0x07000000, &dmc0->directcmd);                   /* NOP */
    writel(0x01000000, &dmc0->directcmd);                   /* PALL */
    writel(0x00020000, &dmc0->directcmd);                   /* EMRS2 */
    writel(0x00030000, &dmc0->directcmd);                   /* EMRS3 */
    writel(0x00010400, &dmc0->directcmd);                   /* EMRS enable DLL */
    writel(0x00000542, &dmc0->directcmd);                   /* DLL reset */
    writel(0x01000000, &dmc0->directcmd);                   /* PALL */
    writel(0x05000000, &dmc0->directcmd);                   /* auto refresh */
    writel(0x05000000, &dmc0->directcmd);                   /* auto refresh */
    writel(0x00000442, &dmc0->directcmd);                   /* DLL unreset */
    writel(0x00010780, &dmc0->directcmd);                   /* OCD default */
    writel(0x00010400, &dmc0->directcmd);                   /* OCD exit */

    writel(0x0FF02030, &dmc0->concontrol);                  /* auto refresh on */
    writel(0xFFFF00FF, &dmc0->pwrdnconfig);
    writel(0x00202400, &dmc0->memcontrol);

    /* DMC1 */
    writel(0x00101000, &dmc1->phycontrol0);
    writel(0x00101002, &dmc1->phycontrol0);                 /* DLL on */
    writel(0x00000086, &dmc1->phycontrol1);
    writel(0x00101003, &dmc1->phycontrol0);                 /* DLL start */

    while ((readl(&dmc1->phystatus) & 0x7) != 0x7);         /* wait DLL locked */

    writel(0x0FFF2350, &dmc1->concontrol);                  /* Auto Refresh Counter should be off */
    writel(0x00202430, &dmc1->memcontrol);                  /* Dynamic power down should be off */
    writel(0x40E01323, &dmc1->memconfig0);

    writel(0xFF000000, &dmc1->prechconfig);
    writel(0xFFFF00FF, &dmc1->pwrdnconfig);

    writel(0x00000618, &dmc1->timingaref);                  /* 7.8us * 200MHz = 1560 = 0x618  */
    writel(0x19233309, &dmc1->timingrow);
    writel(0x23240204, &dmc1->timingdata);
    writel(0x09C80232, &dmc1->timingpower);

    writel(0x07000000, &dmc1->directcmd);                   /* NOP */
    writel(0x01000000, &dmc1->directcmd);                   /* PALL */
    writel(0x00020000, &dmc1->directcmd);                   /* EMRS2 */
    writel(0x00030000, &dmc1->directcmd);                   /* EMRS3 */
    writel(0x00010400, &dmc1->directcmd);                   /* EMRS enable DLL */
    writel(0x00000542, &dmc1->directcmd);                   /* DLL reset */
    writel(0x01000000, &dmc1->directcmd);                   /* PALL */
    writel(0x05000000, &dmc1->directcmd);                   /* auto refresh */
    writel(0x05000000, &dmc1->directcmd);                   /* auto refresh */
    writel(0x00000442, &dmc1->directcmd);                   /* DLL unreset */
    writel(0x00010780, &dmc1->directcmd);                   /* OCD default */
    writel(0x00010400, &dmc1->directcmd);                   /* OCD exit */

    writel(0x0FF02030, &dmc1->concontrol);                  /* auto refresh on */
    writel(0xFFFF00FF, &dmc1->pwrdnconfig);
    writel(0x00202400, &dmc1->memcontrol);
}
#endif
```

### 16、drivers/gpio/s5p_gpio.c

```c
{ .compatible = "samsung,exynos5420-pinctrl" },
{ .compatible = "samsung,s5pv210-pinctrl" },
{ }
```

### 17、drivers/serial/serial_s5p.c

```c
{ .compatible = "samsung,exynos4210-uart" },
{ .compatible = "samsung,s5pv210-uart" },
{ }

...

#ifdef CONFIG_SPL_BUILD
static inline struct s5p_uart *s5p_get_base_uart(int dev_index)
{
	u32 offset = dev_index * sizeof(struct s5p_uart);
	return (struct s5p_uart *)(samsung_get_base_uart() + offset);
}

void serial_setbrg_dev(const int dev_index)
{
	struct s5p_uart *const uart = s5p_get_base_uart(dev_index);
	u32 uclk = get_uart_clk(dev_index);
	u32 baudrate = gd->baudrate;
	u32 val;

	val = uclk / baudrate;

	writel(val / 16 - 1, &uart->ubrdiv);

	if (s5p_uart_divslot())
		writew(udivslot[val % 16], &uart->rest.slot);
	else
		writeb(val % 16, &uart->rest.value);
}

/*
 * Initialise the serial port with the given baudrate. The settings
 * are always 8 data bits, no parity, 1 stop bit, no start bits.
 */
int serial_init_dev(const int dev_index)
{
	struct s5p_uart *const uart = s5p_get_base_uart(dev_index);

	/* reset and enable FIFOs, set triggers to the maximum */
	writel(0, &uart->ufcon);
	writel(0, &uart->umcon);
	/* 8N1 */
	writel(0x3, &uart->ulcon);
	/* No interrupts, no DMA, pure polling */
	writel(0x245, &uart->ucon);

	serial_setbrg_dev(dev_index);

	return 0;
}

static int serial_err_check(const int dev_index, int op)
{
	struct s5p_uart *const uart = s5p_get_base_uart(dev_index);
	unsigned int mask;

	/*
	 * UERSTAT
	 * Break Detect	[3]
	 * Frame Err	[2] : receive operation
	 * Parity Err	[1] : receive operation
	 * Overrun Err	[0] : receive operation
	 */
	if (op)
		mask = 0x8;
	else
		mask = 0xf;

	return readl(&uart->uerstat) & mask;
}

/*
 * Read a single byte from the serial port. Returns 1 on success, 0
 * otherwise. When the function is succesfull, the character read is
 * written into its argument c.
 */
int serial_getc_dev(const int dev_index)
{
	struct s5p_uart *const uart = s5p_get_base_uart(dev_index);

	/* wait for character to arrive */
	while (!(readl(&uart->utrstat) & 0x1)) {
		if (serial_err_check(dev_index, 0))
			return 0;
	}

	return (int)(readb(&uart->urxh) & 0xff);
}

/*
 * Output a single byte to the serial port.
 */
void serial_putc_dev(const char c, const int dev_index)
{
	struct s5p_uart *const uart = s5p_get_base_uart(dev_index);

	/* wait for room in the tx FIFO */
	while (!(readl(&uart->utrstat) & 0x2)) {
		if (serial_err_check(dev_index, 1))
			return;
	}

	writeb(c, &uart->utxh);

	/* If \n, also do \r */
	if (c == '\n')
		serial_putc('\r');
}

/*
 * Test whether a character is in the RX buffer
 */
int serial_tstc_dev(const int dev_index)
{
	struct s5p_uart *const uart = s5p_get_base_uart(dev_index);

	return (int)(readl(&uart->utrstat) & 0x1);
}

void serial_puts_dev(const char *s, const int dev_index)
{
	while (*s)
		serial_putc_dev(*s++, dev_index);
}

/* Multi serial device functions */
#define DECLARE_S5P_SERIAL_FUNCTIONS(port) \
int s5p_serial##port##_init(void) { return serial_init_dev(port); } \
void s5p_serial##port##_setbrg(void) { serial_setbrg_dev(port); } \
int s5p_serial##port##_getc(void) { return serial_getc_dev(port); } \
int s5p_serial##port##_tstc(void) { return serial_tstc_dev(port); } \
void s5p_serial##port##_putc(const char c) { serial_putc_dev(c, port); } \
void s5p_serial##port##_puts(const char *s) { serial_puts_dev(s, port); }

#define INIT_S5P_SERIAL_STRUCTURE(port, name) { \
	name, \
	s5p_serial##port##_init, \
	NULL, \
	s5p_serial##port##_setbrg, \
	s5p_serial##port##_getc, \
	s5p_serial##port##_tstc, \
	s5p_serial##port##_putc, \
	s5p_serial##port##_puts, }

DECLARE_S5P_SERIAL_FUNCTIONS(0);
struct serial_device s5p_serial0_device =
	INIT_S5P_SERIAL_STRUCTURE(0, "s5pser0");
DECLARE_S5P_SERIAL_FUNCTIONS(1);
struct serial_device s5p_serial1_device =
	INIT_S5P_SERIAL_STRUCTURE(1, "s5pser1");
DECLARE_S5P_SERIAL_FUNCTIONS(2);
struct serial_device s5p_serial2_device =
	INIT_S5P_SERIAL_STRUCTURE(2, "s5pser2");
DECLARE_S5P_SERIAL_FUNCTIONS(3);
struct serial_device s5p_serial3_device =
	INIT_S5P_SERIAL_STRUCTURE(3, "s5pser3");

__weak struct serial_device *default_serial_console(void)
{
#if defined(CONFIG_SERIAL0)
	return &s5p_serial0_device;
#elif defined(CONFIG_SERIAL1)
	return &s5p_serial1_device;
#elif defined(CONFIG_SERIAL2)
	return &s5p_serial2_device;
#elif defined(CONFIG_SERIAL3)
	return &s5p_serial3_device;
#else
#error "CONFIG_SERIAL? missing."
#endif
}
#endif
```



## 18.添加SPL工具 tools/mktiny210spl.c

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define BUFSIZE                 (8*1024)
#define IMG_SIZE                (8*1024)
#define SPL_HEADER_SIZE         16
//#define FILE_PERM               (S_IRUSR | S_IWUSR | S_IRGRP \
                                | S_IWGRP | S_IROTH | S_IWOTH)
#define SPL_HEADER              "S5PC110 HEADER  "

int main (int argc, char *argv[])
{
	FILE		*fp;
	char		*Buf, *a;
	int		BufLen;
	int		nbytes, fileLen;
	unsigned int	checksum, count;
	int		i;

//////////////////////////////////////////////////////////////
	if (argc != 3)
	{
		printf("Usage: mkbl1 <source file> <destination file>\n");
		return -1;
	}

//////////////////////////////////////////////////////////////
	BufLen = BUFSIZE;
	Buf = (char *)malloc(BufLen);
	if (!Buf)
	{
		printf("Alloc buffer failed!\n");
		return -1;
	}

	memset(Buf, 0x00, BufLen);

//////////////////////////////////////////////////////////////
	fp = fopen(argv[1], "rb");
	if( fp == NULL)
	{
		printf("source file open error\n");
		free(Buf);
		return -1;
	}

	fseek(fp, 0L, SEEK_END);
	fileLen = ftell(fp);
	fseek(fp, 0L, SEEK_SET);
#if 0
	if ( BufLen > fileLen )
	{
		printf("Usage: unsupported size\n");
		free(Buf);
		fclose(fp);
		return -1;
	}
#endif
	count = (fileLen < (IMG_SIZE - SPL_HEADER_SIZE))
		? fileLen : (IMG_SIZE - SPL_HEADER_SIZE);

	memcpy(&Buf[0], SPL_HEADER, SPL_HEADER_SIZE);

	nbytes = fread(Buf + SPL_HEADER_SIZE, 1, count, fp);

	if ( nbytes != count )
	{
		printf("source file read error\n");
		free(Buf);
		fclose(fp);
		return -1;
	}

	fclose(fp);

//////////////////////////////////////////////////////////////
	a = Buf + SPL_HEADER_SIZE;
	for(i = 0, checksum = 0; i < IMG_SIZE - SPL_HEADER_SIZE; i++)
		checksum += (0x000000FF) & *a++;

	a = Buf + 8;
	*( (unsigned int *)a ) = checksum;

//////////////////////////////////////////////////////////////
	fp = fopen(argv[2], "wb");
	if (fp == NULL)
	{
		printf("destination file open error\n");
		free(Buf);
		return -1;
	}

	a	= Buf;
	nbytes	= fwrite( a, 1, BufLen, fp);

	if ( nbytes != BufLen )
	{
		printf("destination file write error\n");
		free(Buf);
		fclose(fp);
		return -1;
	}

	free(Buf);
	fclose(fp);

	return 0;
}
```



## 19.添加tiny210.h

```c
#ifndef __CONFIG_H
#define __CONFIG_H

/* High Level Configuration Options */
#define CONFIG_SAMSUNG			1	/* SAMSUNG core*/
#define CONFIG_S5P				1	/* S5P Family */
#define CONFIG_S5PV210			1	/* which is in a S5PV210 SoC */

#include <asm/arch/cpu.h>		/* get chip and board defs */

/* Enable DEBUG for get more information from uart */
#define DEBUG	/* it will print the log that debug() to uart */

#define CONFIG_ARCH_CPU_INIT

#define CONFIG_MCP_SINGLE		1
#define CONFIG_EVT1				1		/* EVT1 */



#define BOOT_ONENAND		0x1
#define BOOT_NAND		0x2
#define BOOT_MMCSD		0x3
#define BOOT_NOR		0x4
#define BOOT_SEC_DEV		0x5

/* Keep L2 Cache Disabled */
#define CONFIG_SYS_DCACHE_OFF           1

#define CONFIG_SYS_SDRAM_BASE           0x20000000
#define CONFIG_SYS_TEXT_BASE            0x23E00000

#define MEMORY_BASE_ADDRESS	CONFIG_SYS_SDRAM_BASE

/* input clock of PLL: MINI210 has 24MHz input clock */
#define CONFIG_SYS_CLK_FREQ		24000000

#ifndef CONFIG_SYS_DCACHE_OFF
#define CONFIG_ENABLE_MMU
#endif

#undef CONFIG_USE_IRQ				/* we don't need IRQ/FIQ stuff */


#define CONFIG_SETUP_MEMORY_TAGS
#define CONFIG_CMDLINE_TAG
#define CONFIG_INITRD_TAG
#define CONFIG_CMDLINE_EDITING

/* MACH_TYPE_MINI210 macro will be removed once added to mach-types */
#define CONFIG_MACH_TYPE		MACH_TYPE_TINY210

/* Size of malloc() pool */
#define CONFIG_SYS_MALLOC_LEN			(CONFIG_ENV_SIZE + 896*1024)

/* select serial console configuration */
#define CONFIG_SERIAL_MULTI				1
#define CONFIG_SERIAL0					1	/* use SERIAL 0 */
#define CONFIG_BAUDRATE					115200
#define S5PC210_DEFAULT_UART_OFFSET		0x020000

/* SD/MMC configuration */
#define CONFIG_GENERIC_MMC		1
#define CONFIG_MMC			1
#define CONFIG_S5P_SDHCI		1

/* PWM */
#define CONFIG_PWM			1

/* allow to overwrite serial and ethaddr */
#define CONFIG_ENV_OVERWRITE

/* Command definition*/
/* #include <config_cmd_default.h> */


#define CONFIG_CMD_PING
#define CONFIG_CMD_ELF
#define CONFIG_CMD_DHCP
#define CONFIG_CMD_MMC
#define CONFIG_CMD_FAT
#if 0
//#undef CONFIG_CMD_NET
//#undef CONFIG_CMD_NFS
#endif

/* Miscellaneous configurable options */
#define CONFIG_SYS_LONGHELP             /* undef to save memory */
#define CONFIG_SYS_PROMPT_HUSH_PS2      "> "
/*
 * #define CONFIG_SYS_PROMPT              "[FriendlyLEG-TINY210]# "
 */
#define CONFIG_SYS_CBSIZE               256     /* Console I/O Buffer Size*/
#define CONFIG_SYS_PBSIZE               384     /* Print Buffer Size */
#define CONFIG_SYS_MAXARGS              64      /* max number of command args */
/* Boot Argument Buffer Size */
#define CONFIG_SYS_BARGSIZE				CONFIG_SYS_CBSIZE
/* memtest works on */
#define CONFIG_SYS_MEMTEST_START	MEMORY_BASE_ADDRESS
#define CONFIG_SYS_MEMTEST_END		(MEMORY_BASE_ADDRESS + 0x3E00000)		/* 256 MB in DRAM	*/
#define CONFIG_SYS_LOAD_ADDR		(PHYS_SDRAM_1 + 0x1000000)	/* default load address	*/

/* the PWM TImer 4 uses a counter of 41687 for 10 ms, so we need */
/* it to wrap 100 times (total 4168750) to get 1 sec. */
/*
#define CONFIG_SYS_HZ			1000		// at PCLK 66MHz
*/

/* valid baudrates */
#define CONFIG_SYS_BAUDRATE_TABLE	{ 9600, 19200, 38400, 57600, 115200 }

/* Stack sizes */
#define CONFIG_STACKSIZE	0x40000		/* regular stack 256KB */

#ifdef CONFIG_USE_IRQ
#define CONFIG_STACKSIZE_IRQ	(4*1024)	/* IRQ stack */
#define CONFIG_STACKSIZE_FIQ	(4*1024)	/* FIQ stack */
#endif

#define CONFIG_SPL_STACK		0xD0037FFF
#define CONFIG_SYS_INIT_SP_ADDR (CONFIG_SYS_LOAD_ADDR - GENERATED_GBL_DATA_SIZE)

/* MINI210 has 4 bank of DRAM */
#define CONFIG_NR_DRAM_BANKS	1
#define SDRAM_BANK_SIZE		0x20000000	/* 512 MB */
#define PHYS_SDRAM_1		MEMORY_BASE_ADDRESS
#define PHYS_SDRAM_1_SIZE	SDRAM_BANK_SIZE

/* FLASH and environment organization */
#define CONFIG_SYS_NO_FLASH		1
#undef CONFIG_CMD_IMLS

#define CONFIG_ENV_IS_IN_MMC		1
#define CONFIG_SYS_MMC_ENV_DEV		0
#define CONFIG_ENV_SIZE				0x4000	/* 16KB */
#define RESERVE_BLOCK_SIZE              (512)
#define BL1_SIZE                        (8 << 10) /*8 K reserved for BL1*/
#define CONFIG_ENV_OFFSET               (RESERVE_BLOCK_SIZE + BL1_SIZE + ((16 + 512) * 1024))
#define CONFIG_DOS_PARTITION		1

#if 0
//#define CONFIG_CLK_667_166_166_133
//#define CONFIG_CLK_533_133_100_100
//#define CONFIG_CLK_800_200_166_133
//#define CONFIG_CLK_800_100_166_133
#endif
#define CONFIG_CLK_1000_200_166_133
#if 0
//#define CONFIG_CLK_400_200_166_133
//#define CONFIG_CLK_400_100_166_133
#endif

#if defined(CONFIG_CLK_667_166_166_133)
#define APLL_MDIV       0xfa
#define APLL_PDIV       0x6
#define APLL_SDIV       0x1
#elif defined(CONFIG_CLK_533_133_100_100)
#define APLL_MDIV       0x215
#define APLL_PDIV       0x18
#define APLL_SDIV       0x1
#elif defined(CONFIG_CLK_800_200_166_133) || \
	defined(CONFIG_CLK_800_100_166_133) || \
	defined(CONFIG_CLK_400_200_166_133) || \
	defined(CONFIG_CLK_400_100_166_133)
#define APLL_MDIV       0x64
#define APLL_PDIV       0x3
#define APLL_SDIV       0x1
#elif defined(CONFIG_CLK_1000_200_166_133)
#define APLL_MDIV       0x7d
#define APLL_PDIV       0x3
#define APLL_SDIV       0x1
#endif

#define APLL_LOCKTIME_VAL	0x2cf

#if defined(CONFIG_EVT1)
/* Set AFC value */
#define AFC_ON		0x00000000
#define AFC_OFF		0x10000010
#endif

#if defined(CONFIG_CLK_533_133_100_100)
#define MPLL_MDIV	0x190
#define MPLL_PDIV	0x6
#define MPLL_SDIV	0x2
#else
#define MPLL_MDIV	0x29b
#define MPLL_PDIV	0xc
#define MPLL_SDIV	0x1
#endif

#define EPLL_MDIV	0x60
#define EPLL_PDIV	0x6
#define EPLL_SDIV	0x2

#define VPLL_MDIV	0x6c
#define VPLL_PDIV	0x6
#define VPLL_SDIV	0x3

/* CLK_DIV0 */
#define APLL_RATIO	0
#define A2M_RATIO	4
#define HCLK_MSYS_RATIO	8
#define PCLK_MSYS_RATIO	12
#define HCLK_DSYS_RATIO	16
#define PCLK_DSYS_RATIO 20
#define HCLK_PSYS_RATIO	24
#define PCLK_PSYS_RATIO 28

#define CLK_DIV0_MASK	0x7fffffff

#define set_pll(mdiv, pdiv, sdiv)	(1<<31 | mdiv<<16 | pdiv<<8 | sdiv)

#define APLL_VAL	set_pll(APLL_MDIV,APLL_PDIV,APLL_SDIV)
#define MPLL_VAL	set_pll(MPLL_MDIV,MPLL_PDIV,MPLL_SDIV)
#define EPLL_VAL	set_pll(EPLL_MDIV,EPLL_PDIV,EPLL_SDIV)
#define VPLL_VAL	set_pll(VPLL_MDIV,VPLL_PDIV,VPLL_SDIV)

#if defined(CONFIG_CLK_667_166_166_133)
#define CLK_DIV0_VAL    ((0<<APLL_RATIO)|(3<<A2M_RATIO)|(3<<HCLK_MSYS_RATIO)|(1<<PCLK_MSYS_RATIO)\
			|(3<<HCLK_DSYS_RATIO)|(1<<PCLK_DSYS_RATIO)|(4<<HCLK_PSYS_RATIO)|(1<<PCLK_PSYS_RATIO))
#elif defined(CONFIG_CLK_533_133_100_100)
#define CLK_DIV0_VAL    ((0<<APLL_RATIO)|(3<<A2M_RATIO)|(3<<HCLK_MSYS_RATIO)|(1<<PCLK_MSYS_RATIO)\
			|(3<<HCLK_DSYS_RATIO)|(1<<PCLK_DSYS_RATIO)|(3<<HCLK_PSYS_RATIO)|(1<<PCLK_PSYS_RATIO))
#elif defined(CONFIG_CLK_800_200_166_133)
#define CLK_DIV0_VAL    ((0<<APLL_RATIO)|(3<<A2M_RATIO)|(3<<HCLK_MSYS_RATIO)|(1<<PCLK_MSYS_RATIO)\
			|(3<<HCLK_DSYS_RATIO)|(1<<PCLK_DSYS_RATIO)|(4<<HCLK_PSYS_RATIO)|(1<<PCLK_PSYS_RATIO))
#elif defined(CONFIG_CLK_800_100_166_133)
#define CLK_DIV0_VAL    ((0<<APLL_RATIO)|(7<<A2M_RATIO)|(7<<HCLK_MSYS_RATIO)|(1<<PCLK_MSYS_RATIO)\
			|(3<<HCLK_DSYS_RATIO)|(1<<PCLK_DSYS_RATIO)|(4<<HCLK_PSYS_RATIO)|(1<<PCLK_PSYS_RATIO))
#elif defined(CONFIG_CLK_400_200_166_133)
#define CLK_DIV0_VAL    ((1<<APLL_RATIO)|(3<<A2M_RATIO)|(1<<HCLK_MSYS_RATIO)|(1<<PCLK_MSYS_RATIO)\
			|(3<<HCLK_DSYS_RATIO)|(1<<PCLK_DSYS_RATIO)|(4<<HCLK_PSYS_RATIO)|(1<<PCLK_PSYS_RATIO))
#elif defined(CONFIG_CLK_400_100_166_133)
#define CLK_DIV0_VAL    ((1<<APLL_RATIO)|(7<<A2M_RATIO)|(3<<HCLK_MSYS_RATIO)|(1<<PCLK_MSYS_RATIO)\
			|(3<<HCLK_DSYS_RATIO)|(1<<PCLK_DSYS_RATIO)|(4<<HCLK_PSYS_RATIO)|(1<<PCLK_PSYS_RATIO))
#elif defined(CONFIG_CLK_1000_200_166_133)
#define CLK_DIV0_VAL    ((0<<APLL_RATIO)|(4<<A2M_RATIO)|(4<<HCLK_MSYS_RATIO)|(1<<PCLK_MSYS_RATIO)\
			|(3<<HCLK_DSYS_RATIO)|(1<<PCLK_DSYS_RATIO)|(4<<HCLK_PSYS_RATIO)|(1<<PCLK_PSYS_RATIO))
#endif

#define CLK_DIV1_VAL	((1<<16)|(1<<12)|(1<<8)|(1<<4))
#define CLK_DIV2_VAL	(1<<0)

#if defined(CONFIG_CLK_533_133_100_100)

#if defined(CONFIG_MCP_SINGLE)

#define DMC0_TIMINGA_REF	0x40e
#define DMC0_TIMING_ROW		0x10233206
#define DMC0_TIMING_DATA	0x12130005
#define	DMC0_TIMING_PWR		0x0E100222

#define DMC1_TIMINGA_REF	0x40e
#define DMC1_TIMING_ROW		0x10233206
#define DMC1_TIMING_DATA	0x12130005
#define	DMC1_TIMING_PWR		0x0E100222

#else

#error "You should define memory type (AC type or H type or B type)"

#endif

#elif defined(CONFIG_CLK_800_200_166_133) || \
	defined(CONFIG_CLK_1000_200_166_133) || \
	defined(CONFIG_CLK_800_100_166_133) || \
	defined(CONFIG_CLK_400_200_166_133) || \
	defined(CONFIG_CLK_400_100_166_133)

#if defined(CONFIG_MCP_SINGLE)

#define DMC0_MEMCONTROL		0x00202400	// MemControl	BL=4, 1Chip, DDR2 Type, dynamic self refresh, force precharge, dynamic power down off
#define DMC0_MEMCONFIG_0	0x20E00323	// MemConfig0	256MB config, 8 banks,Mapping Method[12:15]0:linear, 1:linterleaved, 2:Mixed
#define DMC0_MEMCONFIG_1	0x00E00323	// MemConfig1
#if 0
#define DMC0_TIMINGA_REF	0x00000618	// TimingAref	7.8us*133MHz=1038(0x40E), 100MHz=780(0x30C), 20MHz=156(0x9C), 10MHz=78(0x4E)
#define DMC0_TIMING_ROW		0x28233287	// TimingRow	for @200MHz
#define DMC0_TIMING_DATA	0x23240304	// TimingData	CL=3
#define	DMC0_TIMING_PWR		0x09C80232	// TimingPower
#else
#define DMC0_TIMINGA_REF        0x00000618      // TimingAref   7.8us*133MHz=1038(0x40E), 100MHz=780(0x30C), 20MHz=156(0x9C), 10MHz=78(0x4E)
#define DMC0_TIMING_ROW         0x2B34438A      // TimingRow    for @200MHz
#define DMC0_TIMING_DATA        0x24240000      // TimingData   CL=3
#define DMC0_TIMING_PWR         0x0BDC0343      // TimingPower
#endif

#define	DMC1_MEMCONTROL		0x00202400	// MemControl	BL=4, 2 chip, DDR2 type, dynamic self refresh, force precharge, dynamic power down off
#define DMC1_MEMCONFIG_0	0x40F00313	// MemConfig0	512MB config, 8 banks,Mapping Method[12:15]0:linear, 1:linterleaved, 2:Mixed
#define DMC1_MEMCONFIG_1	0x00F00313	// MemConfig1
#if 0
#define DMC1_TIMINGA_REF	0x00000618	// TimingAref	7.8us*133MHz=1038(0x40E), 100MHz=780(0x30C), 20MHz=156(0x9C), 10MHz=78(0x4
#define DMC1_TIMING_ROW		0x28233289	// TimingRow	for @200MHz
#define DMC1_TIMING_DATA	0x23240304	// TimingData	CL=3
#define	DMC1_TIMING_PWR		0x08280232	// TimingPower
#else
#define DMC1_TIMINGA_REF        0x00000618      // TimingAref   7.8us*133MHz=1038(0x40E), 100MHz=780(0x30C), 20MHz=156(0x9C), 10MHz=78(0x4E)
#define DMC1_TIMING_ROW         0x2B34438A      // TimingRow    for @200MHz
#define DMC1_TIMING_DATA        0x24240000      // TimingData   CL=3
#define DMC1_TIMING_PWR         0x0BDC0343      // TimingPower
#endif
#if defined(CONFIG_CLK_800_100_166_133) || defined(CONFIG_CLK_400_100_166_133)
#define DMC0_MEMCONFIG_0	0x20E01323	// MemConfig0	256MB config, 8 banks,Mapping Method[12:15]0:linear, 1:linterleaved, 2:Mixed
#define DMC0_MEMCONFIG_1	0x40F01323	// MemConfig1
#define DMC0_TIMINGA_REF	0x0000030C	// TimingAref	7.8us*133MHz=1038(0x40E), 100MHz=780(0x30C), 20MHz=156(0x9C), 10MHz=78(0x4E)
#define DMC0_TIMING_ROW		0x28233287	// TimingRow	for @200MHz
#define DMC0_TIMING_DATA	0x23240304	// TimingData	CL=3
#define	DMC0_TIMING_PWR		0x09C80232	// TimingPower

#define	DMC1_MEMCONTROL		0x00202400	// MemControl	BL=4, 2 chip, DDR2 type, dynamic self refresh, force precharge, dynamic power down off
#define DMC1_MEMCONFIG_0	0x40C01323	// MemConfig0	512MB config, 8 banks,Mapping Method[12:15]0:linear, 1:linterleaved, 2:Mixed
#define DMC1_MEMCONFIG_1	0x00E01323	// MemConfig1
#define DMC1_TIMINGA_REF	0x0000030C	// TimingAref	7.8us*133MHz=1038(0x40E), 100MHz=780(0x30C), 20MHz=156(0x9C), 10MHz=78(0x4
#define DMC1_TIMING_ROW		0x28233289	// TimingRow	for @200MHz
#define DMC1_TIMING_DATA	0x23240304	// TimingData	CL=3
#define	DMC1_TIMING_PWR		0x08280232	// TimingPower
#endif

#else

#error "You should define memory type (AC type or H type)"

#endif

#else

#define DMC0_TIMINGA_REF	0x50e
#define DMC0_TIMING_ROW		0x14233287
#define DMC0_TIMING_DATA	0x12130005
#define	DMC0_TIMING_PWR		0x0E140222

#define DMC1_TIMINGA_REF	0x618
#define DMC1_TIMING_ROW		0x11344309
#define DMC1_TIMING_DATA	0x12130005
#define	DMC1_TIMING_PWR		0x0E190222

#endif


#if defined(CONFIG_CLK_533_133_100_100)
#define UART_UBRDIV_VAL		26
#define UART_UDIVSLOT_VAL	0x0808
#else
#define UART_UBRDIV_VAL		34
#define UART_UDIVSLOT_VAL	0xDDDD
#endif

/* MMC SPL */
/*
 * #define CONFIG_SPL
 */

/* Modified by lk for dm9000*/
#define DM9000_16BIT_DATA
#define CONFIG_CMD_NET
#define CONFIG_DRIVER_DM9000       1
#define CONFIG_NET_MULTI               1
#define CONFIG_NET_RETRY_COUNT 1
#define CONFIG_DM9000_NO_SROM 1
#ifdef CONFIG_DRIVER_DM9000  
#define CONFIG_DM9000_BASE		(0x88001000)
#define DM9000_IO			(CONFIG_DM9000_BASE)
#if defined(DM9000_16BIT_DATA)
#define DM9000_DATA			(CONFIG_DM9000_BASE+0x300C)
#else
#define DM9000_DATA			(CONFIG_DM9000_BASE+1)
#endif
#endif
/****************************/


/***Modified by lk ***/
#define CFG_PHY_UBOOT_BASE	MEMORY_BASE_ADDRESS + 0x3e00000
#define CFG_PHY_KERNEL_BASE	MEMORY_BASE_ADDRESS + 0x8000



/* For s5p_sdhci */
#define SDHCI_MAX_HOSTS 4


#endif	/* __CONFIG_H */
```



## 20.tiny210_defconfig

```makefile
CONFIG_ARM=y
CONFIG_ARCH_S5PV210=y
CONFIG_TARGET_TINY210=y
CONFIG_IDENT_STRING=" for TINY210"
CONFIG_DEFAULT_DEVICE_TREE="s5pv210-tiny210"
CONFIG_BOOTDELAY=3
CONFIG_HUSH_PARSER=y
CONFIG_SYS_PROMPT="TINY210 # "
CONFIG_CMD_CACHE=y
CONFIG_CMD_FAT=y
```