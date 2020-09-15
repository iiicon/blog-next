---
title: js正则
date: 2020-03-06 16:56:13
tags: [RegExp]
categories: js
comments: false
---

## 前言

最近系统性地过了一遍 js 正则，参考[《JavaScript 正则迷你书》](https://juejin.im/post/59cc61176fb9a00a437b290b),
很多人说，正则是反应程序员水平的侧面标准，这本书知识点非常到位，把核心的东西都梳理得很好

- [正则表达式字符匹配](https://github.com/iiicon/RegExp/blob/master/reg1.js)
- [正则表达式匹配位置](https://github.com/iiicon/RegExp/blob/master/reg2.js)
- [正则表达式括号的应用](https://github.com/iiicon/RegExp/blob/master/reg3.js)
- [正则表达式回溯法原理](https://github.com/iiicon/RegExp/blob/master/reg4.js)
- [正则表达式的拆分](https://github.com/iiicon/RegExp/blob/master/reg5.js)
- [正则表达式的构建](https://github.com/iiicon/RegExp/blob/master/reg6.js)
- [正则表达式的编程](https://github.com/iiicon/RegExp/blob/master/reg7.js)

## 实例

### 匹配 16 进制颜色值

    const reg = /#([0-9a-fA-F]{6})|([0-9a-fA-F]{3})/g;

### 匹配时间 hh:mm

    const reg = /^[01][0-9]|[2][0-3]:[0-5][0-9]$/g;

### 匹配日期 yyyy-mm-dd

    const reg = /^[0-9]{4}-(0[1-9]|1[12])-(0[1-9]|[12][0-9]|[3][01])$/g;

### windows 操作文件路径

    const reg = /^[a-zA-Z]:\\([^\\:*<>|"?\r\n/]+\\)*([^\\:*<>|"?\r\n/]+)?$/;

### 匹配 id

    const reg = /id="[^"]*"/;

### 匹配位置

    const reg = /^|$/g;

### 不匹配任何东西

    const reg = /.^/

### 数字千位分隔符表示法

    const reg = /(?!^)(?=(\d{3})+$)/g

### 验证密码

    const reg = /(?=.*[0-9])(?=.*[a-zA-Z])^[0-9a-zA-Z]{6,12}$/;
    const reg1 = /(?!^[0-9]{6,12}$)(?!^[a-z]{6,12}$)(?!^[A-Z]{6,12}$)^[0-9A-Za-z]{6,12}$/;

### 字符串trim方法模拟

    const reg = /^\s+|\s+$/g

### 将每个单词的首字母转换为大写

    const reg = /(?:^|\s)\w/g
    const string = "i am gerritv".replace(reg, c => c.toUpperCase());

### 驼峰化

    const reg = /[-_\s]+(.)?/g;
    const string = "_si-uiw".replace(reg, (match, c) => {
        c ? c.toUpperCase() : "";
    });

### 中线化

    const reg = /([A-Z])/g;
    const reg1 = /[-_\s]+/g;
    const string = "MozTransform"
        .replace(reg, "-$1")
        .replace(reg1, "-")
        .toLowerCase();

### 身份证正则

    const reg = /^\d{15}|\d{17}[\dxX]$/;

### IPV4地址

    const reg = /^((0{0,2}\d|0?\d{2}|1\d{2}|2[0-4]\d|25[0-5])\.){3}(0{0,2}\d|0?\d{2}|1\d{2}|2[0-4]\d