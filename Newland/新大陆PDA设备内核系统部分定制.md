# **新大陆PDA设备内核部分定制说明V1.5**

------------------------------

## 一、**系统分区修改说明**

| 序号 | 功能   |
| ---- | :-----  |
| 1| 添加unrecoveryable分区，该分区支持烧写工具烧写IMG|
| 2| data分区下添加newland文件夹，需要读写及selinux权限|
| 3| init.rc中添加nlserver 服务的启动，备注1|
| 4| nlserver及解码服务中需要localsocket selinux权限支持|
| 5| 系统设置中，添加新大陆设置标签，里面包含内容为按键映射和扫描设置|
| 5| 提供system.img解包及封包工具|

> **备注1** :

```c
service nlsservice /system/bin/nlsserver
	class late_start
	user root
	group root system inet input graphics
	socket nlscan stream 0777 system system
```

```c
MT6735 selinux 权限
# system_server
allow system_server MTK_SMI_device:chr_file  { open read write ioctl };
allow system_server camera_isp_device:chr_file  { open read write ioctl };
allow system_server device:chr_file { open read write ioctl execute };
allow system_server kd_camera_hw_device:chr_file { open read write ioctl };
allow system_server kd_camera_flashlight_device:chr_file { read write };
allow system_server softd_device:chr_file { rw_file_perms execute };
# init
allow init fuse:dir search;
# system_app
allow system_app system_server:unix_stream_socket connectto;

# file_context
/dev/softd                  u:object_r:softd_device:s0
# device.te
type softd_device, dev_type;
```

## 二、**Property_set接口修改说明**

```c
int property_set(const char *key, const char *value)
{
...
    if(strcmp("switch_on", name) == 0)
    {
        property_switch = 1;
        return 0;
    }

    if(strcmp("switch_off", name) == 0)
    {
        property_switch = 0;
        return 0;
    }

    pi = (prop_info*) __system_property_find(name);

    if(pi != 0) {
        /* ro.* properties may NEVER be modified once set */
        if(property_switch == 1)
        {
            if(!strncmp(name, "ro.", 3)) return -1;
        }
...}
}
```
> **property_set函数中添加开关，可支持允许设置ro属性**

## 三、**硬件版本号说明**

| 序号 | 功能   |
| ---- | :-----  |
| 1| 系统中可通过ro.boot.hw_ver获取版本号|
| 2| recovery更新时使用的更新包中需包含支持的固件版本信息，在更新前recovery判断固件是否与当前硬件版本号相匹配|


## 四、CPU RAM ROM修改说明

> 系统中解析unrecoverable/xml/systeminfo.xml文件（文件内容如下），来对CPU等信息进行修改
> Path 路径根据不同驱动可做调整，无需按照下面名称进行修改
```xml
<?xml version="1.0" encoding="UTF-8"?>
<systeminfo>
    <info name="RAM">
		<path>/sys/bus/platform/drivers/k660dl/virtual_meminfo</path>
		<switch>1</switch><!-- 0 - off   1 - on --> 开关
        <value>1048576</value><!-- KB -->
    </info>
    <info name="ROM">
		<path>/sys/bus/platform/drivers/k660dl/virtual_rominfo</path>
		<switch>1</switch>
        <value>4949278720</value><!-- Byte -->
    </info>
    <info name="CPU">
		<path>/sys/bus/platform/drivers/k660dl/cpu_virtual_cpufreq</path>
		<switch>0</switch>
        <value>1300000</value><!-- Hz -->
    </info>
</systeminfo>
```
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

## 五、**键盘自动定义内核部分修改说明**

### 1.矩阵键盘
> 键盘布局下发接口,设备驱动名称必须为**/dev/nls_keyboard**
> 以下为范例说明：

