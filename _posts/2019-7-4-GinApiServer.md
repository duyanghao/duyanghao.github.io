---
layout: post
title: GinApiServer Framework
date: 2019-7-4 16:33:31
category: 技术
tags: 后台开发
excerpt: GinApiServer框架介绍……
---

## 简介

`GinApiServer`是一个基于[gin](https://github.com/gin-gonic/gin)框架写的ApiServer框架，主要用于企业生产环境中的快速开发

## 组成

`GinApiServer`框架的核心就是pkg包，下面主要针对改包结构进行描述：

```bash
pkg/
├── config
│   ├── config.go
│   ├── key.go
│   ├── model.go
│   └── opt_defs.go
├── controller
│   ├── ping.go
│   ├── todo.go
│   └── version.go
├── log
│   └── log.go
├── middleware
│   ├── basic_auth_middleware.go
├── models
│   └── common.go
├── route
│   └── routes.go
├── service
│   └── todo.go
├── store
└── util
```

* config：主要用于配置文件，实现：文件+环境变量+命令行参数读取
* controller: 对应MVC中Controller，调用service中的接口进行处理，自己只拼接数据
* service: 负责主要的逻辑实现
* log: 日志模块，实现：模块名(文件名)+函数名+行数+日志级别
* middleware: 中间件，负责通用的处理，例如：鉴权
* models: 对应MVC中的model
* route: gin路由
* store: 存储模块，可以添加MySQL、Redis等
* util: 通用的库函数

## 使用

实际使用中，通常需要将`GinApiServer`替换成业务需要的后台server名称，可以执行如下命令：

```bash
grep -rl GinApiServer . | xargs sed -i 's/GinApiServer/youapiserver/g' 
```

接下来运行脚本启动服务：

```bash
bash start.sh
```

## 构建镜像

执行如下命令可以自动构建镜像：

```bash
make dockerfiles.build
```

## 特性

* 1、支持configmap reload api
* 2、支持ping-pong健康检查
* 3、支持版本获取