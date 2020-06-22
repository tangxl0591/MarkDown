# Ubuntu编译配置

##### 一、更新软件

**Ubuntu 16.04**

```bash
sudo apt-get install git-core gnupg flex bison gperf libsdl1.2-dev libesd0-dev build-essential zip curl libncurses5-dev zlib1g-dev valgrind libreadline-gplv2-dev sharutils u-boot-tools default-jdk gcc-multilib g++-multilib libncurses5-dev libz1 libncurses5 libbz2-1.0 libreadline-gplv2-dev libz-dev libswitch-perl x11proto-core-dev libx11-dev zlib1g-dev libgl1-mesa-dev g++-multilib  tofrodos libxml2-utils xsltproc samba  libssl-dev 
```

**Ubuntu 18.04**

```bash
sudo apt-get install git gnupg flex bison gperf libsdl1.2-dev build-essential zip curl libncurses5-dev zlib1g-dev valgrind libreadline-gplv2-dev sharutils u-boot-tools default-jdk gcc-multilib g++-multilib libncurses5-dev zlib1g libncurses5 libbz2-1.0 libreadline-gplv2-dev libswitch-perl x11proto-core-dev libx11-dev zlib1g-dev libgl1-mesa-dev g++-multilib  tofrodos libxml2-utils xsltproc samba  libssl-dev vim samba net-tools lib32z1 dos2unix
```

**2、ssh key**

```bash
ssh-keygen -t rsa
```

##### 3、Git 

```bash
git config --global user.name "tangxl"
git config --global user.email "txl@newlandcomputer.com"
```

**4、JDK**

```bash
sudo apt-get install openjdk-8-jdk
sudo update-alternatives –-config java
```

##### 二、Samba配置

> 添加Samba密码 sudo smbpasswd -a 用户名

**smb.conf**

```bash
########## Home LinkFile ###########
   follow symlinks = yes
   wide symlinks = yes
   unix extensions = no

[Share]
   comment = Shared Folder
   path = /share
   public = yes
   writable = yes
   valid users = txl0591
   create mask = 0700
   directory mask = 0700
   available = yes
   browseable = yes
```

##### 三、ubuntu分辨率设置

```bash
sudo xrandr --newmode "1440x900" 106.50 1440 1528 1672 1904 900 903 909 934 -hsync +vsync
sudo xrandr --addmode VGA1 1440x900
sudo xrandr --output VGA1 --mode 1440x900
```

##### 四、table 无法使用

```bash
usermod -s /bin/bash 用户名 
```