```c
#define NLSKEY_MAGIC       0x78        // 该值不允许修改
#define NLSKEY_SET_TYPE        _IOW(NLSKEY_MAGIC, 0x01, unsigned int) //设置按键布局
#define NLSKEY_GET_TYPE        _IOW(NLSKEY_MAGIC, 0x02, unsigned int) //获取按键布局
#define NLSKEY_GET_SHITF       _IOW(NLSKEY_MAGIC, 0x03, unsigned int) //获取SHIFT操作模式
#define NLSKEY_SET_SHITF       _IOW(NLSKEY_MAGIC, 0x04, unsigned int) //设置SHIFT操作模式

typedef struct
{
    int type;               // 键盘类型 默认0
    int shifttype;          // shift操作类型 默认0  0 按下后切换状态 1 长按有效，放开无效
    int count;              // 按键个数
    int button[100];        // 键值（内核）
}tca1116_dev_keys;

static int tca1116_dev_open(struct inode *inode, struct file *filp)
{
	struct tca1116_chip_data *tca1116_dev = container_of(filp->private_data,
						struct tca1116_chip_data,
						tca1116_device);

	filp->private_data = tca1116_dev;

	pr_info("%d,%d \n", imajor(inode), iminor(inode));

	return 0;
}

static ssize_t tca1116_dev_write(struct file *filp, const char __user *buf,
		size_t count, loff_t *offset)
{
	struct tca1116_chip_data  *tca1116_dev;
	tca1116_dev_keys devkeys;
    int i;

	tca1116_dev = filp->private_data;

	if (count > sizeof(tca1116_dev_keys))
	{
	    count = sizeof(tca1116_dev_keys);
    }

	if (copy_from_user(&devkeys, buf, count))
    {
		return -EFAULT;
	}

    for (i = 0 ; i < tca1116_dev->nbutton; i++)
    {
        tca1116_dev->buttons[i].code = devkeys.button[i];
    }
    tca1116_dev->keyboard_type = devkeys.type;
    tca1116_dev->shift_type = devkeys.shifttype;
	return count;
}

static ssize_t tca1116_dev_read(struct file *filp, char __user *buf,
		size_t count, loff_t *offset)
{
	struct tca1116_chip_data *tca1116_dev = filp->private_data;
	tca1116_dev_keys devkeys;
	int i;

    for (i = 0 ; i < tca1116_dev->nbutton; i++)
    {
        devkeys.button[i]= tca1116_dev->buttons[i].code;
    }
    devkeys.count = tca1116_dev->nbutton;
    devkeys.type = tca1116_dev->keyboard_type;
    devkeys.shifttype = tca1116_dev->shift_type;
	if (copy_to_user(buf, &devkeys, sizeof(tca1116_dev_keys))) {
		return -EFAULT;
	}

	return count;

}

static long tca1116_dev_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret = 0;
	struct tca1116_chip_data *tca1116_dev = filp->private_data;

	switch (cmd)
    {
    	case NLSKEY_SET_TYPE:
            tca1116_dev->keyboard_type = (int)arg;
    		break;

        case NLSKEY_GET_TYPE:
            ret = tca1116_dev->keyboard_type;
    		break;

        case NLSKEY_SET_SHITF:
            tca1116_dev->shift_type = (int)arg;
    		break;

        case NLSKEY_GET_SHITF:
            ret = tca1116_dev->shift_type;
    		break;

    	default:
    		pr_info("bad ioctl %d \n", cmd);
    		return -EINVAL;
	}

	return ret;
}

static const struct file_operations tca1116_dev_fops = {
	.owner	= THIS_MODULE,
	.llseek	= no_llseek,
	.read	= tca1116_dev_read,
	.write	= tca1116_dev_write,
	.open	= tca1116_dev_open,
	.unlocked_ioctl  = tca1116_dev_ioctl,
```
> 应用通过IOCTRL接口，下发键盘数组列表进行定制，IOCTRL中所有命令必须全部支持

### 2.独立IO按键

> 如果扫描按键及侧边扫描按键为独立IO接口，可以使用独立IO接口下发该键上报内核键值
> 系统必须有可以下发键值的sys文件接口,接口名称位置不限

