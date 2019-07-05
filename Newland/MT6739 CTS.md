# **MT6739 CTS修改说明**

------------------------------

## 1. 去掉前摄后还有相关配置问题  
```c
android.app.cts.SystemFeaturesTest#testCameraFeatures   
fail:junit.framework.AssertionFailedError: Device does not have front-facing camera but reports either the FEATURE_CAMERA_FRONT or FEATURE_CAMERA_EXTERNAL feature
```
分析原因：没有前摄，但是仍有与前摄相关的配置打开了， 关掉前摄相关的配置
修改位置：android.hardware.camera.xml文件 去掉android.hardware.camera.front


## 2.虚拟按键问题
```c
android.accessibilityservice.cts.AccessibilityWindowQueryTest#testTraverseAllWindows    
android.server.cts.SplashscreenTests#testSplashscreenContent 
android.app.uiautomation.cts.UiAutomationTest#testWindowContentFrameStats    
```
分析原因：应该是扫码按钮导致的fail项    
修改位置：虚拟按键默认关闭

## 3.StorageHostTest问题
```c
android.appsecurity.cts.StorageHostTest#testCache    
android.appsecurity.cts.StorageHostTest#testVerifyAppStats    
android.appsecurity.cts.StorageHostTest#testVerifyStats    
android.appsecurity.cts.StorageHostTest#testVerifyStatsExternal    
android.appsecurity.cts.StorageHostTest#testVerifyStatsExternalConsistent    
android.appsecurity.cts.StorageHostTest#testVerifyStatsMultiple 
```
分析原因：内核CONFIG中缺少CONFIG_SDCARD_FS
修改位置：内核配置文件添加CONFIG_SDCARD_FS 创元代码中添加位置在Project.keng.mk和Project.kuser.mk中将CONFIG_SDCARD_FS宏控打开

## 4.APP DebuggableTest问题
```c
android.permission.cts.DebuggableTest#testNoDebuggable     
fail:junit.framework.AssertionFailedError: Packages marked debuggable: [com.newland.quicksetting, com.nlscan.android.scansettings, com.nl.nlscandemo, com.nlscan.providers.scan, 　　　com.nlscan.tool.touch]
```
分析原因：APP 非release


