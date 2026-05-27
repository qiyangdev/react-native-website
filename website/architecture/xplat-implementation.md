---
id: xplat-implementation
title: 跨平台实现
---

import FabricWarning from './\_fabric-warning.mdx';

<FabricWarning />

#### React Native 渲染器使用一个可跨平台共享的核心渲染实现。

在 React Native 之前的渲染系统中，**[React Shadow Tree](architecture-glossary.md#react-shadow-tree-and-react-shadow-node)**、布局逻辑和 **[View Flattening](view-flattening.md)** 算法都需要为每个平台各自实现一次。当前渲染器被设计为跨平台方案，它共享一套核心 C++ 实现。

React Native 团队计划把动画系统纳入渲染系统，也计划把 React Native 渲染系统扩展到 Windows、游戏主机和电视等操作系统以及更多新平台。

使用 C++ 实现核心渲染系统有几个优势。单一实现降低了开发和维护成本。它提升了创建 React Shadow Tree 和进行布局计算的性能，因为在 Android 上，[Yoga](architecture-glossary.md#yoga-tree-and-yoga-node) 与渲染器集成时的开销被最小化了，也就是 Yoga 不再需要经过 [JNI](architecture-glossary.md#java-native-interface-jni)。最后，与从 Kotlin 或 Swift 分配对象相比，每个 React Shadow Node 在 C++ 中的内存占用更小。

团队还利用 C++ 中强制不可变性的特性，确保不会出现与并发访问共享但未受保护资源相关的问题。

需要认识到，在 Android 上使用渲染器时，以下两个主要场景仍然会产生 [JNI](architecture-glossary.md#java-native-interface-jni) 成本：

- 复杂视图（例如 `Text`、`TextInput` 等）的布局计算需要通过 JNI 发送 props。
- 挂载阶段需要通过 JNI 发送 mutation operations。

团队正在探索用一种基于 `ByteBuffer` 的新序列化机制替换 `ReadableMap`，以降低 JNI 开销。我们的目标是把 JNI 开销降低 35-50%。

渲染器的 C++ API 分为两侧：

- **(i)** 与 React 通信
- **(ii)** 与宿主平台通信

对于 **(i)**，React 与渲染器通信，用于**渲染**一棵 React Tree，并“监听”**事件**，例如 `onLayout`、`onKeyPress`、touch 等。

对于 **(ii)**，React Native 渲染器与宿主平台通信，把宿主视图挂载到屏幕上，包括创建、插入、更新或删除宿主视图；同时它也监听由用户在宿主平台上触发的**事件**。

![跨平台实现图](/docs/assets/Architecture/xplat-implementation/xplat-implementation-diagram.png)
