# 在 Ubuntu 16.04 LTS 安装 WordPress 全程记录

## 安装 MySQL 并配置

```
apt-get install mysql-server mysql-client
```

使用以下命令访问MySQL服务控制台。

```
mysql -u root -p
```

在 MySQL 控制台上，您需要执行以下任务：

- 创建名为 mndatabase 的数据库，注意字符串使用 `utf8mb4`不然没法支持 emoji 表情编码；
- 创建名为 mnuser/mnpassword 的 MySQL 用户帐户；
- 完全控制 mndatabase 数据库到 mnuser 用户；

```
CREATE DATABASE mndatabase DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'mnuser'@'%' IDENTIFIED BY 'mnpassword';
GRANT ALL PRIVILEGES ON mndatabase.* TO 'mnuser'@'%';
FLUSH PRIVILEGES;
quit;
```



## 安装 PHP 并配置

```
sudo apt-get install php-fpm php-mysql
```

因为ubuntu16.04 只支持 php7，所以安装 php7，上面命令直接安装就是 PHP7.0

上面只是最小的 PHP 配置，还需要安装 WordPress 相关的 PHP 插件

```
sudo apt-get install php-curl php-gd php-mbstring php-mcrypt php-xml php-xmlrpc
```

配置 PHP FPM 的文件消除相似路径漏洞

```
sudo vim /etc/php/7.0/fpm/php.ini
```

搜索文件里面的 `cgi.fix_pathinfo` 去掉其分号注释，并将其设置为 `0`

```
cgi.fix_pathinfo=0
```

然后重启 PHP

```
sudo service php7.0-fpm restart
```



## 安装 Nginx 并配置

```
sudo apt-get install nginx
```

直接打开看到 `Welcome to nginx!` 即时安装成功

编辑 `sudo vim /etc/nginx/sites-available/default` 默认站点文件，增加配置

```
server {

    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;

    # 这一行中添加了index.php
    index index.php index.html index.htm index.nginx-debian.html;

    # 此处修改你的域名，如果没有则不需要修改源文件
    server_name server_domain_or_IP;

    location / {
        try_files $uri $uri/ =404;
    }

    # 以下所有内容需要添加，其实就是去掉原来的注释即可
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.0-fpm.sock;
    }

    # 防止暴露 Apache .htaccess 文件而特别增加的规则
    location ~ /\.ht {
        deny all;
    }

}
```

执行以下命令检查 Nginx 的配置文件的正确性，并重启 Nginx 系统服务

```
sudo nginx -t
sudo service nginx reload
```

可以新建以下文件进行测试

```
sudo vim /var/www/html/index.php

# 文件内容如下：

<?php
	phpinfo();
	
```

记得测试完删除文件！！！

## 安装 WordPress 

### 下载 WordPress 并整理其结构

其实直接下载可以了，没有什么技术含量的，建议直接下载中文版本，免得后面还继续找中文语言包

```
# 临时目录下面操作
cd /tmp

# 下载最新的代码包
curl -O https://cn.wordpress.org/latest-zh_CN.tar.gz

# 解压文件
tar xzvf latest-zh_CN.tar.gz

# 重命名 wp-config-sample.php 文件为正常的 wp-config.php 文件方便配置
mv /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php

# 提前建立 upgrade 文件夹，以便WordPress在更新其软件后尝试自行执行此操作时不会遇到权限问题
mkdir /tmp/wordpress/wp-content/upgrade

# 整体拷贝到站点目录，我们使用该-a 标志来确保维护我们的权限。
# 我们在源目录的末尾使用一个点来表示应该复制目录中的所有内容，包括任何隐藏文件
sudo cp -a /tmp/wordpress/. /var/www/html
```

### 配置 WordPress 目录权限

我们需要完成的一件大事是建立合理的文件权限和所有权。我们需要能够以普通用户的身份写入这些文件，并且我们需要 Web 服务器也能够访问和调整某些文件和目录才能正常运行。

我们首先将文档根目录中的所有文件的所有权分配给用户名。我们将 root 在本指南中使用我们的用户名，但您应该更改此选项以匹配您的 sudo 用户调用。我们会将群组所有权分配给 www-data 群组

其实 `www-data` 就是 Nginx Worker 默认使用的用户！！！

