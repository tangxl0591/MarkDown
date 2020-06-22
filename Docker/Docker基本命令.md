# **Docker 基本命令**



## 1.**查看镜像**

```bash
sudo docker image ls
```



## 2.删除镜像

```bash
sudo docker rmi -f ubuntu:latest (REPOSITORY:TAG)
```



## 3.**运行镜像**

```bash
sudo docker run -it ubuntu:16.04

# docker 打开adb
sudo docker run -ti --name ubuntu16.04_GMS --privileged -v /dev/bus/usb:/dev/bus/usb -v /home/linux/Tool/gms:/userdata  ubuntu16.04-gms-v1

# docker arm-qemu
sudo docker run -ti --name ubuntu-qemu-arm --privileged -v ubuntu16.04-arm-qemu:/userdata  ubuntu:16.04
```



## 4.**保存容器为镜像**

```bash
//退出当前的容器
exit
//查看运行的容器
docker ps -a
//将容器转化为镜像
docker commit -m "Init Commit Android Cts" -a "tangxl" 06f58a5799f4 ubuntu-android9-cts
```



## 5.**列出所有容器ID**

```bash
docker ps -aq
```




## 7.**停止容器ID**

```bash
sudo docker stop (Commit ID) 
sudo docker stop $(sudo docker ps -aq)
```



## 8.**删除容器**

```bash
sudo docker rm $(sudo docker ps -aq)
```



## 9.打开容器

```bash
sudo docker start "name"
```



## 10.进入容器 /bin/bash

```bash
sudo docker attach "name"
sudo docker exec -it "name" /bin/bash
```



## 11.创建数据卷

```bash
docker volume create "name"
```



## 12.查询镜像

```bash
sudo docker search "name"
```





## **Ubuntu16.04安装基础包**

```bash
apt-get install net-tools android-tools-adb openjdk-8-jdk samba usbutils inetutils-ping openssh-server udev vim
```

