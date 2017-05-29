---
title: 破解 Mac 版 Ulysses 试用
date: 2017-05-29 15:19:03
tags: [逆向]
---

本着“如无必要，勿增实体”的原则，一直以来都是直接使用 vim 或者 sublime 编写 markdown 文档的。但是，作为一个程序员，总有一颗不安分的心，现在市面上有那么多优秀的 markdown 编辑器，不去试用一番，总觉得缺了点什么。

经过一轮试用，最终决定使用 Ulysses。其实，像 MWeb，Bear，MarkEditor 这些在编辑方面都是不相上下的，最终选择 ulysses 只是因为它的一个小功能，那就是文档的数字统计和目标功能。对于字数统计之前就一直想要有这个功能，但是 vim 里的 airline 不支持中文字数的统计，而且在提过 issue 后作者表明没有打算实现中文字数统计的功能，因此一直也是个缺憾。至于目标功能，对于像我这种不能坚持写文章的人也有奇效，实在不想写的时候，就给自己定一个字数，然后在写的过程中看着字数一点点朝着目标前进，也是一种相当有效的自我激励方式。

<!-- more -->

但是，Ulysses 只有 10 小时的试用时间，这些时间还不够我下决心花 283 人民币去买一个编辑器，毕竟我也不是专业写文章的。因此，我就想用自己的逆向技术，试试看能否为这个试用时间续几秒。

## 找到入手点

要找到破解的入手点就要看这个软件的试用版相比正式版做了哪些限制。对于 ulysses 来说，试用版跟正式版的功能上是没有差别的，只是它有试用时间，软件的使用时间超过试用时间之后，就无法再使用了。

因此，要入手的地方很明显，要么修改试用的剩余时间，要么修改判断过期的逻辑。

## 代码分析

找到了入手方向之后，就打开 Hopper 分析下代码吧。先到官网上去下载 Ulysses 的试用版，然后使用 Hopper 打开二进制文件。

对于限制试用时间的软件来说，`expiration（过期）`是一个关键詞，我们直接从这个词下手。直接在 Hopper 中进行搜索，你会发现输入到 `expir` 时就会出现一个很明显的方法 `[ULDemoRegistration isExpired]`。