```
# 将整个文件夹设置为 www-data 只读
sudo chown -R root:www-data /var/www/html

# 配置 setgit 位
# setgid 它使得在所述目录内创建的文件，不属于创建者所属的组，而是属于父目录所属的组。
sudo find /var/www/html -type d -exec chmod g+s {} \;

# 为 wp-content 目录提供组写访问权限，以便Web界面可以进行主题和插件更改
sudo chmod g+w /var/www/html/wp-content

# 为Web服务器提供对这两个目录中所有内容的写访问权限
# 其实完全配置好的站点，长期运行的话，此项可以不执行，或者在需要的修改主题和插件的时候再执行！！！
sudo chmod -R g+w /var/www/html/wp-content/themes
sudo chmod -R g+w /var/www/html/wp-content/plugins
```

### 修改 WordPress 自身配置

其实就是修改 `/var/www/html/wp-config.php`  文件中关于 MySQL 数据库的连接信息

```
# 修改数据库连接配置
# 直接使用 root 权限来修改，因为此文件是只读权限的！
sudo vi /var/www/html/wp-config.php
```

里面除了数据库连接的信息修改，还有部分细节需要注意

首先是修改部分的加密 KEY salt

```
define('AUTH_KEY',         'put your unique phrase here');
define('SECURE_AUTH_KEY',  'put your unique phrase here');
define('LOGGED_IN_KEY',    'put your unique phrase here');
define('NONCE_KEY',        'put your unique phrase here');
define('AUTH_SALT',        'put your unique phrase here');
define('SECURE_AUTH_SALT', 'put your unique phrase here');
define('LOGGED_IN_SALT',   'put your unique phrase here');
define('NONCE_SALT',       'put your unique phrase here');
```

可以使用 WordPress API 来直接生成各种样式的 KEY salt，每次请求一次就生成上面的一整段

```
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

直接使用鼠标复制粘贴足够

记得修改默认的 Wordpress 数据库数据表前缀 `WP_` 随便改一个都比默认的好！

```
$table_prefix = 'mn_';
```

然后就可以打开 Web 界面来配置整个 WordPress 啦！

注意默认用户不要配置为常见的 admin 或者 root 这样容易猜解的用户名

### 升级 WordPress 全站

更新的时候由于权限问题可能无法更新。当要升级全站 WordPress 程序时候，记得修改整个文件夹的目录权限

升级前，去除只读权限

```
sudo chown -R www-data /var/www/html
```

升级后，恢复只读权限，锁定权限以确保安全性

```
sudo chown -R sammy /var/www/html
```



## Nginx 下的 WordPress 静态化

Nginx 下面实现 WordPress 静态化很简单，对比上面的 Nginx 的规则，只需加入下面的规则即可

```
server {

    listen 80 default_server;
    listen [::]:80 default_server;
    
    # 网站对应的根目录
    root /var/www/html;

    # 这一行中添加了index.php
    index index.php index.html index.htm index.nginx-debian.html;

    # 此处修改你的域名，如果没有则不需要修改源文件
    server_name server_domain_or_IP;

	# 注意：这里的规则跟前面的 Nginx 默认配置的规则不同
	# 这里已经针对 index.php 进行静态化改写
    location / {
        try_files $uri $uri/ /index.php?$args; 
    }

    # 以下所有内容需要添加，其实就是去掉原来的注释即可
    location ~ \.php$ {
    	# 必须在 php.ini 里面配置 "cgi.fix_pathinfo = 0;"
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.0-fpm.sock;
    }

    # 防止暴露 Apache .htaccess 文件而特别增加的规则
    location ~ /\.ht {
        deny all;
    }

}
```



`try_files $uri $uri/ /index.php?$args;` 已经针对 `wp-admin` 部分需要单独的规则来实现其静态化，因为 `$uri/` 在访问 `/wp-admin` 的时候，会重定向到 `/wp-admin/` 从而关联访问到 `/wp-admin/index.php`

## WordPress 自身安全加固

### 隐藏 WordPress 版本信息

```
function wpbeginner_remove_version(){
　　return '';
}
add_filter('the_generator','wpbeginner_remove_version');
```

### 使用口令保护后台登陆链接

```
function login_protection(){
　　if($_GET['word']!='press') {
　　    header('Location:http://www.baidu.com/');
　　}
}
add_action('login_enqueue_scripts','login_protection');
```

这样一来，后台登录的唯一地址就是 http://yoursite/wp-login.php?word=press，如果不是这个地址，就会自动跳转到 http://www.baidu.com/ 

### 避免显示 WordPress 用户名

想要知道WordPress的管理员用户名？很简单，只要在网站的域名后面加 `/?author=1` 就行了。

如果 `/?author=1` 显示404界面，那很可能是以前有过 admin 用户，后来站长发现用默认帐户 admin 太不安全了，就新建了一个管理员帐户，并删除了 admin 帐户。这种情况下，用 `/?author=2` 就能显示出用户名了。如果使用 admin 帐户，确实不安全，但是如果你的博客使用一个复杂的用户名，却经不起这么简单的一个URL 的考验，这和使用 `admin` 帐户没有根本上的区别。既然存在漏洞，那么就要去填补它。要填补这个漏洞，倒还真的不是什么难事。我的思路就是，只要访问主页 url 后头有 `author`参数就让他跳到主页

将下面的代码添加到当前主题的`functions.php`文件

```
function hidden_author_link() {
    return home_url('/');
}
add_filter('author_link', 'hidden_author_link');
```

其实更好的办法是强制设置所有的用户只显示用户的昵称，禁止显示其默认的用户名！！！

## Nginx 针对 WordPress 安全加固

增加部分 Nginx 规则控制 WordPress 的安全性

### 禁止访问 WP 安装接口

WordPress 在 Web 界面安装完成之后，其安装文件是完全多余的，然而，WP 每次更新后都会自动带上这货色，所以最好的方法就是直接在 Nginx 中禁止访问

```
location ~ ^/wp-admin/install\.php {
	deny all;
	log_not_found off;
	access_log off;
}
```



### 修改登录入口

如果项目有确定的登录地址，那么可以隐藏掉 WordPress 默认的登录入口地址。

WP 的默认登录文件地址必须在 Nginx 里重定向，因为每次更新 WordPress，安装包都会重写这个地址，所以相对来说，直接在 Nginx 中禁用并 rewrite 新的后台登录入口是极其优秀的做法。

```
# 默认登录入口
location ~* /wp-login.php$ {
    allow 127.0.0.1;
    deny all;
}
# 默认注册入口
location ~* /wp-signup.php$ {
    allow 127.0.0.1;
    deny all;
}
# 新的登录入口
location ~* /YOUR-NEW-LOGIN-ADDR$ {
    
}

