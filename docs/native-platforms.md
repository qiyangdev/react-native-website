---
id: native-platform
title: 原生平台
---

你的应用可能需要访问一些平台功能，而这些功能无法直接通过 react-native 或社区维护的数百个[第三方库](https://reactnative.directory/)获得。也许你想在 JavaScript 运行时中复用已有的 Objective-C、Swift、Java、Kotlin 或 C++ 代码。不管原因是什么，React Native 都提供了一组强大的 API，用来把原生代码连接到 JavaScript 应用代码。

本指南将介绍：

- **原生模块（Native Modules）：** 不向用户提供用户界面（UI）的原生库，例如持久化存储、通知、网络事件等。用户可以通过 JavaScript 函数和对象访问这些能力。
- **原生组件（Native Component）：** 原生平台的视图、组件和控制器，可通过 React 组件暴露给应用的 JavaScript 代码使用。

:::note
你以前可能熟悉：

- [旧版原生模块](./legacy/native-modules-intro)；
- [旧版原生组件](./legacy/native-components-android)；

这些是已经弃用的原生模块和组件 API。借助我们的互操作层，你仍然可以在新架构中使用许多旧版库。你应该考虑：

- 使用替代库；
- 升级到对新架构提供一等支持的新版本库；或
- 自行将这些库迁移为 Turbo Native Modules 或 Fabric Native Components。

:::

1. 原生模块（Native Modules）
   - [Android & iOS](turbo-native-modules.md)
   - [使用 C++ 跨平台](the-new-architecture/pure-cxx-modules.md)
   - [高级：自定义 C++ 类型](the-new-architecture/custom-cxx-types.md)
2. Fabric 原生组件（Fabric Native Components）
   - [Android & iOS](fabric-native-components.md)
