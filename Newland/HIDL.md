# **HIDL**

------------------------------

## 一、**编译环境及语法**

### 1.hidl-gen编译工具

*hidl-gen源码路径：system/tools/hidl*

```c
make  hidl-gen 

PACKAGE=vendor.mediatek.hardware.scancamera@1.0
LOC=vendor/mediatek/proprietary/hardware/interfaces/scancamera/1.0/default/

hidl-gen -o $LOC  -Lc++-impl -rvendor.mediatek.hardware:vendor/mediatek/proprietary/hardware/interfaces -randroid.hidl:system/libhidl/transport $PACKAGE 

hidl-gen -o $LOC -Landroidbp-impl -rvendor.mediatek.hardware:vendor/mediatek/proprietary/hardware/interfaces -randroid.hidl:system/libhidl/transport $PACKAGE

touch vendor.mediatek.hardware.scancamera@2.0-service.rc
touch sercive.cpp

./vendor/mediatek/proprietary/hardware/interfaces/update-makefiles.sh

生产hash加入current.txt
hidl-gen -L hash -rvendor.mediatek.hardware:vendor/mediatek/proprietary/hardware/interfaces -randroid.hidl:system/libhidl/transport $PACKAGE

```

*vendor.mediatek.hardware.scancamera@2.0-service.rc 内容*
```c

```


### 2.HIDL 的目录结构和软件包名称

ROOT-DIRECTORY

    MODULE
        SUBMODULE（可选，可以有多层）
            VERSION
                Android.mk
                IINTERFACE_1.hal
                IINTERFACE_2.hal
                …
                IINTERFACE_N.hal
                types.hal（可选）

*下表列出了软件包前缀和位置：*

| 软件包前缀        | 位置   |
| --------   |  :----:  |
| android.hardware.*     |  hardware/interfaces/*     |
| android.frameworks.*     |  frameworks/hardware/interfaces/*   |
| android.system.*     |  system/hardware/interfaces/*   |
| android.hidl.*     |  system/libhidl/transport/*   |
