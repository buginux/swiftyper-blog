---
title: 如何优雅地使用 Playground
tags: [Playground, Xcode]
categories: [iOS]
date: 2016-02-02 13:09:27
---

苹果为推动 Swift 的发展花了很大的力气，Playground 就是其中的一个。Playground 大大降低了 Swift 的学习门槛，它可以实时执行 swift 代码，并将结果显示出来，并且还有各种交互来加深你对 swift 语法的理解。

但是你要是以为这就是 Playground 的全部功能，你就错了。其实 Playground 还有很多其它的高级用法，特别是在 Xcode 7 之后，苹果再一次对 Playground 进行了改进，让它的功能更加强大了。

<!-- more -->

## Sources 目录

通常情况下，我们直接在 Playground 上面写代码，然后编译器会实时编译我们代码，并将结果显示出来。这很好，我们可以实时得到代码的反馈。

但是这也会产生一个问题，如果我们写了一个函数，或者自定义了一个 view，这部分代码一般情况下是不会变的，而编译器却会一次又一次地去编译这些代码，最终的结果就是导致效率的低下。

这时，Sources 目录就派上用场了，使用 `Cmd + 1` 打开项目导航栏(Project Navigator)，可以看到一个 _Sources_ 目录。放到此目录下的源文件会被编译成模块(module)并自动导入到 Playground 中，并且这个编译只会进行一次(或者我们对该目录下的文件进行修改的时候)，而非每次你敲入一个字母的时候就编译一次。  这将会大大提高代码执行的效率。

> 注意：由于此目录下的文件都是被编译成模块导入的，只有被设置成 public 的类型，属性或方法才能在 Playground 中使用。

## 资源

由于 Playground 并没有使用沙盒机制，所以我们无法直接使用沙盒来存储资源文件。

但是，这并不意味着在 Playground 中没办法使用资源文件，Playground 提供了_两个_地方来存储资源，一个是每个 Playground 都独立的资源，而另一个是所有 Playground 都共享的资源。

### 独立资源

在打开的项目导航栏中可以看到有一个 _Resources_ 目录，放置到此目录下的资源是每个 Playground 独立的。

这个目录的资源是直接放到 `mainBundle` 中的，可以使用如下代码来获取资源路径：

```
if let path = NSBundle.mainBundle().pathForResource("example", ofType: "json") {
    // do something with json
    // ...
}
```

如果是图片文件，也可以直接使用`UIImage(named:)`来获取。

### 共享资源

共享资源的目录是放在用户目录的 _Documents_ 目录下的。在代码中可以直接使用
`XCPlaygroundSharedDataDirectoryURL`
来获取共享资源目录的 URL(需要先导入 `XCPlayground` 模块)。

```
import XCPlaygroud

let sharedFileURL = XCPlaygroundSharedDataDirectoryURL.URLByAppendingPathComponent("example.json")
```

> 注意：你需要创建`~/Documents/Shared Playground Data`目录，并将资源放到此目录下，才能在 Playground 中获取到

## 异步执行

Playground 中的代码是顶层代码(top-level code)，也就是它是在于全局作用域中的。这些代码将会从上到下执行，并在执行完毕之后立即停止。

我们的异步回调代码一般都无法在程序结束之前获得执行，因此如果我们在 Playground 执行网络，或者其它耗时的异步操作，都无法获得我们想要的结果(之前我在 Playgroud 中学习网络操作的时候，就被这个特性折磨到发狂，我一直以为是自己的网络代码写错了)。

为了让程序在代码执行结束后继续执行，我们可以使用如下代码：

```
XCPlaygroundPage.currentPage.needsIndefiniteExecution = true
```

这句代码会让 Playground 永远执行下去 ，当我们获取了需要的结果后，可以使用`XCPlaygroundPage.currentPage.finishExecution()`停止 Playground  的执行：

```
import XCPlayground

XCPlaygroundPage.currentPage.needsIndefiniteExecution = true

let url = NSURL(string: "http://httpbin.org/image/png")!
let task = NSURLSession.sharedSession().dataTaskWithURL(url) {
    data, _, _ in
    let image = UIImage(data: data!)

    XCPlaygroundPage.currentPage.finishExecution()
}
task.resume()
```

## 支持 Markdown

Playground 已经原生支持 markdown 注释渲染，只需要在单行或多行注释的后面添加冒号`:`，标记这是一个 markdown 注释。

```
//: This line will have **bold** and *italic* text.

/*:
## Headers of All Sizes

### Lists of Links

- [NSHipster](http://nshipster.com)
- [ASCIIwwdc](http://asciiwwdc.com)
- [SwiftDoc](http://swiftdoc.org)
*/
```

可以到菜单 `Editor → Show Rendered Markup` 下切换是否进行 markdown 渲染。

![markdown渲染](http://7xqonv.com1.z0.glb.clouddn.com/playground_advance_usage_0.png)

## 多页面

Playground 支持多页面，这可以让我们将不同类别的代码分别写到不同的页面下，并可以在多个页面之间进行跳转。

![playground多页面](http://7xqonv.com1.z0.glb.clouddn.com/playground_advance_usage_1.png)

在项目导航栏中，选择`File &gt; New &gt; Playground Page`， 就可以新建一个页面，或者选择项目导航栏左下角的 + 号选择 `New Page`。

> 注意：如果本 Playground 是第一次新建 Page，则系统会产生两个 Page，一个代表你当前的页面，一个是新建的页面。

### 页面跳转

Playground 支持三种方式的页面跳转：

1.  上一页
2.  下一页
3.  跳转到指定页

页面顺序都是根据它们在项目文件中的排序来决定的。

上一页与下一页的语法：

```
//: ["Go to Next Page"](@next)

//: ["Go to Previous Page"](@previous)
```

与 markdown 中的超链接类似，方括号中的代码要显示的文字，而小括号则代表跳转的目的地。

指定页跳转：

```
//: ["Go to The End"](PageName)
```

PageName 代表目标页面的名称，如果页面名称中有空格，则需要使用%20来代替，这是 ASCII 中空格的符号。如下：

```
//: ["Go to The End"](Last%20Page)
```

## 总结

Playground 中还有很多其它的高级功能，这里只是列出了我自己比较经常用到的功能，如果你比较感兴趣，可以[参考官方文档](https://developer.apple.com/library/ios/recipes/Playground_Help/Chapters/AboutPlaygrounds.html)，或者[这本书](https://leanpub.com/playgroundsecretsandpowertips)。