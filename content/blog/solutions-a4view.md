---
external: false
title: 解决A4纸分页边界元素被截断的问题
description: 解决A4纸分页边界元素被截断的问题
date: 2023-07-25
---

## 前言
由于自己的项目需要一个能够将`HTML`结构实时预览为 `PDF` 效果图的功能，然后社区里找了很久也没发现有合适的插件能够支持，没办法，只能自己动手造了（痛～）。

## 分页思路
### 绝对定位 + 偏移位
首先A4纸每一页的宽高分别是 `210mm` 和 `297mm`，换算成像素是 `794px` 和 `1123px`，如果当前 `HTML` 内容的高度超过了一页`A4`纸的高度，那么就需要创建新的一页，将超出的内容放置到下一页，以此类推。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3d407622c56440298dff5c1af6d9ce8~tplv-k3u1fbpfcp-watermark.image?)

### 代码实现

- `re-render` 为需要渲染分页内容的容器
- `jufe-wrapper-page` 为 A4 纸大小的容器
- `jufe-wrapper-page-item` 为真实渲染区域

```ts
const A4_HEIGHT = 1123
// 分割视图 传入一个需要分页的元素
export function splitPage(renderCV: HTMLElement) {
  let page = 0,
    realHeight = 0
  const target = renderCV.clientHeight,
    reRender = document.querySelector('.re-render') as HTMLElement
  reRender.innerHTML = ''

  while (target - realHeight > 0) {
    const wrapper = createDIV(),
      resumeNode = renderCV.cloneNode(true) as HTMLElement
    wrapper.classList.add('jufe-wrapper-page')
    // 创建A4纸里面真正需要渲染的内容 且最小化高度
    const realRenderHeight = Math.min(target - realHeight, A4_HEIGHT)
    const wrapperItem = createDIV()
    wrapperItem.classList.add('jufe-wrapper-page-item')
    wrapperItem.style.height = realRenderHeight + 'px'

    resumeNode.style.position = 'absolute'
    // 计算当前偏移位
    resumeNode.style.top = -page * A4_HEIGHT + 'px'
    resumeNode.style.left = 0 + 'px'

    wrapperItem.appendChild(resumeNode)
    wrapper.appendChild(wrapperItem)

    realHeight += A4_HEIGHT
    page++
    reRender?.appendChild(wrapper) // 将当前生成的 A4 纸添加到容器中
  }
}
```

## 不足之处
现在的分页效果已经有了，但是分页过程中会出现一个体验非常差的问题，那就是**边界内容会被截断**，怎么理解？比如一段文字，它的上半部分显示在第一页，下半部分显示在第二页，这就是内容截断。为什么会出现这种情况？出现这种情况是因为定位造成的，定位只关注具体的像素，而识别不了边界位置是否有元素存在。

## 解决方案
即然在边界的内容被截断了，那么我们需要想办法**找到边界那个被截断的元素**，在该元素前面**创建一个空白元素**，**高度为被截断的上半部分内容的高度**，然后在**分页之前**，把该元素挤到下一页去，确保分页的时候内容被完全显示，画个图理解一下。

### 没有处理边界元素之前

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29ce177d8b404352bc160d101e5a9d19~tplv-k3u1fbpfcp-watermark.image?)

### 处理了边界元素之后

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca32404c43c64bb28f6a40676cbd3631~tplv-k3u1fbpfcp-watermark.image?)

### 代码实现
有了思路，试试写出代码

