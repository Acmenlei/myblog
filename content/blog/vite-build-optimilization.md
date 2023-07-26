---
external: false
title: vite打包优化将包体积压缩75%
description: vite打包优化包体积压缩75%
date: 2023-06-15
---

## 先说结论

优化前是 `7.8M`，优化后是 `1.9M`，整体来说还是不错的，因为这个项目图片比较少，能做的优化还是非常有限的
![image.png](https://z4a.net/images/2023/07/26/vite_build_1.png)

## 怎么优化

想要实现优化，首先我得先知道，是什么占了这么大的空间？是图片？是库？还是其他静态资源？

- 可以将文件分别归类，将 `js`，`css` 等资源目录分别打包到对应的文件夹下

```ts
build: {
      rollupOptions: {
        output: {
          chunkFileNames: 'js/[name]-[hash].js',
          entryFileNames: 'js/[name]-[hash].js',
          assetFileNames: '[ext]/[name]-[hash].[ext]'
        }
      }
    }
```

- 查看项目的依赖，找出大块头，这里使用`rollup-plugin-visualizer`作为分析工具，

  > `rollup-plugin-visualizer`是一个打包体积分析插件，对应`webpack`中的`webpack-bundle-analyzer`。配置好后运行构建命令会生成一个`stats.html`。

- 安装依赖

```bash
npm i rollup-plugin-visualizer -D
```

- 在`vite.config.ts`文件中引入

```ts
import { visualizer } from "rollup-plugin-visualizer";
```

- 在 `plugins` 选项中注册该插件

```ts
defineConfig({
  plugins: [visualizer({ open: true })], // open表示生成后自动打开依赖分析文件
});
```

- 运行打包命令后就可以看到打包文件的体积明细了

```base
npm run build
```

![image.png](https://z4a.net/images/2023/07/26/vite_build_2.png)

从体积能看到，这里光`textbus`就已经占据了 1.3M 了，是时候该做点什么了。

## 开始优化

### 拆包

> 如果不同模块使用的插件基本相同那就尽可能打包在同一个文件中，减少 http 请求，如果不同模块使用不同插件明显，那就分成不同模块打包。这里使用的是最小化拆分包。如果是前者可以直接选择返回'vendor'。

```ts
rollupOptions: {
  output: {
    chunkFileNames: 'js/[name]-[hash].js',
    entryFileNames: 'js/[name]-[hash].js',
    assetFileNames: '[ext]/[name]-[hash].[ext]',
    manualChunks(id) {
      if (id.includes("node_modules")) {
        // 让每个插件都打包成独立的文件
        return id.toString() .split("node_modules/")[1] .split("/")[0] .toString();
      }
    }
  }
}
```

### 删除生产环境的 debugger/console

这里我使用的是`esbuild`进行压缩，所以我在这里配置`esbuild`即可，如果你是使用的`terser`做压缩，那么你需要另外安装`terser`，这里不做过多的赘述，可以自行查看`rollup`文档。

```ts
    // mode为vite传递给
    export default ({ mode }) => {
      const env = loadEnv(mode, process.cwd())
      return defineConfig({
        plugins: [...],
        // 生产环境删除debugger console语句 这里的VITE_DROP_CONSOLE是env文件中配置的环境变量
        esbuild: {
          drop: env?.VITE_DROP_CONSOLE === 'true' ? ['console', 'debugger'] : []
        },
      })
    }
```

### CDN 加速

内容分发网络（Content Delivery Network，简称 CDN）就是让用户从最近的服务器请求资源，提升网络请求的响应速度。同时减少应用打包出来的包体积，利用浏览器缓存，不会变动的文件长期缓存。(**不建议使用第三方 CDN，第三方 CDN 稳定性差，响应较慢，影响用户体验，这里仅为做学习讨论使用**)

- 安装使用到的依赖包，这个插件可以告诉 `vite`，哪些依赖包是不需要打包进生产的，使用全局变量替代

```base
npm i rollup-plugin-external-globals -D
```

- 在 vite 中引入

```ts
import externalGlobals from "rollup-plugin-external-globals";
```

- 配置需要在打包阶段排除的依赖包，这里其实有个点需要注意一下，很多同学搞不懂，这个映射的变量应该填什么，特别是`npm`包和`cdn`的使用方式不同的情况，不理解其中的映射关系的话就会导致无从下手，笔者自己也是尝试了很多次才真正理解，比如`@textbus/editor`这个包使用的就是`editor`变量，而很多同学可能会将他们的映射关系写成`'@textbus/editor': 'textbus'`，因为 CDN 暴露的全局变量是`textbus`，那我们想当然的以为对应的就是`textbus`，结果打包就出现了各种错误提示，但其实`externalGlobals`插件只帮助我们处理映射关系，所以`@textbus/editor`它对应的应该是`textbus`变量下的`editor`实例，就如`@textbus/xxx`对应`textbus.xxx`，我们需要指定好他们之间的关系才不会出错。

```ts
const globals = externalGlobals({
  jspdf: 'jspdf',
  '@textbus/editor': 'textbus.editor',
  axios: 'axios',
  html2canvas: 'html2canvas'
})

rollupOptions: {
    external: ['jspdf', '@textbus/editor', 'axios', 'html2canvas'],
    plugins: [globals],
    output: {
      chunkFileNames: 'js/[name]-[hash].js',
      entryFileNames: 'js/[name]-[hash].js',
      assetFileNames: '[ext]/[name]-[hash].[ext]',
      manualChunks(id) {
        if (id.includes('node_modules')) {
          return id.toString().split('node_modules/')[1].split('/')[0].toString()
        }
      }
    }
  }
```

- 在 `html` 中引入依赖包的对应的 `CDN`

```html
<link rel="stylesheet" href="https://textbus.io/lib/textbus.min.css" />
<script
  defer
  src="https://cdn.jsdelivr.net/npm/axios@0.21.1/dist/axios.min.js"
></script>
<script defer src="https://textbus.io/lib/textbus.min.js"></script>
<script
  defer
  src="https://unpkg.com/jspdf@2.5.1/dist/jspdf.umd.min.js"
></script>
<script
  defer
  src="https://unpkg.com/html2canvas@1.4.1/dist/html2canvas.js"
></script>
```

这样我们打包的效果就已经出来了，`textbus`、`jspdf` 等重量级选手就已经被移出了我们的打包文件
![image.png](https://z4a.net/images/2023/07/26/vite_build_3.png)

## 为什么不移除 vue 和 element-plus？

本项目有多个依赖包依赖了`vue`，比如`element-plus`依赖`vue`，如果将`vue`移除在打包过程中会出现如下情况：
![image.png](https://z4a.net/images/2023/07/26/vite_build_4.webp)
这是因为将 `vue` 移除后，别的依赖包找不到 `vue` 了，随之要做的就是要将这个依赖包也移除，层层套娃，反而变得复杂起来，所以暂时不考虑将 `vue` 移除。

## 文件压缩

- 安装`vite-plugin-compression`插件

```bash
npm i vite-plugin-compression -D
```

- 在`vite`中引入

```ts
// 设置位置：build.rollupOptions.plugins[]
viteCompression({
  verbose: true, // 是否在控制台中输出压缩结果
  disable: false,
  threshold: 10240, // 如果体积大于阈值，将被压缩，单位为b，体积过小时请不要压缩，以免适得其反
  algorithm: "gzip", // 压缩算法，可选['gzip'，' brotliccompress '，'deflate '，'deflateRaw']
  ext: ".gz",
});
```

## 图片资源压缩

- 安装`vite-plugin-imagemin`

```bash
yarn add vite-plugin-imagemin -D
```

or

```bash
npm i vite-plugin-imagemin -D
```

- 在`vite`中引入并在`plugins`注册使用

```ts
import viteImagemin from "vite-plugin-imagemin";

plugin: [
  viteImagemin({
    gifsicle: {
      optimizationLevel: 7,
      interlaced: false,
    },
    optipng: {
      optimizationLevel: 7,
    },
    mozjpeg: {
      quality: 20,
    },
    pngquant: {
      quality: [0.8, 0.9],
      speed: 4,
    },
    svgo: {
      plugins: [
        {
          name: "removeViewBox",
        },
        {
          name: "removeEmptyAttrs",
          active: false,
        },
      ],
    },
  }),
];
```

> 需要注意的是`viteImagemin`在国内比较难安装，容易出现报错，可以尝试一下下面几种解决方案。

- 使用 `yarn` 在 `package.json` 内配置(推荐)

```json
"resolutions": {
  "bin-wrapper": "npm:bin-wrapper-china"
}
```

- 使用 `npm` 在电脑 `host` 文件加上如下配置即可

```bash
199.232.4.133 raw.githubusercontent.com
```

- 执行打包后可以看到最终的压缩效果，有一张图片从`1000多kb`压缩到了`36kb`，可以说是非常给力了!
  ![image.png](https://z4a.net/images/2023/07/26/vite_build_5.png)

## 本章 Vite 优化结束
