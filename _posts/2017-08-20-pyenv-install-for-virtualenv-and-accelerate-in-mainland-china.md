---
layout: post
title: "Pyenv 的安装配置与国内镜像加速"
description: "Pyenv 的安装配置与国内镜像加速，含安装过程、virtualenv、以及在中国大陆如何加速整个安装过程"
date: "2017-08-20 16:36:23"
categories:
    - Weblogs
tags:
    - Python
    - Pyenv
    - Virtualenv
    - Weblogs
---

> **UPDATE 2020/02/22** 针对文章中的部分细节有更新，尤其是国内 `PYTHON BUILD MIRRORS` 的问题

使用 [Python](https://www.python.org/) 的人时常会精神错乱，因为 Python 的版本太多了，有些 Python Package 还会挑版本，这让开发、维护甚至生产环境出现很多很恼人的问题。于是针对这个问题，一系列的 Python 独立包环境出现了，比如有名的 virtualenv 使用空间换时间的战术，通过复制一份已有的 Python 环境，修改系统，特别是 Linux 系统的 `PATH` 变量使得在该虚拟包环境中的 Python 路径指向自己，这样就可以不受系统的 Python 版本影响。

但是 virtualenv 的出现只是部分地解决 Python 的独立环境问题，并没有完全地解决其独立环境的构建问题。如果我需要在同一个系统里，同时存在 Python2.6 、Python2.7 、Python3.5 的版本环境，甚至是 jython、 pypy 这样的环境，并且可以根据需要来切换需要的 Python 版本，使用 virtualenv 会比较麻烦。

这个就是 [Pyenv](https://github.com/pyenv/pyenv) 出现的原因，作为一个 Python 的版本管理工具，实现无缝的 Python 版本切换，并且整合 pyenv-virtualenv 的插件，也使得 [Pyenv](https://github.com/pyenv/pyenv) 具备 virtualenv 一样的创建具体 Python 版本的虚拟包环境的能力。


### pyenv 原理

[Pyenv](https://github.com/pyenv/pyenv) 实现的原理其实很简单。如果你有仔细地观察过 Linux 系统的 `PATH` ，可以在 Linux Shell 中使用命令 `$ echo $PATH ` 查看，那么你就会发现 [Pyenv](https://github.com/pyenv/pyenv) 简单的魔法。简单说来，就是 [Pyenv](https://github.com/pyenv/pyenv) 会在系统的 Shell 环境 PATH 变量中首先插入类似的路径 `~/.pyenv/shims;~/.pyenv/plugins/pyenv-virtualenv/shims` 所有在该系统 Shell 环境执行的 Python 程序都会首先第被这个 `shims` 路径截获，而 `shims` 路径指向哪个 Python 的版本，那就看你激活的是哪个 Python 的版本啦！


### pyenv 安装

[Pyenv](https://github.com/pyenv/pyenv) 的安装异常简单，只需要使用作者提供的一键安装脚本即可

    # Github
    # curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash
    # recommand
    curl -L https://github.com/pyenv/pyenv-installer/raw/master/bin/pyenv-installer | bash
    # simple
    curl https://pyenv.run | bash

然后将 Pyenv 的相关初始化实现放入自己的 `~/.bashrc` 以方便使用

    echo 'export PATH="$HOME/.pyenv/bin:$PATH"' >> ~/.bashrc
    echo 'eval "$(pyenv init -)"'               >> ~/.bashrc
    echo 'eval "$(pyenv virtualenv-init -)"'    >> ~/.bashrc

世界就是这么简单！


### pyenv 国内加速

虽然是安装好了 Pyenv ，但是当你需要具体的 Python 的环境，也是需要下载对应的 Python 二进制程序包来安装的啊！这时候，非常蛋疼的事情就出现了，因为 Pyenv 是直接从 `python.org` 官网上下载对应版本的 Python 程序包的。你想想那网速，还有那众所周知原因不时发神经的网络，目测喝几天的咖啡都不一定能安装好。

其实呢， Pyenv 下载各种 Python 的二进制程序包，都是会首先放到自己的这个 `~/.pyenv/cache` 目录下面的。在需要下载什么文件之前， Pyenv 会先到这个目录找以前是否下载过了，如果已经下载好就直接使用这个目录里面对应的文件。

有上面的思路就很简单啦！利用国内的各种 Python 镜像来下载 Python 的二进制程序包，然后放入到 Pyenv 对应的这个目录里面！这样就可以简单地加速 Pyenv 的安装过程。

国内的 Python 镜像有很多，大概罗列如下，自己找自己对应的速度最快的吧

    http://mirrors.sohu.com/python/ (UPDATE 2020/02/22 时常无法访问)
    # 强烈推荐
    https://npm.taobao.org/mirrors/python/
    

顺便提供一键安装的脚本，用于简化这个世界

    $ export v=2.7.6; wget https://npm.taobao.org/mirrors/python/$v/Python-$v.tar.xz -P ~/.pyenv/cache/; pyenv install $v 

解释一下这个意思，就是从 `https://npm.taobao.org/mirrors/python/` 上下载对应版本的 Python，放入到 `~/.pyenv/cache` 目录里面，然后使用 `pyenv install $v` 即可，`$v` 变量代表 Python 的版本，自己需要哪个版本，就把这个改成自己需要的版本号！

> **注意** `~/.pyenv/cache` 需要自己建立，假如不存在的话 (UPDATE 2020/02/22）

### pip 安装国内加速

既然说到国内安装加速，还是顺便说说这个 pip 安装 Python 程序包加速的问题吧。pip 在国内有很多的镜像，比较稳定有

    http://mirrors.aliyun.com/pypi/simple
    http://pypi.douban.com/simple
    http://pypi.v2ex.com/simple
    https://pypi.tuna.tsinghua.edu.cn/simple （特别推荐）

根据自己所在位置的速度来选用吧，一般说来，TUNA 的速度是比较不错的

只使用镜像来加速一个 Python 程序包的安装可以一个命令搞定，比如说安装 [Tornado](https://www.tornadoweb.org) Web 框架

    pip install tornado -i http://pypi.douban.com/simple

但如果为了方便工作，可以配置成默认的 pip 安装镜像，创建或者修改以下文件 `~/.pip/pip.conf` 写入以下文件内容

    [global]
    index-url = https://pypi.tuna.tsinghua.edu.cn/simple
    [install]
    trusted-host = pypi.tuna.tsinghua.edu.cn

这样在使用 pip 来安装时，会默认调用该镜像。


以上，就是我使用 pyenv 一些小总结，希望你看到的话，能觉得有用。
