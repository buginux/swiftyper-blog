---
title: iOS 逆向实战 - 钉钉签到打卡破解
date: 2017-02-15 09:08:46
tags: [逆向]
---

作为程序员，大家应该都碰到过这样的问题：公司要求加班到 10 点才算加班费或者报销打车费，而你在 9 点多的时候就把活干完了。这时，你是选择强行加班到 10 点，还是选择提前走人不要加班费呢。

所谓鱼和熊掌不可得兼，不过在这个问题上，如果公司恰巧使用了钉钉的考勤系统，我们还是可以做一点手脚的。而这就是这两天[我的一个朋友](http://www.saitjr.com/)对我提出的需求，伪装钉钉的 GPS 定位，实现躺在宿舍床上也能打卡签到的功能。

我用了最简单粗暴的方法完成了这个需求。现在，再次应这位朋友的需求，把整个问题的解决过程记录一下，顺便把使用 theos 写 tweak 的整个流程再梳理一遍。

> 源码已经上传到 Github ：[https://github.com/buginux/DingTalkGPSFaker](https://github.com/buginux/DingTalkGPSFaker)

<!-- more -->

## 准备阶段

先上 iOS 逆向的三板斧，这次每次要做逆向破解前都需要做的步骤。

首先，请确保你的手机已经越狱，同时已经安装好 theos 开发环境，具体流程可以[参考这里](http://www.swiftyper.com/2016/01/25/ios-tweak-install-guide/)。

### 砸壳

如果是从 AppStore 上面下载的 App，都要先进行砸壳，否则是没办法进行下一步的。如果你懒得做这一步，就直接去其它助手类应该上面进行下载，那些都是砸过壳的，可以直接使用。

砸壳可以使用的工具有两种：

1. [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)，使用方法[请参考我之前的博客](http://www.swiftyper.com/2016/05/02/iOS-reverse-step-by-step-part-1-class-dump/)
2. [Clutch](https://github.com/KJCracks/Clutch)，使用方法[请参考我之前的另一篇博客](http://www.swiftyper.com/2016/12/26/wechat-redenvelop-tweak-for-non-jailbroken-iphone/)

在这里，我就以 Clutch 进行演示。

```bash
mac $ ssh root@<your.device.ip>
iphone $ clutch -i
# Installed apps:
# 1:   TestFlight <com.apple.TestFlight>
# 2:   微信读书 <com.tencent.weread>
# 3:   DingTalk <com.laiwang.DingTalk> # 不是很懂这个 laiwang 是啥意思
# 4:   1Password - Password Manager and Secure Wallet <com.agilebits.onepassword-ios>

iphone $ clutch -d com.laiwang.DingTalk
# com.laiwang.DingTalk contains watchOS 2 compatible application. It's not possible to dump watchOS 2 apps with Clutch 2.0.4 at this moment.
# Zipping DingTalk.app
# Swapping architectures..
# ASLR slide: 0x9d000
# Dumping <DingTalk> (armv7)
# ...
# DONE: /private/var/mobile/Documents/Dumped/com.laiwang.DingTalk-iOS7.0-(Clutch-2.0.4).ipa
# Finished dumping com.laiwang.DingTalk in 67.9 seconds
```
上面那句关于 watchOS 的警告是因为 Clutch 现在还不支持 extension 的砸壳功能，不过对我们暂时没影响，先忽略它就可以了。

同时，可以看到，clutch 将砸过后的 ipa 文件放到了 /private/var/mobile/Documents/Dumped/ 目录下。

我们将它改成一个比较简单的名字，然后拷回电脑上：

```bash
iphone $ cd /private/var/mobile/Documents/Dumped/
iphone $ mv com.laiwang.DingTalk-iOS7.0-\(Clutch-2.0.4\).ipa DingTalk.ipa
mac $ scp root@<your.device.ip>:/private/var/mobile/Documents/Dumped/DingTalk.ipa ~/Desktop/DingTalk
```
这里，我们将 ipa 从手机上拷贝到了电脑桌面上。

接下来，就是使用 class-dump 导出头文件了。

### 使用 class-dump 导出头文件

关于 class-dump 工具的安装，还是老样子，可以[参考之前文章](http://www.swiftyper.com/2016/05/02/iOS-reverse-step-by-step-part-1-class-dump/)中的 class-dump 小节。

先将 ipa 解压，然后导出头文件：

```bash
mac $ cd ~/Desktop
mac $ unzip DingTalk.ipa -d DingTalk
mac $ cd DingTalk
mac $ class-dump -S -s -H Payload/DingTalk.app/DingTalk -o Headers/
```

现在，钉钉的头文件已经被导出到 Headers 目录中了。接下来就是要开始分析，界面与功能了。

### UI 分析

一般来讲，在这步我都是直接使用 Reveal 找到我想要分析的那个界面，拿到它的控制器，然后在头文件中寻找相关的方法。利益于 Objective-C 的命名风格，一般通过名字大致就能判断出需要进行 Hook 的方法了。

在越狱机器上使用 Reveal 来分析界面是很简单的，只要在 cydia 市场搜索 Reveal Loader 并进行安装。之后就可以在设置中看到一个 Reveal 选项，进去之后在里面选择需要启动 Reveal 分析的应用，以后启动这些应用的时候，就可以在 reveal 中直接分析这些应用的界面了。

回到正题，继续钉钉界面的分析。

我很无奈地发现钉钉的签到界面是使用 WebView 实现的，这就对界面的分析造成了很大的困难，只好另寻它路了。

## 从定位入手

涉及到 WebView 的功能分析就比较麻烦了，从钉钉的头文件里面可以看出来，他们也是使用 WebViewJavascriptBridge 进行原生与 JS 交互的，这就涉及到了 URL 的拦截等问题。如果从这边入手的话，复杂度就超出了我的预期，并且为了一个简单的定位功能去分析整个协议也太不值得了。

既然这条路走不通，那就试试从另一个角度来分析。

既然涉及到定位，就肯定会用到 `didUpdateLocations` 这个回调方法，于是使用全局搜索一搜，发现了五个有关的回调方法：

```objc
[AMapLocationCLMDelegate locationManager:didUpdateLocations:]
[AMapLocationManager locationManager:didUpdateLocations:]
[DTCLocationManager locationManager:didUpdateLocations:]
[LALocationManager locationManager:didUpdateLocations:]
[MAMapView locationManager:didUpdateLocations:]
```

由于最后一个类是有关 MapView 的，而签到界面是没有地图的，所以可以放心地将其排除在外。

找到了相关类了，就可以使用 theos 附带的 logify 来进行分析了。说到 logify，那可真的是个强大的工具，它可以跟踪函数的调用，并输出 log。这样一来，就大大方便了我们跟踪某个功能的流程了。

首先，我们需要先在手机上安装 `syslogd to /var/log/syslog` 这个插件，它可以将我们代码中的 log 输出到 /var/log/syslog 文件中，方便我们进行观察。只要在 cydia 市场中搜索 syslogd to /var/log/syslog 并安装就可以了。

接下来就是利用 logify 来获取定位功能的流程了，不过首先要先创建一个 Tweak 工程。

```bash
mac $ /opt/theos/bin/nic.pl
# NIC 2.0 - New Instance Creator
# ------------------------------
#   [1.] iphone/activator_event
#   [2.] iphone/application_modern
#   ...
#   [11.] iphone/tweak
#   [12.] iphone/xpc_service
# Choose a Template (required): 11
# Project Name (required): DingTalkGPSFaker
# Package Name [com.yourcompany.dingtalkgpsfaker]: com.swiftyper.dingtalkgpsfaker
# Author/Maintainer Name [wordbeyondyoung]: 小锅
# [iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]: com.laiwang.DingTalk
# [iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]: DingTalk
# Done
```

在 Tweak 工程创建好之后，就可以使用 logify 来生成一个 Tweak.xm 文件了：

```bash
mac $ /opt/theos/bin/logify.pl Headers/AMapLocationCLMDelegate.h > dingtalkgpsfaker/Tweak.xm
mac $ /opt/theos/bin/logify.pl Headers/AMapLocationManager.h >> dingtalkgpsfaker/Tweak.xm
mac $ /opt/theos/bin/logify.pl Headers/DTCLocationManager.h >> dingtalkgpsfaker/Tweak.xm
mac $ /opt/theos/bin/logify.pl Headers/LALocationManager.h >> dingtalkgpsfaker/Tweak.xm
```
上面的这些命令，把 logify 生成的代码都输出到 Tweak.xm 这个文件中。

接着，就可以编译，并将这个 Tweak 安装到机器上了，不过，在些之前还需要对 Makefile 进行一下修改，导入 CoreLocation 框架。同时，由于 logify 生成的代码里面包含了太多的内容，其它我们只是要根据上面提到的四个方法而已，可以将其它无关的内容删除。

```bash
mac $ cd dingtalkgpsfaker
mac $ vim Makefile
mac $ vim Tweak.xm
```

修改后的 Makefile 内容如下：

```bash
THEOS_DEVICE_IP = <your.device.ip>
THEOS_DEVICE_PORT = <your.device.port>
ARCHS = armv7 arm64
TARGET = iphone:latest:7.0

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = DingTalkGPSFaker
DingTalkGPSFaker_FILES = Tweak.xm
DingTalkGPSFaker_FRAMEWORKS = CoreLocation

include $(THEOS_MAKE_PATH)/tweak.mk

after-install::
	install.exec "killall -9 DingTalk"
```

修改后的 Tweak.xm 内容如下：

```objc
#import <CoreLocation/CoreLocation.h>

%hook AMapLocationCLMDelegate
- (void)locationManager:(id)arg1 didUpdateLocations:(id)arg2 { %log; %orig; }
%end

%hook AMapLocationManager
- (void)locationManager:(id)arg1 didUpdateLocations:(id)arg2 { %log; %orig; }
%end

%hook DTCLocationManager
- (void)locationManager:(id)arg1 didUpdateLocations:(id)arg2 { %log; %orig; }
%end

%hook LALocationManager
- (void)locationManager:(id)arg1 didUpdateLocations:(id)arg2 { %log; %orig; }
%end
```

编译，安装到手机上：

```bash
mac $ export THEOS=/opt/theos
mac $ make package install
# > Making all for tweak DingTalkGPSFaker…
# make[2]: Nothing to be done for `internal-library-compile'.
# > Making stage for tweak DingTalkGPSFaker…
# dm.pl: building package `com.swiftyper.dingtalkgpsfaker:iphoneos-arm' in `./debs/com.swiftyper.dingtalkgpsfaker_0.0.1-4+debug_iphoneos-arm.deb'
# ==> Installing…
# (Reading database ... 4037 files and directories currently installed.)
# Preparing to unpack /tmp/_theos_install.deb ...
# Unpacking com.swiftyper.dingtalkgpsfaker (0.0.1-4+debug) over (0.0.1-3+debug) ...
# Setting up com.swiftyper.dingtalkgpsfaker (0.0.1-4+debug) ...
# install.exec "killall -9 DingTalk"
```

在开始操作流程之前，先把手机上的 /var/log/syslog 清空，防止其它无关信息的干扰：

```bash
iphone $ cat /dev/null > /var/log/syslog
```

在手机上操作完之后，将 log 拷回电脑上：

```bash
mac $ scp root@<your.device.ip>:/var/log/syslog ~/Desktop
mac $ vim ~/Desktop/syslog
```

在 log 文件中全局搜索 `didUpdateLocations:`，可以发现 `AMapLocationManager` 类里面的方法是最后被调动到的，所以可以确认我们需要 Hook 的是 `AMapLocationManager` 中的 `locationManager:didUpdateLocations:` 方法。

为了最快到检验这个方法，我使用了最简单和粗暴的方法——直接修改 `didUpdateLocations` 传进来的参数，并将其写死。

在还未进行修改之前，界面上显示的是“当前不在考勤范围内”，并且打卡按钮显示的是“外勤打卡”，如图：

![out-of-range](http://7xqonv.com1.z0.glb.clouddn.com/dingtalk-fake-gps-pic-1.jpg)

接下来，对 Tweak.xm 进行修改：

```objc

#import <CoreLocation/CoreLocation.h>

%hook AMapLocationManager

- (void)locationManager:(id)arg1 didUpdateLocations:(id)arg2 { 

	CLLocation *location = [[CLLocation alloc] initWithLatitude:<自己填> longitude:<自己填>];
	arg2 = @[location];

    // NSString *message = [NSString stringWithFormat:@"arg1 -- %@, arg2 -- %@", arg1, arg2];
    // UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"提示" message:message delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil];
    // [alert show];

	%orig; 
}

%end
```

注释掉的部分可以用于直观地检验我们的方法是否被调用，并把传入的参数使用 Alert 显示出来，方便观察。

重新编译安装后，再进入到打卡界面，可以看到我们已经进入正确的打卡范围了：

![in-range](http://7xqonv.com1.z0.glb.clouddn.com/dingtalk-fake-gps-pic-2.jpg)

关于 GPS 坐标的获取，这里再多说两句，由于众所周知的原因，国内的地图 GPS 都是存在偏移的。因此每家不同地图公司拿到的 GPS 都是不一样的，由于钉钉使用的是高德地图（毕竟同一个爸爸），所以我们需要在高德家的[坐标拾取](http://lbs.amap.com/console/show/picker)中获取的 GPS 才是钉钉上可以正常使用的。

## 小结

这个方法比较粗暴，直接修改 `didUpdateLocations:` 方法里面的坐标，难免在其它的场景会出现“误伤”的问题。但是，毕竟它确实是最快地解决我们当前问题的方法。

在逆向领域中，有时为了更快更方便地实现某个需求，就需要“不择手段”。毕竟我们没有源码，有时为了一个更优雅的解决方案，就要付出多几倍的时间和精力，我们要自己衡量这究竟值不值得。

现在，最核心的问题已经解决了，其它的就是增强和美化了，我们可以使用弹窗来让用户输入坐标，或者直接出一个地图选择界面让用户直接选择地点，而非把坐标写死。这些都是可以自由发挥的地方了，在这里就不多啰嗦了。

毕竟程序员大多数情况下是被压榨的，并且人品都是值得肯定的，所以我写一篇这样“歪门邪路”的文章，心里并不会觉得有所愧疚。

同时，这篇文章只是探究了一种可能性，这个 Tweak 没有经过全面的测试，也有可能存在其它的问题，所以请要使用这个方法的读者自己做好准备。这个锅，作者是不背的。

## 赞赏

如果这篇文章让你在公司的付出得到了更多对应的回报，你想请作者喝杯 koi，我是不拒绝的。

![wechatpaying](http://7xqonv.com1.z0.glb.clouddn.com/wechatpaying.png)



