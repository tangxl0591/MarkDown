# **Android9 系统修改说明**

### 1、修改系统相机分辨率显示个数

```java
vendor/mediatek/proprietary/packages/apps/Camera2/feature/setting/picturesize/src/com/mediatek/camera/feature/setting/picturesize/PictureSizeSelector.java:61:    private static final int MAX_COUNT = 3;
```

### 2、修改ASSOCIATION_REJECTION失败后重试及进入黑名单修改

> **frameworks\opt\net\wifi\service\java\com\android\server\wifi\WifiConfigManager.java**

```java
重试次数修改
public static final int[] NETWORK_SELECTION_DISABLE_THRESHOLD = {
    -1, //  threshold for NETWORK_SELECTION_ENABLE
    1,  //  threshold for DISABLED_BAD_LINK
    5,  //  threshold for DISABLED_ASSOCIATION_REJECTION
    5,  //  threshold for DISABLED_AUTHENTICATION_FAILURE
    5,  //  threshold for DISABLED_DHCP_FAILURE
    5,  //  threshold for DISABLED_DNS_FAILURE
    1,  //  threshold for DISABLED_NO_INTERNET_TEMPORARY
    1,  //  threshold for DISABLED_WPS_START
    6,  //  threshold for DISABLED_TLS_VERSION_MISMATCH
    1,  //  threshold for DISABLED_AUTHENTICATION_NO_CREDENTIALS
    1,  //  threshold for DISABLED_NO_INTERNET_PERMANENT
    1,  //  threshold for DISABLED_BY_WIFI_MANAGER
    1,  //  threshold for DISABLED_BY_USER_SWITCH
    1   //  threshold for DISABLED_BY_WRONG_PASSWORD
};

重新恢复网络时间
public static final int[] NETWORK_SELECTION_DISABLE_TIMEOUT_MS = {
    Integer.MAX_VALUE,  // threshold for NETWORK_SELECTION_ENABLE
    15 * 60 * 1000,     // threshold for DISABLED_BAD_LINK
    5 * 60 * 1000,      // threshold for DISABLED_ASSOCIATION_REJECTION
    5 * 60 * 1000,      // threshold for DISABLED_AUTHENTICATION_FAILURE
    5 * 60 * 1000,      // threshold for DISABLED_DHCP_FAILURE
    5 * 60 * 1000,      // threshold for DISABLED_DNS_FAILURE
    10 * 60 * 1000,     // threshold for DISABLED_NO_INTERNET_TEMPORARY
    0 * 60 * 1000,      // threshold for DISABLED_WPS_START
    Integer.MAX_VALUE,  // threshold for DISABLED_TLS_VERSION
    Integer.MAX_VALUE,  // threshold for DISABLED_AUTHENTICATION_NO_CREDENTIALS
    Integer.MAX_VALUE,  // threshold for DISABLED_NO_INTERNET_PERMANENT
    Integer.MAX_VALUE,  // threshold for DISABLED_BY_WIFI_MANAGER
    Integer.MAX_VALUE,  // threshold for DISABLED_BY_USER_SWITCH
    Integer.MAX_VALUE   // threshold for DISABLED_BY_WRONG_PASSWORD
};
```

>**frameworks\opt\net\wifi\service\java\com\android\server\wifi\WifiStateMachine.java**

```java
+3900
switch (message.what) {
    case WifiMonitor.ASSOCIATION_REJECTION_EVENT:
        mWifiDiagnostics.captureBugReportData(
            WifiDiagnostics.REPORT_REASON_ASSOC_FAILURE);
        didBlackListBSSID = false;
        bssid = (String) message.obj;
        timedOut = message.arg1 > 0;
        reasonCode = message.arg2;
        Log.d(TAG, "Assocation Rejection event: bssid=" + bssid + " reason code="
              + reasonCode + " timedOut=" + Boolean.toString(timedOut));
        if (bssid == null || TextUtils.isEmpty(bssid)) {
            // If BSSID is null, use the target roam BSSID
            bssid = mTargetRoamBSSID;
        }
        if (bssid != null) {
            // If we have a BSSID, tell configStore to black list it
            didBlackListBSSID = mWifiConnectivityManager.trackBssid(bssid, false,
                                                                    reasonCode);
            //屏蔽此处代码
        }
```

