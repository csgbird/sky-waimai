# 管理端前端运行环境

这里保存课程已经打包好的管理端页面，只用于和 Java 后端联调，不作为前端学习源码。

## 为什么不使用资料中的 `nginx.exe`

课程提供的是 Windows 32 位 Nginx，不能在当前 macOS（Apple Silicon）上执行。因此这里只保留静态页面，并使用本机安装的 Nginx 托管。

## 安装和启动

首次使用时安装 Nginx：

```bash
brew install nginx
```

在项目根目录启动：

```bash
nginx -p "$PWD/frontend-runtime/" -c conf/nginx.conf
```

打开：<http://localhost:8081>

管理端的 `/api/` 请求会转发到 `http://127.0.0.1:8080/admin/`，因此联调时 Java 后端需要监听 8080 端口。

前端使用 8081 端口，是为了避开 macOS 的特权端口 80，同时避免与后端 8080 冲突。

## 停止与重载

```bash
nginx -p "$PWD/frontend-runtime/" -c conf/nginx.conf -s stop
nginx -p "$PWD/frontend-runtime/" -c conf/nginx.conf -s reload
```

配置检查：

```bash
nginx -p "$PWD/frontend-runtime/" -c conf/nginx.conf -t
```

命令保留引号，可以避免路径以后出现空格或特殊字符时执行失败。
