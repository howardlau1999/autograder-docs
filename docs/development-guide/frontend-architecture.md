# 前端架构介绍

本系统前端使用 [Angular](https://angular.io) 框架以及 [Angular Material](https://material.angular.io) UI 框架开发，为 SPA 应用。

前端路由存放在 `src/app/app-routing.module.ts` 文件中，为了减小首页体积，采用了懒加载技术，将不同页面分为不同子模块，并在首页加载完毕后再进行预载，优化响应速度。其他页面都在自己的 `*-routing.module.ts` 文件里定义子路由。

## Markdown 编辑器

Markdown 编辑器使用 `@toast-ui/editor` 编辑器，相关代码位于 `src/app/common/markdown-editor/markdown-editor.component.ts` 文件中。考虑到编辑器模块较大，使用了 WebPack 的 `import()` 调用异步加载。

## LaTeX 显示

前端所有的 LaTeX 公式都使用 `katex` 包以及 `katex/contrib/auto-render.js` 进行渲染。该功能封装在了 `ngx-markdown-latex` 包中的 `renderMathInElement(element, options)` 函数中，对需要渲染公式的 HTML 元素，该函数会获取其 `innerHTML`，传递给 `auto-render.js` 包进行渲染，并用渲染结果替换原来的 HTML。

界定符和其他 KaTex 配置可以使用 `options` 选项传递，目前系统使用的行内公式界定符为 `$`，块公式界定符为 `$$`。 

## VCD 波形查看器

为了实现在线波形查看，本系统前端集成了 `vcdrom`、`vcd-stream` 和 `@wavedrom/doppler` 库，分别用于解析和显示 `.vcd` 波形文件。相关模块位于 `src/app/common/vcd-viewer` 目录下。波形查看器会使用 `fetch` 获取 URL 指定的文件，并在下载过程中流式解析 `.vcd` 文件。为了实时展示进度，使用了 `requestIdleCallback` 调用，在 UI 刷新后才触发解析，避免页面卡顿。

由于 `vcd-stream` 不是 ES6 模块，也没有 `.d.ts` 类型声明，不能直接 `import`，所以使用了 `require` 加载。而 `vcd-stream` 使用了 WASM 技术来加速解析，会依赖 `path` 等 Node.js 环境下的库，所以还需要额外安装它们对应的浏览器版本的包，并配置 TypeScript 编译器使用它们替换原来的包。WebPack 则会自动处理 `require` 调用。

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