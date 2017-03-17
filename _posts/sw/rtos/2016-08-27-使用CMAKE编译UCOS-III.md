---
tags : [ RTOS , cmake ]
category : [ TECH ]
---


最近下载了 Linux Simulation 版 [uCOS-III][1] 的源代码，但这只是纯代码，需要自己编译。以前单位编译操作系统都是自己编写 Makefile ，如果目录、文件比较多则编写 Makefile 比较麻烦，所以我就考虑使用 cmake 自动生成 Makefile ，刚巧前不久在编译应用的时候就使用了 cmake ，现在刚好可以深入使用 cmake 了。

从官网下载源代码需要注册、登陆，我已经把可以编译的系统源代码上传到了 [github][2] 上了，可以直接在 ubuntu 16.04 x64 系统上编译，配合 gcc 5.4 和 cmake 3.5.1 使用（其它版本应该也没问题，不过没做实验）。

下载的 Linux Simulation uCOS-III 的代码是可以直接作为 eclipse 工程使用（工程文件位于 ` `），使用 eclipse 导入工程然后就可以编译、运行，它直接使用 makefile 编译。接下来我们就要使用 cmake 编译 uCOS 了。 

cmake 有自己的语法来生成 makefile 。要用到的包括设置编译器和参数、头文件路径、源码路径、库文件等。

- 第一步，分析代码的结构，准备如何编写 cmake 脚本。
  
  下载好的 uCOS 的代码结构如下：

  ```
➜  Micrium tree 
.
├── Examples
│   └── POSIX
│       └── GNU
│           └── OS3
│               ├── app.c
│               ├── app_cfg.h
│               ├── cpu_cfg.h
│               ├── eclipse-GCC
│               ├── lib_cfg.h
│               ├── os_app_hooks.c
│               ├── os_app_hooks.h
│               ├── os_cfg_app.h
│               └── os_cfg.h
├── Micrium POSIX Readme.pdf
└── Software
    ├── uC-CPU
    │   ├── cpu_cache.h
    │   ├── cpu_core.c
    │   ├── cpu_core.h
    │   ├── cpu_def.h
    │   └── Posix
    │       └── GNU
    │           ├── cpu_c.c
    │           └── cpu.h
    ├── uC-LIB
    │   ├── lib_ascii.c
    │   ├── lib_ascii.h
    │   ├── lib_def.h
    │   ├── lib_math.c
    │   ├── lib_math.h
    │   ├── lib_mem.c
    │   ├── lib_mem.h
    │   ├── lib_str.c
    │   └── lib_str.h
    └── uCOS-III
        ├── Ports
        │   └── POSIX
        │       └── GNU
        │           ├── os_cpu_c.c
        │           └── os_cpu.h
        └── Source
            ├── os_cfg_app.c
            ├── os_core.c
            ├── os_dbg.c
            ├── os_flag.c
            ├── os.h
            ├── os_int.c
            ├── os_mem.c
            ├── os_mon.c
            ├── os_msg.c
            ├── os_mutex.c
            ├── os_pend_multi.c
            ├── os_prio.c
            ├── os_q.c
            ├── os_sem.c
            ├── os_stat.c
            ├── os_task.c
            ├── os_tick.c
            ├── os_time.c
            ├── os_tmr.c
            ├── os_type.h
            └── os_var.c
  ``` 
  
  系统代码明显可以分为两部分 `Examples` 和 `Software` ，后者是单纯的系统代码、libc 库，以及处理器框架代码，也就是说这部分代码和实际的硬件是无关的，arm 版 、 ppc 版等各个版本的这部分代码都是一样的，而前者则包含了测试程序和硬件相关代码。所以我就以 `Examples` 为主代码（`main()` 入口）开始编译。

