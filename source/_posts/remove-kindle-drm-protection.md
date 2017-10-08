---
title: 去除 Kindle 电子书的 DRM 保护
date: 2017-10-08 11:25:38
tags: [kindle]
---

众所周知，中亚与美亚的帐号是不互通的，也就是说，如果我们同时从中国亚马逊和美国亚马逊购买了正版的 Kindle 电子书，是无法在同一台 Kindle 设备（Kindle 手机应用也是一样）上进行阅读的。

要解决这个问题，有两种方法。

一是我们可以通过切换帐号的方式来达到书籍共享的目的，但是这种方法有很多问题，kindle 手机应用无法直接选择登录的地区，必须通过切换系统语言的方式来登录对应的中亚或美亚帐号，同时在切换帐号的时候，前一个帐号的书籍会被抹掉。虽然在 Kindle 设备上切换帐号会保留上一个帐号的书籍，但是在 Kindle 设备那种响应速度下频繁进行帐号切换操作也是劳民伤财，费时费力。

第二种方法就是将直接下载或者导出购买过的 Kindle 电子书，然后使用邮件的方式推送到指定的账户。这种方法相对来说比较方便，我们可以直接在亚马逊网站上下载到购买过的电子书，或者也可以直接从设备中导出。但是，事情可没有那么简单，kindle 上所有的电子书都是有 DRM 保护的，直接传到 Kindle 里是打不开的。

这时，就要求我们对 DRM 进行破解，这也就是本篇文章的主题。

> 本方法只用于自己购买的正版书籍，并只用于本人阅读，请勿用于盗版电子书。

<!-- more -->

## 获取电子书

首先，我们先获取需要进行破解的 azw 或 azw3 格式的电子书，总共有三种方法。

### 一、从亚马逊官网下载

登录[亚马逊官网](https://www.amazon.cn)，从右上角“我的帐户”中选择“管理我的内容和设备”。

![屏幕快照_2017-10-08_下午12_11_30](http://7xqonv.com1.z0.glb.clouddn.com/remove-kindle-drm-protection-pic-1.png)

然后，从当中选择你想下载的书籍，点击“操作”中的“...”按钮，选择“通过电脑下载USB传输”。

![Amazon_cn：管理我的内容和设备](http://7xqonv.com1.z0.glb.clouddn.com/remove-kindle-drm-protection-pic-2.png)

然后，在弹窗中，选择点击“下载”，即可将电子书保存到本地。

![Amazon_cn：管理我的内容和设备](http://7xqonv.com1.z0.glb.clouddn.com/remove-kindle-drm-protection-pic-3.png)

### 二、使用 Kindle 桌面端进行下载

首先，去[官网下载 kindle 桌面端软件](https://www.amazon.cn/gp/digital/fiona/kcp-landing-page/ref=klp_mn)并安装。

安装完成后，登录对应的亚马逊帐号，然后下载想要破解的电子书，从电子书的存放路径中找到它们，并将其拷贝到桌面（或其它你想要的目录）。

Kindle for Windows 的电子书存放路径为：`C:\Users\你的用户名\Documents\My Kindle Content`。

Kindle for Mac 的电子书存放路径为：`/Users/你的用户名/Library/Application Support/Kindle/My Kindle Content`。

> 使用这种方法，电子书的命名由应用自行生成，类似 `B00FF3JVAC_EBOK.azw`，识别起来比较麻烦。

![屏幕快照 2017-10-08 下午12.28.18](http://7xqonv.com1.z0.glb.clouddn.com/remove-kindle-drm-protection-pic-4.png)

### 三、直接从 Kindle 设备中进行拷贝

使用 USB 线将 Kindle 连接到电脑，找到对应的 Kindle 目录，在其下的 Documents 文件夹内找到 azw 或 azw3 格式的文件，将其拷出。

## 移除 Kindle 电子书的 DRM

破解 DRM 其实很简单，只需要使用到 DeDRM_tools 工具即可，这个工具是开源的，可以[直接在 GitHub 上下载到](https://github.com/apprenticeharper/DeDRM_tools/releases)。

这个工具提供了两种方法来破解 DRM，第一种是直接使用它提供的 Mac 或 Windows 应用，第二种是使用 Calibre 插件，并配合 [Calibre](https://calibre-ebook.com/) 进行破解。

这里推荐使用第二种方法，因为第一种方法还需要到 Kindle 设备序列号，比较麻烦。

### 安装 Calibre 应用

Calibre 是一款开源的电子书管理应用，详细信息可以[参考它的官网](https://calibre-ebook.com/)。安装方法也很简单，直接在官网下载安装文件，并进行安装即可。

### 安装去除 DRM 保护的插件

Calibre 是支持插件的，并且它还有一个插件仓库用于方便安装各种插件。但是 Calibre 本身是不支持去除 DRM 的，并且官方插件仓库也没有提供这个插件。

到 [DeDRM_tools 的 releases](https://github.com/apprenticeharper/DeDRM_tools/releases)中下载最新的 zip 包，解压，在 `DeDRM_tools_6.5.4/DeDRM_calibre_plugin` 目录下有一个 readme 文档，还有一个 calibre 插件包 `DeDRM_plugin.zip`。

打开 Calibre，点击首选项（Mac 快捷键，cmd + ,），在“高级选项”中点选“插件”。接着，点击“从文件加载插件”，选择刚刚解压得到的 `DeDRM_plugin.zip` 进行安装。Calibre 会有一些安全警告，直接选择忽略，并一路下一步即可。

**安装完后，需要重启下 calibre**

### 使用插件

安装完插件之后，直接将要破解的 azw 或 azw3 文件添加到 Calibre 就可以了。插件会自动将有 DRM 保护的文件转换成原始格式。

有了原始格式的文件，无论是推送到自己的 Kindle 设备，还是使用 Calibre 将其转换成 epub 或者 txt 格式都是没有问题的。

### 其它方式

除了使用 Calibre 插件之外，DeDRM 还提供 Mac 及 Windows 的应用，我们也可以直接使用它们进行 DRM 的破解。

![DeDRM Application](http://7xqonv.com1.z0.glb.clouddn.com/remove-kindle-drm-protection-pic-5.png)

不过，这两个应用都需要使用到 Kindle 设备序列号，序列号可以直接在设备中的系统信息中找到。不同版本的 Kindle 位置不一样，但是基本上都是在“设置“中找到”设备信息“进行查看。

有了序列号之后，就可以打开应用，并进行相关的配置即可，过程比较简单，这里就不详细讲了。

> 再次声明，本篇文章，只供个人购买的正版书籍进行参考使用。请勿使用于传播盗版书籍，支持正版，人人有责！

## 参考资料

* [用Calibre导入Kindle电子书并去除DRM保护](https://www.librehat.com/importing-kindle-books-with-calibre-and-remove-drm-protection/)
* [用 DeDRM 破解去除 AZW 格式电子书 DRM 保护 （Kindle 图书破解）](http://www.jianshu.com/p/d838f593c8ca)
* [DeDRM](https://apprenticealf.wordpress.com/2012/09/10/drm-removal-tools-for-ebooks/)


