

#### **一、 Docker Portainer可视化管理工具**1.安装配置

```bash
docker pull docker.io/portainer/portainer
```

```bash
docker run -d -p 9000:9000 \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/userdata \
    --name prtainer-manager \
    docker.io/portainer/portainer
```

#### 二、 gitlab 

```bash
docker pull gitlab/gitlab-ce
```

```bash
sudo docker run -d  -p 443:443 -p 80:80 -p 222:22 --name gitlab --restart always -v /home/olddisk/docker/gitlab/config:/etc/gitlab -v /home/olddisk/docker/gitlab/logs:/var/log/gitlab -v /home/olddisk/docker/gitlab/data:/var/opt/gitlab --privileged=true gitlab/gitlab-ce:latest

编辑 gitlab.rb配置文件
vim /srv/gitlab/config/gitlab.rb

添加external_url,配置http协议所使用的访问地址,不加端口号默认为80
external_url 'http://192.168.74.158'

添加访问地址和端口
配置ssh协议所使用的访问地址和端口 gitlab_rails['gitlab_ssh_host'] = '192.168.74.158''
此端口是run时22端口映射的222端口 gitlab_rails['gitlab_shell_ssh_port'] = 222

docker restart gitlab
```

#### 三、Ubuntu

```bash
docker pull ubuntu:14.04
```

```bash
docker run -ti --privileged --name ubuntu -v /dev/bus/usb:/dev/bus/usb ubuntu:latest bash

```