- 第二步，为各个源码目录编写 cmake 脚本，脚本名均为 `CMakeLists.txt` 。

  编写好的 cmake 脚本列表如下：

  ```
├── CMakeLists.txt
├── Examples
│   └── POSIX
│       └── GNU
│           └── OS3
└── Software
    ├── uC-CPU
    │   ├── CMakeLists.txt
    │   └── Posix
    │       └── GNU
    │           └── CMakeLists.txt
    ├── uC-LIB
    │   └── CMakeLists.txt
    └── uCOS-III
        ├── CMakeLists.txt1
        ├── Ports
        │   └── POSIX
        │       └── GNU
        │           └── CMakeLists.txt
        └── Source
            └── CMakeLists.txt
  ``` 

  因为 `Examples` 是我们的主代码，所以这个目录下没有 `CMakeLists.txt` ， 主目录下的 `CMakeLists.txt` 负责编译 `Example` 的代码。各个子目录的 `CMakeLists.txt` 内容类似，以 `uCOS-III/Source` 为例：

  ```
    AUX_SOURCE_DIRECTORY(. LIBOS3_SRC)

    INCLUDE_DIRECTORIES(/workspace/repos/github/rtos/ucos/Micrium/Software/uC-CPU)
    INCLUDE_DIRECTORIES(/workspace/repos/github/rtos/ucos/Micrium/Software/uC-LIB)
    INCLUDE_DIRECTORIES(/workspace/repos/github/rtos/ucos/Micrium/Software/uCOS-III/Ports/POSIX/GNU)
    INCLUDE_DIRECTORIES(/workspace/repos/github/rtos/ucos/Micrium/Examples/POSIX/GNU/OS3)
    INCLUDE_DIRECTORIES(../../uC-CPU)
    INCLUDE_DIRECTORIES(../../uC-CPU/Posix/GNU)

    add_library(libos1  ${LIBOS3_SRC})
  ```

  这部分脚本有三个意思：
  
  1. `AUX_SOURCE_DIRECTORY` 将目录下的代码列表赋给 `LIBOS3_SRC` ；
  2. `INCLUDE_DIRECTORIES` 指定了头文件引用路径 ；
  3. `add_library` 将源代码添加到名为 `libos1` 的静态库。

  其它目录下的 `CMakeLists.txt` 内容也就这些了。

  根目录下的 `CMakeLists.txt` 就比较复杂，首先定义了工程名和使用的 cmake 版本要求：

  ```
    PROJECT (USOS3)
    CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
  ```

  然后设置编译选项，并且指定使用 `Debug` 版的选项：

  ```
    SET(CMAKE_C_FLAGS_DEBUG "$ENV{CFLAGS} -static -O0 -g3 -Wall -fmessage-length=0  ")  #set debug mode c flags
    SET(CMAKE_C_FLAGS_RELEASE "$ENV{CFLAGS} -O3 -Wall ")            #set release mode c flags
    SET(CMAKE_BUILD_TYPE "Debug")                                   #set Debug or Release
  ```

  接着定义头文件、子目录路径（注意，子目录路径不要包含 `Examples` ）和 `Examples` 的源码：

  ```
    INCLUDE_DIRECTORIES(/workspace/repos/github/rtos/ucos/Micrium/Examples/POSIX/GNU/OS3)
    ...
    ADD_SUBDIRECTORY(Software/uC-CPU/Posix/GNU)
    ...
    AUX_SOURCE_DIRECTORY(./Examples/POSIX/GNU/OS3 LIBSIM_SRC)
  ```

  最后编译 `Examples` 并将由各个子目录编译的的库文件链接进来，生成可执行文件：

  ```
    ADD_EXECUTABLE(${TARGET_EXE_Z} ${LIBSIM_SRC})  #生成可执行文件
    TARGET_LINK_LIBRARIES(${TARGET_EXE_Z} libos1)  #链接库文件
    TARGET_LINK_LIBRARIES(${TARGET_EXE_Z} pthread)
    ...
  ```
  
  注意一点，在 Linux 上运行 ucos 要用到 pthread 库，所以一定要将 pthread 添加到链接库中。

- 第三步，编译 uCOS。
  直接在代码目录使用 cmake 编译的话会生成很多临时文件，不好清理，cmake 官方也没提供有效的办法清理这些临时文件，所以需要使用源码外编译（out of source builds）。在根目录创建一个 `build` 目录，然后在该目录下执行 `cmake ..` 即可完成编译，之后只要删除了该目录就可以清理全部临时文件了，而我的则编写了一个简单脚本 `build.sh` 来执行编译：

  ```
    rm -rf ./build/*
    cd build
    cmake ..
    make VERBOSE=0
  ```

  最后，执行脚本 `sh build.sh` 就可以完成编译了，会在 `build` 目录下生成 `ucos` 可执行文件(注：我使用的这个版本并不运行在开发板上，而是在 Linux 上运行的的一个仿真器，对操作系统来说就是一个特殊点的 ELF 可执行文件。)。运行 ucos 要注意，因为需要将 RTPRIO 设置为 unlimited ，所以需要在 root 用户下运行 `ulimit -r unlimited` 解除限制，然后再执行 ucos `./build/ucos` 。


[1]: https://www.micrium.com/download/micrium_posix/
[2]: https://github.com/oska874/ucos3.git
