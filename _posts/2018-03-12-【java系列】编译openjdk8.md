很多人想尝试编译openjdk，但是编译的过程中会遇到各种问题。这里记录下我编译的过程，希望可以帮助大家。

# 环境
操作系统: ubuntu 17.10 内核 4.13.0-36-generic  
gcc版本: 7.2.0  
jdk版本: openjdk8  

# 准备工作

## 下载源码
大家可以使用官方提供的方法，
```
hg clone http://hg.openjdk.java.net/jdk8/jdk8
```
但是由于防火墙等因素，下载经常会失败。推荐大家在github上查找一些jdk的镜像库，比如
```
https://github.com/dmlloyd/openjdk.git
```

# 编译

## configure
```
bash ./configure
```
运行上面的命令后可能会提示你一些软件没有安装，按照提示安装即可。
```
sudo apt-get install openjdk-8-jdk
sudo apt-get install libx11-dev libxext-dev libxrender-dev libxtst-dev libxt-dev
sudo apt-get install libcups2-dev
sudo apt-get install libfreetype6-dev
sudo apt-get install libasound2-dev
```
需要注意的是在安装x11的时候，安装程序给出的提示是
> You might be able to fix this by running ‘sudo apt-get install libX11-dev libxext-dev libxrender-dev libxtst-dev libxt-dev’

你可能需要把```libX11-dev```修改为```libx11-dev```，否则无法安装。

所有软件都安装完成后，重新运行下面的命令
```
bash configure --enable-debug --with-freetype-include=xxx --with-freetype-lib=/usr/local/lib/ 
```
上面的xxx填入你安装的freetype的路径即可。--enable-debug模式是fastdebug，如果需要开启slowdebug需要在configure命令后面添加--with-debug-level=slowdebug。

## make
```
make all
```

如果你使用的gcc版本比较新，可能会在编译的时候报错
> /opt/openjdk/jdk8u/hotspot/src/os/linux/vm/os_linux.inline.hpp:127:42: error: 'int readdir_r(DIR*, dirent*, dirent**)' is deprecated [-Werror=deprecated-declarations]
   if((status = ::readdir_r(dirp, dbuf, &p)) != 0) {
                                          ^
In file included from /opt/openjdk/jdk8u/hotspot/src/os/linux/vm/jvm_linux.h:44:0,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/prims/jvm.h:30,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/utilities/debug.hpp:29,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/runtime/globals.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/memory/allocation.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/memory/iterator.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/memory/genOopClosures.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/oops/klass.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/runtime/handles.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/memory/universe.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/code/oopRecorder.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/asm/codeBuffer.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/asm/assembler.hpp:28,
                 from /opt/openjdk/jdk8u/hotspot/src/share/vm/precompiled/precompiled.hpp:29:
/usr/include/dirent.h:183:12: note: declared here  
 extern int readdir_r (DIR *__restrict __dirp,  
            ^~~~~~~~~  
cc1plus: all warnings being treated as errors  
vm.make:295: recipe for target 'precompiled.hpp.gch' failed   

这是因为代码中的某些函数已经过期了，同时编译的脚本里面配置了Werror，这个配置会把warning当做error来处理。你需要做的是修改文件
```
hotspot/make/linux/makefiles/gcc.make
```
把WARNINGS_ARE_ERRORS = -Werror这一行注释掉。

还有一点需要注意的是，如果你使用的是虚拟机来进行编译，你需要给虚拟机分配大一点的内存。否则编译的时候会失败

## 使用

编译好后，可以在源码目录下找到build文件夹，然后在build/linux-x86_64xxxx/jdk/bin目录下可以看到对应的编译好的文件。运行
```
./java -version
```
可以看到对应输出
