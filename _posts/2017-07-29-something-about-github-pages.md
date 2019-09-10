---
layout: post
title: "关于 Github Pages 的有趣的事情"
description: "关于 Github Pages 的有趣的事情，文章描述了 Github Pages CNAME 的新玩法；多个仓库不同源部署的实现等"
date: "2017-07-29 12:18:13"
categories:
    - Weblogs
tags:
    - Github Pages
    - Weblogs
---


关于如何将静态博客部署到 Github Pages 网络上似乎有着无数的教程和实现，随便一搜索，简直就是烂大街。然而，这些教程和文章，很多都是匆忙的实现过程，并没有对一些细节有认真的理解和实现。

## Github Pages 的种类

Github Pages 的种类，Github 官方有个 [详细的文档](https://help.github.com/articles/user-organization-and-project-pages/)，简单说来，分为两个大类 `User/Organization Pages` 和 `Project Pages` 

* User/Organization Pages，这类页面称为用户主页或者组织主页，页面的默认域名是 `username.github.io` 对应用户主页，`orgname.github.io` 对应组织主页。并且必须在该账户名下创建一个名为 `username.github.io` 的仓库，并且只展示其 `master` 分支；

* Project Pages，这类页面称为项目主页，页面的默认域名是 `username.github.io/projectname`，默认展示这个项目仓库里面的 `master`，`gh-pages`分支，或者展示在 `master` 分支中的 `/docs`文件夹；


两种不同的页面种类，要尤其注意一下，不然会莫名其妙，觉得 Github Pages 很混乱。


## Github Pages 自定义域名

这个打算说说自定义域名。Github Pages 的自定义域名，同样，Github 官方也有个 [详细的文档](https://help.github.com/articles/using-a-custom-domain-with-github-pages/) ，简单来说，你必须在自己的仓库添加一个名为 `CNAME` 的文件，文件名就是这个，不可以有小写或者其他的名字。

这个名为 `CNAME` 的文件，里面填写自己想要的域名，大约有下面的例子：

* `example.com` 称为裸域，简短，符合现在不要 www 的潮流；
* `www.example.com` www 子域名，其实跟最后这种自定义子域名并没有实质的区别，不过由于这个是默认的子域名，需要单独提及一下；
* `blog.example.com` 纯粹的子域名，事实上你可以将 `blog` 这个单词换成任何你想要的字母；

这个 `CNAME` 设置倒是没有好说的，唯一需要注意的是，Github pages 会根据你设置的自定义域名进行跳转的：

* 如果你的 `CNAME` 文件包含 `example.com`，那么 `www.example.com` 会重定向到 `example.com`
* 如果你的 `CNAME` 文件包含 `www.example.com`，那么 `example.com` 会重定向到 `www.example.com`

不同的 `CNAME` 设置要配合不同的 DNS 记录设置，这个不想说了，什么 `A` `ALIAS` `ANAME` `CNAME` 等各种域名设置，自己去学习一遍回来就知道怎么设置了。

特别说一下就是域名 `CNAME` 记录的设置，这个域名 `CNAME` 是可以直接 `CNAME` 到自己的 `username.github.io.` （敲黑板：要注意这个后面多了一个点啊！），这样可以充分利用 Github CDN 进行自己的网站内容进行加速，如果你使用命令 `dig` 一下可以看到

    $ dig www.example.com +nostats +nocomments +nocmd
    ;www.example.com.                     IN      A
    www.example.com.              3592    IN      CNAME   YOUR-USERNAME.github.io.
    YOUR-USERNAME.github.io.      43192   IN      CNAME   < GITHUB-PAGES-SERVER >.
    < GITHUB-PAGES-SERVER >.      22      IN      A       199.27.XX.XXX

这样就是经过了 Github CDN 进行分发的内容，是可以加速你的主页的访问的。

不过，如果你使用了 [Cloudflare](https://www.cloudflare.com/) 这样的 DNS 和 CDN 一体化解决方案的提供商的话，直接使用 Cloudflare 的静态缓存就可以啦，还可以免费设置全站的 HTTPS，简单到点点鼠标就完成设置了，真心居家旅行杀人必备。人生已经很艰难了，能简单点为什么不简单点，省下来的时间可以去好好享受生命的美好啊！



