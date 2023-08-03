---
external: false
title: serverless + puppeteer 的实践与填坑
description: 一次 serverless + puppeteer 的实践与填坑
date: 2023-08-03
---

## 从生成PDF需求引起的问题
因为项目中涉及到了 `PDF` 文件的生成，目前生成 `PDF` 使用的纯前端 `html2canvas + jspdf`的方案，这种方案虽然不需要服务端的支持，但用户体验也大打折扣，最终权衡以下弊端舍弃了该方案，转而拥抱 `puppeteer`.

**弊端：**
1. 内容中文字**不能选中**，链接也**不能点击**，本质上就是先绘制图片，再将图片插入到`PDF`中
2. 图像本身的属性比如 `obejct-fit` 设置也不起效果，导致导出后的图像被**拉伸**，
3. 伪元素的效果设置也无效，且导出的 `PDF` 内容中伪元素大小显示不一致，比如 `li:marker`
4. 生成的**文件过大**，因为是图片，生成的内容如果不通过放大增加分辨率，最终声称的内容将会非常模糊，但是放大之后文件就变得非常大，同样的内容使用服务端导出大小在 `300KB` 左右，前端导出在`10M` 左右，这是一个**非常不能接受**的缺点

## 为什么选择 puppeteer
虽说前端还有 `print` 方法可以生成质量不错的 `PDF`，但是在用户体验上来说还是差了些，需要用户多次点击按钮操作，弹出的一些选项需要用户自己进行勾选，且打印效果非常难调整，导出的效果与预览效果的排版并不一致，所以综合考虑还是使用无头浏览器来将这一系列操作都自动化！

## 解决服务器问题 
要在服务端生成 `PDF`，首先得有一个服务器来放后端服务吧，但是自己又不想买服务器，碍于域名、备案等等一系列事情，想想就头大。所以最终我选择直接挂到 `serverless` 云函数上，在挑选了一通免费服务后我选择了 `Netlify Functions`


## 使用puppeteer实现PDF导出
进入正题，根据自己的项目需求，实现了一个大致的导出框架
### 代码实现
```js
const puppeteer = require("puppeteer");

exports.handler = async function (event, context) {
  const { content, style, link } = JSON.parse(event.body);
  // 开启一个浏览器进程
  const browser = await puppeteer.launch({ headless: "new" });
  // 打开一个页面
  const page = await browser.newPage();
  // 设置页面内容
  await page.setContent(
       `<html>
            <head>
                <meta charset="UTF-8" />
                <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            </head>
            <body>${content}</body>
        </html>`
  );
  // 给页面添加样式
  await page.addStyleTag({ url: link });
  // 生成pdf
  const pdf = await page.pdf({
    width: 794,
    height: 1123,
    printBackground: true
  });
  // 关闭浏览器
  browser.close();
  return {
    statusCode: 200,
    body: JSON.stringify({
      msg: "导出成功~",
      pdf
    }),
  };
};

```
### 效果测试
以上代码测试过后没有任何问题，可以导出，就是速度不是很理想（但毕竟是免费的，还要什么自行车呢），测试没问题之后直接就部署该函数

![image.png](https://z4a.net/images/2023/08/03/b19af910846b4ddb9e2988879c8dab07tplv-k3u1fbpfcp-watermark.png)


## serverless环境使用puppeteer报错
上线后使用导出功能结果抛出了一个错误：“错误的启动浏览器进程...”

![image.png](https://z4a.net/images/2023/08/03/puppeteer-error.png)

查阅文档会发现 `puppeteer` 依赖 `chromuim` 浏览器的环境，所以如果要使用它的话，那么前提是服务所处环境有`chromium`浏览器，那么我猜测在`serverless`环境是没有`chromuim`的，那怎么办？这个时候就需要给它提供一个环境了，怎么提供？

## puppeteer执行环境配置
本地开发和线上部署使用不同配置

### 本地环境

在本地开发的时候，我们提供本机的 `chrome` 执行文件地址给 `puppeteer`，访问 **chrome://version**

![image.png](https://z4a.net/images/2023/08/03/chrome_execute_path.png)

### 线上环境
线上环境自定义`chromuim`路径，查阅了许多文档都说使用 `chrome-aws-lambda` 这个库可以实现在云上使用 `chromium`，但是经过我的多次尝试发现并不可行，`@sparticuz/chromium`这个库是行得通的


### 调整代码
ok，准备就绪，现在需要对代码进行一些改动，同时还有一个需要注意的点，现在可以使用`puppeteer-core` 来替代 `puppeteer`，因为 `puppeteer` 会自动下载与之匹配的 `chromium` 版本，但是
目前我们并不需要它帮我们下载了，我们自己提供给它，也就只需要使用它的核心代码部分即可

```js
const chromium = require("@sparticuz/chromium");
const puppeteer = require("puppeteer-core");

const isDev = process.env.CONTEXT === "dev";

exports.handler = async function (event, context) {
// 省略不相关代码...
const browser = await puppeteer.launch({
      args: chromium.args,
      defaultViewport: chromium.defaultViewport,
      executablePath: isDev
        ? "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
        : await chromium.executablePath(), // 生产环境使用我们自定义的chromium路径
      headless: chromium.headless,
    });
// 省略不相关代码...
};
```
`args` 参数用于传递命令行参数给 `chromium` 浏览器实例，以配置其行为和性能设置，而 `defaultViewport` 参数用于设置浏览器视口的默认大小，`executablePath` 用于指定 `chromium` 的执行路径。改动完之后部署上线再测试一波

### 部署测试

![image.png](https://z4a.net/images/2023/08/03/deploy_effect_525435.png)

测试过后没有问题，现在就可以愉快的白嫖了～

## 总结
其实看起来真的没多少内容，但是如果刚接触的朋友自己去实践这块内容会踩很多坑，社区的相关解决方案大多是过期的，现在已经是不实用了，我自己查阅了两天国内外的文档才解决这个头疼的问题，所以把这个方案也分享一下，有遇到相同问题的朋友可以参考一下，给这个 `bug` 画上一个完美的句号。
