---
layout: post
title: "Python Web 框架中 Context 机制"
description: "Python Web 框架中 Context 机制"
date: "2016-02-28 14:24:52"
categories:
    - Weblogs
tags:
    - Python
    - Weblogs
---

在 Python 世界里的各种 Web framework 摸爬打滚了好些日子，也是时候该总结一下自己的看法了，这篇看法里没有方法论，只有通篇的扯淡。

Web 请求的实现是需要有上下文 context 的，如何保存和传递 context 则是一个框架和库需要解决的核心问题。简单来说，一个简单的 HTTP 请求到达 Server 后，Web framework 需要处理这个 HTTP 请求的数据吧，如何传递和保存这个 HTTP 请求的数据，方便后面的 Handler 来处理，这里就可以简单理解为 HTTP 请求处理过程的上下文，即 Context。

## 1、Function-based Context

Function-based Context 就是利用函数的上下文来传递和保存数据的形式，现实中的 Web framework 有 Flask/Bottle/Django 等，利用 threading.local 这样（或着近似）的全局或者局部变量来传递和保存 Context 的。

## 2、Class-based Context

Class-based Context 就是利用类实例的上下文来传递和保存数据的形式，现实中的 Web framework 有 Tornado/Web.py 等，利用 class 来传递和保存 Context 的。

Context 作为一个 Handler 存在的环境，其意义一直被大家所忽略。而能从 context 去理解一个处理过程，其价值和意义就超出了一个框架和库，进入到更加广阔的线程/进程/协程等一堆的任务处理的状态隔离的术语中去了！融会贯通这个词，在这里可见一斑。