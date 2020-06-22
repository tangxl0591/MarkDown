

# GMS测试说明

### 一.GMS测试环境

```bash
apt-get install openjdk-8-jdk python-dev lib32stdc++6 lib32z1 python-protobuf protobuf-compiler python-virtualenv python-pip python-numpy python-matplotlib python-scipy python-opencv
```

### 二.设备设置

> 1、确保语言选择英语
> 2、打开数据连接
> 3、连接 wifi，请确保网络信号稳定并且能够翻墙（部分要求 IPV6 的翻墙网络）
> 4、恢复出厂设置（刚刷机的手机丌需要此步骤）
> Settings-Backup&reset-Factoty data reset
> 5、打开开发者模式，并且请勾选以下选项
> Settings > Developer options > USB debugging
> Settings > Developer options > Stay awake
> Settings > Developer options > Disable HW overlays
> 6、Sleep 时间*Settings-Display-Sleep* 勾选 *30minutes*
> 7、时间选择 12 小时格式
> 8、请确保手机无锁屏，并且跑 CTS 的时候停留在 home 界面
> 9、 确保打开 location，手机测试时前后摄不能被遮挡  


## 一.GSI测试

```bash
1.MTK User load boot up to home screen
Please enable OEM unlocking in settings

2.LH:下面这部也可以直接通过  adb reboot bootloader ，进入
Press volume up key + power key into fastboot mode
Connect phone to PC and then type following commands

3.fastboot flashing unlock   注意按提示按下上侧键
4.fastboot erase system
5.fastboot flash system system.img (system.img is GSI)
6.fastboot --disable-verification flash vbmeta vbmeta.img
   (vbmeta.img is MTK's vbmeta.img. Please get it from MTK load.)

7.fastboot reboot

```

### 二.CTS测试

#### 1.测试单个模块

```bash
run cts -m <module>
```

#### 2.测试单个测试项

```bash
run cts -m <module> -t test_name
例如：
run cts -m CtsOsTestCases -t android.os.cts.BuildVersionTest#testBuildFingerprint
```

#### 3.测试添加subplan项

```bash
add subplan --session 0 --result-type passed --result-type failed --name 0
run cts --subplan newland_cts
```

> 此处的session 为l r查询到的ID信息，subplan中的xml相互间隔只有一个空格

```xml
<?xml version='1.0' encoding='UTF-8' standalone='no' ?>
<SubPlan version="2.0">
  <Entry include="CtsJniTestCases android.jni.cts.JniStaticTest" />
  <Entry include="arm64-v8a CtsJniTestCases" />
  <Entry include="armeabi-v7a CtsJniTestCases" />
</SubPlan>
```



#### 三.GMS   Sepolicy测试

```bas
Usage: sepolicy_tests -l $(ANDROID_HOST_OUT)/lib64/libsepolwrap.so -f vendor_file_contexts -f plat_file_contexts -p policy [--test test] [--help]

Options:
  -h, --help            show this help message and exit
  -f FILE, --file_contexts=FILE
  -p FILE, --policy=FILE
  -l FILE, --library-path=FILE
  -t TEST, --test=TEST  Test options include ['TestDataTypeViolators',
                        'TestProcTypeViolations', 'TestSysfsTypeViolations',
                        'TestDebugfsTypeViolations',
                        'TestVendorTypeViolations',
                        'TestCoreDataTypeViolations']



out/host/linux-x86/bin/sepolicy_tests -l out/host/linux-x86/lib64/libsepolwrap.so -f out/target/product/bird_k62v1_64_bsp/obj/ETC/plat_file_contexts_intermediates/plat_file_contexts  -f out/target/product/bird_k62v1_64_bsp/obj/ETC/vendor_file_contexts_intermediates/vendor_file_contexts  -p out/target/product/bird_k62v1_64_bsp/obj/ETC/sepolicy_intermediates/sepolicy 

out/host/linux-x86/bin/sepolicy_tests -l out/host/linux-x86/lib64/libsepolwrap.so -f out/target/product/bird_k62v1_64_bsp/obj/ETC/plat_file_contexts_intermediates/plat_file_contexts  -f out/target/product/bird_k62v1_64_bsp/obj/ETC/vendor_file_contexts_intermediates/vendor_file_contexts  -p out/target/product/bird_k62v1_64_bsp/obj/ETC/sepolicy_intermediates/sepolicy -t TestDataTypeViolations





```

