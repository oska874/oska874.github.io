---
tags : 工具 
catagery : [ 说明 ]
---


### 1. 准备工作

1. 树莓派3 一个
2. 8G tf卡（推荐16G以上）一个，用来运行系统
3. ubuntu mate 16.04 for raspberry pi [镜像](https://ubuntu-mate.org/raspberry-pi/ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz)
3. 移动硬盘一个，用来存放同步文件

### 2. 安装操作系统

将系统镜像直接写到sd卡上。
```
sudo dd if=ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img of=/dev/sdc
```
连接显示器安装系统，配置用户名

使用网线或无线连接网络

连接移动硬盘用来存储文件

安装openssh-server

```
sudo apt-get install openssh-server
```

修改 /etc/rc.local 开机时启动 ssh 服务器和挂载移动硬盘

```
/bin/mount /dev/sda3 /home/ezio/owncloud
/etc/init.d/ssh start
```


ps:

推荐使用 arch，只需要将系统文件复制到SD卡上，上电就可以直接使用ssh远程连接，不用连接额外的显示和键鼠，但是arch对arm的支持不如ubuntu，在安装过程中可能会遇到一些不好解决的问题（本来使用想在arch上装seafile的，结果没有可用的libselinux.so 不得不切换成ubuntu，然后又切换成owncloud了），而且各种软件对ubuntu的支持好过arch太多了。

### 3. 安装owncloud

1. 安装 owncloud

```
sudo curl https://download.owncloud.org/download/repositories/stable/Ubuntu_16.04/Release.key | sudo apt-key add -
echo 'deb https://download.owncloud.org/download/repositories/stable/Ubuntu_16.04/ /' | sudo tee /etc/apt/sources.list.d/owncloud.list
sudo apt-get update
sudo apt-get install owncloud
```

2. 配置 MYSQL

```
To get started, log into MySQL with the administrative account:

mysql -u root -p
Enter the password you set for the MySQL root user when you installed the database server.

ownCloud requires a separate database for storing administrative data. While you can call this database whatever you prefer, we decided on the name owncloud to keep things simple.

CREATE DATABASE owncloud;

Next, create a separate MySQL user account that will interact with the newly created database. Creating one-function databases and accounts is a good idea from a management and security standpoint. As with the naming of the database, choose a username that you prefer. We elected to go with the name owncloud in this guide.

GRANT ALL ON owncloud.* to 'owncloud'@'localhost' IDENTIFIED BY 'set_database_password';

With the user assigned access to the database, perform the flush-privileges operation to ensure that the running instance of MySQL knows about the recent privilege assignment:

FLUSH PRIVILEGES;
This concludes the configuration of MySQL, therefore we will quit the session by typing:

exit
With the ownCloud server installed and the database set up, we are ready to turn our attention to configuring the ownCloud application.
```

3. 配置owncloud

修改文件存储位置到移动硬盘

```
/var/www/owncloud/config/config.php
```

```
<?php
$CONFIG = array (
...
  'datadirectory' => '/home/ezio/owncloud/owncloud-data/data',
...
);
```

4. 启动
直接启动apache2 即可

sudo service apache2 start


### 4. 访问服务器

网页

http://192.168.3.13/owncloud/

客户端

android

oCloud

ios

swiftowncloud

桌面

ubuntu 16.04

```
sudo sh -c "echo 'deb http://download.opensuse.org/repositories/isv:/ownCloud:/desktop/Ubuntu_16.04/ /' > /etc/apt/sources.list.d/owncloud-client.list"
sudo apt-get update
sudo apt-get install owncloud-client
```

### 5. 外网访问

1. 如果有独立ip，则可以在路由器上配置 dmz或者虚拟主机
2. 使用内网穿透的方式，比如花生壳、ngrok等



### reference
1. https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-owncloud-on-ubuntu-16-04
2. https://software.opensuse.org/download/package?project=isv:ownCloud:desktop&package=owncloud-client
