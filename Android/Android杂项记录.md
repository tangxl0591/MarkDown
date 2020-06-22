# **Android杂项记录**

------------------------------

## 1.**编译的代码获取Commit ID**

```bash
repo forall -c "~/getcommitid.sh" > commitid.txt
```

> getcommitid.sh的脚本代码如下

```bash
COMMITID=$(git log -1 --pretty=format:%H)
echo "PROJECT=${REPO_PROJECT};PATH=${REPO_PATH};COMMITID=${COMMITID}"
```

## 2.**OTA生成脚本**

**Android 7.0**

```bash
./build/tools/releasetools/ota_from_target_files --block -p ./out/host/linux-x86 -k ./build/target/product/security/testkey -s ./device/mediatek/build/releasetools/mt_ota_from_target_files -i  
```

**Android 8.1**

```bash
./build/tools/releasetools/ota_from_target_files --block -p ./out/host/linux-x86 -k ./build/target/product/security/testkey -s  ./vendor/mediatek/proprietary/scripts/releasetools/mt_ota_from_target_files -i
```

**Android9.1**

```bash
./build/tools/releasetools/ota_from_target_files -k ./build/target/product/security/testkey -i
```

## 3.**repo下载指定TAG**

```bash
repo init -u ssh://gerrit/MT90_6739_81/platform/manifest.git --repo-url=ssh://gerrit/repo.git --no-repo-verify -m MT90_6739_81_GYY_V3.00.003.7182.xml
```

## 4.**monkey命令执行**

```bash
adb shell monkey  --ignore-crashes --ignore-timeouts  --ignore-security-exceptions --monitor-native-crashes  --throttle 100 100000000
```

## 5. **git 提交出现钩子异常**

```bash
git push origin HEAD:refs/for/master
git commit --amend
```

## 6. **addr2line使用**

```bash
addr2line -C -f -e libnetfilter.so 7ffff7bcd1d2
```

## 7. **Android.bp预编译so**

```bash
cc_prebuilt_library_shared {
    name: "libnlsutils",
    compile_multilib: "both",
    multilib: {
        lib64: {
            srcs: ["./libnlsutils/arm64/libnlsutils.so"],
        },
        lib32: {
            srcs: ["./libnlsutils/arm/libnlsutils.so"],
        },        
    },

    strip: {
        none: true,
    },    
}
```

## 8. **Android Zip包 APK签名脚本**

**Android 8.1**

```bash
java -Xmx2048m -Djava.library.path=out/host/linux-x86/lib64 -jar out/host/linux-x86/framework/signapk.jar -w build/target/product/security/testkey.x509.pem build/target/product/security/testkey.pk8 
```

**Android 9**

```bash
java -Xmx2048m -Djava.library.path=out/host/linux-x86/lib64 -jar out/host/linux-x86/framework/signapk.jar -w build/target/product/security/testkey.x509.pem build/target/product/security/testkey.pk8 
```

## 9. **Android Studio 打包jar**

```bash
gradlew makejar
```

## 10.sdkmanager更新SDK

```bash
sdkmanager --list  查看
sdkmanager "platform-tools" "build-tools;28.0.3" "platforms;android-28" 安装
sdkmanager --update  更新
```

## 11.ADB Tcp Debug

```bash
setprop service.adb.tcp.port 5555 
```

## 12.Android.mk转Android.bp

```bash
androidmk  Android.mk > Android.bp
```

## 13.设备解锁verity

> 开发者选项中选择OEM unlock
>
> fastboot flashing unlock

## 14.生成testkey

```bash
#!/bin/bash
openssl genrsa -3 -out $1.pem 2048
openssl req -new -x509 -key $1.pem -out $1.x509.pem -days 10000 \
    -subj '/C=US/ST=FuJian/L=Fuzhou View/O=Platfrom_KEY/OU=NewLand_NFT10_BIRD_Android9ReleaseKey/CN=AutoID/emailAddress=tangxl0591@gmail.com'
openssl pkcs8 -in $1.pem -topk8 -outform DER -out $1.pk8 -nocrypt
```

```bash
development/tools/make_key mykey '/C=US/ST=California/L=Mountain View/O=Android/OU=Android/CN=Android/emailAddress=android@android.com'
```

## 15.verity说明及生成方式

| 名称            | 作用                                             |
| --------------- | ------------------------------------------------ |
| verity.pk8      | private key used to sign boot.img and system.img |
| verity.x509.pem | certificate include public key                   |
| verity_key      | public key used in dm verity for system.img      |

**生成 generate_verity_key**

```bash
source build/envsetup.sh
choosecombo
make generate_verity_key (mmm system/extras/verity/)
```

**将 *.x509.pem 转换成 verity key**

```bash
out/host/linux-x86/bin/generate_verity_key -convert mykey.x509.pem verity_key
```

**拷贝并重命名**

> 拷贝mykey.pk8，mykey.x509.pem，verity_key.pub 至 build/target/product/security/ 目录，将其重命名: verity.pk8， verity.x509.pem，verity_key ，并替换默认的开发 key。

## 16.环境变量配置

**Android SDK**

```bash

```

## 17.JACK环境设置

**.jack-settings**

```bash
# Server settings
SERVER_HOST=localhost
SERVER_PORT_SERVICE=8100 --修改处
SERVER_PORT_ADMIN=8101 --修改处

# Internal, do not touch
SETTING_VERSION=4
```

**jack-server/config.properties**

```bash
#Thu Dec 13 16:31:05 CST 2018
jack.server.idle=180
jack.server.max-service.by-mem=1\=2147483648\:2\=3221225472\:3\=4294967296
jack.server.shutdown=21600
jack.server.time-out=7200
jack.server.max-jars-size=104857600
jack.server.service.port=8100 --修改处
jack.server.admin.port=8101 --修改处
jack.server.config.version=4
jack.server.max-service=4
jack.server.deep-idle=900

```

