---
title: Hexo 踩坑 - 不要在标题中开头使用方括号
date: 2017-07-30 09:24:06
tags: [Hexo]
---

一个刚刚踩过的 Hexo 的坑，浪费了一个多小时，不过幸好最后解决了。事实证明是一个很弱智的问题，跟写代码时漏掉逗号，找一个多小时一样的弱智。

不过还是记录下，也许还有人会踩到同样的坑，谁知道呢？

## 场景

刚刚完成一篇新文章，不过由于是翻译的，想在在标题的开头注明是翻译文章。我自己的习惯是使用方括号来进行注明，因此文章的标题就成了这样：

```
[译]这是一篇翻译的文章
```

不幸的是，Hexo 的 meta 数据可以使用方括号来进行指定，因此 yaml 会尝试去解析这个 `[译]`，结果就是一堆的报错调用栈（我把全部报错都贴进来，以供搜索引擎抓取）：

```
ERROR Process failed: _posts/translated_article.md
Error
    at generateError (/Users/buginux/blogs/site-blog/node_modules/js-yaml/lib/js-yaml/loader.js:162:10)
    at throwError (/Users/buginux/blogs/site-blog/node_modules/js-yaml/lib/js-yaml/loader.js:168:9)
    at readBlockMapping (/Users/buginux/blogs/site-blog/node_modules/js-yaml/lib/js-yaml/loader.js:1045:9)
    at composeNode (/Users/buginux/blogs/site-blog/node_modules/js-yaml/lib/js-yaml/loader.js:1331:12)
    at readDocument (/Users/buginux/blogs/site-blog/node_modules/js-yaml/lib/js-yaml/loader.js:1493:3)
    at loadDocuments (/Users/buginux/blogs/site-blog/node_modules/js-yaml/lib/js-yaml/loader.js:1549:5)
    at Object.load (/Users/buginux/blogs/site-blog/node_modules/js-yaml/lib/js-yaml/loader.js:1566:19)
    at parseYAML (/Users/buginux/blogs/site-blog/node_modules/hexo-front-matter/lib/front_matter.js:80:21)
    at parse (/Users/buginux/blogs/site-blog/node_modules/hexo-front-matter/lib/front_matter.js:56:12)
    at /Users/buginux/blogs/site-blog/node_modules/hexo/lib/plugins/processor/post.js:52:18
    at tryCatcher (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/util.js:16:23)
    at Promise._settlePromiseFromHandler (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:509:35)
    at Promise._settlePromise (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:569:18)
    at Promise._settlePromise0 (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:614:10)
    at Promise._settlePromises (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:693:18)
    at Promise._fulfill (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:638:18)
    at PromiseArray._resolve (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise_array.js:126:19)
    at PromiseArray._promiseFulfilled (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise_array.js:144:14)
    at PromiseArray._iterate (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise_array.js:114:31)
    at PromiseArray.init [as _init] (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise_array.js:78:10)
    at Promise._settlePromise (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:566:21)
    at Promise._settlePromise0 (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:614:10)
    at Promise._settlePromises (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:693:18)
    at Promise._fulfill (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:638:18)
    at PromiseArray._resolve (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise_array.js:126:19)
    at PromiseArray._promiseFulfilled (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise_array.js:144:14)
    at Promise._settlePromise (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:574:26)
    at Promise._settlePromise0 (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:614:10)
    at Promise._settlePromises (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:693:18)
    at Promise._fulfill (/Users/buginux/blogs/site-blog/node_modules/bluebird/js/release/promise.js:638:18)
```

## 解决

一开始认为是文件的问题，就把旧文件删除了，再新建一篇文章，然后发现刚新建完的时候是好的，但只要把文章内容贴上去，就会造成报错。

在这一个多小时时，我尝试修改了 hexo 的 `_config.yml` 文件，尝试修改了主题的 `_config.yml` 文件，尝试过切换成别的主题，都没有解决这个问题。

后来感觉可能是文件内容，或者 Markdown 的格式有错误，就用了最笨的方法，每次删除一点，然后再重新生成，结果反复删了七八次，最后把整篇文章的内容都删除了，还是会报错！

<div style="text-align: center">
	<img src="http://7xqonv.com1.z0.glb.clouddn.com/struggle-with-evernote-api-pic-1.jpg" alt="are-you-kidding-me">
</div>

直到这时，我还是没有察觉是标题的问题，还感觉可能是 tags 格式的问题，最后也是重复试了十几遍，各种 google，发现还是不好使，一怒之下，把 tags 也删除了。结果就是，问题依然没有解决。

<div style="text-align: center">
	<img src="http://7xqonv.com1.z0.glb.clouddn.com/hexo-tip-do-not-using-square-brace-in-title-pic-1.jpg" alt="are-you-kidding-me">
</div>

这里就比较尴尬了，貌似已经走投无路了，我仿佛听到了 Hexo 的讪笑声：

> 终于把你逼到绝路了！

正如[坂田银时](https://zh.wikiquote.org/zh/%E9%8A%80%E9%AD%82#.E5.9D.82.E7.94.B0.E9.8A.80.E6.99.82)说过的：

> 有阳光的地方就会有阴影，所以有阴影的地方也一定会有阳光。绝望的颜色越是浓厚，在哪里也一定会存在耀眼的希望之光。

有热血技能傍身的我，怎么可能会认输：

> 开什么玩笑 我是故意走到绝路的！

事实上，当时有点自暴自弃，绝望之下把标题也删了，然后重新生成发现，错误不见了！这时，再重新回过去审视原来的标题，才发现就是方括号导致的问题。这时，把方括号再改成小括号，重新生成，确实不会再报错了。

## 小结

如果，你碰巧也遇到了这个问题，碰巧查到了这篇文章，那么，劳烦你在文章底下留下你的足迹。因为，我不希望我是唯一一个碰到这种很傻的问题，而且还花了这么长时间才解决的人。

## 赞赏

如果本篇文章对你有帮助，可以进行小额赞助，鼓励作者写出更好的文章。

![Paying](http://7xqonv.com1.z0.glb.clouddn.com/wechatpaying.png)



