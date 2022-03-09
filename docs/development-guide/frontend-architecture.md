# 前端架构介绍

本系统前端使用 [Angular](https://angular.io) 框架以及 [Angular Material](https://material.angular.io) UI 框架开发，为 SPA 应用。

## 项目结构简介

前端路由存放在 `src/app/app-routing.module.ts` 文件中，为了减小首页体积，采用了懒加载技术，将不同页面分为不同子模块，并在首页加载完毕后再进行预载，优化响应速度。其他页面都在自己的 `*-routing.module.ts` 文件里定义子路由。

在 `src/app/service` 中存放了如用户服务、提交服务等公共服务，可以使用依赖注入的方式在页面组件中使用。

在 `src/app/common` 中存放了一些公用组件，以及对一些第三方应用如 Markdown 编辑器的 Angular 封装。

其余的文件夹均为页面组件。

## API 通信

前端使用 `@improbable-eng/grpc-web` 包和后端服务使用 gRPC 协议通信。对于单次请求单次回复的 RPC 调用，将转换为一次 `POST` 请求。对于涉及流式传输的调用，都将使用 WebSocket 进行通信。所有的请求和回复都编码为 Protobuf 二进制格式。所有 API 调用都封装为 `Observable` 了。

所有的 API 请求都应该携带 `authorization` 头，内容为 `bearer ${token}`，其中 `token` 是登录后获取的 `token`。另外，如果 `token` 有效，所有的回复都会携带一个新的 `token` 在 `token` 头中，这个新的 `token` 延长了原有 `token` 的有效期，可以替换原来的 `token`。

相关代码位于 `src/app/api` 文件夹中，`src/app/api/proto` 存放的是 Protobuf 生成的代码。

## 文件上传

提交文件上传使用了 `webkitdirectory` 属性以便上传文件夹，同时也使用了 `@zip.js/zip.js` 包来解压 `.zip` 文件，同样达到上传文件夹的目的。在文件上传前，应该先通过 `CreateManifest` API 获取 `manifest_id`，然后再上传文件。上传每个文件都需要先调用 `InitUpload` 获取上传 Token，然后再使用该 Token 向服务器发送一个普通的 Multipart Form `POST` 请求上传文件。文件应该设置在 FormData 的 `file` 属性上。上传大量小文件的时候需要注意限流。目前前端会自动重试上传 5 次，作为触发服务器限流和上传过程出错的兜底策略。

## 第三方包处理

为了减小首屏体积，以及增加下载并发度，一些第三方库，例如 KaTeX、Prism 等都是在 `index.html` 中单独加载。相关的文件在 Angular 编译的时候负责拷贝到输出目录，也就是修改了 `angular.json` 中的 `assets` 配置。

## Markdown 编辑器

Markdown 编辑器使用 `@toast-ui/editor` 编辑器，相关代码位于 `src/app/common/markdown-editor/markdown-editor.component.ts` 文件中。考虑到编辑器模块较大，使用了 WebPack 的 `import()` 调用异步加载。

另外，为了实现代码高亮功能，还使用了 `@toast-ui/editor-plugin-code-syntax-highlight` 插件，调用 `Prism` 库来实现代码高亮。因为 Markdown 显示器也用了 `Prism` 库，就直接使用 `window.Prism` 实例用来渲染了。

## Markdown 显示

Markdown 渲染使用了 `ngx-markdown` 包。由于原来的包对 LaTeX 公式的分隔符自定义不太友好，所以我 Fork 了一份发布为 [`ngx-markdown-latex`](https://github.com/howardlau1999/ngx-markdown)。[主要改动](https://github.com/howardlau1999/ngx-markdown/commit/2c002eadd643d54546298716b321a375c4527e98)是将原来包里的使用正则匹配查找 LaTeX 公式的逻辑改为使用 KaTeX 的 [`auto-render` 功能](https://katex.org/docs/autorender.html)渲染。

## LaTeX 显示

前端所有的 LaTeX 公式都使用 `katex` 包以及 `katex/contrib/auto-render.js` 进行渲染。该功能封装在了 `ngx-markdown-latex` 包中的 `renderMathInElement(element, options)` 函数中，对需要渲染公式的 HTML 元素，该函数会获取其 `innerHTML`，传递给 [`auto-render`](https://katex.org/docs/autorender.html) 包进行渲染，并用渲染结果替换原来的 HTML。

界定符和其他 KaTex 配置可以使用 `options` 选项传递，目前系统使用的行内公式界定符为 `$`，块公式界定符为 `$$`。 


## 日志流显示

日志流使用了 `xterm` 包显示终端输出，页面组件就是简单地调用了初始化 API，并将收到的数据写入到 `xterm` 中。还使用了 `xterm-addon-fit` 插件，来让终端宽度自适应。

## VCD 波形查看器

为了实现在线波形查看，本系统前端集成了 `vcdrom`、`vcd-stream` 和 `@wavedrom/doppler` 库，分别用于解析和显示 `.vcd` 波形文件。相关模块位于 `src/app/common/vcd-viewer` 目录下。波形查看器会使用 `fetch` 获取 URL 指定的文件，并在下载过程中流式解析 `.vcd` 文件。为了实时展示进度，使用了 `requestIdleCallback` 调用，在 UI 刷新后才触发解析，避免页面卡顿。

`vcd-stream` 在 Windows 下无法正常安装，所以没有保存到 `package.json` 文件里。但是可以通过 `npm install --ignore-scripts --no-save vcd-stream` 命令来安装，否则 Angular 会因为缺少相关文件无法正常编译。

由于 `vcd-stream` 不是 ES6 模块，也没有 `.d.ts` 类型声明，不能直接 `import`，所以使用了 `require` 加载。而 `vcd-stream` 使用了 WASM 技术来加速解析，会依赖 `path` 等 Node.js 环境下的库，所以还需要额外安装它们对应的浏览器版本的包，并配置 TypeScript 编译器使用它们替换原来的包。WebPack 则会自动处理 `require` 调用。

另外，`vcd-stream` 使用的 WASM 文件，需要配置 `angular.json` 在编译的时候拷贝到输出目录。

=== "tsconfig.json"
    ```json
    {
        "compilerOptions": {
            "paths": {
                "crypto": ["./node_modules/crypto-browserify"],
                "stream": ["./node_modules/stream-browserify"],
                "path": ["./node_modules/path-browserify"],
                "fs": ["./node_modules/fs-web"]
            },
        }
    }
    ```

调库封装代码位于 `src/app/common/vcd-viewer/vcdrom.js` 文件中。