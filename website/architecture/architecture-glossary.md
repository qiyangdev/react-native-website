---
id: architecture-glossary
title: 术语表
slug: /glossary
---

## Dev Menu（开发菜单）

应用内开发者菜单，在开发构建中可用，用于访问各种开发和调试操作。[在文档中进一步了解 Dev Menu](/docs/debugging)。

## Fabric Renderer（Fabric 渲染器）

React Native 和 Web 端 React 执行的是同一套 React 框架代码。不过，React Native 渲染到通用的平台视图（宿主视图），而不是 DOM 节点（可以把 DOM 节点视为 Web 的宿主视图）。Fabric Renderer 让渲染到宿主视图成为可能。Fabric 让 React 能够与各个平台通信，并管理各自的宿主视图实例。Fabric Renderer 存在于 JavaScript 中，并面向由 C++ 代码暴露的接口。[这篇博客进一步介绍了 React 渲染器。](https://overreacted.io/react-as-a-ui-runtime/#renderers)

## Host platform（宿主平台）

嵌入 React Native 的平台，例如 Android、iOS、macOS、Windows。

## Host View Tree（宿主视图树）和 Host View（宿主视图）

宿主平台中的视图树表示，例如 Android 和 iOS。在 Android 上，宿主视图是 `android.view.ViewGroup`、`android.widget.TextView` 等实例，它们构成了宿主视图树。每个宿主视图的大小和位置基于 Yoga 计算出的 `LayoutMetrics`，每个宿主视图的样式和内容则基于 React Shadow Tree 中的信息。

## JavaScript Interfaces（JSI）

一种轻量级 API，用于把 JavaScript 引擎嵌入 C++ 应用。Fabric 使用它在 Fabric 的 C++ 核心和 React 之间通信。

## Java Native Interface（JNI）

一个用于[编写 Java native 方法的 API](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/)，用于在 Fabric 的 C++ 核心和以 Java 编写的 Android 之间通信。

## React Component（React 组件）

一个 JavaScript 函数或类，用来描述如何创建 React Element。[这篇博客进一步介绍了 React 组件和元素。](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)

## React Composite Components（React 复合组件）

这类 React Component 的 `render` 实现会进一步归约为其他 React Composite Component 或 React Host Component。

## React Host Components（React 宿主组件）或 Host Components（宿主组件）

视图实现由宿主视图提供的 React Component，例如 `<View>`、`<Text>`。在 Web 上，ReactDOM 的 Host Component 对应的就是 `<p>`、`<div>` 这类组件。

## React Element Tree（React 元素树）和 React Element（React 元素）

_React Element Tree_ 由 React 在 JavaScript 中创建，由多个 React Element 组成。_React Element_ 是一个普通 JavaScript 对象，用来描述屏幕上应该出现什么内容。它包含 props、样式和 children。React Element 只存在于 JavaScript 中，可以表示 React Composite Component 或 React Host Component 的实例化结果。[这篇博客进一步介绍了 React 组件和元素。](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)

## React Native Framework（React Native 框架）

React Native 允许开发者使用 [React 编程范式](https://react.dev/learn/thinking-in-react)向原生目标交付应用。React Native 团队专注于创建适用于原生应用开发中最通用场景的**核心 API** 和**功能**。

把原生应用交付到生产环境通常还需要一组工具和库。它们默认并不属于 React Native 的一部分，但对于发布生产应用仍然很关键。例如：支持把应用发布到专门的应用商店，或者支持路由与导航机制。

当这些工具和库被组合起来，形成一个构建在 React Native 之上的完整框架时，我们将其定义为 **React Native Framework**。

[Expo](https://expo.dev/) 就是一个开源 React Native Framework 的例子。

## React Shadow Tree（React 影子树）和 React Shadow Node（React 影子节点）

_React Shadow Tree_ 由 Fabric Renderer 创建，由多个 React Shadow Node 组成。React Shadow Node 是一个对象，表示一个将被挂载的 React Host Component，并包含来自 JavaScript 的 props。它也包含布局信息（x、y、width、height）。在 Fabric 中，React Shadow Node 对象存在于 C++ 中。在 Fabric 之前，这些对象存在于移动端运行时堆中，例如 Android JVM。

## Yoga Tree（Yoga 树）和 Yoga Node（Yoga 节点）

_Yoga Tree_ 由 [Yoga](https://www.yogalayout.dev/) 用来为 React Shadow Tree 计算布局信息。每个 React Shadow Node 通常都会创建一个 _Yoga Node_，因为 React Native 使用 Yoga 来计算布局。不过，这不是硬性要求。Fabric 也可以创建不使用 Yoga 的 React Shadow Node；每个 React Shadow Node 的实现决定了它如何计算布局。
