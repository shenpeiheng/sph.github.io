---
url: /guide/assets.md
---
# 静态资源处理 {#static-asset-handling}

* 相关: [公共基础路径](./build#public-base-path)
* 相关: [`assetsInclude` 配置项](/config/shared-options.md#assetsinclude)

## 将资源引入为 URL {#importing-asset-as-url}

服务时引入一个静态资源会返回解析后的公共路径：

```js twoslash
import 'vite/client'
// ---cut---
import imgUrl from './img.png'
document.getElementById('hero-img').src = imgUrl
```

例如，`imgUrl` 在开发时会是 `/src/img.png`，在生产构建后会是 `/assets/img.2d8efhg.png`。

行为类似于 Webpack 的 `file-loader`。区别在于导入既可以使用绝对公共路径（基于开发期间的项目根路径），也可以使用相对路径。

* `url()` 在 CSS 中的引用也以同样的方式处理。

* 如果 Vite 使用了 Vue 插件，Vue SFC 模板中的资源引用都将自动转换为导入。

* 常见的图像、媒体和字体文件类型被自动检测为资源。你可以使用 [`assetsInclude` 选项](/config/shared-options.md#assetsinclude) 扩展内部列表。

* 引用的资源作为构建资源图的一部分包括在内，将生成散列文件名，并可以由插件进行处理以进行优化。

* 较小的资源体积小于 [`assetsInlineLimit` 选项值](/config/build-options.md#build-assetsinlinelimit) 则会被内联为 base64 data URL。

* Git LFS 占位符会自动排除在内联之外，因为它们不包含它们所表示的文件的内容。要获得内联，请确保在构建之前通过 Git LFS 下载文件内容。

* 默认情况下，TypeScript 不会将静态资源导入视为有效的模块。要解决这个问题，需要添加 [`vite/client`](./features#client-types)。

::: tip 通过 `url()` 内联 SVG
当在 JS 中手动构造 `url()` 并传入一个 SVG 的 URL 时，应该用双引号将变量包裹起来。

```js twoslash
import 'vite/client'
// ---cut---
import imgUrl from './img.svg'
document.getElementById('hero-img').style.background = `url("${imgUrl}")`
```

:::

### 显式 URL 引入 {#explicit-url-imports}

未被包含在内部列表或 `assetsInclude` 中的资源，可以使用 `?url` 后缀显式导入为一个 URL。这十分有用，例如，要导入 [Houdini Paint Worklets](https://developer.mozilla.org/en-US/docs/Web/API/CSS/paintWorklet_static) 时：

```js twoslash
import 'vite/client'
// ---cut---
import workletURL from 'extra-scalloped-border/worklet.js?url'
CSS.paintWorklet.addModule(workletURL)
```

### 显式内联处理 {#explicit-inline-handling}

可以分别使用`?inline`或`?no-inline`后缀，明确导入带内联或不带内联的静态资源。

```js twoslash
import 'vite/client'
// ---cut---
import imgUrl1 from './img.svg?no-inline'
import imgUrl2 from './img.png?inline'
```

### 将资源引入为字符串 {#importing-asset-as-string}

资源可以使用 `?raw` 后缀声明作为字符串引入。

```js twoslash
import 'vite/client'
// ---cut---
import shaderString from './shader.glsl?raw'
```

### 导入脚本作为 Worker {#importing-script-as-a-worker}

脚本可以通过 `?worker` 或 `?sharedworker` 后缀导入为 web worker。

```js twoslash
import 'vite/client'
// ---cut---
// 在生产构建中将会分离出 chunk
import Worker from './shader.js?worker'
const worker = new Worker()
```

```js twoslash
import 'vite/client'
// ---cut---
// sharedworker
import SharedWorker from './shader.js?sharedworker'
const sharedWorker = new SharedWorker()
```

```js twoslash
import 'vite/client'
// ---cut---
// 内联为 base64 字符串
import InlineWorker from './shader.js?worker&inline'
```

查看 [Web Worker 小节](./features.md#web-workers) 获取更多细节。

### `public` 目录 {#the-public-directory}

如果你有下列这些资源：

* 不会被源码引用（例如 `robots.txt`）
* 必须保持原有文件名（没有经过 hash）
* ...或者你压根不想引入该资源，只是想得到其 URL。

那么你可以将该资源放在指定的 `public` 目录中，它应位于你的项目根目录。该目录中的资源在开发时能直接通过 `/` 根路径访问到，并且打包时会被完整复制到目标目录的根目录下。

目录默认是 `<root>/public`，但可以通过 [`publicDir` 选项](/config/shared-options.md#publicdir) 来配置。

请注意，应该始终使用根绝对路径来引入 `public` 中的资源 —— 举个例子，`public/icon.png` 应该在源码中被引用为 `/icon.png`。

## new URL(url, import.meta.url)

[import.meta.url](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import.meta) 是一个 ESM 的原生功能，会暴露当前模块的 URL。将它与原生的 [URL 构造器](https://developer.mozilla.org/en-US/docs/Web/API/URL) 组合使用，在一个 JavaScript 模块中，通过相对路径我们就能得到一个被完整解析的静态资源 URL：

```js
const imgUrl = new URL('./img.png', import.meta.url).href

document.getElementById('hero-img').src = imgUrl
```

这在现代浏览器中能够原生使用 - 实际上，Vite 并不需要在开发阶段处理这些代码！

这个模式同样还可以通过字符串模板支持动态 URL：

```js
function getImageUrl(name) {
  // 请注意，这不包括子目录中的文件
  return new URL(`./dir/${name}.png`, import.meta.url).href
}
```

在生产构建时，Vite 才会进行必要的转换保证 URL 在打包和资源哈希后仍指向正确的地址。然而，这个 URL 字符串必须是静态的，这样才好分析。否则代码将被原样保留、因而在 `build.target` 不支持 `import.meta.url` 时会导致运行时错误。

```js
// Vite 不会转换这个
const imgUrl = new URL(imagePath, import.meta.url).href
```

::: details 工作原理

Vite 会将 `getImageUrl` 函数改造为：

```js
import __img0png from './dir/img0.png'
import __img1png from './dir/img1.png'

function getImageUrl(name) {
  const modules = {
    './dir/img0.png': __img0png,
    './dir/img1.png': __img1png,
  }
  return new URL(modules[`./dir/${name}.png`], import.meta.url).href
}
```

:::

::: warning 注意：无法在 SSR 中使用
如果你正在以服务端渲染模式使用 Vite 则此模式不支持，因为 `import.meta.url` 在浏览器和 Node.js 中有不同的语义。服务端的产物也无法预先确定客户端主机 URL。
:::
