---
layout: post
title: "如何开始配置你的 Linux Server"
description: "如何开始配置你的 Linux Server：记录一些自己在初始化配置自己的 Linux Server 上的一些心得和小技巧"
date: "2017-10-05 23:45:23"
categories:
    - Weblogs
tags:
    - Linux
    - Weblogs
---

* toc
{:toc}



面对一个全新的 Linux Server，新手该如何快速配置快速可用是一个 Linux 初学者面对的问题。网络上有很多的乱七八糟的新手配置的教程，一上来就是各种 iptables、sshd_config 的配置，往往让人不知道所以然，而很多基础的配置，比如 Linux 的 resolv.conf 作为 Linux 基础的 DNS 配置，一旦没有弄好往往连域名都解析不了，这通常就会让初学者一头雾水不知道所以然。

我也是一路过来的菜鸟，对，至今我都仍然认为自己是个菜鸟，很多不懂不会的东西遇到了或者顺便看到，于是就去 Google 自己学习，觉得自己也该写一个属于自己总结的小教程，也不为了帮助谁，而是，为了记录一下自己的一些心得体会。好记性不如烂笔头嘛，写下来方便查找，也方便梳理自己的思路。

**注意**：以下这些 Shell 操作均是在 Ubuntu(Debian) 下面完成的，其他 Linux 发行版均类似，需要的话请自行学习！

**注意**：以下这些命令的运行，没有特别说明，均是在 root 权限下运行！

## 安装所有的软件更新

首先安装所有的软件更新，尤其是操作系统对应的安全更新

    # ubuntu 14.04
    apt-get update && apt-get upgrade


## 设置 hostname

设置 hostname 是为了方便记录服务器的名字和用途，很简单，一句话的命令就搞定啦！

    # ubuntu 14.04
    echo "example_hostname" > /etc/hostname
    hostname -F /etc/hostname

顺便说，设置了新的 hostname 请自己去 `/etc/hosts` 中检查有没有到新的 hostname 对应的 IP 的映射，比如这样：

    127.0.0.1           example_hostname
    YOUR_IPv4_ADDRESS   example_hostname

### 设置 hostname 的一些小经验

为自己的 Linux Server 设置一个名字，这个名字一般来说随自己喜欢的来设置，但是我个人推荐使用以下方式来命名，以方便你自己使用

    [服务器用途]-[服务器机房位置]-[服务器提供商的名称]

比如说，我有一台亚马逊 AWS 的 N.California 用于网站应用的服务器，那么可以这样命名：`web-app-cal-us-aws`

这些规则都是个人方便使用，不必强行按照我的方式来，你按照自己习惯和喜欢的最好！

#### 服务器的用途

服务器的用途有很多啊，比如说应用服务器、数据库服务器、队列服务器、负载均衡服务器等，大约按照以下方法来:

* app 应用服务器，也可以自己扩展为 web-app，app-container 等
* cache 缓存服务器
* db 数据库服务器，有 db-master，db-slave 等
* lb 负载均衡
* task 队列服务器

#### 服务器机房位置

当出现区域性的网络故障的时候，用服务器机房位置来定位网络故障是挺好的做法。这个位置可以自己随意啊，比如按照飞机场的编号来命名，比如广州某电信数据中心服务器，可以自己用 tel-can 这样自己一眼就能知道哪里的服务器啦！

#### 服务器提供商的名称

通常都是使用云服务商的主机的，比如上面的 aws


## 设置时区和 NTP 对时

默认情况下，几乎多数的服务器镜像都会默认设置为 UTC 时间（又称格林威治时间），如果你需要本地化你的时间，请务必自行设置你自己的时区。

    # ubuntu 14.04
    dpkg-reconfigure tzdata

## 增加自己常用的用户

服务器上最好不要使用 `root` 来管理服务器，使用一个普通用户，然后 `sudo` 到  `root` 来临时获得高级的权限，是比较安全的做法。

    # ubuntu 14.04
    adduser example_user
    adduser example_user sudo

## 配置 SSH

关于 SSH 的配置，都可以写出一本书来了，不过个人使用，不必要求太多，够用就好。

### 生成你的 RSA key-pair