| 名称       | 功能   |
| --------   | :-----  |
| ro.keyboard.uhf      | UHF位置键值定义 |
| ro.keyboard.scanmain | ScanMain位置键值定义 |
| ro.keyboard.scanside | ScanSide位置键值定义 |

```xml
例如：主扫描键键值设置成240,侧边扫描键设置成241

echo 240 > /sys/bus/platform/driver/k660dl/main_scanner
echo 241 > /sys/bus/platform/driver/k660dl/side_scanner
```
### 3.功能键自定义

> 当前键盘布局功能键中有3种，系统中需要获取ro.build.shiftkey，获取当前功能键行为进行做出修改
> 根据键盘布局上的丝印显示功能进行执行

| 值     | 功能   |
| --------   | :-----  |
| 0 | 按下一次，切换状态(数字-大写-小写) |
| 1 | 长按有效，放开无效 shift+F1 输出F1|
| 2 | 按下后仅一次有效 shift 按下一次，再次按下F1 输出F1|

> **ro.build.FnKeyValue** 功能键键值 KeyEvent中（默认KEYCODE_SHIFT_LEFT），系统监控该键值作为功能键切换

### 4、按键映射
```xml
<?xml version="1.0" encoding="utf-8"?>
<KeyBoard Version="1.0">
<Property>
	<Keyboard name = "KEYBOARD_TYPE">0</Keyboard>
	<Keyboard name = "SHIFTKEY_TYPE">0</Keyboard>
	<Keyboard name = "SHIFTKEY_VALUE">KEYCODE_SHIFT_RIGHT</Keyboard>
	<Keyboard name = "KEYBOARD_MAX">25</Keyboard>
</Property>

<Kernel>
	<Keyboard name = "1" attr = "modify">KEYCODE_DEL</Keyboard>
	<Keyboard name = "2" attr = "modify">KEYCODE_BACK</Keyboard>
	<Keyboard name = "3" attr = "modify">KEYCODE_F2</Keyboard>
	<Keyboard name = "4">INVALID_KEY</Keyboard>
	<Keyboard name = "5">KEYCODE_MOVE_HOME</Keyboard>
	<Keyboard name = "6">INVALID_KEY</Keyboard>
	<Keyboard name = "7">INVALID_KEY</Keyboard>
	<Keyboard name = "8">INVALID_KEY</Keyboard>

	<Keyboard name = "9" attr = "modify">KEYCODE_F4</Keyboard>
	<Keyboard name = "10" attr = "modify">KEYCODE_F3</Keyboard>
	<Keyboard name = "11">KEYCODE_2</Keyboard>
	<Keyboard name = "12" attr = "modify">KEYCODE_F1</Keyboard>
	<Keyboard name = "13" attr = "modify">KEYCODE_MENU</Keyboard>
	<Keyboard name = "14">INVALID_KEY</Keyboard>
	<Keyboard name = "15">INVALID_KEY</Keyboard>
	<Keyboard name = "16">INVALID_KEY</Keyboard>

	<Keyboard name = "17" attr = "modify">KEYCODE_ENTER</Keyboard>
	<Keyboard name = "18" attr = "modify">KEYCODE_SPACE</Keyboard>
	<Keyboard name = "19">KEYCODE_3</Keyboard>
	<Keyboard name = "20">KEYCODE_1</Keyboard>
	<Keyboard name = "21" attr = "modify">KEYCODE_SHIFT_RIGHT</Keyboard>
	<Keyboard name = "22">INVALID_KEY</Keyboard>
	<Keyboard name = "23">INVALID_KEY</Keyboard>
	<Keyboard name = "24">INVALID_KEY</Keyboard>


	<Keyboard name = "25">KEYCODE_0</Keyboard>
	<Keyboard name = "26">KEYCODE_6</Keyboard>
	<Keyboard name = "27">KEYCODE_5</Keyboard>
	<Keyboard name = "28">KEYCODE_4</Keyboard>
	<Keyboard name = "29" attr = "modify">KEYCODE_CALL</Keyboard>
	<Keyboard name = "30">INVALID_KEY</Keyboard>
	<Keyboard name = "31">INVALID_KEY</Keyboard>
	<Keyboard name = "32">INVALID_KEY</Keyboard>

	<Keyboard name = "33">KEYCODE_MOVE_END</Keyboard>
	<Keyboard name = "34">KEYCODE_PERIOD</Keyboard>
	<Keyboard name = "35">KEYCODE_9</Keyboard>
	<Keyboard name = "36">KEYCODE_8</Keyboard>
	<Keyboard name = "37">KEYCODE_7</Keyboard>
	<Keyboard name = "38">INVALID_KEY</Keyboard>
	<Keyboard name = "39">INVALID_KEY</Keyboard>
	<Keyboard name = "40">INVALID_KEY</Keyboard>

	<Keyboard name = "1" attr = "modify">KEYCODE_HOME</Keyboard>
	<Keyboard name = "2" attr = "modify">KEYCODE_SCAN_SIDE</Keyboard>
</Kernel>

<MapKey>
    <enum name="KEYCODE_0" value="7" />
    <enum name="KEYCODE_1" value="8" />
    <enum name="KEYCODE_2" value="9" />
    <enum name="KEYCODE_3" value="10" />
    <enum name="KEYCODE_4" value="11" />
    <enum name="KEYCODE_5" value="12" />
    <enum name="KEYCODE_SCAN_MAIN" value="240" />
    <enum name="KEYCODE_SCAN_SIDE" value="241" />
</MapKey>
```
>**脚本位置及名称**，查找顺序从上到下

