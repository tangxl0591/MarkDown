# **新大陆Android杂项记录**

------------------------------

## 1.**编译的代码获取Commit ID**

repo forall -c "~/getcommitid.sh" > commitid.txt

getcommitid.sh的脚本代码如下

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