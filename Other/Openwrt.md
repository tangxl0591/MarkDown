# OpenWrt

#### 1.安装SSR

```bash
https://github.com/tangxl0591/luci-app-shadowsocksr
https://github.com/tangxl0591/openwrt-shadowsocksr

LEDE_17.01.6，在package/libs/mbedtls/patches/200-config.patch下第163行，补丁注释掉了MBEDTLS_CAMELLIA_C宏："+//#define MBEDTLS_CAMELLIA_C"。
去掉双斜杠表示启用宏，保存重新make即可："+#define MBEDTLS_CAMELLIA_C"

```

