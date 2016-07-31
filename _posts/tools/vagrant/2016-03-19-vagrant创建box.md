---
tags : vagrant 
category : [ 说明 ]
---

0. 先安装virtualbox，再装ubuntu（其他发行版也可以，习惯问题），注意，资源分配足够（内存、cpu）
      略

1. 将vagrant添加到admin组

      `sudo usermod -G admin vagrant`

2. 修改sudoers文件，让vagrant 用户使用sudo 不用输密码

```
sudo vim /etc/sudoers      
Defaults env_keep="SSH_AUTH_SOCK"
%admin ALL=NOPASSWD: ALL
```

3. 安装ssh server，ubuntu 默认不装ssh 服务器
      
      `sudo apt-get install openssh-server`

4. 安装vagrant的public keys

```
mkdir ~/.ssh/
cd ~/.ssh
wget http://github.com/mitchellh/vagrant/raw/master/keys/vagrant
wget http://github.com/mitchellh/vagrant/raw/master/keys/vagrant.pub
mv vagrant.pub authorized_keys
```

5. 安装virtualbox 增强包，菜单栏就有选项

6. 关闭虚拟机，进入自己的工作目录（比如我的就是 e:\vagarnt)
  
      `vagrant package --output ubuntu_64.box --base vagrant`

7. 使用自己创建的vagrant

     `vagrant init vagrant d:/vagrant/ubuntu_64.box`

8. 常用的vagrant命令：

```
 $ vagrant box add NAME URL    #添加一个box
 $ vagrant box list            #查看本地已添加的box
 $ vagrant box remove NAME virtualbox #删除本地已添加的box，如若是版本1.0.x，执行$ vagrant box remove  NAME
 $ vagrant init NAME          #初始化，实质应是创建Vagrantfile文件
 $ vagrant up                   #启动虚拟机
 $ vagrant halt                 #关闭虚拟机
 $ vagrant destroy            #销毁虚拟机
 $ vagrant reload             #重启虚拟机
 $ vagrant package            #当前正在运行的VirtualBox虚拟环境打包成一个可重复使用的box
 $ vagrant ssh                 #进入虚拟环境
```

9. 共享文件夹：vagrant 默认将Vagrantfile 所在目录共享到ubuntu 的/vagrant ，也可以自己添加，修改Vagrantfile：

```
  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder "srd", "/vagrant_data"
```

10. vagrant 的大部分配置都要修改Vagrantfile，文件里面说的很直白，一看就会：

```
# -*- mode: ruby -*-
# vi: set ft=ruby :


# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.


  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "puphpet/ubuntu1404-x64"


  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false


  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080


  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"


  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"


  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder "srd", "/vagrant_data"


  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.


  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end


  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   sudo apt-get update
  #   sudo apt-get install -y apache2
  # SHELL
end 
```
