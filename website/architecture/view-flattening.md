---
id: view-flattening
title: 视图扁平化
---

import FabricWarning from './\_fabric-warning.mdx';

<FabricWarning />

#### 视图扁平化是 React Native 渲染器用来避免过深布局树的一种优化。

React API 被设计为声明式，并且可以通过组合复用。这为直观开发提供了很好的模型。不过在实现层面，API 的这些特性会产生很深的 [React Element Tree](architecture-glossary.md#react-element-tree-and-react-element)，其中大量 React Element Node 只影响 View 的布局，并不会在屏幕上渲染任何内容。我们把这类节点称为 **“Layout-Only”** 节点。

从概念上说，React Element Tree 中的每个节点都和屏幕上的视图存在 1:1 关系。因此，如果一个很深的 React Element Tree 由大量 “Layout-Only” 节点组成，渲染性能就会变差。

下面是一个会受到 “Layout Only” 视图成本影响的常见用例。

假设你想渲染一张图片和一个标题，标题由 `TitleComponent` 处理，并且你把这个组件作为 `ContainerComponent` 的子组件，而 `ContainerComponent` 带有一些 margin 样式。组件拆分后，React 代码会像这样：

```jsx
function MyComponent() {
  return (
    <View>                          // ReactAppComponent
      <View style={{margin: 10}} /> // ContainerComponent
        <View style={{margin: 10}}> // TitleComponent
          <Image {...} />
          <Text {...}>This is a title</Text>
        </View>
      </View>
    </View>
  );
}
```

作为渲染过程的一部分，React Native 会生成以下几棵树：

![图一](/docs/assets/Architecture/view-flattening/diagram-one.png)

注意，视图 (2) 和 (3) 是 “Layout Only” 视图。它们会被渲染到屏幕上，但它们只是在子节点之上渲染 `10 px` 的 `margin`。

为了提升这类 React Element Tree 的性能，渲染器实现了 View Flattening 机制。它会合并或扁平化这类节点，从而减少屏幕上实际渲染的[宿主视图](architecture-glossary.md#host-view-tree-and-host-view)层级深度。这个算法会考虑 `margin`、`padding`、`backgroundColor`、`opacity` 等 props。

View Flattening 算法在设计上就是渲染器 diff 阶段的一部分。这意味着我们不会额外消耗 CPU 周期来先优化 React Element Tree、再扁平化这些视图。和核心的其余部分一样，View Flattening 算法用 C++ 实现，它的收益默认在所有支持的平台上共享。

在前面的例子中，视图 (2) 和 (3) 会作为 “diffing algorithm” 的一部分被扁平化，结果是它们的样式会被合并到视图 (1) 中：

![图二](/docs/assets/Architecture/view-flattening/diagram-two.png)

需要注意的是，这项优化让渲染器避免创建和渲染两个宿主视图。从用户视角看，屏幕上不会有任何可见变化。
