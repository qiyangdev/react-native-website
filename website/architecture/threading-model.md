---
id: threading-model
title: 线程模型
---

import FabricWarning from './\_fabric-warning.mdx';

<FabricWarning />

#### React Native 渲染器会把[渲染流水线](render-pipeline.md)的工作分布到多个线程上。

下面我们会定义线程模型，并通过几个例子说明渲染流水线如何使用线程。

React Native 渲染器被设计为线程安全。在高层上，线程安全依靠框架内部的不可变数据结构来保证，这一点由 C++ 的 “const correctness” 特性强制执行。这意味着 React 中的每次更新都会在渲染器中创建或克隆新对象，而不是更新已有数据结构。这样，框架就可以向 React 暴露线程安全且同步的 API。

渲染器使用两个不同的线程：

- **UI 线程**（通常称为 main）：唯一可以操作宿主视图的线程。
- **JavaScript 线程**：React 的渲染阶段和布局都在这里执行。

下面回顾每个阶段支持的执行场景：

<figure>
  <img src="/docs/assets/Architecture/threading-model/symbols.png" alt="线程模型符号" />
</figure>

## 渲染场景

### 在 JS 线程中渲染

这是最常见的场景，渲染流水线的大部分工作都发生在 JavaScript 线程上。

<figure>
	<img src="/docs/assets/Architecture/threading-model/case-1.jpg" alt="线程模型用例一" />
</figure>

---

### 在 UI 线程中渲染

当 UI 线程上发生高优先级事件时，渲染器可以在 UI 线程上同步执行整条渲染流水线。

<figure>
	<img src="/docs/assets/Architecture/threading-model/case-2.jpg" alt="线程模型用例二" />
</figure>

---

### 默认或连续事件中断

这个场景展示了 UI 线程上的低优先级事件如何中断渲染阶段。React 和 React Native 渲染器可以中断渲染阶段，并把它的状态与在 UI 线程上执行的低优先级事件合并。在这种情况下，渲染过程会继续在 JS 线程中执行。

<figure>
	<img src="/docs/assets/Architecture/threading-model/case-3.jpg" alt="线程模型用例三" />
</figure>

---

### 离散事件中断

渲染阶段可以被中断。这个场景展示了 UI 线程上的高优先级事件如何中断渲染阶段。React 和渲染器可以中断渲染阶段，并把它的状态与在 UI 线程上执行的高优先级事件合并。随后，渲染阶段会在 UI 线程上同步执行。

<figure>
	<img src="/docs/assets/Architecture/threading-model/case-4.jpg" alt="线程模型用例四" />
</figure>

---

### C++ State 更新

更新源自 UI 线程，并跳过渲染阶段。更多细节见 [React Native 渲染器状态更新](render-pipeline.md#react-native-renderer-state-updates)。

<figure>
	<img src="/docs/assets/Architecture/threading-model/case-6.jpg" alt="线程模型用例六" />
</figure>
