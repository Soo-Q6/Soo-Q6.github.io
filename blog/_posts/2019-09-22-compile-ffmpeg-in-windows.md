---
title: Window编译和单步调试FFmpeg
# image: /assets/img/blog/...
description: >
  本文主要记录一下如何在window使用visual studio编译FFmpeg源码，生成window环境可调用的库文件，如何将生成的库文件集成到自己的项目当中，并且可以单步调试FFmpeg的源码。此外，本文还记录了一些编译和调试过程的坑。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags:
---
本文的环境是window 10 pro 1809, [FFmpeg](http://ffmpeg.org/)是2019-08-15提交的版本

需要安装的软件：
visual studio 2015，msys2和yasm
## 安装visual studio
官网下载安装
## 安装msys2
[在这里下载](http://www.msys2.org/)，选择下载msys2-x86_64-20190524.exe，默认安装就好。
为了使用pacman安装编译所需环境，需要**修改源**和**设置下载timeout**（否则会下载错误的）。
首先，修改源，在C:\msys64\etc\pacman.d中，打开：
![修改源]({{site.data.strings.blog_url}}9565345-5d75570b48a3aed6.png)<br>
在三个文件中分别添加以下镜像源（清华和中科大二选一）：
清华大学
```
Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/i686
Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/x86_64
Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/msys/$arch
```
中科大
```
Server = http://mirrors.ustc.edu.cn/msys2/mingw/i686/
Server = http://mirrors.ustc.edu.cn/msys2/mingw/x86_64/
Server = http://mirrors.ustc.edu.cn/msys2/msys/$arch/
```
然后，设置关闭下载timeout，在C:\msys64\etc中打开pacman.conf文件，在[options]下添加：
```
[options]
DisableDownloadTimeout
```
然后安装编译所需环境`pacman -S make gcc diffutils pkg-config`
重命名C:\msys64\usr\bin\link.exe ，避免和MSVC 的link.exe冲突。

## 安装yasm
[下载yasm](http://yasm.tortall.net/Download.html)，将下载的*.exe文件修改为yasm.exe并放置在C:\msys64\usr\bin目录中。


## 下载ffmpeg源码
如果git clone出现下载错误，可以使用浅拷贝方法（网上说是git的缓存不够，但是修改了**http.postBuffer**好像没什么用）：
```bash
git clone --depth=1 https://git.ffmpeg.org/ffmpeg.git ffmpeg
git fetch --unshallow
```
## 使用vs2015编译ffmpeg
首先，修改C:\msys64\msys2_shell.cmd中的：
```
rem set MSYS2_PATH_TYPE=inherit
```
为（rem为cmd文件中的注释）：
```
set MSYS2_PATH_TYPE=inherit
```
然后在开始菜单中，执行 `Visual Studio 2015-> VS2015 x64 本机工具命令提示符`，在命令窗口下，执行`cd \msys64`，执行`msys2_shell.cmd`，执行：
![查看所使用的工具使用已经安装]({{site.data.strings.blog_url}}9565345-f65d50cfeaa402fa.png)<br>
接着修改窗口的编码方式为GBK，以免log显示编码错误
最后，进入ffmpeg的文件目录，执行（过程十分缓慢）：
```
./configure  --toolchain=msvc  --arch=x86  --enable-yasm  --enable-asm --enable-shared  --disable-static
make
make install
```
>`./configure`命令可以配置要编译的模块，这根据你的项目的需要来配置,有的模块的编译需要遵循某些开源协议。比如`libpostproc`就需要添加`--enable-postproc --enable-gpl`

编译生成的执行文件和DLL文件在目录C:\msys64\usr\local\bin中，头文件在C:\msys64\usr\local\include中。
**注意**，如果报错了，
![报错了]({{site.data.strings.blog_url}}9565345-812c2e34ad0420fb.png)<br>
应该是`CC_IDENT`这个宏有问题，将：
```c
//cmdutils.c
av_log(NULL, level, "%sbuilt with %s\n", indent, CC_IDENT);

//ffprobe.c
print_str("compiler_ident", CC_IDENT);
```
注释掉就好了

## 库文件集成到项目(visual studio)
如果单纯是想使用这些库文件的话，只需要将生成的可执行文件和DLL文件，头文件的引入到vs的项目就可以了。如果是想要了解FFmpeg执行的原理，需要对FFmpeg源码进行单步调试，则需要将对应的PDB(Program Database File)文件添加到项目里(拷贝到附加库目录就行), 这个需要个人进行编译生成的，一般会生成在ffmpeg源码文件里面(_他人的pdb文件是无法使用的，即使你二进制可执行文件也是使用他人生成的_)。<br>
* * *
初学者可以看看[雷霄烨](http://leixiaohua1020.github.io/)的项目，很有帮助。