| 序号     | 功能   |
| --------   | :-----  |
| 0 | /unrecoverable/xml/KeyBoard.xml |
| 1 | /system/usr/xml/KeyBoard.xml |

> 1.系统中需要解析attr = "modify"字段，包含这个字段的节点按键，允许被映射
> 2.INVALID_KEY为无效按键，不允许映射
> 3.MapKey节点那些按键映射，被允许映射的按键范围。
> 4.Scanner为独立按键及扫描键自定义字段
> 5.Ctrl Alt需要实现黏滞效果
> 6.添加矩阵个数 KEYBOARD_MAX
> 需要在系统设置中添加按键映射菜单


## 五、Camera获取图像接口

> 系统中需实现libScanCamera.so的库文件，实现功能为获取图像数据，**请特别注意Camera库与后置摄像头互斥问题**
> 在CameraService中添加接口仅判断Camera 0,打开关闭都通知系统上层

```c
typedef enum
{
    STREAM_Y   =   0x01,
    STREAM_U   =   0x02,
    STREAM_V   =   0x03,
    STREAM_UV  =   0x04,
	  STREAM_RAW8 =   0x05,
    STREAM_RAW10 =   0x06,
    STREAM_RAW12 =   0x07,
}STREAM_FMT;

typedef enum
{
    MM_CAMERA_NOTIFY_NONE   = 0x00,
    MM_CAMERA_NOTIFY_KILL   = 0x01,
    MM_CAMERA_NOTIFY_DIED   = 0x02,
    MM_CAMERA_NOTIFY_CLOSE  = 0x03, // 摄像头关闭
    MM_CAMERA_NOTIFY_OPEN   = 0x04, // 摄像头打开
}MM_CAMERA_NOTIFY;

typedef enum
{
    CAMERA_BACK  = 0x00,
    CAMERA_FRONT = 0x01,
}CAMERA_DIR;

typedef int (*module_stream_proc)(STREAM_FMT fmt, char* Buf, unsigned long BufLen); // 数据回调
typedef int (*module_nofity_proc)(int state, int param1, void* param2);  // 状态回调

/*************************************************
 Function:		cam_init
 Descroption:	摄像头初始化接口
 Input:
	1.channel   通道 CAMERA_DIR
	2.format    格式 STREAM_FMT
	3.Width     宽
	4.Height    高
	5.proc      数据回调
	6.notify    状态回调
 Output:
 Return: 	    内部私有指针
 Other:
*************************************************/
void* cam_init(int channel, int format, int Width, int Height, module_stream_proc proc, module_nofity_proc notify);

/*************************************************
 Function:		cam_open
 Descroption:	摄像头打开
 Input:
	1.camera    私有指针
 Output:
 Return: 	    0 成功 非0失败
 Other:
*************************************************/
int cam_open(void* camera);

/*************************************************
 Function:		cam_close
 Descroption:	摄像头关闭
 Input:
	1.camera    私有指针
 Output:
 Return: 	    0 成功 非0失败
 Other:
*************************************************/
int cam_close(void* camera);

/*************************************************
 Function:		cam_start
 Descroption:	启动传图
 Input:
	1.camera    私有指针
 Output:
 Return: 	    0 成功 非0失败
 Other:
*************************************************/
int cam_start(void* camera);

/*************************************************
 Function:		cam_stop
 Descroption:	停止传图
 Input: 		None
 Output:
 Return: 	    0 成功 非0失败
 Other:
*************************************************/
int cam_stop(void* camera);

/*************************************************
 Function:		cam_ioctrl
 Descroption:	控制接口(保留接口，目前未使用)
 Input:
	1.camera    私有指针
	2.cmd
	3.param1
	4.param2
 Output:
 Return: 	0 成功 非0失败
 Other:
*************************************************/
int cam_ioctrl(void* camera, int cmd, int param1, void* param2);
```

