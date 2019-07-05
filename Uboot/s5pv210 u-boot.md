# **S5VP210 U-BOOT Porting**

----------

## 一、编译命令

> ** 1、make O=输出目录**
> ** 2、make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi- smdkv210_defconfig


## 二、修改步骤

### 1、arch/arm/Kconfig添加ARCH_S5PV2XX

```c
config ARCH_S5PV2XX
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
machine-$(CONFIG_ARCH_S5PV2XX)  += s5pv2xx
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

#ifdef CONFIG_SPL_BUILD
void s5pv210_clock_init(void)
{
    u32 val = 0;

    struct s5pv210_clock *const clock = (struct s5pv210_clock *)samsung_get_base_clock();

    writel(0xFFFF, &clock->apll_lock);
    writel(0xFFFF, &clock->mpll_lock);
    writel(0xFFFF, &clock->epll_lock);
    writel(0xFFFF, &clock->vpll_lock);

    writel((3  << 8) | (125 << 16) | (1 << 0) | (1 << 31), &clock->apll_con0);      /* FOUT_APLL = 1000MHz */
    writel((12 << 8) | (667 << 16) | (1 << 0) | (1 << 31), &clock->mpll_con);       /* FOUT_MPLL = 667MHz */
    writel((3  << 8) | (48  << 16) | (2 << 0) | (1 << 31), &clock->epll_con0);      /* FOUT_EPLL = 96MHz */
    writel((6  << 8) | (108 << 16) | (3 << 0) | (1 << 31), &clock->vpll_con);       /* FOUT_VPLL = 54MHz */

    while (!(readl(&clock->apll_con0) & (1 << 29)));
    while (!(readl(&clock->mpll_con) & (1 << 29)));
    while (!(readl(&clock->apll_con0) & (1 << 29)));
    while (!(readl(&clock->epll_con0) & (1 << 29)));
    while (!(readl(&clock->vpll_con) & (1 << 29)));

    /* MOUT_MSYS = SCLKAPLL = FOUT_APLL = 1000MHz
    ** MOUT_DSYS = SCLKMPLL = FOUT_MPLL = 667MHz
    ** MOUT_PSYS = SCLKMPLL = FOUT_MPLL = 667MHz
    ** ONENAND = HCLK_PSYS
    */

    writel((1 << 0) | (1 << 4) | (1 << 8) | (1 << 12), &clock->clk_src0);

    val =   (0 << 0)  |     /* APLL_RATIO = 0, freq(ARMCLK) = MOUT_MSYS / (APLL_RATIO + 1) = 1000MHz */
            (4 << 4)  |     /* A2M_RATIO = 4, freq(A2M) = SCLKAPLL / (A2M_RATIO + 1) = 200MHz */
            (4 << 8)  |     /* HCLK_MSYS_RATIO = 4, freq(HCLK_MSYS) = ARMCLK / (HCLK_MSYS_RATIO + 1) = 200MHz */
            (1 << 12) |     /* PCLK_MSYS_RATIO = 1, freq(PCLK_MSYS) = HCLK_MSYS / (PCLK_MSYS_RATIO + 1) = 100MHz */
            (3 << 16) |     /* HCLK_DSYS_RATIO = 3, freq(HCLK_DSYS) = MOUT_DSYS / (HCLK_DSYS_RATIO + 1) = 166MHz */
            (1 << 20) |     /* PCLK_DSYS_RATIO = 1, freq(PCLK_DSYS) = HCLK_DSYS / (PCLK_DSYS_RATIO + 1) = 83MHz */
            (4 << 24) |     /* HCLK_PSYS_RATIO = 4, freq(HCLK_PSYS) = MOUT_PSYS / (HCLK_PSYS_RATIO + 1) = 133MHz */
            (1 << 28);      /* PCLK_PSYS_RATIO = 1, freq(PCLK_PSYS) = HCLK_PSYS / (PCLK_PSYS_RATIO + 1) = 66MHz */
    writel(val, &clock->clk_div0);
}
#endif

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

static inline void s5p_set_cpu_id(void)
{
	s5p_cpu_id = readl(S5PV210_PRO_ID);
	s5p_cpu_rev = s5p_cpu_id & 0x000000FF;
	s5p_cpu_id = 0xC000 | ((s5p_cpu_id & 0x00FFF000) >> 12);
}

...

IS_SAMSUNG_TYPE(s5pc100, 0xc100)
IS_SAMSUNG_TYPE(s5pc110, 0xc110)
IS_SAMSUNG_TYPE(s5pv210, 0xc000)

#ifdef CONFIG_ARCH_S5PV2XX
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
#ifdef CONFIG_ARCH_S5PV2XX
SAMSUNG_BASE(dmc0,DMC0_BASE)
SAMSUNG_BASE(dmc1,DMC1_BASE)
#endif

```

> **arm\mach-s5pv2xx\include\mach\clock.h**

```c
struct s5pv210_clock {
    unsigned int    apll_lock;
    unsigned char   res1[0x04];
    ···
};
```
> **arch\arm\mach-s5pv2xx\Kconfig**

```c
if ARCH_S5PV2XX

choice
	prompt "S5PV2XX board select"
	optional

config TARGET_SMDKV210
	bool "Support smdkv210 board"
	select OF_CONTROL
	select SUPPORT_SPL
	select SPL

endchoice

config SYS_SOC
	default "s5pv2xx"

source "board/samsung/smdkv210/Kconfig"

endif
```
### 4、修改arch/arm/cpu/armv7/Makefile 添加s5pv2xx

```c
ifneq (,$(filter s5pc1xx s5pv2xx exynos,$(SOC)))
```
### 5、修改arch/arm/cpu/armv7/s5p-common/cpu_info.c

```c
unsigned int s5p_cpu_id = 0xC000;
```

### 6、修改arch/arm/dts/Makefile

```c
dtb-$(CONFIG_S5PV210) += s5pv2xx-smdkv210.dtb
```

### 7、添加arch/arm/dts/s5pv2xx-smdkv210.dts

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
	compatible = "samsung,smdkv210", "samsung,s5pv210";

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

```c
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
### 12、添加board\samsung\smdkv210\Kconfig

```c
if TARGET_SMDKV210

config SYS_BOARD
	default "smdkv210"

config SYS_VENDOR
	default "samsung"

config SYS_SOC
	default "s5pv2xx"

config SYS_CONFIG_NAME
	default "smdkv210"

endif
```
### 13、添加board\samsung\smdkv210\Makefile

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

hostprogs-y := tools/mksmdkv210spl
always := $(hostprogs-y)
endif

obj-y	:= smdkv210.o
obj-$(CONFIG_SAMSUNG_ONENAND)	+= onenand.o
obj-y	+= lowlevel_init.o

obj-$(CONFIG_SPL_BUILD) += smdv210-spl.o
```

### 14、修改board\samsung\smdkc100\lowlevel_init.S

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

### 15、board\samsung\smdkv210\smdkv210.c 参考smdkc100.c

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