### 3、屏幕支持180度旋转

```java
frameworks/base/core/res/res/values/config.xml 

 <!-- If true, the screen can be rotated via the accelerometer in all 4
         rotations as the default behavior. -->
    <bool name="config_allowAllRotations">true</bool>   这里默认是false的， 改成true，就支持180了
```

### 4、CPU RAM ROM修改说明

```c
drivers/cpufreq/cpufreq.c  修改CPU主频

#define show_one(file_name, object)			\
static ssize_t show_##file_name				\
(struct cpufreq_policy *policy, char *buf)		\
{							\
	return sprintf(buf, "%u\n", policy->object);	\
}

#define show_one_ex(file_name, object)			\
static ssize_t show_##file_name				\
(struct cpufreq_policy *policy, char *buf)		\
{							\
	return sprintf(buf, "%u\n", object);	\
}

show_one(cpuinfo_min_freq, cpuinfo.min_freq);
show_one_ex(cpuinfo_max_freq, 1190400);        //修改CPU主频位置
show_one(cpuinfo_transition_latency, cpuinfo.transition_latency);
show_one(scaling_min_freq, min);
show_one(scaling_max_freq, max);
show_one(scaling_cur_freq, cur);
show_one(cpu_utilization, util);

fs/proc/meminfo.c 修改RAM

@@ -105,7 +105,7 @@ static int meminfo_proc_show(struct seq_file *m, void *v)
                "AnonHugePages:  %8lu kB\n"
                ,
-               K(i.totalram),
+               K(i.totalram)+296932,
                K(i.freeram),
                K(i.bufferram),
                K(cached),

kernel/fs/statfs.c 修改ROM

SYSCALL_DEFINE2(statfs64, const char __user *, pathname, size_t, sz, struct statfs64 __user *, buf)
或者
SYSCALL_DEFINE3(statfs64, const char __user *, pathname, size_t, sz, struct statfs64 __user *, buf)
{
        struct kstatfs st;
        int error;
        if (sz != sizeof(*buf))
                return -EINVAL;
        error = user_statfs(pathname, &st);
        if (!error)
        {
                //NRE要求，修改ROM大小为8G，即/storgae/sdcard0目录大小增加4G
                if ((strcmp("/data", pathname) == 0) || (strcmp("/data/", pathname) == 0))
                {
                        st.f_blocks += (((u64)4*1024*1024*1024)/4096);
                        st.f_bfree += (((u64)4*1024*1024*1024)/4096);
                        st.f_bavail += (((u64)4*1024*1024*1024)/4096);
                }
                error = do_statfs64(&st, buf);
        }
        return error;
}
```

### 5.WIFI修改仅5G或者2.4G

> **vendor/mediatek/kernel_modules/connectivity/wlan/core/gen4m/rlm_domain.c**  

```bash
中国是  COUNTRY_CODE_CH g_u2CountryGroup4
附件是我 删了5G的
[SOLUTION] 
在rlm_domain.c 中 找到国家对应的group，再根据需求修改group 对应信道。

以秘鲁(peru)为例，若需要去掉2.4G channel 12 和 channel 13

1.先在rlm_domain.h 中搜索 peru, 找到COUNTRY_CODE_PE
在rlm_domain.c找到COUNTRY_CODE_PE 对应group为： g_u2CountryGroup5

2.修改 arSupportedRegDomains[] 中g_u2CountryGroup5 包含的channel 即可。
{81, BAND_2G4, CHNL_SPAN_5, 1, 13, FALSE} 改为{81, BAND_2G4, CHNL_SPAN_5, 1, 11, FALSE}

{81, BAND_2G4, CHNL_SPAN_5, 1, 13, FALSE} 中栏位说明如下，可查看struct _DOMAIN_SUBBAND_INFO
ucRegClass：81
ucBand:  BAND_2G4 (2.4G)
ucChannelSpan:  CHNL_SPAN_5(channel 以5MHz 为单位递增)
ucFirstChannelNum:  1 
ucNumChannels: 13 
fgDfs:  FALSE
各channel 对应频率 可以在网上搜索WLAN信道列表
```