## 六、主界面快捷方式定制

> 系统解析/unrecoverable/xml/default_workspace.xml 文件进行判断桌面快捷方式可以自定义设置

```xml
<favorites xmlns:launcher="http://schemas.android.com/apk/res-auto/com.android.launcher3">
    <!-- Far-left screen [0] -->
    <appwidget
        launcher:packageName="com.android.deskclock"
        launcher:className="com.android.alarmclock.DigitalAppWidgetProvider"
        launcher:screen="0"
        launcher:x="0"
        launcher:y="0"
        launcher:spanX="4"
        launcher:spanY="2" />

    <favorite
        launcher:packageName="suncere.zongzhan.androidapp"
        launcher:className="suncere.zongzhan.androidapp.LanuchActivity"
        launcher:screen="0"
        launcher:x="0"
        launcher:y="3" />

    <favorite
        launcher:packageName="suncere.suncere.yunwei.app"
        launcher:className="suncere.suncere.yunwei.app.MainActivity"
        launcher:screen="0"
        launcher:x="1"
        launcher:y="3" />

    <favorite
        launcher:packageName="com.newland.appstore"
        launcher:className="com.newland.appstore.MainActivity"
        launcher:screen="0"
        launcher:x="2"
        launcher:y="3" />
```

## 七、NLSConfig.xml通用配置解析
| 字段名称    | 功能   |
| --------   | :-----  |
| LibExDecode       | 外部解码库 |
| LibExDecodeType   | 外部解码库,码制类型 |
| LibU3Focus       | 外部校正库 |
| DebugLevel       | BUG级别输出 |
| DebugFile        | BUG输出文件位置 |

## 八、内核扫描头驱动部分参数
| 名称 |I2c | ID | Width | Height | Line | Frame | FPS | Settle | Lane | Format |
| ---  | :--- | :------ | :--- | :----- | :--- | :--- | :-- | :--- | :-- | :----- |
| CM36 | 0xC0  | 0x300A/0x7750   | 640  | 480  | 800  | 500  | 54 |  85	| 1 |  RAW10 |
| CM48 | 0x12  | 0xF0/0x0A   | 1280   | 960    | 1588 | 997  | 52 |  85 | 1 |  RAW8  |
| CM46 | 0x12  | 0xF0/0x08   | 752   | 480    | 846   | 525  | 54 |  85	| 1 |  YUV420  |
| SE4750 | 0xB8  | 0x7A/0x82 | 1360  | 960    | 1388  | 997  | 54 |  45	| 1 |  RAW8  |
| CM60 | 0x12  | 0xF0/0x0B   | 1280   | 800    | 1588 | 997  | 54 |  85 | 1/2 |  RAW8  |

