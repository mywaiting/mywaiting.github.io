---
layout: post
title: "Introducing Nginx NJS"
description: "Introducing Nginx NJS"
date: "2018-08-18 21:09:13"
lastmod: "2018-12-31 18:55:13"
categories:
    - Weblogs
tags:
    - Nginx
    - NJS
    - Weblogs
---

* toc
{:toc}


![](/media/introducing-nginx-njs/introducing-nginx-njs-768x461.jpg)

[Nginx](https://nginx.org) 作为世界最为流行、性能最为强大的 Web 服务器，一直在各种架构中担任前端反向代理进行流量转发的重要组成部分。考虑到 Nginx 其良好的代码结构，其实是可以将其视为高性能的异步编程框架来看待的，只是 Nginx 的编程需要略为深厚的 C 语言编程功底，这在一定程度上阻止了很多满腔热情的同学（比如我）想在这上面搞些小动作的想法。

考虑到我天性愚笨，我能想到的事情，肯定有人想到了！Nginx+Lua 这个组合 [OpenResty](https://openresty.org) 已经日趋成熟，说起来，其背后主要创始人 [章亦春](http://agentzh.org/) 和前期创始人 王晓哲（花名清无，淘宝网技术专家）都是国人，算是国人里为数不多的原创技术的骄傲。通过揉和众多设计良好的 Nginx 模块，OpenResty 有效地把 Nginx 服务器转变为一个强大的 Web 应用服务器，基于它开发人员可以使用 Lua 编程语言对 Nginx 核心以及现有的各种 Nginx C 模块进行脚本编程，构建出可以处理过万并发请求的高性能的 Web 应用。不过本文并非针对 OpenResty 进行介绍，需要进一步了解的同学可以自行搜索相关资料进一步学习。

为什么要提及 OpenResty 呢？因为我是通过关注 OpenResty 进入到将 Nginx 转为强大的 Web 应用服务器这个议题当中的，而且我也确实使用 OpenResty 编写了一些小应用，从个人的开发体验来说，我觉得 OpenResty 不是那么的方便，当然了，这里很大一部分的原因是因为我对 Lua 编程和 OpenResty 不熟悉所导致的。而且在编译部署方面，OpenResty 携带的 LuaJIT 库也让我这样的代码洁癖的深感麻烦。多方了解之后，Nginx 官方实现的 NJS 走进了我的视线。

## 初识 Nginx NJS

NJS 是什么？其实就是 Nginx + JavaScript 的缩写。很多年前，Nginx 官方就已经开始这个项目了，那时候还叫 nginxScript，后来还混合叫，一会是 nginxScript 一会是 NJS，最近才定下只叫 NJS 的，如果你觉得有疑问，可以看看他们的代码仓库的 commit 记录 [Renaming remnants of "nJScript" and "njscript" to "njs"](http://hg.nginx.org/njs/rev/e38c3f59e30c)，简单来说，就是 Nginx 里面可以运行 JavaScript，用 JavaScript 来构建动态的 Web 应用啦！

- Nginx NJS 的 [官方介绍页面](https://nginx.org/en/docs/njs/)
- Nginx NJS 的 [changelog](https://nginx.org/en/docs/njs/changes.html)，可以围观一下项目进展情况
- Nginx NJS 的 [API 文档](https://nginx.org/en/docs/njs/reference.html)，里面有一些 NJS Demo 例子，可以参考看看
- Nginx NJS 的 [官方代码仓库](https://hg.nginx.org/njs)，竟然不是流行的 Git 而是 Mercurial 仓库


Nginx NJS 实际上是 Nginx 官方折腾的一个完全独立的 JavaScript 的执行环境(VM)，说到 JS 那肯定免不了也提及 JS 的语言规范，目前 NJS 是有限实现的 JavaScript，简单来说，Nginx NJS 并不是一个完整的与 [Node.JS](https://nodejs.org) 和 [Google V8](https://developers.google.com/v8/) 这样的 JavaScript 解析引擎。由于项目还在不断开发当中，请自行到 [官方介绍页面](https://nginx.org/en/docs/njs/) 进行查看当前支持哪些 JS 的特性。

Nginx NJS 包含两个 Nginx 扩展模块，具体如下

- [ngx_http_js_module](https://nginx.org/en/docs/http/ngx_http_js_module.html)
- [ngx_stream_js_module](http://nginx.org/en/docs/stream/ngx_stream_js_module.html)

这两个模块目前还没有默认集成到 Nginx mainline 的发布里面，但是你可以通过代码编译安装或者作为 Nginx 动态模块来安装。

## Nginx NJS 的安装

这里不详细地说明通过代码编译安装 Nginx NJS，只是简单说明作为 Nginx 动态模块来安装。 以下的操作环境都是基于 Ubuntu Linux 下面实现，其他的 Linux 发行版本请自行了解。

首先要在 Ubuntu 中添加 Nginx 官方的 APT 仓库，可以便于未来通过 `apt-get update` 来安装或者更新软件包

    # 首先导入 Nginx 官方的 PGP Key
    wget -qO - https://nginx.org/keys/nginx_signing.key | sudo apt-key add -

    # 添加 Nginx 官方 APT 仓库到 `/etc/apt/sources.list`
    sudo add-apt-repository -y "deb http://nginx.org/packages/mainline/ubuntu/ $(lsb_release -sc) nginx"
    sudo add-apt-repository -y "deb-src http://nginx.org/packages/mainline/ubuntu/ $(lsb_release -sc) nginx"

    # 更新 APT 列表
    sudo apt-get update

如果你先前已经增加过 Nginx 的官方 APT仓库，那么上面这个步骤是可以省略的，请执行下面的命令确认 Nginx 和 NJS module 已经可以进行 `apt-get install` 进行安装

    sudo apt-cache search nginx | grep nginx

 返回的结果一般包含以下的 module 

    nginx-module-geoip - nginx GeoIP dynamic modules
    nginx-module-geoip-dbg - debug symbols for the nginx-module-geoip
    nginx-module-image-filter - nginx image filter dynamic module
    nginx-module-image-filter-dbg - debug symbols for the nginx-module-image-filter
    nginx-module-njs - nginx njs dynamic modules
    nginx-module-njs-dbg - debug symbols for the nginx-module-njs
    nginx-module-perl - nginx Perl dynamic module
    nginx-module-perl-dbg - debug symbols for the nginx-module-perl
    nginx-module-xslt - nginx xslt dynamic module
    nginx-module-xslt-dbg - debug symbols for the nginx-module-xslt

这个时候就可以执行安装啦

    sudo apt-get install nginx nginx-module-njs

执行这个命令会自动安装 Nginx 和 Nginx NJS module 的，如果你去到 `/etc/nginx/` 这个 Nginx 默认的配置文件存放目录见到里面包含了 `modules` 的软连接的时候，应该就是安装成功了。

    mywaiting@ubuntu:/etc/nginx$ ls -al
    ...
    drwxr-xr-x   2 root root 4096 Jan  9  2018 conf.d
    lrwxrwxrwx   1 root root   22 Jan  9  2018 modules -> /usr/lib/nginx/modules
    ...
    -rw-r--r--   1 root root  846 Jan  9  2018 nginx.conf
    ...
    drwxr-xr-x   2 root root 4096 Jan  9  2018 sites-available
    drwxr-xr-x   2 root root 4096 Jan  9  2018 sites-enabled

## Nginx NJS 的配置

Nginx 在 `Nginx 1.9.11`(release at 2016-02-09) 版本中增加了 `Nginx Dynamic Module` 的支持，你可以在 [官方的文档](http://nginx.org/en/docs/ngx_core_module.html#load_module) 看到其最低支持的版本。 Nginx 官方还写了一篇很长的 Blog 来介绍这个功能，链接见这里 [Compiling Third-Party Dynamic Modules for NGINX and NGINX Plus](https://www.nginx.com/blog/compiling-dynamic-modules-nginx-plus/)，官方的实现，省去了重新编译加载 Nginx 的麻烦，就是跟 Apache DSO 一个卵样的功能吧。

Nginx 中，要加载动态模块很简单，直接将动态模块的共享对象文件的路径指定为配置文件中 load_module 指令的值即可

    load_module modules/ngx_http_js_module.so;
    load_module modules/ngx_stream_js_module.so;

但是，请注意 Nginx 的动态模块在 `/etc/nginx/nginx.conf` 配置文件中是有设置的前后关系的，`load_module` 这个指令必须在每个 `block` （包括 `events`、`http`、`stream`、`mail`）之前对其进行描述，举个例子：

    user  nginx;
    worker_processes  1;

    events {
        worker_connections  1024;
    }

    load_module "modules/ngx_stream_module.so";
    load_module "modules/ngx_http_geoip_module.so";

    http {
    }

修改配置文件成这样，执行 `sudo nginx -t` 来测试配置文件的正确性的时候，会出现这样的错误：

    nginx: [emerg] "load_module" directive is specified too late in /etc/nginx/nginx.conf:13
    nginx: configuration file /etc/nginx/nginx.conf test failed

所以 `load_module` 必须在全局配置后进行配置，如下例子就是正确的：

    user  nginx;
    worker_processes  1;

    load_module "modules/ngx_stream_module.so";
    load_module "modules/ngx_http_geoip_module.so";

    events {
        worker_connections  1024;
    }

    http {
    }

就这么简单，调换个顺序，Nginx 就不会报错了。其实这样设计，也很容易理解吧，毕竟 `load_module` 是需要在每个 `block` （包括 `events`、`http`、`stream`、`mail`）之前进行加载并运行的，虽然说 `/etc/nginx/nginx.conf` 是声明式的配置文件，但是必须这样声明在一定程度也有利于告诉开发者其顺序的重要性吧。

这里贴一个 Nginx NJS 实例的 `/etc/nginx/nginx.conf` 文件，带中文注释，方便参考

    mywaiting@ubuntu:/etc/nginx$ cat nginx.conf

    user  nginx;
    worker_processes  1;

    # 必须在此处 load_module 原因上面解释过！
    load_module modules/ngx_http_js_module.so;
    load_module modules/ngx_stream_js_module.so;

    # 错误 log 修改为 debug 级别，方便开发时候查看错误日志。产品环境请修改为 warn 级别
    error_log  /var/log/nginx/error.log debug;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        keepalive_timeout  65;

        #gzip  on;

        # Nginx NJS
        # Nginx NJS 配置开始 ... NJS config start ...
        #
        # 此处准备加入 Nginx NJS 的配置内容，备注！ NJS config here ...
        #
        # Nginx NJS 配置结束 ... NJS config end ...

        include /etc/nginx/conf.d/*.conf;
    }

## Nginx NJS 的使用

有了上面折腾的基础，想玩转 Nginx NJS 那就是分分钟的事情啦。请在上面示例配置文件中的 Nginx NJS 的注释处加入下面的配置


    js_include http.js;

    js_set $foo     foo;
    js_set $summary summary;

    server {

        # 绑定到 8000 端口访问
        listen 8000;

        location / {
            add_header X-Foo $foo;
            js_content baz;
        }

        location = /summary {
            return 200 $summary;
        }

        location = /hello {
            js_content hello;
        }
    }

同时在 `/etc/nginx/` 目录下面新建 `http.js` 并输入以下内容

    function foo(r) {
        r.log("hello from foo() handler");
        return "foo";
    }

    function summary(r) {
        var a, s, h;

        s = "JS summary\n\n";

        s += "Method: " + r.method + "\n";
        s += "HTTP version: " + r.httpVersion + "\n";
        s += "Host: " + r.headersIn.host + "\n";
        s += "Remote Address: " + r.remoteAddress + "\n";
        s += "URI: " + r.uri + "\n";

        s += "Headers:\n";
        for (h in r.headersIn) {
            s += "  header '" + h + "' is '" + r.headersIn[h] + "'\n";
        }

        s += "Args:\n";
        for (a in r.args) {
            s += "  arg '" + a + "' is '" + r.args[a] + "'\n";
        }

        return s;
    }

    function baz(r) {
        r.status = 200;
        r.headersOut.foo = 1234;
        r.headersOut['Content-Type'] = "text/plain; charset=utf-8";
        r.headersOut['Content-Length'] = 15;
        r.sendHeader();
        r.send("nginx");
        r.send("java");
        r.send("script");

        r.finish();
    }

    function hello(r) {
        r.return(200, "Hello world!");
    }

打开浏览器访问对应地址的 8000 端口，就能看到 Nginx NJS 返回的内容啦！

简单说，Nginx NJS 就是在 Nginx 的配置文件中 `/etc/nginx/nginx.conf` 增加几条指令

- js_content，执行其中 JS 内容并输出
- js_include，指定特定的文件内的 JS 代码处理请求
- js_set，设置特定的 JS 变量

上面这是 [ngx_http_js_module](https://nginx.org/en/docs/http/ngx_http_js_module.html) 的使用方法，然而 [ngx_stream_js_module](http://nginx.org/en/docs/stream/ngx_stream_js_module.html) 也是基本一样的，就是多了几个指令而已

- js_access，Nginx [Access](http://nginx.org/en/docs/stream/stream_processing.html#access_phase) 阶段执行的 JS 内容
- js_filter，Nginx 输出的数据过滤阶段执行的 JS 内容
- js_include，指定特定的文件内的 JS 代码处理请求
- js_preread，Nginx [Preread](http://nginx.org/en/docs/stream/stream_processing.html#preread_phase)  阶段执行的 JS 内容
- js_set，设置特定的 JS 变量

就这么简单，自己实践一下就明白怎么回事啦！


## Nginx NJS 编程实践

如果熟悉 NodeJS 系列的 Web 开发，那 Nginx NJS 的开发也不是什么难事，毕竟就是 HTTP 整个数据流过程的理解问题，但是由于 Nginx NJS 自身实现的 JavaScript 引擎的特殊性以及 Nginx 数据流生命周期的特殊性，导致 Nginx NJS 也有一些自己的坑，我的经验也有限，在这里略略分享一二。

### NJS HTTP Request read-only

所有 Nginx NJS 的请求都自动绑定到一个 JS 函数

    // Nginx NJS exmaple
    function hello(r) {
        r.return(200, "Hello world!");
    }

而函数变量 `r` 就相当于 Nginx NJS 的 `HTTP Request Object`，具体的 HTTP Request Object 文档请参考这里 [njs API Doc](http://nginx.org/en/docs/njs/njs_api.html#http) ，其中尤其需要注意很多变量都是 `read-only`  的，怎么理解这个只读呢？我举个例子

    // Nginx NJS read-only example
    function hello(r) {
        var args = Object.create(r.args);
        r.return(200, args);
    }

这段 JS 里看起来完全没有问题的代码，在 Nginx NJS 里面运行是会报错的，为什么？原因就是这个 `r.args` 是 `read-only` 的，简单来说，Nginx NJS 里 `read-only` 的变量都是惰性加载的，虽然  `r.args` 是包含所有请求参数的 `object` ，但是这个 `object` 不是 Nginx NJS 引擎初始化的时候就建好的，它只是 Nginx 内部请求参数的 JS 环境的 binding，用过 OpenResty 的同学应该对这个是很熟悉的。

问题是有了，那么我应该如何绕过，或者处理我的代码？答案是复制（ deepcopy ）就可以了

    // Nginx NJS read-only example
    function hello(r) {
        var _args = {}, arg;

        // req.args is readonly, just take a copy to self define object
        for (arg in req.args) {
            _args[arg] = req.args[arg];
        }

        r.return(200, JSON.stringify(_args));
    }

### NJS HTTP Request subrequest

Nginx NJS 实现的这个 子请求，其实是很类似 OpenResty 里面的 `ngx.location.capture` 用于在 NJS HTTP Request 的内部去请求别的资源或者服务，从 Nginx NJS 的这个实现看得出来，Nginx NJS 几乎就要变得跟 OpenResty 几乎一样啦！当然了，目前 Nginx NJS 和 OpenResty 包的成熟度和社区的生态没有可比性。

要使用这样的 子请求很简单，见下面的代码的例子，也是来自 [官方的例子](http://nginx.org/en/docs/njs/njs_api.html#example_subrequest)

    function version(r) {
        r.subrequest('/api/3/nginx', {
                args: '',                  // 注意：此处 args 的类型必须是 string
                method: 'GET',             // GET/POST/PUT/DELETE...
                body: JSON.stringify('')   // 注意：此处 body 的类型也必须是 string
            }, 
            function(res) {
                if (res.status != 200) {
                    r.return(res.status);
                    return;
                }

                var json = JSON.parse(res.responseBody);
                r.return(200, json.version);
        });
    }

请注意注释处的说明，其实 [NJS HTTP Request subrequest](http://nginx.org/en/docs/njs/njs_api.html#subrequest) 的文档也有仔细说明，就是比较容易忽略吧。

另外，进行 subrequest callback 处理的时候，需要特别注意返回数据的大小，如果子请求返回的数据特别大（超过 4KB 或者 8KB），那么记得自己设置一下 [subrequest_output_buffer_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#subrequest_output_buffer_size) 为一个合适的大小，最好是设置为 4KB 和 8KB 的整数倍（系统的内存页的大小跟系统版本和硬件都有关系，一般是 4KB 或者 8KB），以方便内存页对齐。

## Nginx NJS 的未来

从代码仓库的更新看来，Nginx NJS 的实现还是挺有保障的，从 nginxScript 到现在的 njs，感觉 Nginx 是想选择 JavaScript 这门 Web 的原生态语言来作为其 Web 应用开发语言的实现的，毕竟不是那么多的人都会 LUA 及其 OpenResty 的，但 JavaScript 却是每个 Web 开发者多多少少都会的。

目前 Nginx NJS 的版本才  `0.2.3` (截至到 2018-08-18)，其实现还是相当有限的，尤其是开发时候 debug 的支持尤其有限，目前只能通过 error_log 来展示，且错误日志的详细程度很不如人意，但是官方支持，这点就足够了。

从实现上说，Nginx NJS 已经基本具备编写 Web 应用的能力了，Nginx NJS 作为逻辑的实现，subrequest 配合 Nginx Redis module 和 Nginx MySQL module 就能实现基本的数据库访问，什么 session、cookie、template 的实现，借鉴庞大的 Node.JS 的生态，改动改动，适配一下，那都是分分钟的事情。

所以，还是期待一下 Nginx NJS 的未来吧。


## Nginx NJS 常见问题

### Q. Nginx NJS 与 JavaScript 的关系？

A. 简单说来，Nginx NJS 是 JavaScript/ECMAscript 的子集，它实现了大部分的JavaScript语言的能力，但是没有完全遵从 ECMAScript 标准，同时抛弃了 JavaScript 一些比较难懂的部分，但是又引入了 Node.JS 的一些实现的方式，如 Node.JS File System 的一些实现的 API

### Q. Nginx NJS 与 Nginx Lua/OpenResty 的关系？

A.  其实两者没啥关系，但是都是将 Nginx 变为 Web 应用服务器的一种实现。两者的实现，除了编程语言的不同，其实很多的内部概念和实现都有着很大的相似性，理解其中一个对另外一个有一定的辅助作用。

### Q. Nginx NJS 和 Nginx Lua/OpenResty 哪个性能高？

A. 实话实说，我没有测试过，目前也没有测试的价值，因为 Nginx NJS 的代码尚未稳定，还在不断开发，但是从实现，Nginx Lua/OpenResty 的性能应该比 Nginx NJS 的性能要好，但是差距应该有限。

### Q. Nginx NJS 和 Nginx Lua/OpenResty 实现上有何差别？

A. Nginx Lua/OpenResty 是使用 patch nginx 来集成 nginx 的，这也是 Nginx Lua/OpenResty 不建议使用 Nginx 动态模块来加载 nginx-lua-module  的原因，并且，Nginx Lua/OpenResty 使用 luajit 来优化 lua 的代码性能，整个 nginx 共用/复用一个或者几个 lua vm 的实例，这个实现来说，Nginx Lua/OpenResty 的内存占用有赖于 lua vm 的 GC 的效率。

Nginx NJS 从一开始就是针对 Nginx 的环境进行设计的，理论上有着很多的性能优化空间。 Nginx NJS 被设计成了非常轻量化的实现，不论内存占用，还是启动时间，都很大的优势。 Nginx NJS 可以做到每处理一个请求，就创建一个对应的实例来执行代码，这样避免了 GC，内存的使用上比较有效率。

### Q. Nginx NJS 可以在生产环境使用吗？

A. 截至到版本 `0.2.3` (截至到 2018-08-18)，我不建议在生产环境里使用 Nginx NJS，我在测试中发现 NJS 的代码稍有问题，很容易导致 Nginx worker 直接 core dump，这在生产环境是无法想象的。 Nginx NJS 应用到生产环境中尚待时日，但可以作为关注点，长期关注。

如果你打算尝鲜，可以尝试在生产环境里面玩玩，毕竟玩玩嘛，乐趣最重要。

### Q. Nginx NJS 目前具备基本的 Web 应用的能力吗？

A. 基本具备！Nginx NJS 作为逻辑的实现，subrequest 配合 Nginx Redis module 和 Nginx MySQL module 就能实现基本的数据库访问，什么 session、cookie、template 的实现，借鉴庞大的 Node.JS 的生态，改动改动，适配一下，那都是分分钟的事情。

要注意一下的就是，如果实现 Nginx NJS 下的 JavaScript Template 的实现，JS 里常见的 `new Function()` 这样的模板实现不可行，因为 Nginx NJS 常未支持 `new Function()` 这个实现，要实现 JS 模板只能靠 `string.replace()` 重写一下，具体见 Google 的关键字 `js template without new function`

### Q. 如何 debug Nginx NJS 编写的代码？

A. 目前我知道的很有限，只能依靠 Nginx 的 error_log 当然了，你要上 gdb 我也没有阻止你。

最笨的办法，编程的时候可以开两个终端窗口，一个终端执行 `sudo tail -f /var/log/nginx/error.log` 另一个终端执行 `sudo nginx -t & sudo nginx -s reload` 有代码的变动就手动重启 Nginx 好了。

### Q. 如何实现 Nginx NJS 的网页 print 输出各个变量或者代码的值？

A. 其实很简单的，如果你懂得 JavaScript 的话，直接用 `JSON.stringify` 就可以了

简单来说，想要输出任意的 JavaScript 的值，方便调试或者查看的话，直接用以下代码

    // Nginx NJS print example
    function hello(r) {
        var _args = {}, arg;

        // req.args is readonly, just take a copy to self define object
        for (arg in req.args) {
            _args[arg] = req.args[arg];
        }

        r.return(200, JSON.stringify(_args)); // here use JSON.stringify to output any object of js.
    }

JavaScript 里的这个 `JSON.stringify` 简直是 JS 的神器，我觉得可能没有之一。只要你脑洞大开，能做的事情实在太多了。