首先在你的本地的电脑上生成你的 4096-bit RSA key-pair，这个 key-pair 有两个文件：`id_rsa` 和 `id_rsa.pub`

* `id_rsa` 这个是你的私钥，放在你的本地电脑上，千万不要公开出去；
* `id_rsa.pub` 这个是你的公钥，放在你的服务器上，或者你要连接的设备上；

**注意**：以上这两个密钥文件，无论在什么地方，都必须保证其 `chmod 600` 的权限！防止被别人读取！请务必妥善保存好你的密钥文件！

在本地的 Linux 上可以使用以下命令生成 4096-bit RSA key-pair：

    ssh-keygen -b 4096

**注意**：上面生成 key-pair 命令会清空你的 `~/.ssh/id_rsa*` 请运行上面命令前，请你务必清楚并备份你先前的 `~/.ssh/id_rsa*`

### 复制你的 authorized_keys

将上面的公钥 `id_rsa.pub` 通过各种渠道复制到你的服务器上去。

在 Linux 里面，你可以直接使用以下命令即可：

    ssh-copy-id example_user@YOUR_SERVER_IP

如果你使用 windows，那么可以使用 WinSCP 复制到服务器上 `/home/example_user/id_rsa.pub`，然后等同执行上面的命令：

    # ubuntu 14.04
    mkdir ~/.ssh; mv ~/id_rsa.pub ~/.ssh/authorized_keys
    sudo chmod 700 -R ~/.ssh && chmod 600 ~/.ssh/authorized_keys

### 配置好 sshd_config

在 Ubuntu 中，`sshd` 作为 SSH 的守护进程，其配置文件位于 `/etc/ssh/sshd_config` 其配置将直接影响到 SSH 的安全性。

    # ubuntu 14.04
    # vi /etc/ssh/sshd_config

    # listen only on IPv4
    AddressFamily inet

    # Disallow root logins over SSH
    PermitRootLogin no

    # Disable SSH password authentication
    PasswordAuthentication no

SSH 的安全至关重要，请务必保证至少具备上面的几条配置！


## 配置 iptables

Linux kernel 自带了一个 netfilter 模块作为内核级别的网络流量控制和过滤的实现，简单来说就是内核自建的防火墙。而 iptables 就是配置 netfilter 的命令行工具，由于几乎每个 Linux 发行版本都自带，作为用户的命令行界面， iptables 也就经常用于代指 Linux 的内核级防火墙。由于  IPv4 和 IPv6 的不同，iptables 用于 IPv4，而 ip6tables 用于 IPv6。

iptables 可以检测、修改、转发、重定向和丢弃 IPv4 数据包。过滤 IPv4 数据包的代码已经内置于内核中，并且按照不同的目的被组织成 表 的集合。表 由一组预先定义的 链 组成，链 包含遍历顺序规则。每一条规则包含一个谓词的潜在匹配和相应的动作（称为 目标），如果谓词为真，该动作会被执行。也就是说条件匹配。iptables 是用户工具，允许用户使用 链 和 规则。很多新手面对复杂的 linux IP 路由时总是感到气馁，但是，实际上最常用的一些应用案例（NAT 或者基本的网络防火墙）并不是很复杂。

更多的 iptables 知识需要自行去了解。

### iptables IPv4 配置

此处简单给出 IPv4 的 iptables 的示例配置，以方便参考：

    $ sudo cat /etc/iptables/iptables.v4.rules
    *filter

    # Allow all loopback (lo0) traffic and reject traffic
    # to localhost that does not originate from lo0.
    -A INPUT -i lo -j ACCEPT
    -A INPUT ! -i lo -s 127.0.0.0/8 -j REJECT

    # Allow ping.
    -A INPUT -p icmp -m state --state NEW --icmp-type 8 -j ACCEPT

    # Allow SSH connections.
    -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

    # Allow HTTP and HTTPS connections from anywhere
    # (the normal ports for web servers).
    -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
    -A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT

    # Allow inbound traffic from established connections.
    # This includes ICMP error returns.
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    # Log what was incoming but denied (optional but useful).
    -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables_INPUT_denied: " --log-level 7

    # Reject all other inbound.
    -A INPUT -j REJECT

    # Log any traffic that was sent to you
    # for forwarding (optional but useful).
    -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "iptables_FORWARD_denied: " --log-level 7

    # Reject all traffic forwarding.
    -A FORWARD -j REJECT

    COMMIT

