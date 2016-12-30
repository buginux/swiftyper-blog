---
title: iOS 逆向手把手教程之一：砸壳与class-dump
date: 2016-05-02 14:30:58
categories: iOSReverse
tags: [iOS, 逆向]
---

年前在学习 iOS 逆向的时候，为了练手，写了一个**微信抢红包插件**。但是由于过年之后就回老家了，开始新的生活总要分散掉一定的精力，所以关于 iOS 逆向的学习也就放下来了，到现在也有大半年没有动过了。

这段时间发现微信运动挺火的，经常有小伙伴在跟我炫耀说他走了多少步。我就突然想到，要不我来写一个 tweak 来修改微信运动的步数好了，我想调几步就几步，看你们还怎么得瑟。

说干就干，正好又赶上五一放假，除了相亲之外的时间，就都可以拿来写代码啦（笑~）。

<!-- more -->

然而，当我正准备大干一场的时候，我才发现我那一点点少得可怜的逆向知识全忘光了，我甚至连怎么砸壳导出微信的头文件都记不得了。不得已，只好重新开始找书找资料，边参考边做，所以我觉得这次有必要学习重新做个笔记，又觉得既然要开始从头开始做记录，不如写成一个系列的教程好了，这样有兴趣的小伙伴也可以跟着做一下，毕竟能做个这种小东西在小伙伴面前装个逼还是挺有成就感的。

## 前导条件

既然要做基于 iOS 的逆向，我就假定读者是一位有一定经验的 iOS 应用开发者吧，同时你要有一把**越狱的 iPhone 手机**，电脑也应该已经装好了 **theos 开发环境**。

