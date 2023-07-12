---
external: false
title: 认识CSS选择器:where以及:is
description: CSS伪类选择器where以及is的妙用
date: 2023-07-12
---

在 `css` 文件中如果需要对多个元素进行相同的样式设置，那么我们大多数情况下选择的写法都是这样

```css
/* 将a标签以及p标签的颜色设置为红色 */
.box a,
.box p {
  color: red;
}
```

使用这种写法，如果需要设置的元素种类少还好，如果比较多的话那么就会比较麻烦了，这种情况我们的 `:where` 选择器以及 `:is` 选择器就派上用场了

### 使用 where 简化

```css
.box :where(a, p) {
  color: red;
}
```

### 使用 is 简化

```css
.box :is(a, p) {
  color: red;
}
```

它们都达到了相同的效果，但是相比第一种方案，`:where`、`:is`帮助我们简化了代码，是不是很棒～

并且它们的作用不仅如此，如果其中一个选择器如果是错误的，并不会导致所有的选择器效果失效，看下面的例子

```css
.box a,
.box p:abc {
  color: red;
}
```
上面的例子中`:abc`这个伪类是不存在的，这种写法如果其中一个写错了就会导致其他元素效果也失效，所以上面`a`标签的样式也不会生效，如果采用`:where`和`:is`选择器来写的话，就可以避免这种问题

```css
.box :where(a, p:abc) {
  color: red;
}
```

以上写法尽管`p`标签的伪类写法不存在，但是`a`标签的效果并不会失效


### 不同点
官方的解释是`:where`选择器在 `CSS3` 中引入，而 `:is` 选择器在 `CSS4` 中引入。因此，`:where` 选择器的兼容性会更好一些，并且`:is`的权重相比`:where`会更高

**使用is的权重**
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29103cda86bc4c5f983c621741c8054e~tplv-k3u1fbpfcp-watermark.image?)

**使用where的权重** 
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/724bc46f3d2a4a549b5dcb5a83764457~tplv-k3u1fbpfcp-watermark.image?)

好了这篇文章就分享到这里，有任何发现后面再进行补充。