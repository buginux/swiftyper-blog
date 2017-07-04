---
title: 使用 lldb 对应用砸壳
date: 2017-07-04 19:55:34
tags: [逆向, lldb]
---

进行 iOS 越狱开发的第一步，通常就是对从 AppStore 上下载安装的应用进行砸壳。我们常用的工具就是 [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted) 或者 [Clutch](https://github.com/KJCracks/Clutch)，关于这两个工具的使用，在[之前的文章](http://swiftyper.com/archives/) 已经进行过介绍了。

事实上，除了使用工具帮助我们进行砸壳，我们还可以直接使用 lldb 进行手动砸壳。

<!-- more -->

## 原理

手动砸壳的原理其实很简单，在应用被加载完成后，对应的解密工作就已经完成了。

这个时候，我们就可以通过 loadCommand 中的 `LC_ENCRYPTION_INFO` 与 `LC_ENCRYPTION_INFO_64` 信息，找到对应的偏移，然后将解密后的数据从内存中 dump 出来，再使用这份数据对原来的可执行文件进行补丁，这样我们就得到了一个砸过壳的可执行文件了。

了解过原理后，下面就可以直接动手了。

## 准备工作

在开始真正进行砸壳之前，要先将 iphone 设备上的可执行文件拷贝到本地。

由于从 AppStore 安装的应用都是在 Application 目录下的，因此我们可以使用 `ps` 配合 `grep` 命令来找到我们的目标应用。

```bash
iphone$ ps -e | grep Application

  2815 ??         0:02.32 /var/mobile/Containers/Bundle/Application/577995FB-298A-40DC-88BE-FC9C7804D1E8/LuoJiFM-IOS.app/LuoJiFM-IOS
```

接着就可以使用 `scp` 命令将可执行文件拷贝到本机上。

```bash
# 拷贝到 Mac 的桌面上
osx$ scp root@<device_ip>:/var/mobile/Containers/Bundle/Application/577995FB-298A-40DC-88BE-FC9C7804D1E8/LuoJiFM-IOS.app/LuoJiFM-IOS ~/Desktop
```

## 查看可执行文件信息

将可执行文件拷贝到本地之后，我们就可以使用 `otool` 来查看它的信息了。

```bash
osx$ otool -fh LuoJiFM-IOS

Fat headers
fat_magic 0xcafebabe
nfat_arch 2
architecture 0
    cputype 12
    cpusubtype 9
    capabilities 0x0
    offset 16384
    size 21302352
    align 2^14 (16384)
architecture 1
    cputype 16777228
    cpusubtype 0
    capabilities 0x0
    offset 21331968
    size 25417728
    align 2^14 (16384)
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedface      12          9  0x00           2    63       6280 0x00218085
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777228          0  0x00           2    63       7008 0x00218085
```

可以看到这个可执行文件是一个多架构的文件，它包含 32 位与 64 位的架构。选择想要砸壳的架构查看它的加密信息，我这里选择的是 64 位架构。

```bash
osx$ cd ~/Desktop
osx$ otool -arch arm64 -l LuoJiFM-IOS | grep crypt

     cryptoff 16384
    cryptsize 19972096
      cryptid 1
```

其中，`cryptoff` 是加密信息的偏移，`cryptsize` 是加密信息的大小，而 `cryptid` 为 1 证明这个可执行文件是加密的。

有了这些信息后，我们就可以开始进行砸壳了。

## 使用 lldb 砸壳

配合 `debugserver` 并祭出 `lldb` 神器，附加到进程上。

```bash
iphone$ debugserver *:1234 -a LuoJiFM-IOS
# debugserver-@(#)PROGRAM:debugserver  PROJECT:debugserver-320.2.89 for arm64.
# Attaching to process LuoJiFM-IOS...
# Listening to port 1234 for a connection from *...

osx$ lldb
(lldb) process connect connect://<device_ip>:1234

Process 2815 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
    frame #0: 0x0000000196d24e0c libsystem_kernel.dylib` mach_msg_trap  + 8
libsystem_kernel.dylib`mach_msg_trap:->  0x196d24e0c <+8>: ret
libsystem_kernel.dylib'mach_msg_overwrite_trap:    0x196d24e10 <+0>: mov    x16, #-0x20
    0x196d24e14 <+4>: svc    #0x80
    0x196d24e18 <+8>: ret
libsystem_kernel.dylib'semaphore_signal_trap:    0x196d24e1c <+0>: mov    x16, #-0x21
    0x196d24e20 <+4>: svc    #0x80
    0x196d24e24 <+8>: ret
libsystem_kernel.dylib'semaphore_signal_all_trap:    0x196d24e28 <+0>: mov    x16, #-0x22
```

接着，找出可执行文件的偏移地址：

```bash
(lldb) image list LuoJiFM-IOS
[  0] 426D3B7D-5433-3EA3-9E8C-384830B6A104 0x0000000100010000 /private/var/mobile/Containers/Bundle/Application/577995FB-298A-40DC-88BE-FC9C7804D1E8/LuoJiFM-IOS.app/LuoJiFM-IOS (0x0000000100010000)
```

在此例中，偏移地址为 `0x0000000100010000`。

然后，dump 出可执行文件被加密部分的二进制信息。命令如下：

```bash
(lldb) memory read --force --outfile ./decrypted.bin --binary --count <cryptsize> <image offset>+<cryptoff>
```

将以上命令的 `cryptsize`、`image offset` 以及 `cryptoff` 都替换为你自己的数据。对于这个例子而言，实际的命令如下：

```bash
(lldb) memory read --force --outfile ./decrypted.bin --binary --count 19972096 0x0000000100010000+16384

# 19972096 bytes written to './decrypted.bin'
```

有了这个补丁文件，就可以对原来的可执行文件打补丁了。开始之前，先备份下原来的可执行文件。

```bash
# 先退出 lldb 并回到桌面
(lldb) exit
osx$ cd ~/Desktop
osx$ cp LuoJiFM-IOS LuoJiFM-IOS-bak
```

将 `decrypted.bin` 文件的内容写入到可执行文件中：

```bash
osx$ dd seek=21348352 bs=1 conv=notrunc if=./decrypted.bin of=./LuoJiFM-IOS

# 19972096+0 records in
# 19972096+0 records out
# 19972096 bytes transferred in 25.058427 secs (797021 bytes/sec)
```

上面命令中的 21348352=21331968+16384，其中 21331968 是架构的偏移（即上面 otool -fh 打印出的信息中对应架构的 offset 字段），而 16384 是 `cryptoff` 的值。

至此，我们就使用 lldb 完成对应用的手动砸壳。

最后，为了让可执行文件适用于某些应用（如 class-dump-z），我们需要将 `cryptid` 字段手动置为 0。这一步可以通过 MachOView 来完成。

![](/images/14991741609513.jpg)


将上面的 `Cyrpt ID` 改成 0，并保存即可。

## 验证

为了验证我们的砸壳过程是否成功，可以使用 `class-dump` 来测试导出头文件。如果头文件导出成功，则说明我们已经成功砸壳了。

```bash
osx$ class-dump --arch arm64 -H LuoJiFM-IOS -o Headers
osx$ cd Headers
osx$ ls -l | wc -l
#    2989
```

可以看到，我们导出了 2989 个头文件，说明整个砸壳过程是正确的。

## 小结

大概瞄了下 `dumpdecrypted` 的源码，貌似它的原理就是本篇文章中使用的方法。而对于 `Clutch`，由于还没有看过它的源码，因此目前还不知道它的实现方法，之后有时间来学习学习。

## 参考资料

* [Decrypting apps from AppStore](http://codedigging.com/blog/2016-03-01-decrypting-apps-from-appstore/)
* [iOS Hacking Guide](https://web.securityinnovation.com/hubfs/iOS%20Hacking%20Guide.pdf)

## 关注

如果你喜欢这篇文章，可以关注我的公众号，随时获取我最新的博客文章。

![qrcode_for_gh_6e8bddcdfca3_430.jpg](http://upload-images.jianshu.io/upload_images/650096-9155581b667f64b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




