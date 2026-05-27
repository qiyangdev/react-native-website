---
id: fabric-renderer
title: Fabric
---

Fabric 是 React Native 的新渲染系统，可以理解为旧渲染系统在概念上的演进。它的核心原则是：在 C++ 中统一更多渲染逻辑，提升与[宿主平台](architecture-glossary.md#host-platform)的互操作能力，并为 React Native 解锁新的能力。Fabric 的开发始于 2018 年；到 2021 年，Facebook 应用中的 React Native 已经由新的渲染器支撑。

这份文档概述了[新渲染器](architecture-glossary.md#fabric-render)及其概念。它会避免平台细节，也不包含代码片段或指针说明。本文覆盖关键概念、动机、收益，以及不同场景下渲染流水线的概览。

## 新渲染器的动机和收益

新的渲染架构是为了实现旧架构无法支持的更好用户体验。例子包括：

- 通过改进[宿主视图](architecture-glossary.md#host-view-tree-and-host-view)和 React 视图之间的互操作能力，渲染器可以同步测量和渲染 React surface。在旧架构中，React Native 布局是异步的，因此把 React Native 渲染的视图嵌入到*宿主视图*时，可能出现布局“跳动”。
- 支持多优先级事件和同步事件后，渲染器可以优先处理某些用户交互，确保它们被及时响应。
- [与 React Suspense 集成](https://reactjs.org/blog/2019/11/06/building-great-user-experiences-with-concurrent-mode-and-suspense.html)，让 React 应用中的数据获取设计更直观。
- 在 React Native 上启用 React [并发特性](https://github.com/reactwg/react-18/discussions/4)。
- 更容易为 React Native 实现服务端渲染。

新架构还在代码质量、性能和可扩展性方面带来收益：

- **类型安全：** 通过代码生成确保 JS 和[宿主平台](architecture-glossary.md#host-platform)之间的类型安全。代码生成以 JavaScript 组件声明作为事实来源，生成用于保存 props 的 C++ struct。JavaScript 与宿主组件 props 不匹配时会触发构建错误。
- **共享 C++ 核心：** 渲染器用 C++ 实现，核心在多个平台之间共享。这会提升一致性，并让 React Native 更容易接入新平台。
- **更好的宿主平台互操作能力：** 同步且线程安全的布局计算能改善把宿主组件嵌入 React Native 时的用户体验，也意味着更容易集成那些要求同步 API 的宿主平台框架。
- **性能提升：** 有了新的跨平台渲染系统实现，每个平台都能受益于性能改进，即使这些改进最初是为了解决某个平台的限制。例如，视图扁平化最初是 Android 上的性能优化方案，现在 Android 和 iOS 默认都能使用。
- **一致性：** 新渲染系统是跨平台的，因此更容易在不同平台之间保持一致。
- **更快启动：** 宿主组件默认延迟初始化。
- **减少 JS 和宿主平台之间的数据序列化：** React 过去会把 JavaScript 和*宿主平台*之间的数据以序列化 JSON 的形式传递。新渲染器通过 [JavaScript Interfaces（JSI）](architecture-glossary.md#javascript-interfaces-jsi)直接访问 JavaScript 值，改进了数据传输方式。
