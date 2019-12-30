---
title: 从零开始打造属于自己的UI库
date: 2019-12-30 19:44:01
tags:
  - Vue
categories:
  - 前端
---


目前主流的 Vue UI 框架很多, 每个 UI 框架的风格各有各的特点, 国内目前知名度较高的莫过于 Element 和 iView

大概是 16 年的时候，那时候刚开始上手 Vue2.0, 同时也分别使用了 Element 和 iView 两个 UI 框架来实现一些运营系统的前端开发。

从那时候开始就想着，能写出这么完善和优秀的框架得需要多久啊！内心一直很渴望也能够自己写一写 UI 框架，然后自己设计，自己配色。

不求推广到给别人使用！只求能够自己过把瘾！以及在写 UI 框架的过程中能够学习和沉淀多一丁点前端技术。


![Vue](/images/best-ui-banner.png)

> The best-ui is not the best !important;

__PS: Best UI 大部分学习和借鉴了 Element 和 iView 组件库的源码__

## 项目结构

.
+-- build
+-- examples
+-- lib
+-- packages
    +-- components
    +-- theme
+-- src
    +-- mixins
    +-- utils
    +-- index.js
+-- tests
+-- types
+-- babel.config.js
+-- components.json
+-- gulpfile.js
+-- package.json

看过 Element 的源码的同学估计一目了然，这里主要参考了 Element UI 的项目结构

简单讲一下这里的项目目录和文件的作用

- build 主要是和构建相关
- examples 用于编写一些简单的 demo 代码，开发组件调试所用
- lib 是打包构建完成后代码的存放目录
- packages 是每个组件的源码实现, components 表示每个组件独立的目录，theme 存放组件的样式 .scss 文件
- src 是 UI 框架的入口文件存放位置，以及一些工具函数，mixins，plugins 文件的存放
- tests 单元测试和集成测试（不好意思，写这篇文章的时候一个组件的单元测试都还没写，我是个辣鸡）
- types d.ts 的类型声明文件（写这篇文章的时候所有组件的 d.ts 文件还没完成补齐，我是个辣鸡第二遍）
- babel.config.js babel 的配置声明
- components.json 用来指定具体组件的源码位置，用于构建多个组件单独的文件
- gulpfile 用于 gulp 进行样式构建的运行文件

一个前端工程化项目的结构出来了，按照个人的开发习惯，接下来就是要对项目的构建进行配置了，不然光写代码而运行不了，没什么意义。

## Webpack 构建配置

