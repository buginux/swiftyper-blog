---
title: 印象笔记 Python SDK 的 Token 认证问题
date: 2017-02-18 12:23:14
tags: [Memo, Alfred]
---

于我而言，一个笔记工具最重要的两个功能就是信息采集和提取。

人脑是用来思考而非用来记忆的，因此我们需要一个外部系统来存储个人化的信息。这一点的体现就在于，我们能随时随地方便地将看到的有价值的信息保存起来。网上的各种博客文章和微信公众号是我的主要信息来源，我的印象笔记配合上剪藏插件就能完美地解决我信息采集的问题。

提取信息最好的办法是搜索，印象笔记在这一点上也是做得很出色。可以说，有了它的搜索功能，基本上我收藏文章的时候可以不需要进行分类了（然而，作为一个有整理强迫症的人，不分类我是不能忍的）。IBM 做过一个研究，让人去找一封电子邮件，有的人喜欢搜索，有的人喜欢平时就把邮件分类。结果搜索的人平均只需要 17 秒可以找到他想要的邮件，而分类的人则需要 58 秒。

虽然印象笔记的搜索功能已经很好用了，但是每次要搜索的时候还是得先打开印象笔记再进行搜索，这一个流程下来，少说也要花掉三四秒的时间，并且还要把手从键盘上移到鼠标（触摸板）上，实在是太费劲了。

<!-- more -->

## 优化搜索方案