#### 找到处于边界的元素节点
这个节点高度应该尽可能的小，比如一个div它的偏移位加上自己本身的高度，已经超出了当前的页所能容纳的高度，那么它是边界元素，但是如果它里面还有div等块元素存在的话，那么我们需要秉承深度优先的原则，让这个占位符的高度尽可能的最小化
```ts

// 处理边界内容截断
export function handlerWhiteBoundary(renderCV: HTMLElement) {
// 首先我们获取到当前页面的边距，方便后续新页也保持这个边距大小
  const pt = +getComputedStyle(renderCV).getPropertyValue('padding-top').slice(0, -2)
  const pb = +getComputedStyle(renderCV).getPropertyValue('padding-bottom').slice(0, -2)
  const children = Array.from(renderCV.children) as HTMLElement[]
  // 记录页码 必须为引用 因为该变量涉及参数传递
  const pageSize = { value: 1 }
  for (const child of children) {
    // 子元素的外边距也需要参与计算
    const height = calculateElementHeight(child)
    // 当前元素距离最外层容器的高度 通过actualTop可以判断元素是否处于容器的边界
    const actualTop = getElementTop(child, renderCV)
    // 如果总长度已经超出了一页A4纸的高度（除去底部边距的高度） 那么需要找到边界元素
    if (actualTop + height > A4_HEIGHT * pageSize.value - pb) {
      if (child.children.length) {
        // 有子节点 继续往深度查找 最小化空白元素的高度
        findBoundaryElement(child, renderCV, pt, pb, pageSize)
      } else {
        // 没有子节点 计算空白占位符的高度 插入到边界元素的前面
        renderCV.insertBefore(
          createBoundaryWhiteSpace(A4_HEIGHT * pageSize.value - actualTop + pt),
          child
        )
        // A4 纸页数增加
        ++pageSize.value
      }
    }
  }
  return renderCV
}

// 最小化空白占位符的高度，深度优先查找
function findBoundaryElement(
  node: HTMLElement,
  target: HTMLElement,
  paddingTop: number,
  paddingBottom: number,
  pageSize: { value: number }
) {
  const children = Array.from(node.children) as HTMLElement[]
  for (const child of children) {
    const totalHeight = calculateElementHeight(child)
    const actualTop = getElementTop(child, target)
    if (actualTop + totalHeight > A4_HEIGHT * pageSize.value - paddingBottom) {
      // 直接排除一行段落文字 因为在一段普通文本中没必要再进行深入，它们内嵌不了什么其他元素
      if (child.children.length && !['p', 'li'].includes(child.tagName.toLocaleLowerCase())) {
        findBoundaryElement(child, target, paddingTop, paddingBottom, pageSize)
      } else {
        // 找到了边界 给边界元素前插入空白元素 将内容挤压至下一页
        node.insertBefore(
          createBoundaryWhiteSpace(A4_HEIGHT * pageSize.value - actualTop + paddingTop),
          child
        )
        pageSize.value++
      }
    }
  }
}
// 创建空白占位符
function createBoundaryWhiteSpace(h: number) {
  const whiteSpace = createDIV()
  whiteSpace.setAttribute(WHITE_SPACE, 'true')
  // 创建边界空白占位符 加上顶部边距
  whiteSpace.style.height = h + 'px'
  return whiteSpace
}


// 获取元素高度
function calculateElementHeight(element: HTMLElement) {
  // 获取样式对象
  const styles = getComputedStyle(element)
  // 获取元素的内容高度
  // const contentHeight = element.getBoundingClientRect().height
  const contentHeight = element.clientHeight
  // 获取元素的外边距高度
  const marginHeight =
    +styles.getPropertyValue('margin-top').slice(0, -2) +
    +styles.getPropertyValue('margin-bottom').slice(0, -2)
  // 计算元素的总高度
  const totalHeight = contentHeight + marginHeight
  return totalHeight
}

// 获取元素距离目标元素顶部偏移位
function getElementTop(element: HTMLElement, target: HTMLElement) {
  let actualTop = element.offsetTop
  let current = element.offsetParent as HTMLElement

  while (current !== target) {
    actualTop += current.offsetTop
    current = current.offsetParent as HTMLElement
  }
  return actualTop
}
```

到此为止，核心代码已经写完了，现在我们在分页之前先使用 `handlerWhiteBoundary` 函数把边界元素都处理一下

```ts
// 分割视图
export function splitPage(renderCV: HTMLElement) {
// 分页之前先进行边界元素处理
  handlerWhiteBoundary(renderCV)
  let page = 0,
    realHeight = 0
  const target = renderCV.clientHeight,
    reRender = document.querySelector('.re-render') as HTMLElement
  // ...
  // 省略后续代码
}
```
#### 最终效果图
ok，到此为止已经实现了我想要的效果

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f98b9dced7154664b962a7884ab36349~tplv-k3u1fbpfcp-watermark.image?)


