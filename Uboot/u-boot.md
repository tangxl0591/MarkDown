# **U-BOOT**

----------

## 一、定义介绍

> ** 1、u-boot.lds**
> 
```c
    OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
    /*指定输出可执行文件是elf格式，32位ARM指令，小端*/
    OUTPUT_ARCH(arm)
    /*指定输出可执行文件的平台为ARM*/
    ENTRY(_stext)
    /*指定输出可执行文件的起始代码段为_stext*/
    SECTIONS
    {
    /*指定可执行文件的全局入口点，通常这个地址都放在ROM(flash)0x0位置。必须使编译器知道这个地址，通常都是修改此处来完成*/
    . = 0x00000000;
    /*从0x0位置开始*/
    . = ALIGN(4);
    /*代码以4字节对齐*/
    .text :
    /*代码段*/
    {
        *(.__image_copy_start)
        /*u-boot将自己copy到RAM，此为需要copy的程序的start*/
        SOCDIR/start.o (.text*)
        /*./arch/arm/cpu/slsiap/s5p4418/start.S*/
        SOCDIR/vectors.o (.text*)
        /*./arch/arm/cpu/slsiap/s5p4418/vectors.S，异常向量表*/
        *(.text*)
        /*其他的代码段放在这里，即start.S/vector.S之后*/
    }
    . = ALIGN(4);
    /*代码段结束后，有可能4bytes不对齐了，此时做好4bytes对齐，以开始后面的.rodata段*/
    .rodata : { *(SORT_BY_ALIGNMENT(SORT_BY_NAME(.rodata*))) }
    /*在代码段之后，存放read only数据段*/
    . = ALIGN(4);
    /*和前面一样，4bytes对齐，以开始接下来的.data段*/
    .data : {
        *(.data*)
        /*可读写数据段*/
    }
    . = ALIGN(4);
    /*和前面一样，4bytes对齐*/
    . = .;
    . = ALIGN(4);
    .u_boot_list : {
        KEEP(*(SORT(.u_boot_list*)));
        /*.data段结束后，紧接着存放u-boot自有的一些function，例如u-boot command等*/
    }
    . = ALIGN(4);
    .image_copy_end :
    {
        *(.__image_copy_end)
        /*至此，u-boot需要自拷贝的内容结束，总结一下，包括代码段，数据段，以及u_boot_list*/
    }
    .rel_dyn_start :
    /*在老的uboot中，如果我们想要uboot启动后把自己拷贝到内存中的某个地方，只要把要拷贝的地址写给TEXT_BASE即可，然后boot启动后就会把自己拷贝到TEXT_BASE内的地址处运行，在拷贝之前的代码都是相对的，不能出现绝对的跳转，否则会跑飞。在新版的uboot里（2013.07），TEXT_BASE的含义改变了。它表示用户要把这段代码加载到哪里，通常是通过串口等工具。然后搬移的时候由uboot自己计算一个地址来进行搬移。新版的uboot采用了动态链接技术，在lds文件中有__rel_dyn_start和__rel_dyn_end，这两个符号之间的区域存放着动态链接符号，只要给这里面的符号加上一定的偏移，拷贝到内存中代码的后面相应的位置处，就可以在绝对跳转中找到正确的函数。*/
    {
        *(.__rel_dyn_start)
    }
    .rel.dyn : {
        *(.rel*)
        /*动态链接符存放在的段*/
    }
    .rel_dyn_end :
    {
        *(.__rel_dyn_end)
        /*动态链接符段结束*/
    }
    .end :
    {
        *(.__end)
    }
    _image_binary_end = .;
    /*bin文件结束*/
    /*
     * Deprecated: this MMU section is used by pxa at present but
     * should not be used by new boards/CPUs.
     */
    . = ALIGN(4096);
    .mmutable : {  /*for MMU*/
        *(.mmutable)
    }
    /*
     * Compiler-generated __bss_start and __bss_end, see arch/arm/lib/bss.c
     * __bss_base and __bss_limit are for linker only (overlay ordering)
     */
    /*bss段的描述*/
    .bss_start (OVERLAY) : {
        KEEP(*(.__bss_start));
        __bss_base = .;
    }
    .bss __bss_base (OVERLAY) : {
        *(.bss*)
         . = ALIGN(4);
         __bss_limit = .;
    }
    .bss_end __bss_limit (OVERLAY) : {
        KEEP(*(.__bss_end));
    }
    /*bss段的描述结束*/
    .dynsym _image_binary_end : { *(.dynsym) }
    .dynbss : { *(.dynbss) }
    .dynstr : { *(.dynstr*) }
    .dynamic : { *(.dynamic*) }
    .plt : { *(.plt*) }
    .interp : { *(.interp*) }
    .gnu.hash : { *(.gnu.hash) }
    .gnu : { *(.gnu*) }
    .ARM.exidx : { *(.ARM.exidx*) }
    .gnu.linkonce.armexidx : { *(.gnu.linkonce.armexidx.*) }
    }
```
> ** rodata**
> 存放只读数据段
> 
> ** data**
> 存放可读写数据段
> 
> ** bss**
> 通常指用来存放程序中未初始化的全局变量和静态变量的一块内存区域，特点是可读可写
> > 
> 
> ** 2、TEXT_BASE**
> 
> 链接器把UBOOT从地址TEXT_BASE 所有使用的目标地址寻址都是使用当前PC值加减偏移量的方法
  _start是可以动态变化，而TEXT_BASE是链接时就确定的地址，该值是BOOT在RAM中的偏移量
> 