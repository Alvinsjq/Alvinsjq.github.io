---
layout: post
title:  "Github Pages和Jekyll建立自己的个人blog小记"
tags:[Jekyll theme,Blog]
comments: true
keywords: "blog"
date: 2017-02-07
---

一直想给自己建一个小站来写写东西，最近终于有时间动手了。首先想着最好能有一种经济实惠、靠谱稳定的平台，后来上网找了许久，发现Github Pages就是我需要的。于是根据前人的博客和Github Pages上的步骤建了一个十分简单的属于自己的网站。接下来简单记录一下建站的过程。

### 前提
- 首先你需要知道几本的Git知识，具体可以移步这里:[ Mac中Git的简单实用](http://blog.csdn.net/qiyu93422/article/details/46456297)。
- 了解一点Jekyll，我由一篇[博文](http://baixin.io/2016/10/jekyll_tutorials1/)得知了[Jekyll中文快速指南](http://jekyll.com.cn/docs/quickstart/)。

### 环境配置
有了上面的前提，大概就能够开始配置环境了，首先确保电脑中装有
- Git环境
- Ruby
- RubyGems

这里假设是Mac用户，那么一般Mac下都已经装有Ruby以及RubyGems了，当然如果之前装过XCode以及 Command-Line Tools的用户，Git环境也应该有的（怎样装Git请看[这里](http://blog.csdn.net/qiyu93422/article/details/46456297)）。

有了这些环境，就可以根据[Jekyll中文快速指南](http://jekyll.com.cn/docs/quickstart/)在终端中安装Jekyll了。

安装Jekyll

`$ gem install jekyll`

新建博客文件夹并进入文件夹，这里取文件夹名为myBlog

`$ jekyll new myBlog`
`$ cd myBlog`

在本地预览，输入命令回车后就能在浏览器中预览博客界面了，这里的地址默认为http://127.0.0.1:4000

`$ jekyll serve`


### 部署项目到GitHub
登陆GitHub账户，并新建一个repository，命名为username.github.io,这里的username就是Github的用户名。这样就可以将其作为Github Pages的项目，也就是博客的项目。这里是[官方的教程](https://pages.github.com/)。

接下来就是将我们本地的项目部署到Github的username.github.io上:
在Mac终端下，进入博客文件夹目录

`$ cd myBlog`

初始化git

`$ git init`

查看本地git状态并提交，下面的files指在status命令下出现的未添加到git中的文件

```
$ git status
$ git add [files]
$ git commit -m "deploy my blog"
```

根据新建的repositor，将本地push到Github上，这里若有疑问依旧可以在[这里](http://blog.csdn.net/qiyu93422/article/details/46456297)找到解答。

```
$ git remote add origin git@github.com:username/username.github.io.git
$ git push -u origin master
```

这样就能在Github上看到博客项目了，于是在浏览器的界面中输入username.github.io就能够访问个人博客了。


### 个性化设置
由于现在的博客用的依旧是初始化的模版，Jekyll的主题是minima的。为了使它真正成为自己的博客，必须对其进行一些个性化的设置。这里我们仅就该主题，做一些简单的个性化设置。[这里](https://github.com/jekyll/minima)是该主题的详细说明。

##### 修改 _config.yml 文件

```
title: 想多懂点的家伙
email: shijian_ecnu@163.com
description: > 实现梦想的过程中，会失去多少...
baseurl: "" # the subpath of your site, e.g. /blog
url: "" # the base hostname & protocol for your site, e.g. http://example.com
twitter_username: Alvin_sjq
github_username:  Alvinsjq
```

##### 将首页的“Post”换成“最新文章”
这往往也是个性化主题的大概步骤，即在myBlog中新建文件夹_layouts/,在里面添加文件home.html，而home.html其实就是从minima项目中复制后修改而来的。再次提交就可以见到不一样的博客了。

## 致谢
1. [Mac中Git的简单实用](http://blog.csdn.net/qiyu93422/article/details/46456297)

2. [Jekyll搭建个人博客](http://baixin.io/2016/10/jekyll_tutorials1/)

3. 被百度谷歌到的各种资料





