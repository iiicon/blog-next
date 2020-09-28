---
layout: post
title: nodejs基础
date: 2020-09-27 17:22:18
tags: [js, node]
categories: nodejs
---

## 介绍

Node.js 是一个 Javscript 运行环境(runtime)。它让 JavScript 可以开发后端程序，
它几乎能实现其他后端语言能实现的所有功能。

Nodejs 是基于 Gogle V8 引擎，V8 引擎是 Gogle 发布的一款开源的 JavScript 引擎，
原来主要用于 Chrome 浏览器的 JS 解释部分，但是 Ryan Dahl 这哥们，鬼才般的，把这个 V8
引擎搬到了服务器上，用于做服务器的软件。

<!-- more -->

## nodejs 一些优势

1. Nodejs 用户量大
2. Nodejs 是程序员必备技能
3. Nodejs 最擅长高并发
4. Nodejs 简单
5. Nodejs 可实现的功能多：Nodejs 不仅可以像其他后端语言一样写动态网站、写接口，
   还可以应用在云计算平台、游戏开发、区块链开发、即时通讯、跨平台 Ap 开发、桌面应用
   开发（elctron）、云直播、物联网领域等


## 模块

一类是 Node 提供的模块，称为核心模块，另一类是用户编写的模块，称为文件模块

遵照 commonJS 规范, module.exports 导出默认对象，exports.xx 导出对象 {xx}

nodejs 的第三方模块由包组成，用包来管理有依赖关系的一些模块

包目录：
```
package.json :包描述文件。
bin :用于存放可执行二进制文件的目录。
lib :用于存放 JavScript 代码的目录。
doc :用于存放文档的目录。
```

- http
- url

## 问题

- dependence