## 完整代码
```ts
const A4_HEIGHT = 1123
// 分割视图 传入一个需要分页的元素
export function splitPage(renderCV: HTMLElement) {
  handlerWhiteBoundary(renderCV)
  let page = 0,
    realHeight = 0
  const target = renderCV.clientHeight,
    reRender = document.querySelector('.re-render') as HTMLElement
  reRender.innerHTML = ''

  while (target - realHeight > 0) {
    const wrapper = createDIV(),
      resumeNode = renderCV.cloneNode(true) as HTMLElement
    wrapper.classList.add('jufe-wrapper-page')
    // 创建A4纸里面真正需要渲染的内容 且最小化高度
    const realRenderHeight = Math.min(target - realHeight, A4_HEIGHT)
    const wrapperItem = createDIV()
    wrapperItem.classList.add('jufe-wrapper-page-item')
    wrapperItem.style.height = realRenderHeight + 'px'

    resumeNode.style.position = 'absolute'
    // 计算当前偏移位
    resumeNode.style.top = -page * A4_HEIGHT + 'px'
    resumeNode.style.left = 0 + 'px'

    wrapperItem.appendChild(resumeNode)
    wrapper.appendChild(wrapperItem)

    realHeight += A4_HEIGHT
    page++
    reRender?.appendChild(wrapper) // 将当前生成的 A4 纸添加到容器中
  }
}
// 处理边界内容截断
export function handlerWhiteBoundary(renderCV: HTMLElement) {
// 首先我们获取到当前页面的边距，方便后续新页也保持这个边距大小
  const pt = +getComputedStyle(renderCV).getPropertyValue('padding-top').slice(0, -2)
  const pb = +getComputedStyle(renderCV).getPropertyValue('padding-bottom').slice(0, -2)
  const children = Array.from(renderCV.children) as HTMLElement[]
  // 记录页码 必须为引用 因为该变量涉及参数传递
  const pageSize = { value: 1 }
  for (const child of children) {
    // 子元素的外边距也需要参与计算
    const height = calculateElementHeight(child)
    // 当前元素距离最外层容器的高度 通过actualTop可以判断元素是否处于容器的边界
    const actualTop = getElementTop(child, renderCV)
    // 如果总长度已经超出了一页A4纸的高度（除去底部边距的高度） 那么需要找到边界元素
    if (actualTop + height > A4_HEIGHT * pageSize.value - pb) {
      if (child.children.length) {
        // 有子节点 继续往深度查找 最小化空白元素的高度
        findBoundaryElement(child, renderCV, pt, pb, pageSize)
      } else {
        // 没有子节点 计算空白占位符的高度 插入到边界元素的前面
        renderCV.insertBefore(
          createBoundaryWhiteSpace(A4_HEIGHT * pageSize.value - actualTop + pt),
          child
        )
        // A4 纸页数增加
        ++pageSize.value
      }
    }
  }
  return renderCV
}

// 最小化空白占位符的高度，深度优先查找
function findBoundaryElement(
  node: HTMLElement,
  target: HTMLElement,
  paddingTop: number,
  paddingBottom: number,
  pageSize: { value: number }
) {
  const children = Array.from(node.children) as HTMLElement[]
  for (const child of children) {
    const totalHeight = calculateElementHeight(child)
    const actualTop = getElementTop(child, target)
    if (actualTop + totalHeight > A4_HEIGHT * pageSize.value - paddingBottom) {
      // 直接排除一行段落文字 因为在一段普通文本中没必要再进行深入，它们内嵌不了什么其他元素
      if (child.children.length && !['p', 'li'].includes(child.tagName.toLocaleLowerCase())) {
        findBoundaryElement(child, target, paddingTop, paddingBottom, pageSize)
      } else {
        // 找到了边界 给边界元素前插入空白元素 将内容挤压至下一页
        node.insertBefore(
          createBoundaryWhiteSpace(A4_HEIGHT * pageSize.value - actualTop + paddingTop),
          child
        )
        pageSize.value++
      }
    }
  }
}
// 创建空白占位符
function createBoundaryWhiteSpace(h: number) {
  const whiteSpace = createDIV()
  whiteSpace.setAttribute(WHITE_SPACE, 'true')
  // 创建边界空白占位符 加上顶部边距
  whiteSpace.style.height = h + 'px'
  return whiteSpace
}


// 获取元素高度
function calculateElementHeight(element: HTMLElement) {
  // 获取样式对象
  const styles = getComputedStyle(element)
  // 获取元素的内容高度
  // const contentHeight = element.getBoundingClientRect().height
  const contentHeight = element.clientHeight
  // 获取元素的外边距高度
  const marginHeight =
    +styles.getPropertyValue('margin-top').slice(0, -2) +
    +styles.getPropertyValue('margin-bottom').slice(0, -2)
  // 计算元素的总高度
  const totalHeight = contentHeight + marginHeight
  return totalHeight
}

// 获取元素距离目标元素顶部偏移位
function getElementTop(element: HTMLElement, target: HTMLElement) {
  let actualTop = element.offsetTop
  let current = element.offsetParent as HTMLElement

  while (current !== target) {
    actualTop += current.offsetTop
    current = current.offsetParent as HTMLElement
  }
  return actualTop
}
```
