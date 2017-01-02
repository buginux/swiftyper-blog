---
title: iOS微信抢红包Tweak安装教程
tags: [逆向]
categories: [iOS]
date: 2016-01-25 18:53:42
update: 2016-04-19
---
<!-- 下面开始正文 -->

> 2016-12-27 更新：本 Tweak 已经上传到 Bigboss 源，现在只需要在 Cydia 搜索 WeChatRedEnvelop 就可以直接下载安装了。不再需要从源码进行安装。

最近在学习 iOS 逆向开发的时候，为了练手，开发了一个 iOS 版的微信抢红包 tweak，并且已经发布到了 Github 上面。

[微信抢红包 Tweak 的 Github 地址](https://github.com/buginux/WeChatRedEnvelop.git)

但是，很多小伙伴表示不会安装，特此写了这篇iOS tweak 安装教程。

> 说明：本篇文章只是为了说明如何在 iOS 当中安装 tweak，并不会涉及完整的逆向环境的搭建，也不会涉及到 tweak 的开发。如果对这方面有兴趣的童鞋可以参考[iOS应用逆向工程](https://www.amazon.cn/gp/product/B00VFDVY7E/ref=as_li_tf_tl?ie=UTF8&camp=536&creative=3200&creativeASIN=B00VFDVY7E&linkCode=as2&tag=buginux-23)这本书。

<!-- more -->

## 何谓 tweak

tweak 在维基百科上的定义是：对复杂的系统——通常是电子设备——进行微调或修改来增强其功能。

而在 iOS 当中，tweak 是指那些能够增强其它进程功能的 dylib。

可以将 tweak 理解为就是一个外挂，只不过这个外挂是以动态链接库的方式注入到目标应用当中。我们已经很了解外挂就是用来作一些原本的应用无法做到的事情，当然抢红包也属于这样的事。

## 如何安装 tweak

上面已经讲过，本篇文章不会涉及到 tweak  的开发，因此我们直接进入安装的主题。

### 对设备进行越狱

对设备进行越狱是安装 tweak 的首要前提，如果没有越狱设备，并且你不想对自己的设备进行越狱，那么也可以不用继续往下看了。

众所周知，苹果的权限管理是很严格的，在没有越狱的情况下，我们能对设备进行的操控其实是很有限的。而越狱之后我们就可以获得root权限，即最高权限。

现在国内最知名的越狱团队就是[盘古](http://pangu.io/)与[太极](http://www.taig.com/)了，他们都提供了 windows 版与 mac 版的越狱软件，可以进行一键越狱。

需要注意的是，这两家目前支持越狱的最新版本是 **9.0.2**。因此，如果你的设备已经更新到 9.2 了，你可能需要重新找一台比较低版本的设备。

使用工具进行越狱其实很简单，在此也就不赘述了。

### CydiaSubstrate

越狱完的设备上面都会多出一个 App，即 Cydia。

Cydia 可以理解为越狱界的 App Store。只不过 App Store 上面的都是经过苹果审核过的应用。而 Cydia 上面的各式各样的 App，tweak都或多或少使用到苹果在 App Store 审核中禁用的功能特性，比如私有方法。

CydiaSubstrate 是 Cydia 的作者的另一个作品，它的主要功能就是对 App 进行 hook，替换 App 中代码的实现，它是绝大部分 tweak 正常工作的基础，Cydia 上的 tweak 都是基于 CydiaSubstrate 的。

一般情况下，越狱完之后就已经安装了 CydiaSubstrate 了，如果你想看到软件包的详细信息，可以直接在 Cydia  当中搜索 Cydia Substrate。

### 安装 OpenSSH

OpenSSH 会在 iOS 上安装 SSH 服务，以供外界可以远程登录到 iOS 系统当中。

安装 OpenSSH 也很简单，同样在 Cydia 当中搜索 OpenSSH，然后进行安装就行了。

iOS 上的 OpenSSH 的默认用户有 root 和 mobile，默认密码都为 alpine。在这里强烈建议大家对默认密码进行修改，如果没有修改，很多病毒就可以轻易地通过 ssh 以 root 身份远程登录到 iOS 当中，这后果可是非常严重的。

修改密码的步骤：
1\. 确保你的电脑跟你的 iOS 设备在同一个局域网当中
2\. 获取 iOS 设备的 IP：设备 -> 无线局域网 -> 查看当前连接的 WIFI 的详细信息，就可以看到设备的 IP
3\. 在 Mac 上打开终端，执行命令 `ssh root@DeveiceIP`，将 DeveiceIP 替换成你的设备 IP
4\. 输入密码进行登录，注意密码是不会回显的，也就是不会显示普通的密码星号，只要继续输入就行了，输入完后按回车
5\. 登录后，修改root用户密码，执行命令`passwd root`，根据提示输入新密码
6\. 再修改mobile用户密码，执行`passwd mobile`，根据提示输入新密码

至此，iOS 设备上的环境就配置好了。

### 安装 Theos

Theos 是一个越狱开发工具包，它可以生成 iOS 越狱APP以及tweak等程序的框架，并提供makefile来编译、打包和安装。

#### 安装 Xcode 和 Command Line Tools

会来看这篇教程的，我默认大家都是 iOS 开发者了，所以应该都已经安装了Xcode了，Xcode 就已经附带了 Command Line Tools。

#### 从 Github 下载 Theos

打开命令行，进行如下操作：

```bash
export THEOS=/opt/theos
# 如果之前已经安装过 theos，请先删除，然后下载最新版
rm -rf $THEOS
sudo git clone --recursive https://github.com/theos/theos.git $THEOS
```

#### 配置ldid

ldid是用于对 iOS 可执行文具进行签名的工具，可以在越狱 iOS 中替换 Xcode 自带的签名工具。

从 [http://joedj.net/ldid](http://joedj.net/ldid) 下载，将其移动到 `/opt/theos/bin` 目录下，然后设置可执行权限。

```
cd <下载ldid的目录>
sudo mv ldid /opt/theos/bin
sudo chmod 777 /opt/theos/bin/ldid
```

#### 配置CydiaSubstrate

运行 Theos 自动化配置脚本：

```
sudo /opt/theos/bin/bootstrap.sh substrate
```

> 注：最新版的 theos 里面已经没有这个脚本了，可以跳过执行脚本这一步

接下来要做的就是从 iOS 上已经安装的 Cydia Substrate 上复制 cydiaSubstrate 文件到 theos 上。

要想在 Mac 上访问 iOS 设备的文件目录，新手可以直接使用 [iFunBox](http://www.i-funbox.com/en_download.html)，或者如果你觉得不屑使用图形化工具，也可以直接使用 `scp` 命令来进行拷贝。

需要拷贝的文件位于 iOS 上的 "/Library/Frameworks/CydiaSubstrate.framework/CydiaSubstrate"，将其拷贝到 OSX 上，然后重命名为 libsubstrate.dylib 后放到 "/opt/theos/libsubstrate.dylib" 中。

#### 配置 dpkg-deb

deb 是越狱开发安装包的标准格式，而 dpkg-deb 是操作 deb 文件的工具，有了这个工具，Theos 才能将工程正确地打包成 deb 包。

从 [https://raw.githubusercontent.com/DHowett/dm.pl/master/dm.pl](https://raw.githubusercontent.com/DHowett/dm.pl/master/dm.pl)下载dm.pl，将其重命名为 **dpkg-deb** 后，放到 "/opt/theos/bin/" 目录下，然后设置它的可执行权限：

```
sudo chmod 777 /opt/theos/bin/dpkg-deb
```

其实，Theos 已经是一个 tweak 的开发环境了，但是由于这里只是因为需要编译 tweak 而用到它，所以它的很多后续配置也没有详细讲解了。

至此，我们的安装环境就搭建完了，下一步可以正式地开始安装 tweak 了。

### 正式安装 tweak

根据上面的内容，我们大概知道了，如果要安装一个别人的 tweak，最简单的方法就是直接到 Cydia 上面进行下载并自动安装，但是前提就是你想要安装的这个 tweak 的作者已经将这个 tweak 提交到 cydia 源当中了。

~~那你可能会问，那我为什么不直接提交到 Cydia 呢，多方便，多简单，反而还要绕这么一大圈，然后特地又写一篇博客来说明怎么从源码进行安装，是不是太久没装逼憋坏了？我的回答是：因为我还不会提交，哈哈哈。当然那只是其中一部分原因，主要还是因为我做这个插件原来也是出于练手学习的目的，后来有很多小伙伴都跟我要，我才放到 Github 上的。而且，这种作弊式的插件有点破坏游戏平衡了，原本抢红包也都是图个欢乐，如果作弊了就破坏了这种气氛（道貌岸然状）。~~

本 tweak 已经提交到 Cydia 中的 Bigboss 源，现在只需要在 Cydia 搜索 WeChatRedDevelop 即可下载安装了。

所以，经过我“苦口婆心”的劝说，你还是想安装的话，[就到我的Github去下载吧](https://github.com/buginux/WeChatRedEnvelop.git)（如果觉得好用，就点个星鼓励一下吧）。

将仓库拉取下来后，可以看到主目录里有一个 `Tweak.xm` 文件，主要的代码就写在这个文件中，其实里面的代码也就几行，做 tweak  的主要精力还是花在找你的目标方法上，真正写代码其实不会太多。当然，你如果只是单纯想安装的话，这个文件就跟你无关了。

主要修改的是`Makefile`文件，使用编辑器打开Makefile文件，可以看到头两行是这样的：

```
THEOS_DEVICE_IP = localhost
THEOS_DEVICE_PORT = 2222
```

将 **localhost** 替换成你的 iOS 设备的 IP，IP的获取方法在上面已经提过了。然后将端口 **2222** 替换成 **22**。

修改并保存后就可以进行安装了。

在 Mac 下打开终端命令行，并切换到这个仓库的目录，首先确保你的 iOS 设备上的微信是在运行中的，然后执行如下的命令：

```
make package install
```

之后，根据提示，输入两次密码（这个密码就是你刚刚修改过的密码），然后安装就完成了。

就是这么简单。

## 总结

洋洋洒洒写了这么多，其实真正的步骤是很简单的，只是我比较啰嗦，想把每个步骤都尽量讲得详细一点。

如果还有不清楚的地方，可以直接留言提问，或者直接到[我的Github](https://github.com/buginux/WeChatRedEnvelop.git)上提issue。

当然，我也是新手，刚开始学习逆向，可能有些地方理解不准确或有错误，欢迎批评指证。

再多啰嗦一句，这里讲的都是很浅很浅的东西，可以说跟逆向只能搭上一丢丢的边，如果你对逆向特别有兴趣的话，强烈推荐去看下这本书[iOS应用逆向工程](https://www.amazon.cn/gp/product/B00VFDVY7E/ref=as_li_tf_tl?ie=UTF8&camp=536&creative=3200&creativeASIN=B00VFDVY7E&linkCode=as2&tag=buginux-23)。

## 更新

### 2016-04-19

* 更新最新版 theos 安装方法

### 2016-12-27

* Tweak 提交到 Cydia 市场
