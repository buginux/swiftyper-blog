---
title: 通过逆向深入理解 Block 的内存模型
date: 2016-12-16 14:24:50
tags: [逆向, Objective-C]
---

自从对 iOS 的逆向初窥门径后，我也经常通过它来分析一些比较大的应用，参考一下这些应用中某些功能的实现。这个探索的过程乐趣多多，不仅能满足自己对未知的好奇心，还经常能发现一些意外的惊喜。

正常情况下，通过分析界面以及 class-dump 出来头文件就能对某个功能的实现猜个八九不离十。但是 Block 这种特殊的类型在头文件中是看不出它的声明的，一些有 Block 回调的方法名 dump 出来是类似这样的：

```objc
- (void)FM_GetSubscribeList:(long long)arg1 pageSize:(long long)arg2 callBack:(CDUnknownBlockType)arg3;
```

因为这种回调看不到它的方法签名，我们无法知道这个 Block 到底有几个参数，也不知道它函数体的具体地址，因此在使用 lldb 进行动态调试的时候也是困难重重。我也一度被这个困难所阻挡，以为调用到有 Block 的方法就是进了死胡同，没办法继续跟踪下去了。我还因此放弃过好几次对某个功能的分析，特别受挫。

好在，我们还有 Google 这个强大的武器。没有什么问题是一次 Google 不能解决的。如果有，那就两次。

这篇文章就来讲讲如何通过 Block 的内存模型来分析出它的函数体地址，以及函数签名。

<!-- more -->

## Block 的内存结构

