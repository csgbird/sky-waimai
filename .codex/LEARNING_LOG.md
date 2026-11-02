# 苍穹外卖学习记录

用于记录学习进度、亲自完成的内容、遇到的问题和下一步计划。

## 2026-08-30：项目准备

### 已了解

- 课程共 12 天，覆盖环境搭建、基础数据维护、点餐业务和数据统计。
- 后端采用 Maven 多模块结构：`sky-common`、`sky-pojo`、`sky-server`。
- 主要技术栈包括 Java 8、Spring Boot 2.7.3、Spring MVC、MyBatis、MySQL、Druid、JWT 和 Knife4j；后续会使用 Redis、WebSocket、定时任务和 Apache POI。

### 操作记录

- 已检查课程资料目录和 day01 后端初始工程。
- 曾将课程初始代码复制到学习目录，随后按要求全部删除。
- 曾触发 Maven 构建，发现本机 Maven 默认编译目标为 Java 17，而当前安装的是 JDK 8；复制代码删除后，该临时 POM 修改也已一并删除。
- 当前学习项目保持为空，仅保留 `.codex` 学习档案。

### 当前约定

- 后续代码初始化、构建和启动由学习者亲自操作。
- Codex 主要负责讲解、答疑、代码审查和学习记录，除非学习者明确要求代为修改或执行。
- 当前项目以 Java 后端代码为主要学习内容。
- 管理端和微信小程序复用课程已有前端环境，不在当前项目中重复开发或初始化。

### 下一步

- 从 day01 开始，阅读项目概述并亲自导入后端初始工程。
- 配置 MySQL，创建 `sky_take_out` 数据库并导入 `sky.sql`。
- 理解三个 Maven 模块之间的依赖关系和职责。

## 2026-08-30：前端联调环境规划

- 确认课程前端运行环境包含已打包的管理端页面和 Windows 版 `nginx.exe`。
- 当前系统为 Apple Silicon macOS，课程提供的 Windows 可执行文件不能直接使用。
- 将管理端静态页面整理到当前仓库的 `frontend-runtime/`，不引入或学习前端源码。
- 使用 macOS 本机 Nginx 托管页面，并把 `/api/` 请求转发到 Java 后端的 `8080` 端口。
- 已通过 Homebrew 安装 macOS 版 Nginx 1.31.4，前端使用 `8081` 端口运行。
- 已验证 `http://localhost:8081` 返回 HTTP 200；Java 后端尚未运行时，页面接口请求会暂时失败，这是正常现象。
- 已将项目目录由带前导空格的 ` sky-waimai` 修正为 `sky-waimai`。

## 2026-08-30：JWT 专题学习

- 已学习 Token 与 JWT 的关系、JWT 的三段结构以及签名防篡改原理。
- 已结合员工登录理解 JWT 的生成、响应体返回、Cookie 保存、请求头携带和拦截器校验流程。
- 已在 `docs/learning/java/JWT教程.md` 建立项目内专题教程。
- 新建 `docs/learning/README.md` 作为后续 Java、MySQL、Spring、MyBatis、Redis 等学习笔记的统一入口。
