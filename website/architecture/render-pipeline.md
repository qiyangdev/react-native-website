---
id: render-pipeline
title: 渲染、提交和挂载
---

import FabricWarning from './\_fabric-warning.mdx';

<FabricWarning />

React Native 渲染器会执行一系列工作，把 React 逻辑渲染到[宿主平台](architecture-glossary.md#host-platform)。这一系列工作称为渲染流水线，它会发生在初始渲染以及 UI 状态更新时。本文介绍渲染流水线，以及它在这些场景中的差异。

渲染流水线可以拆成三个通用阶段：

1. **Render（渲染）：** React 执行业务逻辑，并在 JavaScript 中创建 [React Element Tree](architecture-glossary.md#react-element-tree-and-react-element)。渲染器根据这棵树在 C++ 中创建 [React Shadow Tree](architecture-glossary.md#react-shadow-tree-and-react-shadow-node)。
2. **Commit（提交）：** React Shadow Tree 完整创建后，渲染器触发一次 commit。这会把 React Element Tree 和新创建的 React Shadow Tree 都**提升**为将要挂载的 “next tree”。同时，它也会调度这棵树的布局信息计算。
3. **Mount（挂载）：** React Shadow Tree 此时已经带有布局计算结果，会被转换成 [Host View Tree](architecture-glossary.md#host-view-tree-and-host-view)。

> 渲染流水线的不同阶段可能发生在不同线程上。更多细节见[线程模型](threading-model.md)文档。

![React Native 渲染器数据流](/docs/assets/Architecture/renderer-pipeline/data-flow.jpg)

---

## 初始渲染

假设你想渲染下面的内容：

```jsx
function MyComponent() {
  return (
    <View>
      <Text>Hello, World</Text>
    </View>
  );
}

// <MyComponent />
```

在上面的例子中，`<MyComponent />` 是一个 [React Element](architecture-glossary.md#react-element-tree-and-react-element)。React 会递归地归约这个 _React Element_：通过调用它（如果用 JavaScript class 实现，则调用它的 `render` 方法），直到每个 _React Element_ 都无法再继续归约，最终得到终端的 [React Host Component](architecture-glossary.md#react-host-components-or-host-components)。这时你就得到了一棵由 [React Host Component](architecture-glossary.md#react-host-components-or-host-components) 组成的 _React Element Tree_。

### 阶段 1：Render

![阶段一：render](/docs/assets/Architecture/renderer-pipeline/phase-one-render.png)

在元素归约过程中，每当一个 _React Element_ 被调用时，渲染器也会同步创建一个 [React Shadow Node](architecture-glossary.md#react-shadow-tree-and-react-shadow-node)。这只会发生在 _React Host Component_ 上，不会发生在 [React Composite Component](architecture-glossary.md#react-composite-components) 上。在上面的例子中，`<View>` 会创建一个 `ViewShadowNode` 对象，`<Text>` 会创建一个 `TextShadowNode` 对象。需要注意的是，永远不会存在一个直接表示 `<MyComponent>` 的 _React Shadow Node_。

每当 React 在两个 _React Element Node_ 之间创建父子关系时，渲染器也会在对应的 _React Shadow Node_ 之间创建相同的关系。_React Shadow Tree_ 就是这样组装起来的。

**补充细节**

- 这些操作（创建 _React Shadow Node_，在两个 _React Shadow Node_ 之间创建父子关系）都是同步且线程安全的操作，通常在 JavaScript 线程上从 React（JavaScript）进入渲染器（C++）执行。
- _React Element Tree_（以及其中的 _React Element Node_）不会永久存在。它是 React 中由 “fiber” 物化出的临时表示。每个代表宿主组件的 “fiber” 都会保存一个指向 _React Shadow Node_ 的 C++ 指针，这由 JSI 实现。[在这篇文档中进一步了解 “fiber”。](https://github.com/acdlite/react-fiber-architecture#what-is-a-fiber)
- _React Shadow Tree_ 是不可变的。为了更新任意 _React Shadow Node_，渲染器会创建一棵新的 _React Shadow Tree_。不过，渲染器提供了克隆操作，让状态更新更高效。更多细节见 [React 状态更新](render-pipeline.md#react-state-updates)。

在上面的例子中，render 阶段的结果如下：

![步骤一](/docs/assets/Architecture/renderer-pipeline/render-pipeline-1.png)

_React Shadow Tree_ 完成后，渲染器会触发 _React Element Tree_ 的 commit。

### 阶段 2：Commit

![阶段二：commit](/docs/assets/Architecture/renderer-pipeline/phase-two-commit.png)

commit 阶段包含两个操作：_Layout Calculation_ 和 _Tree Promotion_。

- **Layout Calculation（布局计算）：** 这个操作会计算每个 _React Shadow Node_ 的位置和大小。在 React Native 中，这会调用 Yoga 来计算每个 _React Shadow Node_ 的布局。实际计算需要每个 _React Shadow Node_ 的样式，而这些样式来自 JavaScript 中的 _React Element_。它还需要 _React Shadow Tree_ 根节点的布局约束，这会决定最终节点可占用的空间。

![步骤二](/docs/assets/Architecture/renderer-pipeline/render-pipeline-2.png)

- **Tree Promotion（New Tree → Next Tree）：** 这个操作会把新的 _React Shadow Tree_ 提升为将要挂载的 “next tree”。这个提升动作表示新的 _React Shadow Tree_ 已经具备挂载所需的全部信息，并且代表 _React Element Tree_ 的最新状态。“next tree” 会在 UI 线程的下一个 “tick” 被挂载。

**补充细节**

- 这些操作会在后台线程上异步执行。
- 大部分布局计算完全在 C++ 中执行。不过，有些组件的布局计算依赖*宿主平台*，例如 `Text`、`TextInput` 等。文本的大小和位置与各个*宿主平台*相关，需要在*宿主平台*层计算。为此，Yoga 会调用一个由*宿主平台*定义的函数来计算组件布局。

### 阶段 3：Mount

![阶段三：mount](/docs/assets/Architecture/renderer-pipeline/phase-three-mount.png)

mount 阶段会把 _React Shadow Tree_（此时已经包含布局计算数据）转换成屏幕上带有渲染像素的 _Host View Tree_。回顾一下，_React Element Tree_ 长这样：

```jsx
<View>
  <Text>Hello, World</Text>
</View>
```

在高层上，React Native 渲染器会为每个 _React Shadow Node_ 创建一个对应的 [Host View](architecture-glossary.md#host-view-tree-and-host-view)，并把它挂载到屏幕上。在上面的例子中，渲染器会为 `<View>` 创建一个 `android.view.ViewGroup` 实例，为 `<Text>` 创建一个 `android.widget.TextView` 实例，并填入 “Hello World”。在 iOS 上则类似，会创建一个 `UIView`，并通过调用 `NSLayoutManager` 填入文本。随后，每个宿主视图都会使用它的 React Shadow Node 中的 props 进行配置，并使用计算出的布局信息配置大小和位置。

![步骤二](/docs/assets/Architecture/renderer-pipeline/render-pipeline-3.png)

更具体地说，mount 阶段包含以下三个步骤：

- **Tree Diffing（树差异比较）：** 这一步完全在 C++ 中计算 “previously rendered tree” 和 “next tree” 之间的 diff。结果是一组将要在宿主视图上执行的原子 mutation operation，例如 `createView`、`updateView`、`removeView`、`deleteView` 等。这一步也会对 React Shadow Tree 进行扁平化，以避免创建不必要的宿主视图。关于这个算法的细节，见[视图扁平化](view-flattening.md)。
- **Tree Promotion（Next Tree → Rendered Tree）：** 这一步以原子方式把 “next tree” 提升为 “previously rendered tree”，这样下一次 mount 阶段就能基于正确的树计算 diff。
- **View Mounting（视图挂载）：** 这一步会把原子 mutation operation 应用到对应的宿主视图上。它会在*宿主平台*的 UI 线程上执行。

**补充细节**

- 这些操作会在 UI 线程上同步执行。如果 commit 阶段在后台线程执行，mounting 阶段会被调度到 UI 线程的下一个 “tick”。另一方面，如果 commit 阶段在 UI 线程执行，mounting 阶段也会在同一线程上同步执行。
- mounting 阶段的调度、实现和执行高度依赖*宿主平台*。例如，mounting 层的渲染器架构目前在 Android 和 iOS 之间有所不同。
- 在初始渲染期间，“previously rendered tree” 为空。因此，tree diffing 步骤得到的 mutation operation 列表只包含创建视图、设置 props、把视图彼此添加起来等操作。处理 [React 状态更新](#react-state-updates)时，tree diffing 对性能会更重要。
- 在当前的生产测试中，一棵 _React Shadow Tree_ 通常包含大约 600-1000 个 _React Shadow Node_（视图扁平化之前）；经过视图扁平化后，树会减少到约 200 个节点。在 iPad 或桌面应用中，这个数量可能增加 10 倍。

---

## React 状态更新

接下来看看当 _React Element Tree_ 的状态更新时，渲染流水线的每个阶段会如何运行。假设你已经在初始渲染中渲染了下面的组件：

```jsx
function MyComponent() {
  return (
    <View>
      <View
        style={{backgroundColor: 'red', height: 20, width: 20}}
      />
      <View
        style={{backgroundColor: 'blue', height: 20, width: 20}}
      />
    </View>
  );
}
```

根据[初始渲染](#initial-render)部分的描述，你会预期创建出下面几棵树：

![渲染流水线 4](/docs/assets/Architecture/renderer-pipeline/render-pipeline-4.png)

注意，**Node 3** 映射到一个带有**红色背景**的宿主视图，**Node 4** 映射到一个带有**蓝色背景**的宿主视图。假设 JavaScript 业务逻辑中的一次状态更新，使第一个嵌套 `<View>` 的背景从 `'red'` 变成 `'yellow'`。新的 _React Element Tree_ 可能会像这样：

```jsx
<View>
  <View
    style={{backgroundColor: 'yellow', height: 20, width: 20}}
  />
  <View
    style={{backgroundColor: 'blue', height: 20, width: 20}}
  />
</View>
```

**React Native 会如何处理这次更新？**

状态更新发生时，渲染器从概念上需要更新 _React Element Tree_，以便更新已经挂载的宿主视图。但为了保持线程安全，_React Element Tree_ 和 _React Shadow Tree_ 都必须是不可变的。这意味着，React 不会直接修改当前的 _React Element Tree_ 和 _React Shadow Tree_，而是会为每棵树创建一个新副本，并把新的 props、样式和 children 合并进去。

下面看看状态更新期间渲染流水线的每个阶段。

### 阶段 1：Render

![阶段一：render](/docs/assets/Architecture/renderer-pipeline/phase-one-render.png)

当 React 创建一棵包含新状态的 _React Element Tree_ 时，它必须克隆所有受这次变更影响的 _React Element_ 和 _React Shadow Node_。克隆完成后，新的 _React Shadow Tree_ 会被提交。

React Native 渲染器利用结构共享来降低不可变性带来的开销。当某个 _React Element_ 被克隆以包含新状态时，从它到根节点路径上的每个 _React Element_ 都会被克隆。**只有当一个 React Element 的 props、style 或 children 需要更新时，React 才会克隆它。** 状态更新未影响的所有 _React Element_ 会由新旧两棵树共享。

在上面的例子中，React 会使用这些操作创建新树：

1. CloneNode(**Node 3**, `{backgroundColor: 'yellow'}`) → **Node 3'**
2. CloneNode(**Node 2**) → **Node 2'**
3. AppendChild(**Node 2'**, **Node 3'**)
4. AppendChild(**Node 2'**, **Node 4**)
5. CloneNode(**Node 1**) → **Node 1'**
6. AppendChild(**Node 1'**, **Node 2'**)

这些操作完成后，**Node 1'** 表示新的 _React Element Tree_ 的根。我们把 “previously rendered tree” 记为 **T**，把 “new tree” 记为 **T'**：

![渲染流水线 5](/docs/assets/Architecture/renderer-pipeline/render-pipeline-5.png)

注意，**T** 和 **T'** 共享 **Node 4**。结构共享可以提升性能，并减少内存使用。

### 阶段 2：Commit

![阶段二：commit](/docs/assets/Architecture/renderer-pipeline/phase-two-commit.png)

React 创建新的 _React Element Tree_ 和 _React Shadow Tree_ 后，必须提交它们。

- **Layout Calculation（布局计算）：** 类似于[初始渲染](#initial-render)中的布局计算。一个重要区别是，布局计算可能会导致共享的 _React Shadow Node_ 被克隆。如果某个共享 _React Shadow Node_ 的父节点发生布局变化，那么这个共享 _React Shadow Node_ 的布局也可能发生变化。
- **Tree Promotion（New Tree → Next Tree）：** 类似于[初始渲染](#initial-render)中的 Tree Promotion。

### 阶段 3：Mount

![阶段三：mount](/docs/assets/Architecture/renderer-pipeline/phase-three-mount.png)

- **Tree Promotion（Next Tree → Rendered Tree）：** 这一步以原子方式把 “next tree” 提升为 “previously rendered tree”，这样下一次 mount 阶段就能基于正确的树计算 diff。
- **Tree Diffing（树差异比较）：** 这一步计算 “previously rendered tree”（**T**）和 “next tree”（**T'**）之间的 diff。结果是一组将要在*宿主视图*上执行的原子 mutation operation。
  - 在上面的例子中，操作包括：`UpdateView(**Node 3**, {backgroundColor: 'yellow'})`
  - diff 可以在任意当前已挂载的树和任意新树之间计算。渲染器可以跳过某些中间版本的树。
- **View Mounting（视图挂载）：** 这一步会把原子 mutation operation 应用到对应的*宿主视图*上。在上面的例子中，只有 **View 3** 的 `backgroundColor` 会被更新为 yellow。

![渲染流水线 6](/docs/assets/Architecture/renderer-pipeline/render-pipeline-6.png)

---

## React Native 渲染器状态更新

对于 _Shadow Tree_ 中的大多数信息，React 是唯一所有者和唯一事实来源。所有数据都源自 React，并且数据流是单向的。

不过，有一个例外，也是一项重要机制：C++ 中的组件可以包含不直接暴露给 JavaScript 的状态，而 JavaScript 不是这些状态的事实来源。C++ 和*宿主平台*会控制这种 _C++ State_。通常，只有当你正在开发一个需要 _C++ State_ 的复杂*宿主组件*时，这才相关。绝大多数*宿主组件*并不需要这个功能。

例如，`ScrollView` 使用这套机制把当前 offset 告诉渲染器。更新由*宿主平台*触发，具体来说，是由代表 `ScrollView` 组件的宿主视图触发。offset 信息会被 [measure](https://reactnative.dev/docs/direct-manipulation#measurecallback) 这类 API 使用。由于这次更新源自宿主平台，并且不会影响 React Element Tree，所以这份状态数据会保存在 _C++ State_ 中。

从概念上说，_C++ State_ 更新类似于上面描述的 [React 状态更新](render-pipeline.md#react-state-updates)。
但有两个重要区别：

1. 它们会跳过 “render phase”，因为 React 不参与。
2. 更新可以源自任意线程，并在任意线程上发生，包括主线程。

### 阶段 2：Commit

![阶段二：commit](/docs/assets/Architecture/renderer-pipeline/phase-two-commit.png)

执行 _C++ State_ 更新时，一段代码会请求更新某个 `ShadowNode`（**N**），把 _C++ State_ 设置为值 **S**。React Native 渲染器会反复尝试获取 **N** 的最新已提交版本，用新状态 **S** 克隆它，并把 **N'** 提交到树中。如果 React 或另一次 _C++ State_ 更新在此期间执行了另一笔 commit，这次 _C++ State_ commit 就会失败，渲染器会多次重试 _C++ State_ 更新，直到 commit 成功。这样可以防止事实来源冲突和竞态。

### 阶段 3：Mount

![阶段三：mount](/docs/assets/Architecture/renderer-pipeline/phase-three-mount.png)

_Mount Phase_ 实际上与 [React 状态更新中的 Mount Phase](#react-state-updates) 相同。渲染器仍然需要重新计算布局、执行 tree diff 等。细节见上面的各节。