在 LLVM 文档中，可以看到 [Block 的实现规范](http://clang.llvm.org/docs/Block-ABI-Apple.html)，其中最关键的地方是对于 Block 内存结构的定义：

```objc
struct Block_literal_1 {
    void *isa; // initialized to &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 {
    	unsigned long int reserved;         // NULL
        unsigned long int size;         // sizeof(struct Block_literal_1)
        // optional helper functions
        void (*copy_helper)(void *dst, void *src);     // IFF (1<<25)
        void (*dispose_helper)(void *src);             // IFF (1<<25)
        // required ABI.2010.3.16
        const char *signature;                         // IFF (1<<30)
    } *descriptor;
    // imported variables
};
```

可以看到第一个成员是 `isa`，说明了 Block 在 Objective-C 当中也是一个对象。我们重点要关注的就是 `void (*invode)(void *, ...);` 和 descriptor 中的 `const char *signature`，前者指向了 Block 具体实现的地址，后者是表示 Block 函数签名的字符串。

## 实战

> 注：本篇文章都是在 64 位系统下进行分析，如果是 32 位系统，整型与指针类型的大小都是与 64 位不一致的，请自行进行修改。

知道了 Block 的内存模型后，就可以直接打开 hopper 和 lldb 进行调试了。

我这里使用了逻辑思维的得到 APP 作为分析的例子。顺便说一句，得到上面的内容都相当不错，很多付费专栏的内容都是很赞的，值得一看。

### 准备

设备：iPhone 5s iOS 8.2 越狱

usbmuxd

```bash
$ tcprelay -t 22:2222 1234:1234
Forwarding local port 2222 to remote port 22
Forwarding local port 1234 to remote port 1234
......
```

ssh 到 iOS 设备并启动 debugserver：

```bash
$ ssh root@localhost -p 2222
iPhone $ debugserver *:1234 -a "LuoJiFM-IOS"
ebugserver-@(#)PROGRAM:debugserver  PROJECT:debugserver-320.2.89
 for arm64.
Attaching to process LuoJiFM-IOS...
Listening to port 1234 for a connection from *...
```

本地打开 lldb 并远程附加进程，进行动态调试：

```bash
$ lldb
(lldb) process connect connect://localhost:1234
```

找到偏移地址：

```bash
(lldb) image list -o -f 
[  0] 0x0000000000074000 /private/var/mobile/Containers/Bundle/Application/D106C0E3-D874-4534-AED6-A7104131B31D/LuoJiFM-IOS.app/LuoJiFM-IOS(0x0000000100074000)
[  1] 0x000000000002c000 /Users/wordbeyond/Library/Developer/Xcode/iOS DeviceSupport/8.2 (12D508)/Symbols/usr/lib/dyld
```

在 Hopper 下找到需要断点的地址：

![Breakpoint Address](http://7xqonv.com1.z0.glb.clouddn.com/debuging-objective-c-blocks-in-lldb-pic-1.png)

下断点：

```bash
(lldb) br s -a 0x0000000000074000+0x0000000100069700
Breakpoint 2: where = LuoJiFM-IOS`_mh_execute_header + 407504, address = 0x00000001000dd700
```

然后在应用中点击*订阅* Tab ，此时会命中断点（如果没有命中，手动下拉刷新下）。

众所周知，Objective-C 方法的调用都会转化成 `objc_msgSend` 调用，因此单步的时候看到 `objc_msgSend` 就可以停下来了：

```bash
->  0x1000dd71c <+431900>: bl     0x100daa2bc               ; symbol stub for: objc_msgSend
    0x1000dd720 <+431904>: mov    x0, x20
    0x1000dd724 <+431908>: bl     0x100daa2ec               ; symbol stub for: objc_release
    0x1000dd728 <+431912>: mov    x0, x21
(lldb) po $x0
<DataServiceV2: 0x17400cea0>

(lldb) po (char *)$x1
"FM_GetSubscribeList:pageSize:callBack:"

(lldb) po $x4
<__NSStackBlock__: 0x16fd88f88>
```

可以看到，第四个参数是个 `StackBlock` 对象，但是 lldb 只为我们打印出了它的地址。接下来，就靠我们自己来找出它的函数体地址和函数签名了。

## 找出 Block 的函数体地址

要找出 Block 的函数体地址很简单，根据上面的内存模型，我们只到找到 `invoke` 这个函数指针的地址，它指向的就是这个 Block 的实现。

在 64 位系统上，指针类型的大小是 8 个字节，而 int 是 4 个字节，如下：

![Block struct members size](http://7xqonv.com1.z0.glb.clouddn.com/debuging-objective-c-blocks-in-lldb-pic-2.png)

因此，invoke 函数指针的地址就是在第 16 个字节之后。我们可以通过 lldb 的 memory 命令来打印出指定地址的内存，我们上面已经得到了 block 的地址，现在就打印出它的内存内容：

```bash
(lldb) memory read --size 8 --format x 0x16fd88f88
0x16fd88f88: 0x000000019b4d8088 0x00000000c2000000
0x16fd88f98: 0x00000001000dd770 0x0000000100fc6610
0x16fd88fa8: 0x000000017444c510 0x0000000000000001
0x16fd88fb8: 0x000000017444c510 0x0000000000000008
```

如前所述，函数指针的地址是在第 16 个字节之后，并占用 8 个字节，所以可以得到函数的地址是 0x00000001000dd770。

有了函数地址之后，就可以对这个地址进行反汇编：

```bash
(lldb) disassemble --start-address 0x00000001000dd770
LuoJiFM-IOS`_mh_execute_header:
->  0x1000dd770 <+431984>: stp    x28, x27, [sp, #-96]!
    0x1000dd774 <+431988>: stp    x26, x25, [sp, #16]
    0x1000dd778 <+431992>: stp    x24, x23, [sp, #32]
    0x1000dd77c <+431996>: stp    x22, x21, [sp, #48]
    0x1000dd780 <+432000>: stp    x20, x19, [sp, #64]
    0x1000dd784 <+432004>: stp    x29, x30, [sp, #80]
    0x1000dd788 <+432008>: add    x29, sp, #80              ; =80
    0x1000dd78c <+432012>: mov    x22, x3
```

也可以直接在 lldb 当中下断点：

```bash
(lldb) br s -a 0x00000001000dd770
Breakpoint 3: where = LuoJiFM-IOS`_mh_execute_header + 407616, address = 0x00000001000dd770
```

再次运行函数，就可以进到回调的 Block 函数体内了。

但是，大多数情况下，我们并不需要进到 Block 函数体内。在写 tweak 的时候，我们更需要的是知道这个 Block 回调给了我们哪些参数。

接下来，我们继续进行探索。

### 找出 Block 的函数签名

要找出 Block 的函数签名，需要通过 `descriptor` 结构体中的 `signature` 成员，然后通过它得到一个 `NSMethodSignature` 对象。

首先，需要找到 `descriptor` 结构体。这个结构体在 Block 中是通过指针持有的，它的位置正好在 `invoke` 成员后面，占用 8 个字节。可以从上面的内存打印中看到 `descriptor` 指针的地址是 **0x0000000100fc6610**。

接下来，就可以通过 `descriptor` 的地址找到 `signature` 了。但是，文档指出并不是每个 Block 都是有方法签名的，我们需要通过 flags 与 block 中定义的枚举掩码进行与判断。还是在刚刚的 llvm 文档中，我们可以看到掩码的定义如下：

```objc
enum {
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25),
    BLOCK_HAS_CTOR =          (1 << 26), // helpers have C++ code
    BLOCK_IS_GLOBAL =         (1 << 28),
    BLOCK_HAS_STRET =         (1 << 29), // IFF BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE =     (1 << 30),
};
```

再次使用 memory 命令打印出 `flags` 的值：

```bash
(lldb) memory read --size 4 --format x 0x16fd8a958
0x16fd8a958: 0x9b4d8088 0x00000001 0xc2000000 0x00000000
0x16fd8a968: 0x000dd770 0x00000001 0x00fc6610 0x00000001
```

由于 `((0xc2000000 & (1 << 30)) != 0)`，因此我们可以确定这个 Block 是有签名的。

> 虽然在文档中指出并不是每个 Block 都有函数签名的。但是我们可以在 [Clang 源码]() 中的 [CGBlocks.cpp]() 查看 `CodeGenFunction::EmitBlockLiteral` 与 `buildGlobalBlock` 方法，可以看到每个 Block 的 flags 成员都是被默认设置了 `BLOCK_HAS_SIGNATURE`。因此，我们可以推断，所有使用 Clang 编译的代码中的 Block 都是有签名的。

为了找出 `signature` 的地址，我们还需要确认这个 Block 是否拥有 `copy_helper` 和 `disponse_helper` 这两个可选的函数指针。由于 `((0xc2000000 & (1 << 25)) != 0)`，因此我们可以确认这个 Block 拥有刚刚提到的两个函数指针。

现在可以总结下：`signature` 的地址是在 `descriptor` 下偏移两个 unsiged long 和两个指针后的地址，即 32 个字节后。现在让我们找出它的地址，并打印出它的字符串内容：

```bash
(lldb) memory read --size 8 --format x 0x0000000100fc6610
0x100fc6610: 0x0000000000000000 0x0000000000000029
0x100fc6620: 0x00000001000ddb64 0x00000001000ddb70
0x100fc6630: 0x0000000100dfec18 0x0000000000000001
0x100fc6640: 0x0000000000000000 0x0000000000000048

(lldb) p (char *)0x0000000100dfec18
(char *) $4 = 0x0000000100dfec18 "v28@?0q8@"NSDictionary"16B24"
```

看到这一串乱码是不是觉得有点崩溃，折腾了半天，怎么打印出这么一串鬼东西，虽然里面有一个熟悉的 NSDictionary，但是其它的东西完全看不懂啊。

不要慌，这确实就是一个函数签名，只是我们需要通过 `NSMethodSignature` 找出它的参数类型：

```bash
(lldb) po [NSMethodSignature signatureWithObjCTypes:"v28@?0q8@\"NSDictionary\"16B24"]
<NSMethodSignature: 0x174672940>
    number of arguments = 4
    frame size = 224
    is special struct return? NO
    return value: -------- -------- -------- --------
        type encoding (v) 'v'
        flags {}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 0, size adjust = 0}
        memory {offset = 0, size = 0}
    argument 0: -------- -------- -------- --------
        type encoding (@) '@?'
        flags {isObject, isBlock}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 1: -------- -------- -------- --------
        type encoding (q) 'q'
        flags {isSigned}
        modifiers {}
        frame {offset = 8, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 2: -------- -------- -------- --------
        type encoding (@) '@"NSDictionary"'
        flags {isObject}
        modifiers {}
        frame {offset = 16, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
            class 'NSDictionary'
    argument 3: -------- -------- -------- --------
        type encoding (B) 'B'
        flags {}
        modifiers {}
        frame {offset = 24, offset adjust = 0, size = 8, size adjust = -7}
        memory {offset = 0, size = 1}
```

> 注意，字符串中的双引号需要对其进行转义。

对我们最有用的 type encoding 字段，这些符号对应的解释可以参考 [Type Encoding 官方文档](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)。

所以，总结来讲就是：这个方法没有返回值，它接受四个参数，第一个是 block （即我们自己的 block 的引用），第二个是 (long long) 类型的，第三个是一个 NSDictionary 对象，第四个是一个 BOOL 值。

最终，我们得到了这个 Block 的函数参数。最初提到的那个方法签名的完整版就是：

```objc
- (void)FM_GetSubscribeList:(long long)arg1 pageSize:(long long)arg2 callBack:(void (^)(long long, NSDictionary *, BOOL)arg3;
```

## 小结

因为想使用真实的例子进行演示，所以本文直接使用逆向的动态分析进行说明。其实上面提到的所有过程，都可以直接在 Xcode 通过自己写的代码进行操作。通过自己动手分析一遍，比看十篇文章来得更有效果。下次如果面试再有人问到 Block 的实现和内存模型，你就可以跟它侃侃而谈了。

## 参考文章

[Tutorial: An In-Depth Guide To Objective-C Block Debugging](https://maniacdev.com/2013/11/tutorial-an-in-depth-guide-to-objective-c-block-debugging)
