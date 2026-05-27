---
id: architecture-overview
title: 架构概览
slug: /overview
---

:::info
欢迎阅读架构概览！如果你刚开始使用 React Native，请先参考<a href="/docs/getting-started">指南</a>部分。继续阅读本文，可以了解 React Native 内部机制是如何工作的。

本节内容仍在持续完善中，未来会加入更多资料。欢迎之后再回来查看新增内容。
:::

架构概览旨在从概念层面介绍 React Native 内部机制的工作方式。目标读者包括库作者和核心贡献者。如果你是应用开发者，熟悉这些内容并不是高效使用 React Native 的前提。不过，这份概览仍然能帮助你理解 React Native 在底层是如何运行的。你可以在本节对应的<a href="https://github.com/reactwg/react-native-new-architecture/discussions/9">工作组讨论</a>中反馈意见。

## 目录

- [关于新架构](landing-page.md)
- 渲染
  - [Fabric](fabric-renderer.md)
  - [渲染、提交和挂载](render-pipeline.md)
  - [跨平台实现](xplat-implementation.md)
  - [视图扁平化](view-flattening.md)
  - [线程模型](threading-model.md)
- 构建工具
  - [Bundled Hermes](bundled-hermes.md)
- [术语表](architecture-glossary.md)
