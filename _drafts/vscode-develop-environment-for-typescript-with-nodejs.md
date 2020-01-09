# 配置 VSCode 实现 TypeScript/Nodejs 开发环境全程记录

为了方便记录 VSCode 下配置 TypeScript/Nodejs 开发环境，特撰写此文档记录

## 环境准备

这里只记录 Windows7/Windows10 下面的开发环境搭建过程，其他系统请自行搜索实现

## 安装并配置 Nodejs 环境

### 下载 Nodejs 并安装

打开 [Nodejs](https://nodejs.org) 网站即可下载其安装包，选择其 `Long Term Support(LTS)` 版本下载即可。下载回来直接双击其可执行文件进行安装。由于考虑到 Nodejs 默认安装路径 `C:\Program Files\nodejs\` 对于部分机器，尤其是公司开发用机器，不一定具备修改 `C:\Program Files\` 文件夹的权限，这里统一按照单独的其他安装目录进行说明

```
  // Nodejs 安装目录，安装到其他目录的请自行修改对应
  D:\Nodejs\
  
```

等 Nodejs 的安装进度条跑完即可。随意打开 Windows 的命令提示符，输入 `node -v` 和 `npm -v` 能看到对应的版本号即代表安装成功。

### 配置 Nodejs 环境变量

首先进入到 Nodejs 安装目录 `D:\Nodejs\` 新建两个文件夹 `node_cache` 和 `node_global` ，前者用于 nodejs 的安装缓存，后者是 nodejs 全局安装下的模块存放文件夹，然后随意打开 Windows 的命令提示符，执行以下命令

```
  npm config set cache "D:\Nodejs\node_cache"
  npm config set prefix "D:\Nodejs\node_global"
```

如果公司网络需要设置 `proxy` 环境，那么可以这样设置 `npm` 通过代理进行下载 nodejs module

```
  // http://localhost:8080 是网络对应的代理 IP 地址
  npm config set proxy "http://localhost:8080"
```

如果觉得官方的 `npm registry` 下载太慢，那么可以使用淘宝的 `npm` 镜像服务

```
  npm config set registry https://registry.npm.taobao.org
```

以上所有的配置可以使用命令 `npm config list` 来重新检查一遍