**注意**：上面这个 IPv4 的配置仅仅是 Web 服务器常用的配置，开通 PING、22 端口（SSH）、80 端口（HTTP）、443 端口（HTTPS），如果需要增加其配置，请自行增加

### iptables IPv6 配置

此处简单给出 IPv6 的 iptables 的示例配置，考虑到 IPv6 的网络越来越多，其安全问题不容忽视，配置网络的时候应尤其注意。

    $ sudo cat /etc/iptables/iptables.v6.rules
    *filter

    # Allow all loopback (lo0) traffic and reject traffic
    # to localhost that does not originate from lo0.
    -A INPUT -i lo -j ACCEPT
    -A INPUT ! -i lo -s ::1/128 -j REJECT

    # Allow ICMP
    -A INPUT -p icmpv6 -j ACCEPT

    # Allow HTTP and HTTPS connections from anywhere
    # (the normal ports for web servers).
    -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
    -A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT

    # Allow inbound traffic from established connections.
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    # Log what was incoming but denied (optional but useful).
    -A INPUT -m limit --limit 5/min -j LOG --log-prefix "ip6tables_INPUT_denied: " --log-level 7

    # Reject all other inbound.
    -A INPUT -j REJECT

    # Log any traffic that was sent to you
    # for forwarding (optional but useful).
    -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "ip6tables_FORWARD_denied: " --log-level 7

    # Reject all traffic forwarding.
    -A FORWARD -j REJECT

    COMMIT

**注意**：这里是 IPv6 的配置，跟 IPv4 的区别就在于 `icmpv6` 和 `loopback` 的地址啦！

合理配置 iptables 可以有效避免主机的端口外漏，尤其是一些 `bind` 到 `0.0.0.0` 全局地址的应用。对于 MySQL、Redis 等应用开放的端口，要严格将其 `bind 127.0.0.1` 如果需要外联，则必须在 iptables 里面严格控制访问主机的 IP 地址，以策安全。


## 其他配置

其他的配置，零零碎碎的配置归纳在这里，有些是有关安全的，有些是有关工作的便捷性的，统统归纳在这里。

### Ubuntu DNS 配置

Ubuntu 的 resolv.conf 作为 Linux 基础的 DNS 配置，很多人会直接往里面写 `dns-server` 然而重启后，这些配置信息就丢失了。其实 `/etc/resolv.conf` 里面，人家文件里都写得清楚明白了：

    $ cat /etc/resolv.conf 
    # Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
    #     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN

那么如果需要修改 `/etc/resolv.conf` 的配置的话，可以这样改：

    $ sudo vim /etc/resolvconf/resolv.conf.d/base

    nameserver 8.8.8.8
    nameserver 8.8.4.4

    $ sudo resolvconf -u 

注意最后记得重启一下 `resolvconf` 使得配置生效。其实如果你愿意深入研究一下 `man resolvconf` 你会发现其实 `/etc/resolvconf/resolv.conf.d/` 下面还有其他配置文件的，那么，为什么只编辑这个 `/etc/resolvconf/resolv.conf.d/base` 来加入 DNS 的配置信息呢？其实帮助文件里面已经写得很明白了，我摘录下来吧：

    /etc/resolvconf/resolv.conf.d/base
          File  containing  basic  resolver  information.  The lines in this 
          file are included in the resolver configuration file even when no
          interfaces are configured.

    /etc/resolvconf/resolv.conf.d/head
          File to be prepended to the dynamically generated resolver 
          configuration file.  Normally this is just a comment line.

    /etc/resolvconf/resolv.conf.d/tail
          File to be appended to the dynamically generated resolver 
          configuration file.  To append nothing, make this  an  empty  
          file.   This file is a good place to put a resolver options line 
          if one is needed, e.g.,

其实 `/etc/resolvconf/resolv.conf.d/head` 这个文件里面也有这么一句，你看一眼就知道什么回事啦！

    $ cat /etc/resolvconf/resolv.conf.d/head
    # Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
    #     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN

自己研究一下就知道 Ubuntu DNS 的配置是什么回事了～

