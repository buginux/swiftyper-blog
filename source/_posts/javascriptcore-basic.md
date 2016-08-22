---
title: JavaScriptCore 基本使用
date: 2016-08-22 14:32:02
categories: iOS
tags: [Hybrid, JavscriptCore]
---

现在的移动开发已经越来越倾向于使用混合开发，而要使用混合开发就要求我们必须能让原生与 JavaScript 进行无缝的交互。

在 iOS 7 之前，我们对 JavaScript 的操作只有 UIWebView 里的一个方法 `stringByEvaluatingJavaScriptFromString`，而从 Javascript 里面调用原生方法只能使用拦截 URL 的方法，类似 [WebViewJavascripBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 就是基于 URL 拦截的原理来进行实现的。

而从 iOS 7 开始，苹果把 JavascriptCore 引进到了 iOS 开发当中，这个框架可以让我们脱离 UIWebView 同时用更方便同时更强大的方法来与 Javascript 进行交互。

## JavaScriptCore 总览

在学习 JavaScriptCore 的使用之前，需要先了解 JavaScriptCore 当中的重要类型以及协议，包括 `JSValue`、`JSContext`、`JSVirtualMachine`、`JSManagedValue` 以及 `JSExport`。

### JSVirtualMachine

Javascript 代码是在虚拟机当中运行的，每一个虚拟机由一个 `JSVirtualMachine` 来表示。一般情况下我们不用去手动创建 JSVirtualMachine 实例，使用系统提供的就足够了。

需要手动创建 `JSVirtualMachine` 的一个主要场景就是当我们需要并发地运行 Javascript 代码时，在单一的 `JSVirtualMachine` 里面是没办法*同时*运行多个线程的。

### JSContext

代表 JavaScript 的运行环境，是一个全局对象，可以理解为 Web 开发中的 window 对象。所有的 `JSValue` 都与 `JSContext` 相关联。

### JSValue

用来表示 JavaScript 实体的本地对象。由于 JavaScript 是弱类型的语言，所以我们使用 `JSValue` 就可以代表所有 Javascript 中的类型。每个 `JSValue` 实例都与它对应的 `JSContext` 相关联。所以从 context 对象中产生的值都是 `JSValue` 类型的。

### JSManagedValue

Objective-C 或者 Swift 的对象都是使用引用计数，而 Javascript 则是使用垃圾回收机制。为了避免两种语言交互时产生的循环引用，需要使用 `JSManagedValue` 进行内存辅助管理。

### JSExport

这是一个协议，使用这个协议可以将本地的对象导出对 JavaScript 对象，所有的本地属性与方法都会直接变成 JavaScript 的属性与方法。可以，这很魔法。

## Objective-C 与 Javascript 之间的交互

从 Objective-C 调用 Javascript 有一种方法，而使用 JavaScript 调用原生则有两种方法，下面分别介绍。

### 原生调用 Javascript

```objective-c
+ (BOOL)isValidNumber:(NSString *)phone
{
    // 1. getting a JSContext
    JSContext *context = [[JSContext alloc] init];

    // 2. defining a JavaScript function
    NSString *jsFunctionText =
    @"var isValidNumber = function(phone) {"
    "    var phonePattern = /^[0-9]{3}[ ][0-9]{3}[-][0-9]{4}$/;"
    "    return phone.match(phonePattern) ? true : false;"
    "}";
    [context evaluateScript:jsFunctionText];

    // 3. calling a JavaScript function
    JSValue *jsFunction = context[@"isValidNumber"];
    JSValue *value = [jsFunction callWithArguments:@[ phone ]];

    return [value toBool];
}
```

使用 Objective-C 调用 JavaScript 的很简单：
1. 创建一个 JSContext 对象
2. 将 JavaScript 代码加载到 context 当中
3. 取到 JavaScript 函数对象，并使用 `callWithArguments:` 进行传参调用 

### JavaScript 调用原生

使用 JavaScript 调用原生有两种方法，使用闭包或者 `JSExport` 协议。

#### 使用闭包

```objective-c

// 1. 
JSContext *context = [[JSContext alloc] init];

// 2.
void (^block)() = ^(NSString *string) {
	NSLog(@"%@", string);
};

// 3.
[context setObject:block forKeyedSubscript:@"print"];

// 4.
[context evaluateScript:@"print('Hello World');"];

```

使用闭包来调用原生方法也很简单：
1. 跟之前一样，首先需要先创建一个 JSContext
2. 定义一个闭包
3. 将闭包保存到 context 当中，也就是转换成了 JavaScript 的方法。这里使用了 `setObject:forKeydSubscript` 方法，其实也可以直接使用下标操作，`context[@"print"] = block;`。
4. 在 JavaScript 当中调用，会直接调用到原生里面的方法。


#### 使用 JSExport

首先，定义一个继承于 JSExport 的协议：

```objective-c

@protocol ViewControllerJSExport <JSExport>

- (void)print:(NSString *)message;

@end

```

然后，让需要在 JavaScript 中调用的那个类遵守这个协议，在这里简单地使用 ViewController：

```objective-c

@interface ViewController () <ViewControllerJSExport>

@end

@implementation ViewController

- (void)viewDidLoad {
  [super viewDidLoad];

  JSContext *context = [[JSContext alloc] init];

  // 1. 
  context[@"myApp"] = self;

  // 2. 
  [context evaluateScript:@"myApp.print('Hello World');"];
}

- (void)print:(NSString *)message {
  NSLog(@"%@", message);
}

@end
```

在这里需要注意的是，我们将类的句柄传给了 context（在这里是直接将 viewController 传过去），然后在 JavaScript 中就需要使用这个句柄来进行调用。

### 捕获异常

JavaScriptCore 允许我们使用一个 Objective-C 闭包来对 JavaScript 中的异常进行捕获：

```objective-c
[context setExceptionHandler:^(JSContext *context, JSValue *value) {
	NSLog(@"%@", value);
}];
```

只要 JavaScript 中产生了异常，异常信息就会被传递到这个闭包当中，我们可以对其进行处理。建议始终实现这个闭包，对于 JavaScript 中出现的异常很难被发现，就算只是单纯地将错误信息打印出来，也可以节省我们大量调试错误的时间的精力。

## 小结

这里只是介绍了最基本的 JavaScriptCore 的使用方法，只能说开了个头，实际使用 JavaScriptCore 其实也有很多坑，包括内存管理，JavaScript 代码的调试。

同时，我们也不会经常直接加载本地的 JavaScript 代码，大多数情况下，都是需要直接与 WebView，与前端的 JavaScript 代码进行交互，这时就需要直接获取 UIWebView 的 JSContext。

还有在 iOS 8 后推出来的 WKWebView 没有办法直接获取它的 JSContext，它使用了另一套方面来与 JavaScript 进行交互。

而这些都是在开发中都会碰到的实际问题，所以在之后的文章中再进行详细介绍（又开一个坑）。

## 参考资料

* [JavaScriptCore by Example](https://www.bignerdranch.com/blog/javascriptcore-example/)
* [JavaScriptCore Tutorial for iOS: Getting Started](https://www.raywenderlich.com/124075/javascriptcore-tutorial)
* iOS 7 by Tutorials