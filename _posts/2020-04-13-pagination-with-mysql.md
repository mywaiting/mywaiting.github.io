---
layout: post
title: "MySQL 数据库中的分页方案及其效率对比"
description: "MySQL 数据库中的分页方案及其效率对比"
date: "2020-04-13 11:03:13"
categories:
    - Weblogs
tags:
    - MySQL
    - Pagination

---

* toc
{:toc}



## 前言

MySQL 分页是个很常见的项目需求，因为数据量的问题，我们既无法一次导出所有的数据给用户，用户也无法一次性看完所有的数据，所以分页实际上是个很常见的需求。

除了感叹中文内容圈子里有用的内容越来越稀缺，我想做点什么外，我也想将自己的找到解决方案记录下来，以便以后自己忘记了，能有据可查。

## 通用分页

对于 MySQL 这样烂大街的数据库，分页的实现有无数的教程和例子，但是大多数的例子就是下面这样的

```
# 对应的 URL 形式
/news?page=1&per_page=30

# MySQL 专门的 limit 语句实现
SELECT * FROM news limit (page * per_page), per_page

# 标准实现的数据库查询语句实现，如 PostgreSQL
SELECT * FROM news OFFSET (page * per_page) limit per_page
```

这样的分页实现，是最为常见的分页方案，也是最简单最容易理解的分页方案。如果要考虑总页数，那就自己增加一个全库的 `COUNT()` 查询总数即可。

但是这样的分页方案也有明显的缺陷，那就是在数据量大的时候，`limit` 语句容易造成全表扫描，因而效率较低

这里的数据量大的程度，是指数据的条目数以百万甚至千万起算，并且这里的查询均以为 `主键` 作为查询的条件，不涉及其他索引和其他语句的查询的前提下。

> 其实数据量小的情况下，直接使用 `limit` 查询最方便，不必去折腾什么高性能的查询分页的实现，一来没有必要，二来过早优化
>
> 在没有用到排序的情况下，直接使用 `limit` 来分页，即使有千万级别的数据，其实性能是相当不错的，能够在秒级别返回查询结果，对于多数的应用来说是够用的

## 子查询分页

针对使用 `limit offset` 在大量数据里的性能问题，如果使用子查询的话，尤其是涉及 `主键` 排序的时候的性能，有个稍微变通的查询方法

```
# 对应的 URL 形式
/news?page=1&per_page=30

# MySQL 专门的 limit 语句实现
SELECT * FROM news WHERE id >= (SELCT id FROM news limit (page * (per_page - 1)), per_page)
```

如果使用 MySQL 5.1 不支持上面带 `limit` 的子查询，可以自己手动改为 `inner join` 的实现

> 这样的子查询的方式，跟前面的最简单的分页方式近似，也兼顾了性能。千万级别的数据能在秒级别返回查询结果，也能一定程度使用带索引的列进行排序。
>
> 这样的方式稍微兼顾了主键排序的结果，还是可以使用的。很多的互联网公司内（比如阿里巴巴，阿里的 JAVA 开发手册内有这个案例）也是将这个语句作为默认的分页实现的

## 流式分页/游标翻页

其实没有什么流式分页这种业界的分页方式，我自己创造的名词实现吧

多数的时候大家将这样的方式叫做游标翻页

对于 `timeline` 这样的数据流，传统的分页显得很不合适，因为时间线的条目是一直增长的，并且增长得很快，你很难去定义第一页第二页这样的概念。传统的分页对于时间线这样的实现，很容易造成漏数据。

但是时间线的数据流是天然自增的数据，因此引入流式分页的概念，严格来说，流式分页并不能算成一种分页的方式，因为它只是针对数据流某个时间间隙之间的数据

流式分页的实现在 Twitter 和 Facebook 的 API 设计上表现得很明显

```
# Twitter
"search_metadata": {
  "max_id": 250126199840518145,
  "since_id": 24012619984051000,
  "refresh_url": "?since_id=250126199840518145&q=php",
  "next_results": "?max_id=249279667666817023&q=php",
  "count": 10,
  "completed_in": 0.035,
  "since_id_str": "24012619984051000",
  "query": "php",
  "max_id_str": "250126199840518145"
}

# Facebook
{
  "data": [
     ... Endpoint data is here
  ],
  "paging": {
    "cursors": {
      "after": "MTAxNTExOTQ1MjAwNzI5NDE=",
      "before": "NDMyNzQyODI3OTQw"
    },
    "previous": "/albums?limit=25&before=NDMyNzQyODI3OTQw"
    "next": "/albums?limit=25&after=MTAxNTExOTQ1MjAwNzI5NDE="
  }
}
```

从两者的 API 接口文档数据，可以充分看得出两者的设计殊途同归

无论是 Twitter 还是 Facebook，其流式分页都是通过指定当前查询的 `min_id` 和 `max_id` 来查询当前所有数据中的某个时间段的数据，通过指定  `min_id` 和 `max_id` 来查询的效率极其高，以 Facebook 的查询来说明编程的实现如下

```
# 对应的 URL 形式，向前翻页
/news?limit=30&before=[min_id]

# MySQL 专门的 limit 语句实现
SELECT * FROM news WHERE id <= [min_id] limit per_page

# 对应的 URL 形式，向后翻页
/news?limit=30&after=[max_id]

# MySQL 专门的 limit 语句实现
SELECT * FROM news WHERE id >= [max_id] limit per_page
```

使用流式翻页的同学一般都知道 `下一页` 是如何实现的，毕竟传入最大的 `max_id` 即可

但是大家一般都会困惑于 `上一页` 如何实现的，其实最简单的方案就是类似 Facebook 这样的针对向前和向后翻页的区别对待，否则的话，要么建立缓存表或者另建立分页表，要么就只能一页一页往下翻，不能往前翻页（类似于手机时间线流动的感觉）

## 其他分页方式

此处暂无，以后我有发现再更新吧