### 三.GMS部分要求

| Prop                    | 名称   |
| ----------------------- | ------ |
| ro.build.display.id     | 版本号 |
| ro.product.model        | 型号   |
| ro.product.brand        | 品牌   |
| ro.product.name         | 名称   |
| ro.product.device       | 设备   |
| ro.product.board        | 制造商 |
| ro.product.manufacturer | 制造商 |

> 1.**ro.product.name** 产品名称 **ro.product.brand** 品牌 **ro.product.device** (设备名称) 保持一致
>
> 2.**ro.product.vendor.name=ro.product.name=ro.build.product**

### 四.NFT10 6762 Android 9 CTS问题总结

#### *CTS 错误*

##### 1.run cts -m CtsJniTestCases -t android.jni.cts.JniStaticTest#test_linker_namespaces

```java
junit.framework.AssertionFailedError: The library "/system/lib64/libnlsutils.so" is not a public library but it loaded.
```

> 错误原因：libnlsutils.so错误是添加到system/etc/public.librariy.txt中，需要区分SO的作用域
>
> 解决办法：编译libnlsutils.so system中使用，libnlsutils_vendor.so在vendor使用。



##### 2.run cts -m CtsAppSecurityHostTestCases -t android.appsecurity.cts.PermissionsHostTest #XXX

| 模块名称                             |
| ------------------------------------ |
| testInteractiveGrant23               |
| testGranted23                        |
| testNoResidualPermissionsOnUninstall |
| testNullAndRealPermission            |

```java
java.lang.AssertionError: Failed to successfully run device tests for com.android.cts.usepermission: Instrumentation run failed due to 'Process crashed.'
```

> 错误原因：权限定义位置不清
>
> 解决办法：厂商定制部分权限，需要加到non-plat中，包括file.te中的定义



##### 3.run cts -m CtsMediaTestCases -t android.media.cts.MediaRecorderTest#testSetMaxFileSize

```java
junit.framework.AssertionFailedError: timed out waiting for MEDIA_RECORDER_INFO_MAX_FILESIZE_REACHED
```

> 错误原因：任何影响摄像头打开速度，休眠唤醒屏幕的操作均会有此条错误
>
> 解决办法：去掉中间耗时动作



##### 4.run cts -m CtsHostsideWebViewTests -t com.android.cts.webkit.WebViewHostSideStartupTest#testStrictMode

> 错误原因：系统代码中在主UI线程不允许过多耗时，例如读写文件
>
> 解决办法：去掉耗时操作或者预先加载



##### 5.run cts -m CtsHiddenApiKillswitchWildcardTestCases -t android.signature.cts.api.WildcardKillswitchTest#testKillswitchMechanism

> 错误原因：HIDL服务中对空指针未判断，为何调用目前未查
>
> 解决办法：添加空指针判断



#### ***CTS-ON-GSI错误***

##### 1.run cts-on-gsi -m CtsSecurityHostTestCases -t android.security.cts.SELinuxHostTest

```java
junit.framework.AssertionFailedError: The following types on /data/ must be associated with the "data_file_type" attribute: nlswl_data_file wificfg_data_file
junit.framework.AssertionFailedError: The following types on /sys/ must be associated with the "sysfs_type" attribute: sysfs_newland
junit.framework.AssertionFailedError: file_contexts was invalid:
junit.framework.AssertionFailedError: The following types on /vendor/ must be associated with the "vendor_file_type" attribute: nlsapp_vendor_data_file nlsserver_exec
```

| 模块名称                |
| ----------------------- |
| testDataTypeViolators   |
| testSysfsTypeViolators  |
| testValidFileContexts   |
| testVendorTypeViolators |

> 错误原因：权限定义位置不清
>
> 解决办法：厂商定制部分权限，需要加到non-plat中，包括file.te中的定义



#### ***VTS错误***

##### 1.run vts -m VtsVndkDependency -t VtsVndkDependency#testElfDependency 

