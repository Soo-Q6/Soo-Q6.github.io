---
title: MSYS2的一些事
# image: /assets/img/blog/...
description: >
  原来是想记录一下在MSYS2上使用git时无法生成ssh-key的小坑，后面的话也会记录一点MSYS2的使用技巧。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags:
---
偶然在msys2中使用git clone一个加密的仓库...<br>

然后也介绍一下如何在Windows中使用MSYS2构建类Unix环境


## MSYS2中竟然无法使用ssh-keygen生成密钥
作为开发人员，想必是一定会使用到`git`的吧。如果你在Windows环境下想做一些Linux的工作(比如上一篇的编译ffmpeg)，又碰巧你还会用到`git`，虽然在MSYS2中是可以使用Windows安装的`git`。但是，当你需要生成ssh-key的时候，发现使用Windows的`ssh-keygen`会报错:<br>
![卡在这里了...]({{site.data.strings.blog_url}}msys2-1.png)<br>

不管你是不是默认的文件保存路径，结果都是一样的，目前原因是什么还不清楚。但是可以这样解决:
```shell
# 在MSYS2中下载openssh
pacman -S openssh

# 检查一下安装位置
which ssh-keygen

cd ~/.ssh/
# 生成公钥并将公钥放置在known_hosts文件
# host是你访问的主机ip，写进文件的内容是 host ssh-rsa(or others) ***=
ssh-keyscan host
echo *** > known_hosts
```
或者你不想生成公钥, 还可以这样(不过不安全)<br>
修改~/.ssh/config文件, 如下:
```
StrictHostKeyChecking no
UserKnownHostsFile /dev/null
```

然后就好了

## MSYS2的包管理工具pacman的下载问题
参考[上一篇](https://soo-q6.github.io/blog/2019-09-22-compile-ffmpeg-in-windows/)中的MSYS2修改源和设置下载timeout

## 在Windows中配置Unix-like环境

在Windows中构建Unix-like环境，主要的方案又Cygwin和MSYS。相比于Cygwin，MSYS主要的优点就是轻量且快， 而且还可以使用包管理工具下载Linux的工具包。

> MSYS2 is a port of a collection of standard Unix/Linux utilities to Windows. It provides shells, development tools and version control system programs.<br>

借助MSYS2可以很方便的在Windows构建一个Unix/Linux环境，可以做一些很有意思的事情，比如编译某些C项目...<br>
1. MSYS2比较方便一点的就是，它可以识别Unix风格的路径, 比如`/c/User/`, 在你执行某些命令的时候，比较方便(但是ssh-keygen生成密钥的时候...就不吐槽了)。
2. MSYS2中的一些工具, 比如`grep`, 是可以在Windows的命令行工具中使用的, 你只需要将MSYS2的`usr/bin`和`usr/local/bin`目录添加到系统环境变量中就可以了。虽然Windows 10中已经有了`grep`的替代品`findstr`但是不是很好用...
3. 利用MSYS2你还可以在Windows上设置一个ssh代理服务器, 像Linux那样可以被ssh连接。<br>

首先，下载`openssh`

```
    pacman -S openssh
```

为ssh-server生成密钥:

```
  ssh-keygen -f /etc/ssh/ssh_host_rsa_key -N '' -t rsa
  ssh-keygen -f /etc/ssh/ssh_host_dsa_key -N '' -t dsa
  ssh-keygen -f /etc/ssh/ssh_host_ecdsa_key -N '' -t ecdsa
  ssh-keygen -f /etc/ssh/ssh_host_ed25519_key -N '' -t ed25519
```

然后修改`/etc/ssh/sshd_config`

```
UsePrivilegeSeparation no
PasswordAuthentication yes
```
然后启动服务器`/usr/bin/sshd`，这里防火墙需要允许一下<br>

接着你就可以使用`ssh -l <your_windows_username> localhost`进行连接了。<br>

BTW, 你还可以设置ssh服务器的开机自启, 后面有时间补上吧