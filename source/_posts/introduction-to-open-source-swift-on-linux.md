---
title: (译)介绍Linux下的开源Swift
tags: [Swift, Linux]
categories: [翻译]
date: 2015-12-10 21:14:15
---

就在不到一周前，Swift 世界贡献了一份提前到达的圣诞节礼物 —— [开源Swift](https://developer.apple.com/swift/blog/?id=34) —— 可以在 Linux 上运行的 Swift。

这份大礼里面甚至又_包含_了另一份礼物：苹果宣布了新的项目以及工具使得 Swift 成为 Linux 开发的一个相当具有实践性的选择。这也产生了很多令人兴奋的可能性，比如说从强大的 Linux 服务器到 5 美元的一台[树莓派Zeros](https://www.raspberrypi.org/blog/raspberry-pi-zero/)，或者其它大量的托管了 Linux 的环境都可以运行 Swift 了。

在这个教程中，我们将会在 Mac 中创建一个 Linux 环境，安装 Swift，并且在这个 Linux 当中编译和运行一些基本的 Swift 示例程序。之后可以感觉一下苹果提供的新工具，然后我们会透过时间的水晶球来预测 Swift 开发未来的走向。

<!-- more -->

此教程不要求读者预先了解任何关于 Linux 的知识，甚至不需要读者安装过 Linux —— 但是如果你已经有一个可以运行的 Ubuntu 系统，那就再好不过了！你所需要的就是运行了最新的 EL Capitan 系统的 Mac 电脑，以及对开源 Swift 兴趣就够了。那就让我们开始吧！

(译者注：我也会补充关于如何在 Windows上安装的资料。所以不用担心！Don't Panic.)

![](http://cdn1.raywenderlich.com/wp-content/uploads/2015/12/IntroToLinuxSwift_Feature.png)

## 在 Linux 上安装 Swift

如果你已经有一个可以运行的 Unbutn 14.04 LTS 或 Ubuntu 15.10，不管是装在实体机上，还是装在 Mac 或 Windows 下的虚拟机上，你都可以简单地按照[swift.org](www.swift.org)上面的教程（或者参考我的[上一篇博客](http://www.swiftyper.com/swift_toolchain_on_ubuntu/)）下载并在 Linux 上安装 Swift，然后直接跳到下一小节。

如果你没有上面所列版本的 Ubuntu 系统，那就按照我下面的教程，在你的 Mac 上安装 VirtualBox 以及 Ubuntu。这会使用小于 1GB 的硬盘空间，并在虚拟机上面安装 Linux —— 不会影响到你当前的 OS X 系统。

### 安装 VirtualBox

**VirtualBox** 是一个免费并且开源的程序，它可以作为虚拟机让我们在运行当前主系统的同时运行其它的操作系统。

前往[VirtualBox下载页](https://www.virtualbox.org/wiki/Downloads)，然后下载 **VirtualBox 5.0.10**或更新版本的镜像。双击下载后的 DMG 文件，接着运行 **VirtualBox.pkg** 安装包：
![](http://cdn5.raywenderlich.com/wp-content/uploads/2015/12/swift-on-linux.VirtualBoxdDiskImage.png)

(刚刚那个下载页也有 Windows 版本的 VirtualBox 安装文件，直接下载，然后双击安装文件，按照传统的下一步下一步安装步骤就可以安装了。—— 译者注)

### 安装及使用 Vagrant

**Vagrant**是一个 VirtualBox 的命令行界面；它可以下载虚拟机镜像，运行配置脚本，并且在主操作系统与虚拟机系统之间设置网络和共享文件夹以使其可以进行交互。

前往[Vagrant下载页](https://www.vagrantup.com/downloads.html)，然后下载 Mac OS X 版本的 Vagrant。双击下载的 DMG 硬盘镜像，接着运行 **Vagrant.pkg** 安装包：
![](http://cdn1.raywenderlich.com/wp-content/uploads/2015/12/swift-on-linux.VagrantDiskImage.png)

现在需要一个 **Vagrantfile** 来告诉 Vagrant 我们具体需要安装哪个版本的 Linux，以及如何在那个版本的 Linux 上安装 Swift。

在你的文档目录下（或者其它你觉得舒服的目录下）创建一个空目录，并且将它全名为 **vagrant-swift**。在那个目录下，创建一个 **Vagrantfile** 然后输入如下内容：

```
Vagrant.configure(2) do |config|
      ## 1
      config.vm.box = "https://cloud-images.ubuntu.com/vagrant/trusty/20151201/trusty-server-cloudimg-amd64-vagrant-disk1.box"

    config.vm.provision "shell", inline: &lt;&lt;-SHELL
        ## 2
        sudo apt-get --assume-yes install clang
        ## 3
        curl -O https://swift.org/builds/ubuntu1404/swift-2.2-SNAPSHOT-2015-12-01-b/swift-2.2-SNAPSHOT-2015-12-01-b-ubuntu14.04.tar.gz
        ## 4
        tar zxf swift-2.2-SNAPSHOT-2015-12-01-b-ubuntu14.04.tar.gz
        ## 5
        echo "export PATH=/home/vagrant/swift-2.2-SNAPSHOT-2015-12-01-b-ubuntu14.04/usr/bin:\"${PATH}\"" &gt;&gt; .profile
        echo "Swift has successfully installed on Linux"
      SHELL
    end
```

现在运行 Vagrantfile。在你的 Mac 上打开终端（可以在 /Applications/Utilties/目录下找到），切换到包含上面的那个文件的 **vagrant-swfit** 目录下，然后输入如下的命令运行这个脚本：

```
vagrant up
```

现在 Vagrant 会根据 Vagrantfile 执行如下步骤，你可以去泡杯咖啡慢慢等：

1. 下载 Ubuntu LTS 14.04 的硬盘镜像。镜像只会被下载一次，然后保存起来，以供后续的使用。你大概会看到类似 "Adding it directly..." 后接着 "Attempting to find and install..." 的信息。
2. 在虚拟机中运行下载完的硬盘镜像。
3. 安装 **clang** 即苹果公司的 C 编译器，这是 Swift 编译器所需要的组件。
4. 下载 Swift 然后对其进行解压。
5. 配置用户家目录的 **PATH** 路径变量，以使其可以使用 swift 二进制工具，REPL以及其它工具。

如果这些步骤都顺利完成了，最后一条信息应该会显示 "Swift has successfully installed on Linux"。

下一步，需要连接上你的 Linux 虚拟机。在同一个 **vagrant-swift** 目录下，运行如下的命令：

```
vagrant ssh
```

现在你可以看到自己在一个 Ubuntu 的提示符下，用户名为 "vagrant"。为了验证你现在可以运行 Swift 并且它的功能是否正确，运行如下的命令：

```
swift --version
```

你可以看到如下的信息：

```
vagrant@vagrant-ubuntu-trusty-64:~$ swift --version
Swift version 2.2-dev (LLVM 46be9ff861, Clang 4deb154edc, Swift 778f82939c)
```

啦啦啦！运行在 Linux 上面的 Swift。这真是一个圣诞对奇迹呀！:]

![We](http://cdn5.raywenderlich.com/wp-content/uploads/2015/12/800px-Tiny-tim-dickens-352x500.jpg)

## 编译一个程序

我们已经在  Linux 上面安装了 Swift. 现在我们可以来编译和运行一些东西了。

当然，神圣的代码传统意味着我们的第一个程序将只有一个功能，那就是输出一句"Hello, world"。依照程序员的性格，我们将以最懒最简单的方式来完成这个功能。:]

切换到 Linux 的命令行环境下并且执行如下的命令：

```
cat > helloworld.swift
```

现在输入下面一行的 Swift 代码：

```
print("Hello, world")
```

按下 **Enter** 键，接着使用 **Ctrl-D** 来创建这个文件。

执行一下 `ls` 命令来列出当前目录的文件，确保我们创建了一个名为 **helloworld.swift** 的文件。

现在在命令行提示符下运行如下的命令来调用 swift 编译器对程序进行编译：

```
swiftc helloworld.swift
```

> 注意：如果你在编译时遇到了错误，你可能需要更新 clang：
> 
> ```
> sudo apt-get install clang-3.6
> sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-3.6 100
> sudo update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-3.6 100
> ```

在命令行下执行 `ls -l h*`命令来查看当前目录下以 **h** 开头的文件：

```
-rwxrwxr-x 1 vagrant vagrant 13684 Dec  4 17:55 helloworld
-rw-rw-r-- 1 vagrant vagrant    22 Dec  4 17:55 helloworld.swift
```

在输出结果的第一列中我们可以看到 **helloworld** 有一个 x 字符，说明它是一个可执行文件。执行如下的命令来运行 **helloworld**：

```
./helloworld
```

你将会看到如下的输出：

```
Hello, world
```

你是电！你是光！你是唯一的神话！我们在 Linux 上运行了第一个 Swift 程序。So easy! :]

但是这_意味_着什么呢？我们究竟可以在 Linux 上用 Swift 来做什么呢？现在就让我们在跨平台开发的高速路上稍微踩下刹车，抽一点时间来重温下苹果究竟开源了哪些东西。

## 苹果开源了哪些东西

到目前为止，苹果开源了五个类目的软件：
**1) Swift 编译器(源代码以及可执行二进制文件)，调试器，以及交互性命令行(如，REPL)**

这是 Swift 语言的核心。苹果开源了 Swift 编译器的源代码，即与 Xcode 中用于编译 iOS 以及 Mac 应用一致的 **swiftc** 程序。

**2) Swift 标准库**

这与 Swift 发布后一直使用的标准库是(几乎)完全一致的。它定义了字符串，集合类型，协议，以及其它基础结构的实现。你是否一直渴望看到标准库对于基础类型的实现，像 **Double** 以及 **Array**？[现在你完全可以看到了](https://github.com/apple/swift/blob/master/stdlib/public/core/Arrays.swift.gyb)。

然而，只有单独的 Swift 标准库并起不了多大的作用。有多少个你写过的 Swift 应用_没有导入 UIKit_？大概没有几个吧。至少，Foundation 的功能，对于开发一些_有用的软件_(而非"Hello, world"，以及千千万的计算阶乘的函数)是十分有必要的。这一切代表着苹果也会开源 ...

**3) Swift 核心库**
这些是针对苹果库的全新，跨平台以及独立于 Objective-C 的重新实现。它们提供了比较高阶的功能比如网络，文件系统交互，日期和时间计算。一个警告：它们_尚未_完全开发完成。但不止于此，苹果还开源了...

**4) Swift 包管理器**
 Swift 包管理器与[Carthage](https://github.com/Carthage/Carthage)有点类似；它可以下载依赖，编译，然后对其进行链接以生成库和可执行文件。

**5) 其它组件**
如果你浏览过[苹果的Github主页](http://github.com/apple)，你还会发现其它的组件：一个基于 Swift 的 [libdispatch包装器](https://github.com/apple/swift-corelibs-libdispatch)(即，Grand Central Dispatch)，[Swift 3.0未来计划](https://github.com/apple/swift-evolution)的说明，甚至还有一个 Markdown 方言 **CommonMark** 的[解析和渲染器](https://github.com/apple/swift-cmark)。

## 苹果所没有开源的东西

苹果开源的东西真是相当多啊 —— 那么，他们没有开源哪些东西呢？

让我们来数数吧，Xcode 没有开源，AppKit 没有开源，UIKit 没有开源。还有 Core Graphics, Core Animation, AVFoundation以及其它我们所熟悉的 Objective-C 核心类库都没有开源。基本上，所有你用来创建漂亮的 iOS 或者 Mac 应用所需要的库都没有开源。

但是，苹果_已经_开源的东西，具有十分重大的意义。考虑一下核心类库这个项目。为什么苹果会费那么多力气重新实现这个他们已经千锤百炼，并且经过重重测试的 Foundation 库呢？

事实就是这个核心类库不需要依赖于 Objective-C 运行时，这意味着苹果在创造一个可以在未来用 Swift _完全_替代 Objective-C 的基础。并且它的跨平台特性意味着苹果十分希望用户使用 Swift 进行 Linux 的软件开发 —— 至少是服务端软件，而非 GUI 应用开发。

但是包管理器是最重要的一个开源组件。在深入解释原因之前，让我们先快速过一下，来了解包管理器的使用，和认识一下它究竟是什么东西。

## 使用 Swift 包管理器

包管理器为任何需要构建成一个可执行文件或者库的 Swift 代码定义了一个简洁，直观的目录结构。我们将在安装过 Ubuntu 和 Vagrant 的 VirtualBox 中一探究竟。

首先，在命令行提示符执行如下的命令来创建一个 **helloworld-project** 目录，并且切换到该目录：

```
mkdir helloworld-project && cd helloworld-project
```

现在执行如下命令用来创建一个空的 `Package.swift` 文件：

```
touch Package.swift
```

这个文件的存在，表明当前目录定义了一个 Swift 的包。

接着，用下面的命令来为源代码创建一个子目录：

```
mkdir Sources
```

下一步，执行如下命令来把之前的旧程序，`hello.swift`，复制到 `Sources/` 目录下，并且重新命名为 `main.swift`：

```
cp ../helloworld.swift Sources/main.swift
```

`helloworld-project` 目录当前的结构定义一个最基础的包，并且它的结构应该是这样的：

```
├── Package.swift
└── Sources/
    └── main.swift
```

执行如下命令来构建这个包：

```
swift build
```

可以看到如下的输出：

```
vagrant@vagrant-ubuntu-trusty-64:~/helloworld-project$ swift build
Compiling Swift Module 'helloworldproject' (1 sources)
Linking Executable:  .build/debug/helloworld-project
```

这里为构建完成的产品创建了一个新的，并且是隐藏的目录—— `.build`。现在目录结构看起来应该是这样的：

```
.
├── .build/
│   └── debug/
│       ├── helloworld-project*
│       ├── helloworld-project.o/
│       │   ├── build.db
│       │   ├── helloworld-project/
│       │   │   ├── main.d
│       │   │   ├── main~partial.swiftdoc
│       │   │   ├── main~partial.swiftmodule
│       │   │   ├── main.swiftdeps
│       │   │   ├── master.swiftdeps
│       │   │   └── output-file-map.json
│       │   ├── home/
│       │   │   └── vagrant/
│       │   │       └── helloworld-project/
│       │   │           └── .build/
│       │   │               └── debug/
│       │   ├── llbuild.yaml
│       │   └── Sources/
│       │       └── main.swift.o
│       ├── helloworldproject.swiftdoc
│       └── helloworldproject.swiftmodule
├── Package.swift
└── Sources/
    └── main.swift
```

可以使用`ls -aR`命令来验证这个隐藏目录的存在。

最重要的文件就是 **.build/debug/helloworld-project**了，它被标记了星号，代表着它是一个可执行文件 。这就是我们所需要的可执行文件，可以在命令行输入它的名字来运行，就像这样：

```
vagrant@vagrant-ubuntu-trusty-64:~/helloworld-project$ .build/debug/helloworld-project
Hello, world
```

相当酷，有没有！但是目录里的那些杂七杂八的东西都有什么作用呢？

所有这些构建中间产物表明了包管理器真正的能力范围。记住——它不只是一个编译器。作为一个构建工具(类似`make`)，包管理器的主要责任是编译单个包所需要的多个源文件，并将它们链接到一起。

作为一个包管理器(类似 Carthage 或 CocoaPods)，它也有将多个包链接到一起，并且到互联网上拉取依赖包的责任。这意味着包需要了解一些_版本与兼容性_(比如，“哪一个依赖可以满足我的需求？”)，以及_位置_(比如，“我应该到哪里去获得这个包呢？”)的概念。

    作为一个比较复杂的例子，我们来看一下苹果提供的示例程序；它模拟了一个扑克牌游戏。

    安装 **Git**，切换到一个新的目录，使用如下的命令来获取一个最顶层的包：

```
cd ~
sudo apt-get --assume-yes install git
git clone https://github.com/apple/example-package-dealer.git
cd example-package-dealer
```

查看一下这个目录，你可以发现里面几乎没有任何东西：

```
.
├── CONTRIBUTING.md
├── main.swift
├── Package.swift
└── README.md
```

但是，这次的 **Package.swift** 不是一个空文件了；它包含了一小段的 Swift 表明了这个包依赖了另一个版本 1 的包，`example-package-deckofplayingcards`，而这个包是存放在[github链接上](https://github.com/apple/example-package-deckofplayingcards.git)的，同时_这个_包又依赖了其它的包，如此类推。

最终的结果就是，如果你对这个示例程序运行了 `swift build` 命令，它会下载超过 400 个的文件，因为构建工具会去下载其它依赖的仓库源，对其进行编译，然后链接。

苹果在 [Swift.org](www.swift.org) 里[对包管理器进行了更加详细的描述](https://swift.org/package-manager/)。下面的列表列出了这个新的包管理器值得注意的一些点：
* 为包的布局使用一个简洁的目录结构，以及一个简单的基于 Swift 的 manifest 文件，相对地， Xcode可以使用任何的目录结构，但是需要一个 Xcode 特有的项目文件用于追踪由自由而造成的混乱的结构
* 使用[语义化版本](http://semver.org/)，一个用于描述包版本以及兼容性的事实标准
* 与 Swift 的_模块_概念紧密集成，而这正是苹果在 Objective-C 上使用和在 C 语言中大力推广的标准
* 与 xcodebuild 构建系统的功能存在一些重叠
* 执行了依赖传递分析，但是还不能对依赖冲突进行处理
* 构建可执行文件和静态库，但是还不支持动态库
* 使用 git，但是还未对其它版本控制系统提供支持
* 包含了一份特别详细周到的[社区文件方案](https://github.com/apple/swift-package-manager/blob/master/Documentation/PackageManagerCommunityProposal.md)，详细描述了这个设计背后的思想。
如果你使用过 CocoaPods, Carthage 或者其它的包管理器，这些对你来说看起来应该非常熟悉。确实，在技术上来讲，这里并没有什么新的东西。但这_对苹果来说是全新的_，并且是十分显著的。
那就让我们穿过时间的迷雾，来探索下 ...

## 这一切意味着什么？

为什么包管理器如此重要？原因在于包管理器不仅仅是一个工具。包管理器与围绕着_技术社区_的软件生态有着紧密的联系，它们定义了人们在创作软件时对于基本事务上的相互帮助是如何交互的。

对于 Ruby 和 Node 社区，如果缺少了 `gem` 和 `npm` 管理器，它们能达到现在这样状态么？并不。而且毫不夸张地说，[CocoaPods](https://cocoapods.org/) 让 iOS 开发者的联系更加紧密了，因为它让共享工作成果与互相帮助变得更加简单。

苹果在与开发者交流方面一直名声不太好；他们没有为开发者的反馈提供频道支持，也很少支持开发者之间进行沟通。一个很明显的例子就是 Radar 漏洞追踪系统，它就像一个星际黑洞，静静地吞噬掉一切信息，却不为外界留下一丝一毫的追踪痕迹。

但是包管理器必将成为_社区_的一个基础设施；它是一个用于开发者与开发者，而不是与苹果，之间进行交流的东西。苹果现在领导发布了这样一个工具，说明它已经开始慢慢在这方面起步了。

这个工具感觉上就像是一个普通的包管理器，它使用了语义化版本，Git，以及其它普通的工具和模式，并没有使用苹果家一贯的封闭原则，这也是一个很好的迹象。苹果甚至为 Swift 提供了一个基于 JIRA 的漏洞追踪系统来替代 Radar，这使得开发者可以实实在在地_看到_自己或其它人提交过的漏洞。

我们能真真切切地感受到 Swift 开发者的诚意！:]

![](http://cdn3.raywenderlich.com/wp-content/uploads/2015/12/WelcomeMat.png)

从包管理器我们可以感觉到苹果隐藏在开源背后更为广阔的想法。对于开源，苹果不仅仅是将源代码公开了事。它为支持 Linux 上的 Swift 生产开发踏出了第一步。不仅如此，他们还在鼓励和_欢迎_其它开发者对此作出贡献上踏出了更大的一步。

### 停止虚拟机

当你使用完了虚拟机后，使用 `logout` 命令退出命令行，然后执行`vagrant suspend`来使它挂起，以备以后的使用，或者你也可以直接使用`vagrant destroy`来让它完全销毁然后释放硬盘空间。

## 接下来呢？

在这个教程当中，你安装了 Linux, 在 Linux 上安装了 Swift，在 Linux 编译了 Swift 代码，并且快速地了解了下苹果的包管理工具。现在完全可以在 Linux 上运行 Swift 的事实已经十分有趣了，但是事实上，苹果也十分_鼓励_我们这样做，它提供了技术以及社区来对其进行支持，这实在是帅呆了！:]

你现在只是快速粗略地了解了一下 Linux 上面的 Swift，但这些新工具大部分也对 OS X 上的 Swift 造成了影响，比如包管理工具；现在我们可以更容易地共享代码以及构建命令行工具。很难想像这不是对 Xcode 未来方向的一个_暗示_。新的代码仓库，新路线图的曝光以及对 Swift 未来开发的讨论，都预示着这是一个十分令人看好——也是十分令人惊喜——的开发之旅。

有趣的日子即将来临！

> 这篇译文可以作为我上篇博客的后续，我本来也打算开始写了，但是突然发现 Ray 上面有了这篇博客，与我打算写的如出一辙（当然肯定是人家的质量要高），于是我第一时间就把这篇文章翻译过来，也算是补了上一篇博客的坑吧。

本文翻译自：[raywenderlich](www.raywenderlich.com)
原文地址：[Introduction to Open Source Swift on Linux](http://www.raywenderlich.com/122189/introduction-to-open-source-swift-on-linux)