```java
9 != 0 Total number of errors: 9
host log:
/vendor/app/ScanViewDemo/lib/arm/libnlstools_jni.so: libandroid.so, libstdc++.so
/vendor/app/NLScanDemo/lib/arm/libwlt2bmp.so: libstdc++.so
/vendor/app/NLScanDemo/lib/arm/libemvjni.so: libnl_ndk.so, libnlposapi.so, libnlemv.so, libstdc++.so
/vendor/app/NLScanDemo/lib/arm/libIdcardMsg.so: libstdc++.so
/vendor/app/NLScanDemo/lib/arm/libsyd_comm.so: libstdc++.so
/vendor/app/NLScanDemo/lib/arm/libdewlt2-jni.so: libwlt2bmp.so, libstdc++.so
/vendor/app/PDATool/lib/arm/libnlstools_jni.so: libandroid.so, libstdc++.so
/vendor/app/ScanViewDemo/lib/arm64/libnlstools_jni.so: libandroid.so, libstdc++.so
/vendor/app/PDATool/lib/arm64/libnlstools_jni.so: libandroid.so, libstdc++.so
```

> 错误原因：第三方编译的APP so不满足vndk规则
>
> 解决办法：
>
> 1.NDK版本需要更新到比较新（R20）
>
> 2.JNI编译开关需要添加 **APP_STL := c++_shared**
>
> 3.JNI需要删除原有system下面SO引用 例如**libandrlod.so**

##### 2.run vts -m VtsTrebleVendorVintfTest -t VtsTrebleVendorVintfTest#DeviceManifest/SingleManifestTest.HalsAreServed/0_64bit

```java
test/vts-testcase/hal/treble/vintf/SingleManifestTest.cpp:45
```

> 错误原因：原有manifest中添加的scancamera HIDL，修改添加位置
>
> 解决办法：**DEVICE_MANIFEST_FILE += device/nlscan/Common/build/manifest/manifest.scancamera.xml**

***GTS错误***

##### 1.run gts -m GtsSecurityHostTestCases -t com.google.android.security.gts.SELinuxHostTest#testNoExemptionsForBinderInVendorBan

```java
junit.framework.AssertionFailedError: Policy exempts vendor domains from ban on Binder: [nlsserver
```

> 错误原因：nlsserver 服务编译在vendor  下，所依赖的SO需要也编译到vendor ，该native服务需要归属于vndmanagerservice
>
> 解决办法：
>
> 1,依赖的SO需要编译到vendor
>
> 2.该服务归属于vndservice
>
> 3.native服务中需要添加一句**ProcessState::initWithDriver("/dev/vndbinder");**

##### 2.run gts -m GtsUnofficialApisUsageTestCases -t com.android.gts.api.UnofficialApisUsageTest#testNonApiReferences

```java
junit.framework.AssertionFailedError: Undefined method ref: android.app.Notification.setLatestEventInfo(android.content.Context,java.lang.CharSequence,java.lang.CharSequence,android.app.PendingIntent)void from: /vendor/app/NLScanDemo/NLScanDemo.apk
```

> 错误原因：framework_class.jar太旧
>
> 解决版本：反射调用或使用同版本编译的JAR

##### 3.run gts -m GtsUnofficialApisUsageTestCases -t com.android.gts.api.UnofficialApisUsageTest#testTargetSdk 

```ja
junit.framework.AssertionFailedError: Vendor package com.nl.nlscandemo is targeting old SDK: 25
```

>  错误原因：SDK版本过低



***GTS错误***

##### 1.run gts -m GtsInstallPackagesWhitelistDeviceTestCases -t com.google.android.installpackageswhitelist.gts.GtsInstallPackagesWhitelistDeviceTest#testInstallerPackagesAgainstWhitelist

> 错误原因：检查哪些apk添加了android.permission.INSTALL_PACKAGES权限
>
> 解决办法：自带APP去除该权限即可



##### 2.run gts -m GtsSecurityHostTestCases -t com.google.android.security.gts.SELinuxHostTest#testNoExemptionsForDataBetweenCoreAndVendor

> 错误原因：Selinux权限归属问题，Data目录 Core与Vendor权限不能混用，也不能进行类型转换
>
> 解决办法：Data目录采用各自权限范围内读写