对于使用 Mac 的人来说，要提高效率，第一反应就是能不能使用 Alfred 来完成这件事。本着不重复造轮子的原则，我立马上 Google 搜索了印象笔记相关的 workflow，结果还真让[我找到一个](https://github.com/shaoshing/alfred-evernote)。

但是由于这个 workflow 使用的 JavasScript SDK 是旧版本的，很多功能都不能正常使用，再加上它针对的是 Evernote 而非印象笔记。与其去大动干戈地修改它里面的内容，还不如仿造这个项目，自己使用 Python 写一个。

由于有一个现成的项目做参照，因此在做 workflow 的时候没有碰到什么大问题。反而是在我自认为可以很快完成的印象笔记 API 上跌入了大坑中，白白浪费不少时间。

整个 API 开发过程中主要就是卡在 Token 认证问题上，主要原因就是官方的开发文档没写清楚，特别是对于印象笔记和 Evernote 间的认证差异问题没有特别说明。这里主要就是把我接入 SDK 过程中一些问题记录下来，希望可以帮助后来人不再翻车。

> workflow 已完成，并上传到 Github：[https://github.com/buginux/evernote-alfred](https://github.com/buginux/evernote-alfred)

## Token 认证问题

由于我们需要的只是搜索自己的笔记，因此也不需要走 OAuth 的整个流程了。只需要在[印象笔记的 Developer Token](https://app.yinxiang.com/api/DeveloperToken.action)的页面上直接获取到 Token 就行了。

根据文档上的说法，只要做下面的操作就可以认证了：

```python
dev_token = "put your dev token here"
client = EvernoteClient(token=dev_token)
userStore = client.get_user_store()
user = userStore.getUser()
print user.username
```

但是，我在做这步操作的时候，死活都是认证不通过。直到我去翻了这个 SDK 的源码后才发现这里面藏着几个坑啊（这也就是开源的好处，源码才是最官方的文档）。

下面是 EvernoteClient 的构造方法：

```python
class EvernoteClient(object):

    def __init__(self, **options):
        self.consumer_key = options.get('consumer_key')
        self.consumer_secret = options.get('consumer_secret')
        self.sandbox = options.get('sandbox', True)
        if self.sandbox:
            default_service_host = 'sandbox.evernote.com'
        else:
            default_service_host = 'www.evernote.com'
        self.service_host = options.get('service_host', default_service_host)
        self.additional_headers = options.get('additional_headers', {})
        self.token = options.get('token')
        self.secret = options.get('secret')
```

可以看到，构造方法有一个 sandbox 参数，默认是 `True`。sandbox 服务器是印象笔记的开发测试服务器，主要用于在应用程序发布到生产环境前进行的开发工作。但是，我们要做的东西只是针对自己的帐号，没有所谓的开发或者生产环境，并且测试服务器里面是没有笔记的，也就没办法进行笔记搜索的测试了。

然而，在我把代码改成 `client = EvernoteClient(token=dev_token, sandbox=False)` 后，还是认证不通过。

无奈之下，只好再回过头看源码，才发现构造方法里面用的服务器地址全是 Evernote 的，并没有印象笔记的服务器地址，我们必须**自己指定 `service_host` 参数**。用印象笔记的 Token 来 Evernote 服务器上进行验证，怎么可能认证通过！

这么关键的一个东西，[开发者文档](https://dev.yinxiang.com/doc/start/python.php)上居然一！个！字！都！没！提！

<div style="text-align: center">
	<img src="http://7xqonv.com1.z0.glb.clouddn.com/struggle-with-evernote-api-pic-1.jpg" alt="are-you-kidding-me">
</div>

好吧，直接指定 `service_host` 的话，sandbox 参数也可以省了，这次代码应该是这样的：

```python
client = EvernoteClient(token=dev_token, service_host='https://www.yinxiang.com')
```

谁知道，这次改过之后直接运行错误了，信息如下：

```python
# ... 前面省略
  File "/usr/local/Cellar/python/2.7.13/Frameworks/Python.framework/Versions/2.7/lib/python2.7/socket.py", line 557, in create_connection
    for res in getaddrinfo(host, port, 0, SOCK_STREAM):
socket.gaierror: [Errno 8] nodename nor servname provided, or not known
```

原来，python 是使用 socket 进行通信的，所以不能使用 `https://` 前缀，网上有很多类似的问题，但是[只有这个解决方案](http://stackoverflow.com/questions/38582990/gaierror-errno-8-nodename-nor-servname-provided-or-not-known)才是针对我这种情况的。

现在，该加的也加了，该删了也删了，应该是可以正常运行了。不过，果然还是太天真了，运行之后出错了另一个错误：

```python
# ... 前面省略
  File "/usr/local/lib/python2.7/site-packages/thrift/protocol/TBinaryProtocol.py", line 137, in readMessageBegin
    name = self.trans.readAll(sz)
  File "/usr/local/lib/python2.7/site-packages/thrift/transport/TTransport.py", line 63, in readAll
    raise EOFError()
EOFError
```
......
......
......

<div style="text-align: center">
	<img src="http://7xqonv.com1.z0.glb.clouddn.com/struggle-with-evernote-api-pic-2.jpg" alt="why">
</div>

网上关于这个问题的回答就更多了，但是都不是我需要的答案。直到，我碰到另一个同样问题的人，它是因为少写了 `www` 才出现的问题。我就想，会不会是我的域名写错了，然后我就到使用 [www.yinxiang.com](www.yinxiang.com) 去访问了下印象笔记的官网，结果发现没毛病啊，完全可以正常访问。

就在我确信已经没有任何办法的时候，我的程序员直觉告诉我，再试最后一次。

于是，我随手登录了我的帐号，然后发现...浏览器地址栏上的 URL 变成了 `https://app.yinxiang.com/Home.action`。

我用着颤抖的双手，把代码改成了：

```python
client = EvernoteClient(token=dev_token, service_host='app.yinxiang.com')
```

然后运行。

终于，控制台上打印出了我的用户名。

<div style="text-align: center">
	<img src="http://7xqonv.com1.z0.glb.clouddn.com/struggle-with-evernote-api-pic-3.jpg" alt="misson-complete">
</div>

## 其它的一些小问题

我在用 python 写 workflow 的时候，使用到了 [alfred-workflow](https://github.com/deanishe/alfred-workflow) 这个库，它对于接受的参数已经做过了 unicode 处理，如果直接传到笔记搜索 API 里面是有问题的。

因此，在使用前需要先进行转码：

```python
args = wf.args
query = args[1].encode('utf-8')
``` 

关于 python2 中的 Unicode 处理，可以参考 [Unicode HOWTO](https://docs.python.org/2/howto/unicode.html)。

## 小结

其实，只要文档上再多加一句说明，我也不用为折腾这个问题浪费大半天时间了。

经过这件事之后，我对其它两件事更加确信了：

1. 坏的文档，不如没有文档
2. 程序员能相信的只能自己和源码

如果你对这个 workflow 感兴趣的话，可以到[我的 Github](https://github.com/buginux/evernote-alfred)上进行下载。

![Show Case](http://7xqonv.com1.z0.glb.clouddn.com/struggle-with-evernote-api-pic-4.png)

## 赞赏

如果这篇文章让你避免了踩坑翻车问题，或者这个 workflow 对你有用的话，可以请作者请杯 koi 哟。

![WeChatPaying](http://7xqonv.com1.z0.glb.clouddn.com/wechatpaying.png)


