---
title: Flutter学习笔记（一）——入门
date: 2021-07-25 10:28:03
toc: true
mathjax: false
categories: 
- 前端
tags: 
- flutter
---

## 一、简介

Flutter 是谷歌开发的的一款开源的、免费的，基于 Dart 语言的 UI 框架，帮助开发者通过一套代码库高效构建多平台精美应用，支持移动、Web、桌面和嵌入式平台。。它最大的特点就是**跨平台**与**高性能**。

<!-- more -->

1、跨平台

Flutter 开发的应用可适用于多个终端：

- 移动端：IOS、Andriod
- Web端：各种浏览器
- 桌面：Windows、Mac
- 嵌入式平台：Linux、Fuchsia

2、高性能

- Flutter 开发的应用的性能接近于原生 App
- Flutter 采用了 GPU 渲染技术
- Flutter 应用的刷新频率可达 120 fps（React Native 的刷新频率只能达 60 fps），因此可以用 Flutter 来开发游戏

## 二、定位

1、移动 App 的开发模式

- 原生开发：原生 App，目前主要指 IOS 和 Andriod 开发
- 混合开发：混合 App，目前主要框架有 React Native、Weex 和 Flutter
- H5开发：Web App，主要通过 HTML、CSS 和 JavaScript 开发 Web 应用，然后再将这个 Web 应用套在 App 里

| 开发模式 | 原生开发  | 混合开发  | Web 开发 |
| ---- | -------- | -------------- | ------ |
| 运行环境    | Andriod、IOS  |      Andriod、IOS   | 浏览器、WebView |
| 编程语言 | Java、Objective-C | JavaScript、Dart | HTML、CSS、JavaScript |
| 可移植性 | 差 | 一般 | 好 |
| 开发速度 | 慢 | 一般 | 快 |
| 性能 | 快 | 较慢 | 慢 |
| 学习成本 | 高 | 一般 | 低 |
