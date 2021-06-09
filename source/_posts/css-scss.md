---
title: SCSS 学习笔记
date: 2021-06-03 19:52:58
toc: true
mathjax: false
categories: 
- 前端
tags:
- css
---

## CSS 预处理器的由来

CSS 语言本身存在如下缺陷：

- 语法不够强大，比如无法嵌套书写导致模块化开发中需要书写很多重复的选择器；
- 没有变量和合理的样式复用机制，使得逻辑上相关的属性值必须以字面量的形式重复输出，导致难以维护；

## CSS 预处理器是什么？

CSS 预处理器用一种专门的编程语言，进行 Web 页面样式设计，然后再编译成正常的 CSS 文件，以供项目使用。CSS 预处理器为 CSS 增加一些编程的特性，无需考虑浏览器的兼容性问题。 比如说：Sass（SCSS）、LESS、Stylus、Turbine 等等都属于 CSS 预处理器。

<!-- more -->

目前使用范围较广的 CSS 预处理器主要有如下三种：

- SASS：2007年诞生，最早也是最成熟的 CSS 预处理器，拥有 ruby 社区的支持和 compass 这一最强大的 CSS 框架，后来受 LESS 影响，已经进化到了全面兼容 CSS 的 SCSS。
- LESS：2009年出现，受 SASS的 影响较大，但又使用 CSS 的语法，让大部分开发者和设计师更容易上手，在 ruby 社区之外支持者远超过 SASS，其缺点是比起 SASS 来，可编程功能不够，不过优点是简单和兼容 CSS，反过来也影响了 SASS 演变到了 SCSS 的时代，著名的 Twitter Bootstrap 就是采用 LESS 做底层语言的。
- Stylus：2010年产生，来自 Node.js 社区，主要用来给 Node 项目进行 CSS 预处理支持，在此社区之内有一定支持者，在广泛的意义上人气还完全不如 SASS 和 LESS，但 Stylus被称为是一种革命性的新语言，提供一个高效、动态、和使用表达方式来生成 CSS，写法更接近于 js，并且同时支持缩进和 CSS 常规样式书写规则。


## 编译环境

使用终端命令实现对 SASS 代码的编译需要全局安装 SCSS 预处理器，在不同的环境我们可以使用了不同的预处理器：

- Node 环境下的 node-sass 模块
- Node 环境下的 dart-sass 模块
- Ruby 环境下的 sass 模块
- Dart 环境下的 sass 模块

SCSS 的开发语言是 Ruby，但是由于对于前端来说最常用的环境是 Node，因此社区为我们提供了 Node 环境下的编译包 node-sass 和 dart-sass，Less 在命令行终端进行编译则需要全局安装 less 库，而 Stylus 本身就是来源于 Node.js 社区，因此只需要安装最新的 stylus 包即可使用。

## 语法糖