关于手机越狱和 theos 环境的安装，可以参考我[之前的博客](http://www.swiftyper.com/2016/01/25/ios-tweak-install-guide/)。

那就开始吧。

## 何为砸壳

正常情况下，如果要导出一个已安装应用的头文件，我们只需要使用 **class-dump**（关于 class-dump 下面会有更详细的介绍） 就可以了。但是当我们直接从 AppStore 上面下载安装应用的时候，这些应用都被苹果进行过加密了，在可执行文件上加了一层壳。

所以在这种情况下，如果我们想要获取头文件，就需要先破坏这一层保护壳，这就是所谓的“砸壳”。

## 砸壳

为了砸壳，我们需要使用到 **dumpdecrypted**，这个工具已经开源并且托管在了 GitHub 上面，我们需要进行手动编译。步骤如下：

1. 从 GitHub 上 clone 源码：
   
   ```bash
   $ cd ~/iOSReverse
   $ git clone git://github.com/stefanesser/dumpdecrypted/
   ```
2. 编译 **dumpdecrypted.dylib**：
   
   ```bash
   $ cd dumpdecrypted/
   $ make
   ```

执行完 `make` 命令之后，在当前目录下就会生成一个 **dumpdecrypted.dylib**，这个就是我们等下要使用到的砸壳工具。

有了砸壳工具之后，下一步就需要找到待砸的猎物了，这里就以微信为例子，来说明如何砸掉微信的壳，并导出它的头文件。

1. 使用 ssh 连上你的 iOS 系统，关于 ssh 的使用在[之前的博客](http://www.swiftyper.com/2016/01/25/ios-tweak-install-guide/)里已经讲过了

   ```bash
   # 注意将 IP 换成你自己 iOS 系统的 IP
   $ ssh root@192.168.1.107
   ```

2. 连上之后，使用 `ps` 配合 `grep` 命令来找到微信的可执行文件
   
   ```bash
   $ ps -e | grep WeChat
   4557 ??         5:35.98 /var/mobile/Containers/Bundle/Application/944128A6-C840-434C-AAE6-AE9A5128BE5B/WeChat.app/WeChat
 4838 ttys000    0:00.01 grep WeChat
   ```
   在这里，因为我们已经知道微信的应用名就叫 WeChat，所以可以直接进行关键字搜索。那如果我们不知道想要找到应用名称叫什么该怎么办呢？这种情况下也有一种简单的方法，那就是把 iOS 上的所有应用都关掉，只留下你想要定位的那个应用，这样使用 `ps` 打印出来就会只有目标应用了。
   
	> 注意：如果输入 ps 之后，系统提示没有这个命令，那就需要到 cydia 当中安装 `adv-cmds` 包。

3. 使用 **Cycript** 找到目标应用的 Documents 目录路径。
	
	> Cycript 是一款脚本语言，可以看成是 Objective-JavaScript，它最好用的地方在于，可以直接附加到进程上，然后直接测试函数的效果。关于 cycript 的更详细介绍可以看[它的官网](http://www.cycript.org/)，之后如果有需要，我也会再写一篇关于 cycript 的文章。
	
	安装 cycript 也很简单，直接在 cydia 上搜索 cycript 并安装就可以了。装完之后就可以来查找目标应用的 Documents 目录了：
	
	```bash
	$ cycript -p WeChat
	cy# NSHomeDirectory()
	@"/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD"
	```
	可以看到，我们打印出了应用的 Home 目录，而 Documents 目录就是在 Home 的下一层，所以在我这里 Documents 的全路径就为 `/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents`
	
	> 注意：可以使用 `Ctrl + D` 来退出 cycript
	
4. 将 **dumpdecrypted.dylib** 拷到 Documents 目录下：
	
	```bash
	# 注意将 IP 和路径换成你自己 iOS 系统上的
	# 注意本步是在电脑端进行的操作
	$ scp ~/iOSReverse/dumpdecrypted/dumpdecrypted.dylib root@192.168.1.107:/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents
	```

5. 砸壳

	```bash
	$ cd /var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents
	# 后面的路径即为一开始使用 ps 命令找到的目标应用可执行文件的路径 
	$ DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Containers/Bundle/Application/944128A6-C840-434C-AAE6-AE9A5128BE5B/WeChat.app/WeChat
	```
	完成后会在当前目录生成一个 WeChat.decrypted 文件，这就是砸壳后的文件。之后就是将它拷贝到 OS X 用 class-dump 来导出头文件啦。
	
## class-dump

### 简介

**class-dump** 是一个工具，它利用了 Objective-C 语言的运行时特性，将存储在 Mach-O 文件中的头文件信息提取出来，并生成对应的 .h 文件。

### 安装 class-dump

到 [class-dump 官网](http://stevenygard.com/projects/class-dump/) 进行下载，目前最新的版本为 3.5。

在个人目录下新建一个 `bin` 目录，并将其添加到 PATH 路径中，然后将下载后的 class-dump-3.5.dmg 里面的 class-dump 可执行文件复制到该 `bin` 目录下，赋予可执行权限：

```bash
$ mkdir ~/bin
$ vim ~/.bash_profile
# 编辑 ~/.bash_profile 文件，并添加如下一行
export PATH=$HOME/bin/:$PATH

# 将 class-dump 拷贝到 bin 目录后执行下面命令
$ chmod +x ~/bin/class-dump
```

### 使用 class-dump

先将刚刚砸壳后的 WeChat.decrypted 拷到本机：

```bash
$ mkdir ~/iOSReverse/WeChat
# 注意替换 IP和路径
$ scp root@192.168.1.107:/var/mobile/Containers/Data/Application/13727BC5-F7AA-4ABE-8527-CEDDA5A1DADD/Documents/WeChat.decrypted ~/iOSReverse/WeChat/
```

终于到最后一步啦，使用 class-dump 来将微信的头文件导出：

```bash
$ mkdir ~/iOSReverse/WeChat/Headers
$ cd ~/iOSReverse/WeChat/
$ class-dump -S -s -H WeChat.decrypted -o Headers/
```

这样，所有的头文件就都被导出到了 `~/iOSReverse/WeChat/Headers` 目录下了。

![](http://7xqonv.com1.z0.glb.clouddn.com/iOS-reverse-step-by-step-part-1-class-dump-dumped-files.png)

## 下一步

将应用的头文件导出来之后就可以大概看到应用的组织架构了，而且由于 Objective-C 的类与方法的命名也都十分清晰，所以可以很容易就找到应用中对应界面的头文件了。

但是，如果要分析一个 APP 光有头文件是远远不够的，我们还需要使用 Hopper 对其进行静态分析，以及使用 lldb 配合 reveal 和 cycript 进行动态分析。

这就是下一部分要学习的内容了。

## 参考文献

* [iOS 应用逆向工程](https://www.amazon.cn/gp/product/B00VFDVY7E/ref=as_li_tf_tl?ie=UTF8&camp=536&creative=3200&creativeASIN=B00VFDVY7E&linkCode=as2&tag=buginux-23)
	
