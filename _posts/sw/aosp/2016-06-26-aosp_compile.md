---
tags : android
---

AOSP 开发流程
============

(in progress)

## 1. get aosp
### 1.0. aosp 源代码获取  
大陆，被墙，可以使用两个国内源 
清华 `http://mirrors.tuna.tsinghua.edu.cn/`
中科大 `https://lug.ustc.edu.cn/wiki/mirrors/help/aosp`

`git clone git://aosp.tuna.tsinghua.edu.cn/android/git-repo.git/`

修改repo
google的地址
`REPO_URL = 'https://gerrit.googlesource.com/git-repo'`
改为清华大学的地址
`REPO_URL = ‘git://aosp.tuna.tsinghua.edu.cn/android/git-repo'`

从清华源获取某个版本
`repo init -u  git://aosp.tuna.tsinghua.edu.cn/android/platform/manifest -b android-5.1.1_r6`
或者全部版本
`repo init -u git://aosp.tuna.tsinghua.edu.cn/android/platform/manifest`

`repo sync` 会同做所有到.repo

`repo sync -c` 只下载当前分支（清华同一个源不能超过4个连接,中科大不超过5个）

从中科大源获取
`repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest`
or
`repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-4.0.1_r1`

### 1.1. sdk
东软的源 [link](http://mirrors.neusoft.edu.cn/)
启动 Android SDK Manager ，打开主界面，依次选择「Tools」、「Options...」，弹出『Android SDK Manager - Settings』窗口；
在『Android SDK Manager - Settings』窗口中，在「HTTP Proxy Server」和「HTTP Proxy Port」输入框内填入mirrors.neusoft.edu.cn和80，并且选中「Force https://... sources to be fetched using http://...」复选框。设置完成后单击「Close」按钮关闭『Android SDK Manager - Settings』窗口返回到主界面；
依次选择「Packages」、「Reload」。

## 2. repo
### 2.0.
查看可切换的分支

```
cd .repo/manifests
git branch -a | cut -d / -f 3
```
以 gingerbread-release 分支为例
```
repo init -b gingerbread-release 
repo sync (not needed if your local copy is up to date)
repo start gingerbread-release --all 
#查看当前的分支
repo branches
```
---

所有的资源都在.repo 下， 切换分支前最好删除当前除了git-repo，.repo 的文件，然后repo sync -c ，否则可能因为有改动无法切换，sync -c 此时很快，所有东西都在.repo下

---

### 2.1.
repo只是google用Python脚本写的调用git的一个脚本，主要是用来下载、管理Android项目的软件仓库。

下载 repo 的地址: http://android.git.kernel.org/repo ，可以用 wget http://android.git.kernel.org/repo 或者 curl http://android.git.kernel.org/repo >~/bin/repo  来下载 repo , chmod a+x ~/bin/repo 
用repo sync 在抓去 android source code 的时候，会经常出现一些错误导致 repo sync 中断，每次都要手动开始。 可以用如下的命令，来自动重复：   $?=1;   while [ $? -ne 0 ] ; do  repo sync ; done
 repo help [ command ] , 显示command 的详细的帮助信息内容
repo init -u URL ,  在当前目录安装 repository ，会在当前目录创建一个目录 ".repo"  -u 参数指定一个URL， 从这个URL 中取得repository 的 manifest 文件。   repo init -u git://android.git.kernel.org/platform/manifest.git
               可以用 -m 参数来选择 repository 中的某一个特定的 manifest 文件，如果不具体指定，那么表示为默认的 namifest 文件 (default.xml)    repo init -u git://android.git.kernel.org/platform/manifest.git -m dalvik-plus.xml

              可以用 -b 参数来指定某个manifest 分支。

               repo init -u git://android.git.kernel.org/platform/manifest.git -b release-1.0

              可以用命令: repo help init 来获取 repo init 的其他用法

        4. repo sync [project-list]

            下载最新本地工作文件，更新成功，这本地文件和repository 中的代码是一样的。 可以指定需要更新的project ， 如果不指定任何参数，会同步整个所有的项目。

           如果是第一次运行 repo sync ， 则这个命令相当于 git clone ，会把 repository 中的所有内容都拷贝到本地。 如果不是第一次运行 repo sync ， 则相当于 git remote update ;  git rebase origin/branch .  repo sync 会更新 .repo 下面的文件。 如果在merge 的过程中出现冲突， 这需要手动运行  git  rebase --continue

      5. repo update[ project-list ]

      上传修改的代码 ，如果你本地的代码有所修改，那么在运行 repo sync 的时候，会提示你上传修改的代码，所有修改的代码分支会上传到 Gerrit (基于web 的代码review 系统), Gerrit 受到上传的代码，会转换为一个个变更，从而可以让人们来review 修改的代码。

       6. repo diff [ project-list ]

        显示提交的代码和当前工作目录代码之间的差异。

       7. repo download  target revision

        下载特定的修改版本到本地， 例如:  repo download pltform/frameworks/base 1241 下载修改版本为 1241 的代码

       8. repo start newbranchname

        创建新的branch分支。 "." 代表当前工作的branch 分支。

       9.  repo prune [project list]

        删除已经merge 的 project

      10. repo foreach [ project-lists] -c command

       对每一个 project 运行 command 命令

      11. repo status

       显示 project 的状态

## 3. compile
### 3.0. 配置编译环境

 $ sudo apt-get install git-core gnupg flex bison gperf build-essential   zip curl libc6-dev libncurses5-dev x11proto-core-dev    libx11-dev libreadline6-dev libgl1-mesa-glx  libgl1-mesa-dev g++-multilib mingw32 tofrodos    python-markdown libxml2-utils xsltproc zlib1g-dev lib32readline6-dev ia32-dev

配置ccache

export USE_CCACHE=1
export export CCACHE_DIR=/mnt/ccache/ar
prebuilts/misc/linux-x86/ccache/ccache -M 50G

### 3.1. 编译

source build/envsetup.sh
lunch
make

问题：
  1. make -jN ，N 越大，速度越快，占用内存越多
  2. ccahe：prebuild 预留了ccahe，提高编译速度，减少重新编译的时间

### 3.2. 自定义
* 定义自己的产品编译项，
  1. 创建公司目录
  
      #mkdir vendor/farsight
  
  2. 创建一个vendorsetup.sh文件，将当前产品编译项添加到lunch里，让lunch能找到用户个性定制编译项
  
      #echo "add_lunch_combo fs100-eng" > vendor/farsight/vendorsetup.sh
  
  3. 仿着Android示例代码，在公司目录下创建products目录
  
      #mkdir -p vendor/farsight/products
  
  4. 仿着Android示例代码，在products目录下创建两个mk文件
  
      #touch vendor/farsight/products/AndroidProduct.mk vendor/farsight/products/fs100.mk

在AndroidProduct.mk里添加如下内容：

* board cfg
build/core/main.mk包含了config.mk，它主要定义了编译全部代码的依赖关系

      build/core/config.mk         定义了大量的编译脚本命令，编译时用到的环境变量，引入了envsetup.mk 文件，加载board相关配置文件。
      build/core/envsetup.mk   定义了编译时用到的大量OUT输出目录，加载product_config.mk文件
      build/core/product_config.mk 定义了Vendor目录下Product相关配置文件解析脚本，读取AndrodProducts.mk生成TARGET_DEVICE变量
      build/target/product          product config
      build/target/board            board config
      build/core/combo             build flags config  

PRODUCT_MAKEFILES := $(LOCAL_DIR)/fs100.mk
表示只有一个产品fs100，它对应的配置文件在当前目录下的fs100.mk。

* borad主要是设计到硬件芯片的配置，比如是否提供硬件的某些功能，比如说GPU等等，或者芯片支持浮 点运算等等。product是指针对当前的芯片配置定义你将要生产产品的个性配置，主要是指APK方面的配置，哪些APK会包含在哪个product中， 哪些APK在当前product中是不提供的。

  1. 在vendor目录下创建自己公司目录，然后在公司目录下创建一个新的vendorsetup.sh，在里面添加上自己的产品编译项
  
  $mkdir vendor/farsight/  
  $touch vendor/farsight/vendorsetup.sh  
  $echo "add_lunch_combo fs100-eng" > vendor/farsight/vendorsetup.sh   
  
  2. 仿着Android示例代码，在公司目录下创建products目录
   $mkdir -p vendor/farsight/products
  3. 仿着Android示例代码，在products目录下创建两个mk文件
  $touch vendor/farsight/products/AndroidProduct.mk vendor/farsight/products/fs100.mk
      在AndroidProduct.mk里添加如下内容：
  PRODUCT_MAKEFILES := $(LOCAL_DIR)/fs100.mk  
      在产品配置文件里添加最基本信息
  
   PRODUCT_PACKAGES := \  
       IM \  
       VoiceDialer     
   $(call inherit-product, build/target/product/generic.mk)     
   # Overrides  
   PRODUCT_MANUFACTURER := farsight  
   PRODUCT_NAME := fs100  
   PRODUCT_DEVICE := fs100  
  
  4. 借鉴build/target/board/generic/AndroidBoard.mk和BoardConfig.mk，创建对应文件。
  $cp build/target/board/generic/AndroidBoard.mk build/target/board/generic/BoardConfig.mk  vendor/farsight/fs100/
  
## 5.problem
### 5.0.
Android repo 出现error.GitError: manifests rev-list ('^12303f87b9f90c07bf4aec4c4353ba514ee70c8a', 'HEAD', '--'): fatal: bad revision 'HEAD'
执行的操作
repo init -u git://192.168.1.183/android/platform/manifest.git -b android-4.2.1_r1.2
或者
repo init -b android-4.2.1_r1.2
出现问题
Traceback (most recent call last): File "/work/information/source_code/google/abc/.repo/repo/main.py", line 414, in <module> _Main(sys.argv[1:]) File "/work/information/source_code/google/abc/.repo/repo/main.py", line 390, in _Main result = repo._Run(argv) or 0 File "/work/information/source_code/google/abc/.repo/repo/main.py", line 138, in _Run result = cmd.Execute(copts, cargs) File "/work/information/source_code/google/abc/.repo/repo/subcmds/init.py", line 347, in Execute self._SyncManifest(opt) File "/work/information/source_code/google/abc/.repo/repo/subcmds/init.py", line 203, in _SyncManifest m.Sync_LocalHalf(syncbuf) File "/work/information/source_code/google/abc/.repo/repo/project.py", line 1120, in Sync_LocalHalf upstream_gain = self._revlist(not_rev(HEAD), revid) File "/work/information/source_code/google/abc/.repo/repo/project.py", line 2006, in _revlist return self.work_git.rev_list(*a, **kw) File "/work/information/source_code/google/abc/.repo/repo/project.py", line 2155, in rev_list p.stderr)) error.GitError: manifests rev-list ('^HEAD', '4c2be345d6bfc25db87f23749912ae9d98d2ad62', '--'): fatal: bad revision '^HEAD'
 
解决方案
删除.repo 目录下的 除 repo 文件夹之外的其它所有文件，重新执行inti和sync
可以执行： rm -rf .repo/manifest*

### 5.1. 
target SharedLib: libwebviewchromium (out/target/product/generic/obj/SHARED_LIBRARIES/libwebviewchromium_intermediates/LINKED/libwebviewchromium.so) collect2: error: ld terminated with signal 9 [Killed


make: *** [out/target/product/generic/obj/SHARED_LIBRARIES/libwebviewchromium_intermediates/LINKED/libwebviewchromium.so] Error 1

出错原因：swap 分区不足
解决办法：增加swap 分区
