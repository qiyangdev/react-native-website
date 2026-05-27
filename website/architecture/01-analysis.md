# 翻译分析

Source: `website/architecture/`
Target language: `zh-CN`
Mode: `normal`
Audience: `technical`
Style: `technical`

## 内容定位

这组文章是 React Native 网站中的架构文档，面向库作者、核心贡献者，以及需要理解 React Native 内部机制的应用开发者。核心主题包括新架构、Fabric 渲染器、渲染流水线、线程模型、视图扁平化、跨平台 C++ 实现，以及 Bundled Hermes 的构建和发布机制。

## 语气和格式

- 保持技术文档风格，句子尽量直接、清晰。
- 保留 Markdown、MDX import、admonition、HTML 标签、图片、代码块、diff、链接和文件路径。
- 术语首次出现时尽量保留英文原词，后续使用稳定中文译名。
- 代码、API、组件名、文件名、命令、环境变量、包名不翻译。
- 翻译标题时保留原始 heading id，避免文档内锚点链接失效。

## 术语表

| English                     | 中文                                |
| --------------------------- | ----------------------------------- |
| New Architecture            | 新架构                              |
| legacy architecture         | 旧架构                              |
| Fabric Renderer             | Fabric 渲染器                       |
| renderer                    | 渲染器                              |
| render pipeline             | 渲染流水线                          |
| render phase                | 渲染阶段                            |
| commit phase                | 提交阶段                            |
| mount phase                 | 挂载阶段                            |
| host platform               | 宿主平台                            |
| host view                   | 宿主视图                            |
| Host View Tree              | 宿主视图树                          |
| React Element Tree          | React 元素树                        |
| React Element               | React 元素                          |
| React Shadow Tree           | React Shadow Tree（React 影子树）   |
| React Shadow Node           | React Shadow Node（React 影子节点） |
| React Host Component        | React 宿主组件                      |
| React Composite Component   | React 复合组件                      |
| Yoga Tree                   | Yoga 树                             |
| Yoga Node                   | Yoga 节点                           |
| View Flattening             | 视图扁平化                          |
| Layout-Only                 | 仅布局                              |
| Tree Diffing                | 树差异比较                          |
| Tree Promotion              | 树提升                              |
| Layout Calculation          | 布局计算                            |
| JavaScript Interface / JSI  | JavaScript Interface（JSI）         |
| Java Native Interface / JNI | Java Native Interface（JNI）        |
| automatic batching          | 自动批处理                          |
| concurrent renderer         | 并发渲染器                          |
| Transitions                 | Transitions                         |
| Bundled Hermes              | Bundled Hermes                      |
| ABI incompatibility         | ABI 不兼容                          |
