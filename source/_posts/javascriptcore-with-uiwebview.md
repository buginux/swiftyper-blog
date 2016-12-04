---
title: 使用 JavascriptCore 与 UIWebView 进行交互
date: 2016-12-03 11:25:56
categories: iOS
tags: [Hybrid, JavscriptCore]
---

> 本篇文章的示例代码可以在[我的Github](https://github.com/buginux/UIWebViewContextDemo)上进行下载。

在[上一篇文章中](http://www.swiftyper.com/2016/08/22/javascriptcore-basic)我们讨论了 JavaScriptCore 的基本使用，如何在脱离 UIWebView 的情况下让 JavaScript 与原生进行交互。

但是，在混合开发过程中，我们需要的是让原生应用能与 UIWebView 进行流畅的交互。就如上一篇文章讲到的，从 iOS 2 以来，我们与 UIWebView 进行交互的唯一方式就是使用 [stringByEvaluatingJavaScriptFromString:](https://developer.apple.com/library/ios/documentation/uikit/reference/UIWebView_Class/Reference/Reference.html#//apple_ref/occ/instm/UIWebView/stringByEvaluatingJavaScriptFromString:)方法拦截请求，像在 Github 上很火的 [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 就是使用这一原理来实现的。不幸的是，这一现状在 iOS 7 以后并没有改变，虽然苹果公司在 iOS 7 之后推出了 JavaScriptCore 这个新工具，但是官方并没有提供获取 UIWebView 的 JSContext 方法。

<!-- more -->

使用 JavaScriptCore 与 UIWebView 结合进行混合开发，这个需求是如此地合理，以致于我相信不会只有我一个人有这种想法。果然，互联网上牛人多，直接使用 Google 一搜，果然让我找到别人提供的两种解决方案。

> 注意：本篇文章所描述的方法并非是苹果官方提供的——可能甚至是他们所不赞成的，这些方法在文章写作的时候还是可以使用的，但是不保证之后会一直好用，请留意。

## 问题描述

在我们需要使用 JavaScript 与原生进行交互的时候，需要一个 JSContext 实例。当我们使用 JavaScript 代码开发自己的功能的时候，我们可以手动创建 JSContext。 

而每个 UIWebView 实例当中都拥有自己的 JSContext 对象，当我们要与 UIWebView 进行交互的时候，第一步就是要获取它们的 JSContext 对象。但是，苹果官方并没有提供获取 UIWebView 中的 JSContext 对象的方法。

经过搜索之后，发现两种比较通用的方法：
1. 使用 KVC
2. 使用 Category

本篇会使用[一个示例](https://github.com/buginux/UIWebViewContextDemo)来进行演示，代码中只用到了第二种方法，因为个人觉得第二种方法比较方便。

## 使用 KVC

使用这个方法很简单，简单到一句代码就可以描述清楚：
```objective-c
JSContext *context = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
```

只要我们能拿到 UIWebView 的实例，然后就可以直接使用 KVC 的方法来获取它的 JSContext 对象，就这么简单。

## 使用分类

第二种方法就是为 NSObject 添加一个分类，并使用这个分类来实现 WebKit 的 `didCreateJavaScriptContext` 回调，这种方法的具体描述可以参考[https://github.com/TomSwift/UIWebView-TS_JavaScriptContext](https://github.com/TomSwift/UIWebView-TS_JavaScriptContext)。

具体实现代码如下：

```objective-c
@implementation NSObject(JSContextTracker)

+ (NSMapTable *)JSContextTrackerMap {
    static NSMapTable *contextTracker;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        contextTracker = [NSMapTable strongToWeakObjectsMapTable];
    });
    return contextTracker;
}

- (void)webView:(id)unused didCreateJavaScriptContext:(JSContext *)ctx forFrame:(id)alsoUnused {
    NSAssert([ctx isKindOfClass:[JSContext class]], @"bad context");
    if (!ctx)
        return;
    NSMapTable *map = [NSObject JSContextTrackerMap];
    static long contexts = 0;
    NSString *contextKey = [NSString stringWithFormat:@"jsctx_%@", @(contexts++)];
    [map setObject:ctx forKey:contextKey];
    ctx[@"JSContextTrackerMapKey"] = contextKey; // store the key to the map in the context itself
}

+ (JSContext *)contextForWebView:(UIWebView *)webView {
    // this will trigger didCreateJavaScriptContext if it hasn't already been called
    NSString *contextKey = [webView stringByEvaluatingJavaScriptFromString:@"JSContextTrackerMapKey"];
    JSContext *ctx = [[NSObject JSContextTrackerMap] objectForKey:contextKey];
    return ctx;
}
@end
```

在项目中增加这个分类之后，以后要获取 UIWebView 的 JSContext 对象，只需要使用 `[NSObject contextForWebView:myWebView]` 就可以了。

## 实例

示例代码是一个联系人列表，项目里有一个 html 文件，显示了一个添加用户的表单，在点击提交之后，将新联系人添加到本地的数组中，并在 UITableView 中显示出来。

同时，代码里面还演示了[上一篇文章中](http://www.swiftyper.com/2016/08/22/javascriptcore-basic)讨论过的使用 JavaScript 代码调用原生的方法。

核心代码在 `XGAddContactWebViewController` 文件中，这个控制器里面有一个 UIWebView 实例，在 `viewDidLoad` 方法中获取了 JSContext 对象：
```objective-c
self.jsContext = [NSObject contextForWebView:self.webView];
```

然后，在 `- (void)webViewDidFinishLoad:(UIWebView *)webView` 回调中，使用这个 JSContext 来调用原生的方法：
```objective-c
- (void)webViewDidFinishLoad:(UIWebView *)webView {
    [self.loadingView stopAnimating];
    
    [self.jsContext setExceptionHandler:^(JSContext *context, JSValue *value) {
        NSLog(@"WEB JS: %@", value);
    }];
    
    self.jsContext[@"myStore"] = self.store;
    self.jsContext[@"XGContact"] = [XGContact class];
    
    NSString *jsPath = [[NSBundle mainBundle] pathForResource:@"add_a_contact" ofType:@"js"];
    NSString *jsCode = [NSString stringWithContentsOfFile:jsPath encoding:NSUTF8StringEncoding error:nil];
    [self.jsContext evaluateScript:jsCode];
}
```

完成的示例代码请参考[我的Github](https://github.com/buginux/UIWebViewContextDemo)。

## 参考文章
* [JavaScriptCore by Example](https://www.bignerdranch.com/blog/javascriptcore-example/)
* [UIWebView-TS_JavaScriptContext](https://github.com/TomSwift/UIWebView-TS_JavaScriptContext)
* [StackOverflow](http://stackoverflow.com/questions/18920536/why-use-javascriptcore-in-ios7-if-it-cant-access-a-uiwebviews-runtime)

