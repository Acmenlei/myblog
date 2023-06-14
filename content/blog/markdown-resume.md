---
external: false
title: 开发了一个专注于程序员的简历制作工具
description: 用写 Markdown 的方式来写简历，让写简历变得更简单.
date: 2023-05-01
---

> 在介绍之前我们先来看看目前现有的简历平台都有哪些缺点以及我是如何萌生的这个想法的.

## 现有的简历平台有什么缺点

- 基于 UI 操作 **(拖拽)** 界面的方式扩展性不够高，模块更改较为繁琐，用户对于简历模板的某个细节点不够满意，不能对其进行调整修改
- 编写一份简历内容不能适配多个简历模板

## 我是怎么想去改善的

在一次项目开发中我使用了`Markdown`转`HTML`的插件，而当时正好我又在调整简历内容，且我使用的简历平台的 UI 操作让我感到非常的不灵活，于是我萌生了一个想法：“**`Markdown`可以用来做笔记，那我用来写简历好像也是可以的?** 想法有了，说干就干！”

## 预期功能

- 双编辑模式，`Markdown模式` 和`内容模式（富文本编辑）`
- `Markdown` 到 `PDF` 的转换
- 简历内容智能一页
- 上传证件照
- 自由控制简历排版，提供自定义编写`CSS`的功能
- 提供边距调节器，自由调节简历内容的边距问题
- 滚动跟随（编辑器和预览内容联动）
- 主题切换

## 开发过程中遇到的问题

社区现有的的插件并不提供多列布局的支持，虽然一些比较成熟的`Markdown`转`HTML`插件也提供了一些接口给用户扩展其他语法，但是，在简历的编写过程中其实只需要用到一些基础的语法支持，这么一看，很多语法的解析都是冗余的，增大了项目的体积大小，所以总结一下就是以下两点：

- `Markdown` 的排版不能实现多列布局，写出来的简历`排版过于单一`
- 如何控制两种模式（`Markdown` 模式 & 内容模式）的内容数据联动

## 解决问题

适合自己的才是最好的，综合以上考虑，我选择自己实现`MD`语法解析器来解析简历中可能会用到的语法，以及扩展多列布局等语法，插件仓库[在这里](https://github.com/Acmenlei/markdown-transform-html)。为了解决`Markdown`模式和内容模式转换的数据联动问题，我还需要编写从`HTML`到`Markdown`的转换器

## 实现效果

### 简历编辑

- 在主题方面有两种主题可以切换，`Light` 和 `Dark` 模式
- 在编辑模式方面是两种，`Markdown模式`和`内容模式`

#### Markdown 模式 & Dark 模式

![Markdown 模式 & Dark模式](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d97bd20df1c846119bdc5d3fc2c46055~tplv-k3u1fbpfcp-watermark.image?)

#### 内容模式 & Light 模式

![内容模式 & Light模式](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6445e7e79cfd4e23972923f3b40ba238~tplv-k3u1fbpfcp-watermark.image?)

## 项目体验
[在线编辑地址1](https://codeleilei.gitee.io/markdown2pdf)
&nbsp;&nbsp;&nbsp;
[在线编辑地址2](https://acmenlei.github.io/markdown-resume-to-pdf/dist/)