![isExpired](http://7xqonv.com1.z0.glb.clouddn.com/ulysses-demo-mac-reverse-pic-1.png)

接下来就很好办了，按照破解迅雷离线下载的办法，直接把 `eax` 寄存改成 0，然后直接 `ret` 就可以了，具体可以[查看我的上一篇文章](http://swiftyper.com/2017/04/19/mac-thunder-reverse/)。

到这里，我们一开始的目的就已经达到了。但是，我还是想看看它判断过期的逻辑，因此需要继续探索。

可以看到 `isExpired` 是 `ULDemoRegistration` 这个类的方法，而且这个类名也很明显地表示出了它就是用来管理试用版软件的类，判断逻辑肯定在这个类里面。

直接使用 Hopper 搜索这个类名，可以看到它的所有方法。

![ULDemoRegistration](http://7xqonv.com1.z0.glb.clouddn.com/ulysses-demo-mac-reverse-pic-2.png)

找到 `[ULDemoRegistration minutesLeftInDemo]` 方法，这个方法是用来计算试用的剩余时间的，我们打开 Hopper 中的伪代码模式，可以看到它的实现很简单：

![minutesLeftInDemo](http://7xqonv.com1.z0.glb.clouddn.com/ulysses-demo-mac-reverse-pic-3.png)

它只是从 `demoDictionary` 中读取 `currentUsageTime` 的值，然后再与本次的使用时间进行计算而已。所以，关键就在 `demoDictionary` 这个字典中，它是从哪里获取到的呢？

如果你刚刚有认真看 `ULDemoRegistration` 类的方法列表的话，只会发现它有一个 `demoDictionary` 的 get 方法。进到这个方法的伪代码里面，可以发现它只是从本地的文件中进行的初始化，而非进行网络请求从服务端获取使用时间的。

![demoDictionary](http://7xqonv.com1.z0.glb.clouddn.com/ulysses-demo-mac-reverse-pic-7.png)

> 一个这么贵的软件，居然使用了这么简单的验证方法，不免让人大跌眼镜。不过话说回来，国外的软件大多情况下也是防君子不防小人的，很多软件个人的 license 也不做设备数验证的，像是 Hopper 跟 Alfred，基本上三五个人用同一个 license 貌似都是没有问题的。

既然软件的使用时间都是记录在本地的，那就直接去修改这个文件，或者更暴力点直接删除这个文件，就可以无限试用了，连改可执行文件都免了。

那么，应该怎么找到这个路径呢。可以看到它还有一个 `demoDictionaryPath` 的方法用来获取本地路径，只要我们拿到这个方法的返回值就可以了。

这时就要用到 lldb 这个利器了。

## 使用 lldb 进行动态调试

由于苹果在 `os x 10.11` 后使用了 `Rootless` 技术，因此我们没办法直接使用 lldb 附加到进程上，因此我们需要先将 `Rootless` 关闭。

### 关闭 Rootless

> Rootless 是苹果为了加强安全性而引入的，关闭它会带来一定的风险。因此请保证你明白自己在做什么之后再将它进行关闭。

要关闭 `Rootless` 也很简单，步骤如下：
1. 重启机器
2. 当系统正在启动时，按住 `Command + R`，直接苹果的启动图标出现为止。这会让我们进入到 `Recovery Mode`。（重装过系统的小伙伴应该对这个界面比较熟悉）
3. 在 `Utilities（实用工具）` 中找到 `Terminal（终端）`并打开
4. 在终端命令行中输入 `csrutil disable; reboot`
5. 系统会自动重启，这时就已经关闭了 `Rootless`

### 使用 lldb 附加进程

如果你已经打开了 Ulysses，先将它彻底退出 `Command + q`。

然后在终端中输入：

```
$ lldb -n Ulysses\ Demo -w
```

注意，中间要有反斜杠，不会空格有问题。`-w` 参数说明要 lldb 等待应用程序启动，因为 `demoDictionary` 是在启动的时候去读取的，如果我们不使用这个参数，就会断不到点。然后，启动 Ulysses，lldb 就会附加到进程上。

找到 `demoDictionaryPath` 方法内的任意一行的偏移地址，然后下断点：

```
(lldb) image list -o -f
# [  0] 0x000000000aff8000 /Applications/Ulysses Demo.app/Contents/MacOS/Ulysses Demo
# [  1] 0x00000001177bc000 /usr/lib/dyld
# [  2] 0x00007fff9d135000 /usr/lib/libiconv.2.dylib
# [  3] 0x00007fff8a363000 /System/Library/Frameworks/Foundation.framework/Versions/C/Foundation
# [  4] 0x00007fff8ebf2000 /System/Library/Frameworks/Security.framework/Versions/A/Security
# [  5] 0x00007fff90cb1000 /System/Library/Frameworks/WebKit.framework/Versions/A/WebKit

(lldb) br s -a 0x000000000aff8000+0x00000001003be2f7
```

我们在 `demoDictionaryPath` 方法中断了点，我们只要等到断点被命中后，使用 `step out` 执行到这个方法的结束，然后打印出返回值就可以了：

```
(lldb) c
# 继续执行后，会命中我们的断点
# Process 5758 resuming
# Process 5758 stopped
# * thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
#     frame #0: 0x0000000101aa12fa Ulysses Demo`___lldb_unnamed_symbol19854$$Ulysses Demo + 7
# Ulysses Demo`___lldb_unnamed_symbol19854$$Ulysses Demo:
# ->  0x101aa12fa <+7>:  sub    rsp, 0x30
#     0x101aa12fe <+11>: mov    rax, qword ptr [rip + 0x1ee3a3] ; (void *)0x00007fffa6fc7060: _NSConcreteStackBlock
#     0x101aa1305 <+18>: mov    qword ptr [rbp - 0x40], rax
#     0x101aa1309 <+22>: mov    dword ptr [rbp - 0x38], 0xc2000000
(lldb) thread step out
# Process 5758 stopped
# * thread #1, queue = 'com.apple.main-thread', stop reason = step out
#     frame #0: 0x0000000101aa1565 Ulysses Demo`___lldb_unnamed_symbol19858$$Ulysses Demo + 60
# Ulysses Demo`___lldb_unnamed_symbol19858$$Ulysses Demo:
# ->  0x101aa1565 <+60>: mov    rdi, rax
#     0x101aa1568 <+63>: call   0x101ba78b8               ; symbol stub for: objc_retainAutoreleasedReturnValue
#     0x101aa156d <+68>: mov    r14, rax
#     0x101aa1570 <+71>: mov    rdi, qword ptr [rip + 0x331339] ; (void *)0x00007fffa3d2ad28: NSFileManager
(lldb) po $rax
# /Users/<username>/Library/Preferences/.com.soulmen.ulysses3.reg
```

在 64 位的 intel 汇编下，返回值是存放在 `rax` 寄存器中的。这时，我们就拿到了这个文件的路径，可以自己写个测试代码来读取下这个文件的内容。

```objc
NSString *path = @"/Users/<username>/Library/Preferences/.com.soulmen.ulysses3.reg";
NSData *data = [NSData dataWithContentsOfFile:path];

id result = [NSPropertyListSerialization propertyListWithData:data options:0 format:0 error:nil];

NSLog(@"%@", result);
```

打印出的内容如下：

![Output](http://7xqonv.com1.z0.glb.clouddn.com/ulysses-demo-mac-reverse-pic-4.png)

接下来，退出 lldb，并把 Ulysses 完全退出，然后删除这个文件，再重新打开应用。可以看到，试用时间恢复了 10 小时，可以确定这个文件就是我们所要找的文件。

![](http://7xqonv.com1.z0.glb.clouddn.com/ulysses-demo-mac-reverse-pic-5.png)
![](http://7xqonv.com1.z0.glb.clouddn.com/ulysses-demo-mac-reverse-pic-6.png)

## 小结

逆向最重要的一个技能就是“猜”，这次的逆向之所以这么顺利，也是因为我们单刀直入，直接猜到了过期验证会用到 `expiration` 这个单词，因此后面直接就没有碰到什么困难了。大胆假设，小心求证，对于逆向来说尤其适用。

## 赞赏

如果本篇文章对你有帮助，可以进行小额赞助，鼓励作者写出更好的文章。

![Paying](http://7xqonv.com1.z0.glb.clouddn.com/wechatpaying.png)
 