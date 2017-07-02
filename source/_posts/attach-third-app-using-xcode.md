---
title: 使用 Xcode 调试第三方应用
date: 2017-07-02 16:37:50
tags: [逆向, Xcode, lldb]
---

要使用 lldb 对应用进行断点调试，首要的前提就是 lldb 能附加到指定的应用上。而要能让 lldb 附加到应用上，就需要 debugserver 工具的帮助。

一直以来，我们都是使用配置过了 `task_for_pid` 权限的 debugserver 来调试越狱环境下的第三方应用。由于 iOS 上原版的 debugserver 不可写，因此我们无法使用修改过的 debugserver 来对其进行覆盖，而 Xcode 上的 lldb 会直接使用原版的 debugserver 来进行应用的附加。这样导致的结果就是，我们只能在命令行上对第三方应用进行断点调试。

<!-- more -->

如果强行使用 Xcode，就会导致如下结果：

![](http://7xqonv.com1.z0.glb.clouddn.com/attach-third-app-using-xcode-pic-1.jpg)

那么问题来了，我们有办法使用 Xcode 来对第三方应用进行断点调试吗？如果你使用过 [IPAPatch](https://github.com/Naituw/IPAPatch) 等工具的话，就会知道答案是”可以“。

其中的关键就在于[上一篇文章](http://swiftyper.com/2017/06/27/ios-app-signer-source-code/)中提到过的 `entitlements.plist` 文件中指定的权限了。

## get-task-allow

如果我们去查看那些我们自己使用开发者证书签名的描述文件，我们可以在其中的 Entitlements 中发现一个特殊的字段 `get-task-allow`：

```bash
<key>Entitlements</key>
<dict>
    <key>get-task-allow</key>
    <true/>
    <key>application-identifier</key>
    <string>AMCXGTYG4X.com.swiftyper.HitList</string>
    <key>com.apple.developer.team-identifier</key>
    <string>AMCXGTYG4X</string>
</dict>
```

这个字段就是决定我们能不能对应用进行断点调试的关键，关于这个字段的解释是这样的，我从 [StackOverflow](https://stackoverflow.com/questions/1003066/what-does-get-task-allow-do-in-xcode) 上找到的解释是这样的：

> get-task-allow, when signed into an application, allows other processes (like the debugger) to attach to your app. Distribution profiles require that this value be turned off, while development profiles require this value to be turned on (otherwise Xcode would never be able to launch and attach to your app).

对于我们自己写的应用，我们当然是可以调试的，所以这个字段是存在的。而对于第三方应用，这个字段已经被关闭了，因此我们也就无法使用 Xcode 对其进行断点调试。

所以，现在的问题变成了，如果为应用程序增加 `get-task-allow` 权限。

## 越狱设备

对于越狱设备来讲，那就很简单了，只需要使用 `ldid` 命令就可以了，在此只做简单的说明。

1. 将应用程序从设备上拷贝到本地
2. 利用 `ldid` 将应用程序的 code sign 导出：`ldid -e WeChat >> WeChat.xml`
3. 在 `WeChat.xml` 文件中添加 `get-task-allow` 权限
4. 利用 `ldid` 对应用进行重签名 `ldid -SWeChat.xml ./WeChat`
5. 将应用拷回设备，将设置可执行权限 `chmod 755 WeChat`
6. 完成

## 非越狱设备

对于非越狱的设备，要修改权限，不可避免地要涉及到重签名的步骤。正好在[上一篇文章](http://swiftyper.com/2017/06/27/ios-app-signer-source-code/)中解析过了 `ios-app-signer` 的源码，这次就可以派上用场了。我们利用文章里面提到的步骤来手动实践下重签名的过程，当然，还是以我们的老朋友微信为例。

### 获取应用程序

这个步骤就不用多说了，可以从自己的越狱机上砸壳后拷到本地，也可以直接从某助手上下载砸过壳的应用。关于这点在[之前的很多文章](http://swiftyper.com/archives/)都有提过了。

### 生成描述文件

对获取到的 `ipa` 文件进行解压，并将其中的 `.app` 文件拷贝到桌面（或其它任意目录）上。

如果你有付费的开发证书，或者企业证书，就直接使用它们生成的描述文件。如果都没有的话，也可以使用免费的个人证书，但是免费证书的过期时间是 7 天，因此在 7 天后再打开重签名的应用会闪退。

这里就以免费证书为例。

随便新建一个 Demo 应用，让 Xcode 进行自动签名，在真机上运行一下。这样，我们就能获得一个免费证书生成的描述文件了。

所有的描述文件都存放在 `~/Library/MobileDevice/Provisioning Profiles` 目录下。一般情况下，这里的描述文件应该不止一个，我们可以使用命令来查看描述文件的内容：

```bash
security cms -D -i 24195625-bc0c-4077-9fdf-e6554f628b3e.mobileprovision
```

但是，使用命令很麻烦。这里介绍一个[好用的工具](https://github.com/chockenberry/Provisioning)，可以直接使用系统的 QuickLook 对描述文件进行查看。

找到我们刚刚的 Demo 应用生成的描述文件后，将其拷贝到 `WeChat.app` 目录下，并重命名为 `embedded.mobileprovision`。

```bash
$ cp ~/Library/MobileDevice/Provisioning\ Profiles\24195625-bc0c-4077-9fdf-e6554f628b3e.mobileprovision ~/Desktop/WeChat.app/embedded.mobileprovision
```

## 生成 entitlements.plist 文件

如前所言，这个文件决定了应用程序可以使用哪些权限。并且这个文件的内容是可以从描述文件中的 `Entitlements` 字段中获取的。

我们这里使用 `PlistBuddy` 来生成该文件：

```bash
$ cd ~/Desktop/WeChat.app
$ /usr/libexec/PlistBuddy -c "Print :Entitlements" mobileprovision.plist -x > entitlements.plist
```

上面的命令将 `mobileprovision.plist` 文件中的 `Entitlements` 字段的内容提取出来，并使用 xml 的格式存放到 `entitlements.plist` 中。

`entitlements.plist` 文件生成后，要确认下里面是否有 `get-task-allow` 字段，如果没有的话，要自己手动添加，否则我们所做的工作就都没有意义了。字段的具体格式，可以参考文章开头中描述文件的 Entitlements 字段内容。


## 删除旧的资源签名

使用 `defaults` 命令直接删除 `Info.plist` 中的 `CFBundleREsourceSpecification` 字段：

```bash
$ cd ~/Desktop/WeChat.app
$ defaults delete CFBundleResourceSpe Info.plist
```

如果系统提示说 `Info.plist` 中没有 `CFBundleREsourceSpecification` 是正常的，不会影响我们后续的操作。

## 选择重签名要使用的证书

现在万事具备了，只要再选择一个用来重签名的证书就可以了，可以使用 `security` 命令列出现在设备上所有的证书：

```bash
$ security find-identity -v -p codesigning
# 1) 28AC348120212577CF465864C9563D9734EA4239 "iPhone Developer: 小锅 (5UG5DFJWF6)
```

## 重签

确认好了证书之后，就可以来进行重签名了：

```bash
$ cd ~/Desktop/
$ codesign -vvv -fs "iPhone Developer: 小锅 (5UG5DFJWF6)" --entitlements=WeChat.app/entitlements.plist --no-strict WeChat.app
```

## 打包

如果一切顺利的话，至此我们就已经完成重签名了，接下来就是将其打包回 `.ipa` 格式的文件了，我们可以直接使用 `zip` 命令来完成。

```bash
$ cd ~/Desktop
# 先新建一个 Payload 目录
$ mkdir Payload
# 将 WeChat.app 拷贝到 Payload 目录中
$ mv WeChat.app Payload
# 使用 zip 命令打包
$ zip -ry WeChat.ipa Payload
```

经过上面几个命令后，我们就能在桌面上发现一个 WeChat.ipa 的文件，这个就是我们重签名过，并打包完成的安装包了，接下来可以直接使用 Xcode 进行安装。

关于如何使用 Xcode 安装 ipa 文件，可以参考我之前的[免越狱红包插件的文章](http://swiftyper.com/2016/12/26/wechat-redenvelop-tweak-for-non-jailbroken-iphone/)。

## 使用 Xcode 进行调试

搞完了这么一大圈之后，我们终于可以使用 Xcode 来对应用进行断点调试了，而且是在非越狱的环境下。使用 Xcode 进行断点调试的步骤如下：

1. 将手机连接到电脑上（Xcode 9 可以无线调试，不过我还没实验过）
2. 使用 Xcode 随便打开一个工程
3. 菜单选择 `Debug -> Attach To Process`
4. 在列表中找到你要调试的应用，这里就是 WeChat
5. 静静等待 Xcode 使用 lldb 附加到进程即可

只要 Xcode 成功附加到进程后，我们就可以直接在 Xcode 中进行调试了，有了熟悉的图形界面，调试起来是不是更带劲了呢。

整体的效果如下：
![屏幕快照 2017-07-02 下午6.31.24](http://7xqonv.com1.z0.glb.clouddn.com/attach-third-app-using-xcode-pic-2.png)

## 小结

为了能使用 Xcode 进行断点调试，我们似乎费了很多的功夫。然而实际上，文章的大多内容是在描述如何进行重签名，这当然不是为了赚稿费，不管我写多少字都没人给我钱。主要的目的还是为了对[上一篇文章]()进行实践验证，所谓“纸上得来终觉浅，绝知此事要躬行"。

因此，这篇文章的两个目的就达到了。一是确认自己对 ios-app-signer 代码的理解是正确的，因为我自己通过实践验证过了，二是确认描述文件中的 `get-task-allow` 的作用，就是决定此 app 能否被 Xcode 所调试。

## 参考资料

* [iOS调试技巧（3）—— Attach to Process with Xcode](http://www.hotobear.com/?p=627)
* [StackOverflow](https://stackoverflow.com/questions/1003066/what-does-get-task-allow-do-in-xcode)

## 关注

如果你喜欢这篇文章，可以关注我的公众号，随时获取我最新的博客文章。

![qrcode_for_gh_6e8bddcdfca3_430.jpg](http://upload-images.jianshu.io/upload_images/650096-9155581b667f64b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