```

### 后台管理限制访问 IP 地址/启用 BASIC AUTH

后台管理文件夹 `/wp-admin/` 其实加上密码限制访问最好的，然而很多第三方实施的项目难以保证再次输入密码，很是尴尬，但是个人或者公司类的项目可以考虑。

```
location /wp-admin {
    allow 202.202.202.0/24; # 国内 IP地址段1
    allow 201.201.201.0/24; # 国内 IP地址段2，以此类推，将所有国内的IP地址加入
    deny all;
}
```



### 禁止外部访问无关的文件展示

默认安装的 WordPress 其根目录下有很多外部访问无需知道的文件，比如什么 `wp-config.php` 或者 `wp-load.php` 这样的文件，外部访问应该一律禁止

```
# 用户注册邮件激活账号需要用到，如果没有用户注册，请直接封禁这个文件的访问
location ~* /wp-activate.php$ {
    allow 127.0.0.1;
    deny all;
}
location ~* /wp-blog-header.php$ {
    allow 127.0.0.1;
    deny all;
}
# 调用评论需要用到，如果全站禁止评论，可以直接封禁这个文件的访问
location ~* /wp-comments-post.php$ {
    allow 127.0.0.1;
    deny all;
}
location ~* /wp-config.php$ {
    allow 127.0.0.1;
    deny all;
}
# WP 定时任务，自己在系统 crontab 加入定时任务，该接口允许本机访问即可
location ~* /wp-cron.php$ {
    allow 127.0.0.1;
    allow 123.123.234.234; # 本机外网 IP
    deny all;
}
# 用于生成 OPML 输出，一般人用不到，直接禁止访问即可
location ~* /wp-links-opml.php$ {
    allow 127.0.0.1;
    deny all;
}
location ~* /wp-load.php$ {
    allow 127.0.0.1;
    deny all;
}
# 用邮件来写博客，一般人用不到，直接禁止访问
location ~* /wp-mail.php$ {
    allow 127.0.0.1;
    deny all;
}
location ~* /wp-settings.php$ {
    allow 127.0.0.1;
    deny all;
}
```



### 禁止其他无关文件的展示

默认安装的 WordPress 其根目录下有很多无关的文件，比如什么 `readme.html` 这对于程序运行不是必要的文件

```
location ~* /license.txt$ {
    allow 127.0.0.1;
    deny all;
}
location ~* /readme.html$ {
    allow 172.0.1.1;
    deny all;
}
```

### 限制访问 XMLRPC 和 Traceback

WordPress中的 XMLRPC 端点（根目录下的 `xmlrpc.php`文件）用于允许外部应用程序与 WordPress 数据交互。例如，它可以允许添加、创建或删除文章。但是，XMLRPC 也是一种常见的攻击媒介，攻击者可以在未经授权的情况下执行这些操作。所以最好允许从您信任的授权IP请求XMLRPC，如下所示

```
location ~* /xmlrpc.php$ {
    allow 172.0.1.1;
    deny all;
}
location ~* /wp-trackback.php$ {
    allow 172.0.1.1;
    deny all;
}
```

添加上述内容后，应该在浏览器中访问 `xmlrpc.php` 时会看到403错误响应代码。

### 限制请求类型

大多数情况下，您的网站可能只执行两种类型的请求 `GET` 或者 `POST`

所以，只允许我们的网站执行这两种请求类型，也是增强安全性的做法。

```
if ($request_method !~ ^(GET|POST)$ ) {
    return 444;
}
```

### 禁止直接访问PHP文件

在神不知鬼不觉的情况下，黑客可能会将PHP文件上传到你的服务器中，然后通过访问该恶意文件执行某些操作，即可在你的网站上创建后门，所以在这些目录中我们**应该禁止直接访问任何php文件**，或者任何 PHP 相关的文件

```
location ~* /(?:uploads|files|wp-content|wp-includes|akismet)/.*.php$ {
    deny all;
    access_log off;
    log_not_found off;
}
```

### 禁止访问某些敏感文件

和PHP文件相似，以点开头的文件，比如 `.htaccess`、`.user.ini`以及`.git`可能包含敏感信息。为了更安全，最好**禁用对这些文件的直接访问**。比较容易忽略的是 `vi` 编辑后留下的缓存文件 `.swp`

```
location ~ /\.(svn|git)/* {
    deny all;
    access_log off;
    log_not_found off;
}
location ~ /\.ht {
    deny all;
    access_log off;
    log_not_found off;
}
location ~ /\.user.ini { 
    deny all; 
    access_log off;
    log_not_found off;
}
location ~ /*\.swp { 
    deny all; 
    access_log off;
    log_not_found off;
}
# SQL文件
location ~ /*\.sql { 
    deny all; 
    access_log off;
    log_not_found off;
}
# 防止Web目录中的敏感文件被下载，一句话搞定，不怕死的直接上！
location ~* \.(rar|zip|gz|tar|tgz|tar.gz|7z|z|bz2|tar.bz2|sql|log|ini|bak|old|conf|idea|DS_Store|swp|svn/entries|git/config)$ {
	deny all;
}
```

### 隐藏Nginx和PHP版本

最好不要对外公开Nginx以及PHP版本，如果特定的Ningx或PHP版本暴露出漏洞，攻击者又发现你的服务器上的存在对应的漏洞版本，那可能就很危险了。以下规则可以隐藏Nginx和PHP版本

```
#隐藏 nginx 版本.
server_tokens off;

#隐藏 PHP 版本
fastcgi_hide_header X-Powered-By;
proxy_hide_header X-Powered-By;
```

### 安全标头

安全标头（ header ）通过指示浏览器行为提供额外的安全层。例如，`X-Frame-Options`，可以防止你的网站被嵌入到iframe框架中进行加载。而`Strict-Transport-Security`会**让浏览器采用HTTPS方式加载站点**。

```
add_header X-Frame-Options SAMEORIGIN;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";

# 没有启用 HTTPS 的站点不能随意添加
add_header Strict-Transport-Security "max-age=31536000";

```

### 限制请求

WordPress登录页面`wp-login.php`是暴力攻击的常见端点。**攻击者会尝试通过批量提交用户名和密码组合进行登录尝试，可能无法破解你的密码，但是对服务器资源占用非常大，可能会导致网站无法访问**。

为此，我们可以应用一个规则来限制页面每秒可以处理的请求数。这里我们**将限制设置为每分钟3个请求**，超过次数的请求将被阻止。

```
limit_req_zone $binary_remote_addr zone=WPRATELIMIT:10m rate=3r/m;
location ~ \wp-login.php$ {
    limit_req zone=WPRATELIMIT;
}
```

### 显式禁用目录列表

最后一旦也非常重要，你应该禁用目录列表，以便攻击者无法知道目录中的内容

```
autoindex off;

```

## PHP 内核日常加固

针对 PHP 部分的日常加固

## MySQL 数据库日常加固

针对 MySQL 数据库日常加固
