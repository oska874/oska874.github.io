---
tags : aosp 
category : [ TECH ]
---


# 0. 环境准备

    - tf 卡（存储系统镜像）
    - usb转串口线（查看串口打印）
    - OS ：Ubuntu 16.04 x64 （编译系统）
    - friendlyarm 的 nanopi2 支持 **android 5.1**（friendlyarm 提供支持）
    
# 1. 下载代码和工具

要用到的代码包括：arm gcc 和烧写镜像工具 ， uboot 、 linux kernel 、 aosp 的源代码。

- 首先需要安装 git 和 mkimage：

```
sudo apt-get install git
sudo apt-get install u-boot-tools
```

- 工具链

```
git clone https://github.com/friendlyarm/prebuilts.git 
sudo mkdir -p /opt/FriendlyARM/toolchain 
sudo tar xf prebuilts/gcc-x64/arm-cortexa9-linux-gnueabihf-4.9.3.tar.xz -C /opt/FriendlyARM/toolchain/
```

- 烧写镜像工具

```
git clone https://github.com/friendlyarm/sd-fuse_nanopi2.git 
cd sd-fuse_nanopi2
```

- uboot

```
git clone https://github.com/friendlyarm/uboot_nanopi2.git 

```

- linux kernel

```
git clone https://github.com/friendlyarm/linux-3.4.y.git 
cd linux-3.4.y 
```

- aosp

  下载、更新 aosp 代码要用到工具 repo ，下载安装 ：

```
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

  然后使用 repo 下载：

```
mkdir android && cd android 
repo init -u https://github.com/friendlyarm/android_manifest.git -b nanopi2-lollipop-mr1
repo sync
```

  直接使用 friendlyarm 在 github 上提供的 manifest.xml 下载代码时会因为 GFW 的原因下载失败，这时可以使用国内的源替代 google 的代码源，比如[清华的镜像][https://mirrors.tuna.tsinghua.edu.cn/]:

```
repo init -u https://github.com/friendlyarm/android_manifest.git -b nanopi2-lollipop-mr1 --repo-url=https://gerrit-google.tuna.tsinghua.edu.cn/git-repo
```

  然后修改代码目录下的文件 `.repo/manifest.xml` ，替换：

```
<remote  name="aosp"
         fetch="https://android.googlesource.com/" />
```

  为：

```
<remote  name="aosp"
         fetch="https://aosp.tuna.tsinghua.edu.cn/" />
```

  然后再执行 `repo sync` 就可以正常下载代码了。

  同时，可以配置一下 aosp 编译环境：

  ```
  sudo apt-get install bison g++-multilib git gperf libxml2-utils make python-networkx zipsudo apt-get install flex libncurses5-dev zlib1g-dev gawk minicom
  ```

# 2. 编译 u-boot

编译 uboot 前需要配置好之前下载的交叉编译工具链，后面编译 kernel 也要使用：

```
export PATH=/opt/FriendlyARM/toolchain/4.9.3/bin:$PATH
export GCC_COLORS=auto
```

```
cd uboot_nanopi2 git checkout nanopi2-lollipop-mr1 
make s5p4418_nanopi2_config 
make CROSS_COMPILE=arm-linux-
```

# 3. 编译内核

```
git checkout nanopi2-lollipop-mr1
make nanopi2_android_defconfig 
touch .scmversion 
make uImage
```

# 4. 编译 aosp

bash

```
source build/envsetup.sh 
lunch aosp_nanopi2-userdebug 
make -j8
```

生成的结果位于 ： `out/target/product/nanopi2/`,编译生成的镜像文件：

```
➜  friendly ls out/target/product/nanopi2
android-info.txt  cache           data          installed-files.txt  previous_build_config.mk  recovery  system
boot              cache.img       dex_bootjars  obj                  ramdisk.img               root      system.img
boot.img          clean_steps.mk  gen           partmap.txt          ramdisk-recovery.img      symbols   userdata.img
```

要用到的文件主要是： `cache.img` 、 `boot.img` 、 `partmap.txt` 、 `ramdisk.img` 、 `system.img`



# 5. 烧写镜像

首先要将编译 aosp 生成的镜像文件拷贝到 `sd-fusing_nanopi2/android/` ，然后将编译 uboot 生成的 bin 文件也拷贝到 `sd-fusing_nanopi2/android/` 

然后插入 tf 卡，然后检索目录 `/dev` ，找出 tf 卡对应的设备文件（比如 `sdc`）。

最后执行命令将镜像拷贝到 tf 卡：

```
sudo ./fusing.sh /dev/sdc
```

或者使用脚本 `mkimage.sh` 将几个小镜像合并为一个整体，然后 `dd` 到 tf 卡。

当然也可以分析 `fusing.sh` 脚本文件，自己使用 `dd` 烧写镜像。


# 6. 总结

制作 android 镜像过程中遇到了几个问题：

1. nanopi2 很挑卡
  nanopi2 很挑卡，最开始编译了 aosp 镜像之后使用工具烧写到 kinston 的 16g sd 卡之后，启动 uboot 报错 找不到内核 ，然后换了三星的 8G 卡就好了，太神奇了，官方也有推荐的 sd 卡。
2. ubuntu 16.04 编译android 5.1 需要做修改
  使用 ubuntu 16.04 编译 android 5.1 时会出现错误 ：

```
...
libnativehelper/JniInvocation.cpp:165: error: unsupported reloc 43
libnativehelper/JniInvocation.cpp:165: error: unsupported reloc 43
...
```

  查了 SO ，出错的原因是 aosp 自带的 ld 和 ubuntu 16.04 不兼容，需要用系统自带的 ld 替换 ：

```
cp /usr/bin/ld.gold prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.6/x86_64-linux/bin/ld
```

  注： 使用 ubuntu 16.04 编译 android 6 的源码也会出现类似问题，只需要替换对应（ `glibc2.15` ）的 ld 即可。


出错解决

替换 ld
修改