## 九、CustLauncher.xml开机自运行功能
> 系统解析/unrecoverable/xml/CustLauncher.xml 文件进行判断开机自动运行APP，如果APP不存在启动默认桌面

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Launchers  Version="1.0">
 <Launcher name = "Launcher1">
  <package>com.genlot.scjk</package>
  <className>com.genlot.scjk.activity.LoginActivity</className>
 </Launcher>
</Launchers>
```

## 十、内核电源控制部分
1、扫描部分（驱动名称需定为scan_switch）

| 名称       | 功能   |
| --------   | :-----  |
| power_on      | 扫描电源开 |
| power_off     | 扫描电源关|
| trigger_on    | 扫描触发开|
| trigger_off    | 扫描触发关|
| aimer_on      | 瞄准灯开|
| aimer_off      | 瞄准灯关|
| illum_on      | 补光灯开|
| illum_off      | 补光灯开|

> 例如:echo power_on > scan_switch

2、扫描串口流控 （驱动名称需定为uart_cts uart_rts）

| 名称       | 功能   |
| --------   | :-----  |
| 1      | 置1 |
| 0      | 置0 |

> 例如:echo 1 > uart_rts

3、PSAM卡控制（驱动名称需定为psam_rst psam_en）

| 名称       | 功能   |
| --------   | :-----  |
| 1      | 置1 |
| 0      | 置0 |

## 十一、内核信息部分
1、硬件版本号部分

```c
/sys/devices/platform/pcba_dev/pcba_version 该节点为硬件版本号节点
```

## 十二、系统预装APP部分

| 名称       | 功能   |
| --------   | :-----  |
| 不可卸载    | 拷贝至/system/app |
| 可卸载       | 系统中放置位置/system/vendor/appback 由程序nlserver拷贝至/data/app 并修改0666权限，Android8之后采用安装器方式从该文件夹下安装 |

# Property属性汇总列表

| 名称       | 功能   |
| --------   | :-----  |
| ro.boot.hw_ver       | 设备硬件版本号 |
| ro.build.display.ex  | 设备实际软件版本号，OTA使用 |
| ro.build.display.id  | 设备软件版本号，显示使用 |
| ro.build.display.config  | 设备配置分区版本号 |
| ro.build.keyboard    | 键盘布局 0 通用 1 GOT布局 2 SC布局 3唯品会|
| ro.build.shiftkey    | shift 行为 0 按下后切换状态 1 长按有效，放开无效 2 按下后仅一次有效|
| ro.build.FnKeyValue  | 功能键键值 KeyEvent中，MT65默认KEYCODE_SHIFT_RIGHT MT65W默认KEYCODE_SHIFT_LEFT|
| ro.cpu.id            | 高通设备修改CPU ID *|
| ro.keyboard.uhf      | UHF位置键值定义 *|
| ro.keyboard.scanmain | ScanMain位置键值定义 *|
| ro.keyboard.scanside | ScanSide位置键值定义 *|
| ro.config.version    | 配置包版本号|
| ro.build.scanner.api | 0 通用 1 体彩		  |
| persist.sys.wait     | CAMERA probe完成探测标志 *|
| persist.sys.type     | OTA升级使用设备型号查询 |
| persist.sys.uhf      | UHF功能 1支持 0不支持 |
| persist.sys.scan.display | 显示识读头型号 |
| persist.sys.scan.info    | 实际识读头型号 |
| persist.sys.induction.image    | 感应图像稳定 |
| persist.sys.induction.sensor    | 感应灵敏度 |
| persist.sys.subcamera| HAL层修改每次打开时候判断该prop获取当前是否能够打开前置摄像头（获取摄像头个数及connect接口需要添加该判断） 0 隐藏 1 打开 2 全部隐藏 |
| persist.sys.logcat.kernel | 抓取KERNEL LOG *|
| persist.sys.logcat.android | 抓取LOGCAT LOG *|
| persist.sys.jni.type| JNI 启动类型 0非阻塞 1阻塞 |
| persist.sys.jni.driver| JNI 中使用的驱动节点判断的xml |
| persist.sys.netsend| ARP发送开关，通用版本默认关闭 |
| persist.sys.netsendtimer| ARP发送时间 5次 |
| persist.sys.decodekey| 与芯片交互KEY状态异常  |
| persist.sys.chipverify| 芯片认证状态  |
| ro.platform.id   | 隐藏的CPU ID  当前MT6735使用用于区别mt6735和mt6737|
| ro.boot.hw_ver  |  硬件版本号  |
| ro.config.scanmode    | 触发模式支持|
| persist.sys.tp.version  |  触摸屏版本号 |
| persist.sys.dhcp.mode  | DHCP 发送模式，0 采用Renew 1 StartDhcp，默认1  |
| persist.sys.dhcp.setaddr  | DHCP 下设置IP，0 不设置 1 设置，默认1  |
| persist.sys.camdisableview  | 系统屏蔽相机view  |
| persist.sys.wpa.skip  | wpa eapol skip check1，0 不跳过 1 跳过，默认0  |
| persist.sys.blist.skip  | Black List skip check1，0 不跳过 1 跳过，默认0  |
| persist.sys.scan.flashled  | 扫描引擎手电，0 不支持 1 支持，默认0,由JNI检测设置 |
| persist.sys.scan.flashstate  | 扫描引擎手电状态  |
| persist.sys.scan.flashswitch | 扫描引擎手电开关设置（用于强行关闭该功能） |
| persist.sys.scan.de3  | 判断是否是DE3扫描头 0 不是 1 是，默认0  |
| persist.sys.nlsgms  | 判断是否是进入GMS测试状态 0 不是 1 是，默认0  |
| persist.sys.forceview  | 是否显示 0 不是 1 是，默认0  |
| persist.sys.screendim  | 背光进入省电POLICY_DIM时间 DimTime  = 显示时间- POLICY_DIM等待时间 如果是0 按原有值进行|
| persist.sys.scan.gms  | 是否GMS测试模式下 0 不是 1 是，默认0  |
| persist.sys.nls.libsuport | 读码服务是否加载lib |
|  |  |

nlsserver native服务

| 名称       | 功能   | 兼容名称   |
| --------   | :-----  | :-----  |
| ro.boot.hw_ver | 硬件版本号 | - |
| ro.boot.hardware.revision | 硬件版本号 | - |
| ro.boot.boardid | 频段ID，从内核读出值  | - |
| ro.boot.boardtype | 频段名称 | - |
| ro.boot.hwversionid | 硬件ID，从内核读出值 | - |
| ro.platform.id   | CPU ID | - |
| ro.build.FnKeyValue |功能键名称| |
| ro.build.shiftkey |功能键类型| |
| ro.build.keyboard |键盘类型| |
| persist.sys.batteryid   | 电池ID | |
| persist.sys.tp.version   | 触摸屏版本 | |
| persist.sys.lcd.version   | LCD版本 | |
| persist.ro.BtMac   | BT地址 | |
| persist.ro.WifiMac  | Wifi地址| |

> 当前键盘布局功能键中有3种，系统中需要获取ro.build.shiftkey，获取当前功能键行为进行做出修改
> 根据键盘布局上的丝印显示功能进行执行

| 值   | 功能                                               |
| ---- | :------------------------------------------------- |
| 0    | 按下一次，切换状态(数字-大写-小写)                 |
| 1    | 长按有效，放开无效 shift+F1 输出F1                 |
| 2    | 按下后仅一次有效 shift 按下一次，再次按下F1 输出F1 |

> **ro.build.FnKeyValue** 功能键键值 KeyEvent中（默认KEYCODE_SHIFT_LEFT），系统监控该键值作为功能键切换