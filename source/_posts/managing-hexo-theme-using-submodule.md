---
title: 使用 submodule 管理 Hexo 主题
date: 2017-07-25 20:20:45
tags: [hexo]
---

很久之前[写过一篇文章](http://swiftyper.com/2016/04/17/deploy-hexo-with-git-hook/)，介绍了如何托管 Hexo 中的 Markdown 及配置文件到 Github，并将其部署到个人的 VPS 上。使用这个方法就可以很容易地在不同电脑上同步博客数据，并进行部署。

但是，Hexo 除了内容外还有一个很重要的部分，那就是主题。现在，大多数 Hexo 主题的文档都是直接叫我们 `clone` 主题到对应的 theme 目录中，然后配置 `_config.yml` 就行了。这种方法的好处就是简单明了，坏处就是主题无法跟随博客数据一起同步。

今天我们就来看看管理 Hexo 主题的正确打开方式。

<!-- more -->

## 错误示范

这里以 [hexo-theme-apollo](https://github.com/pinggod/hexo-theme-apollo) 为例。根据它的文档，安装 apollo 主题的步骤如下：

* 安装
  
    ```
    $ npm install --save hexo-renderer-jade hexo-generator-feed hexo-generator-sitemap hexo-browsersync hexo-generator-archive
    $ git clone https://github.com/pinggod/hexo-theme-apollo.git themes/apollo
    ```

* 更新 `_config.yml` 配置文件

    ```
    theme: apollo
    ```

* 如果需要更新主题

    ```
    $ cd theme/next
    $ git pull
    ```

其实，经过上面的步骤，我们只是将配置好了 Hexo 的主题而已。一般情况下，我们还需要对主题本身进行个性化配置，包括：

* 更新主题中的网站及个人信息
* 配置 Google 或者百度统计
* 配置第三方评论系统
* 其它更高级的个性化配置

这些修改，我们当然也是希望能托管到 Github 上的。但是，如果我们到 Github 上查看博客的仓库，我们会发现主题目录是灰掉并且不能点击，这意味着主题并没有进行同步。

![](http://7xqonv.com1.z0.glb.clouddn.com/managing-hexo-theme-using-submodule-pic-1.jpg)

这种情况的原因在于，我们使用了 `clone` 将主题下载到本地，所以它自己本身也是一个 git 仓库。因此它上层的博客仓库就无法对其进行管理，[这篇文章](https://git-scm.com/book/en/v2/Git-Tools-Submodules)里有关于这种情况的详细解释。

## 正确方法

我们需要对主题进行单独的管理，使其成为博客的一个独立部件，而这就是使用 git submodule 的最佳场景。步骤如下：

1. Fork 一份主题到自己的 Github 上。

    Fork 的目的在于，我们可以对主题进行各种个性化的定制以及修改，并对这些定制进行版本控制。同时，我们还能随时与原主题的更新进行合并。

2. 创建一个 submodule。

    ```
    $ cd blog-hexo
    $ git submodule add https://github.com/pinggod/hexo-theme-apollo themes/apollo
    ```

3. 更新 `_config.yml` 使用修改过的主题。

    ```
    theme: apollo
    ```

4. 这时，我们就拥有了两个独立的仓库，一个是 hexo 博客，另外一个是主题。

    ```
    $ cd blog-hexo
    $ git submodule
    # 6c40f5ec27e1889c5a0a0a999e847634a33aef1c themes/apollo (heads/master)
    ```


    并且，在 github 上也可以看到它指向了正确的地址。

    ![](http://7xqonv.com1.z0.glb.clouddn.com/managing-hexo-theme-using-submodule-pic-2.jpg)


使用 submodule 配置好之后，在不同电脑间进行同步就非常简单了：

```
$ cd blog-hexo
$ git pull
$ git submodule update
```

就算是一台全新的电脑，也可以很轻松地进行配置：

```
$ git clone https://github.com/buginux/swiftyper-blog.git
$ cd swiftyper-blog
$ npm install
$ git submodule update --init
```

使用几行代码就能配置一个博客，是不是感觉相当酷炫。而这其中的便利都是拜 Git 及 Github 所赐，这也是我为何如此喜欢它们的原因。

当然，使用这种方法也有缺点，那就是当原主题更新的时候，我们需要进行手动拉取对方的最新代码，并合并到自己的代码中，而且由于我们修改过主题，所以合并的过程中可能会出现冲突，这就需要我们进行手动解决了。不过总体来说，如果我们选择的是一个比较稳定的主题，出现这种情况的机率还是比较小的，相对于 submodule 的便利，这点付出还是值得的。

## 参考资料

* [How to setup a hexo-based blog](http://fenglu.me/2016/08/12/How-to-setup-a-hexo-based-blog-Part-2/)
* [Git Tools - Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)

## 关注

如果你喜欢这篇文章，可以关注我的公众号，随时获取我最新的博客文章。

![qrcode_for_gh_6e8bddcdfca3_430.jpg](http://upload-images.jianshu.io/upload_images/650096-9155581b667f64b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

