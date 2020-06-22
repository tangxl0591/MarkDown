# **BareBox Tiny201**



## 一.**编译脚本**

```bash
export PATH=/home/nltxl/home4/nltxl/linux/opt/7.4.1/bin:$PATH
cp arch/arm/configs/friendlyarm_tiny210_defconfig .config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```

