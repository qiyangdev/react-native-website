---
id: landing-page
title: 关于新架构
---

自 2018 年以来，React Native 团队一直在重新设计 React Native 的核心内部机制，让开发者能够构建质量更高的体验。截至 2024 年，这一版本的 React Native 已经经过大规模验证，并支撑着 Meta 的生产应用。

_New Architecture_ 这个术语既指新的框架架构，也指把它开源化的相关工作。

从 [React Native 0.68](/blog/2022/03/30/version-068#opting-in-to-the-new-architecture) 开始，新架构已经可以作为实验性选项启用，并且在之后的每个版本中持续改进。团队现在正在努力让它成为 React Native 开源生态的默认体验。

## 为什么需要新架构？

在使用 React Native 构建多年之后，团队发现了一组限制，它们阻碍开发者打造某些高完成度体验。这些限制源自框架既有设计本身，因此新架构是对 React Native 未来的一项投资。

新架构解锁了旧架构无法实现的能力和改进。

### 同步布局和 Effects

构建自适应 UI 体验时，经常需要测量视图的大小和位置，并据此调整布局。

今天，你可以使用 [`onLayout`](/docs/view#onlayout) 事件获取一个视图的布局信息，并进行所需调整。不过，`onLayout` 回调中的状态更新可能会在上一轮渲染已经绘制之后才生效。这意味着用户可能会在初始布局渲染和响应布局测量之间看到中间状态或视觉跳动。

在新架构中，我们可以通过同步访问布局信息，并正确调度更新，完全避免这个问题，让用户看不到任何中间状态。

<details>
<summary>示例：渲染 Tooltip</summary>

测量一个视图并把 tooltip 放到它上方，可以很好地展示同步渲染解锁了什么能力。tooltip 需要知道目标视图的位置，才能决定自己应该渲染在哪里。

在当前架构中，我们使用 `onLayout` 获取视图的测量值，然后根据视图位置更新 tooltip 的定位。

```jsx
function ViewWithTooltip() {
  // ...

  // We get the layout information and pass to Tooltip to position itself
  const onLayout = React.useCallback(event => {
    targetRef.current?.measureInWindow((x, y, width, height) => {
      // This state update is not guaranteed to run in the same commit
      // This results in a visual "jump" as the Tooltip repositions itself
      setTargetRect({x, y, width, height});
    });
  }, []);

  return (
    <>
      <View ref={targetRef} onLayout={onLayout}>
        <Text>Some content that renders a tooltip above</Text>
      </View>
      <Tooltip targetRect={targetRect} />
    </>
  );
}
```

在新架构中，我们可以使用 [`useLayoutEffect`](https://react.dev/reference/react/useLayoutEffect) 在一次 commit 中同步测量并应用布局更新，从而避免视觉“跳动”。

```jsx
function ViewWithTooltip() {
  // ...

  useLayoutEffect(() => {
    // The measurement and state update for `targetRect` happens in a single commit
    // allowing Tooltip to position itself without intermediate paints
    targetRef.current?.measureInWindow((x, y, width, height) => {
      setTargetRect({x, y, width, height});
    });
  }, [setTargetRect]);

  return (
    <>
      <View ref={targetRef}>
        <Text>Some content that renders a tooltip above</Text>
      </View>
      <Tooltip targetRect={targetRect} />
    </>
  );
}
```

<div className="TwoColumns TwoFigures">
 <figure>
  <img src="/img/new-architecture/async-on-layout.gif" alt="一个视图会移动到视口的角落和中心，tooltip 根据位置显示在它上方或下方。tooltip 会在视图移动后短暂延迟再渲染出来" />
  <figcaption>Tooltip 的异步测量与渲染。[查看代码](https://gist.github.com/lunaleaps/eabd653d9864082ac1d3772dac217ab9)。</figcaption>
</figure>
<figure>
  <img src="/img/new-architecture/sync-use-layout-effect.gif" alt="一个视图会移动到视口的角落和中心，tooltip 根据位置显示在它上方或下方。视图和 tooltip 会同步移动。" />
  <figcaption>Tooltip 的同步测量与渲染。[查看代码](https://gist.github.com/lunaleaps/148756563999c83220887757f2e549a3)。</figcaption>
</figure>
</div>

</details>

### 支持并发渲染器和相关特性

新架构支持 [React 18](https://react.dev/blog/2022/03/29/react-v18) 及更高版本中已经发布的并发渲染和相关特性。你现在可以在 React Native 代码中使用 Suspense 数据获取、Transitions 以及其他新的 React API，让 Web 和原生 React 开发中的代码与概念进一步保持一致。

并发渲染器也带来了开箱即用的改进，例如自动批处理，它可以减少 React 中的重复渲染。

<details>
<summary>示例：自动批处理</summary>

使用新架构时，你会通过 React 18 渲染器获得自动批处理。

在这个例子中，一个 slider 会指定要渲染多少个 tile。把 slider 从 0 拖到 1000 会快速触发一连串状态更新和重新渲染。

对比[同一份代码](https://gist.github.com/lunaleaps/79bb6f263404b12ba57db78e5f6f28b2)在不同渲染器下的效果时，可以直观看到新渲染器提供了更流畅的 UI，中间 UI 更新更少。来自原生事件处理器的状态更新，例如这个原生 Slider 组件，现在会被批处理。

<div className="TwoColumns TwoFigures">
 <figure>
  <img src="/img/new-architecture/legacy-renderer.gif" alt="一个视频演示应用根据 slider 输入渲染大量视图。slider 的值从 0 调整到 1000，UI 缓慢追上并渲染 1000 个视图。" />
  <figcaption>使用旧渲染器渲染频繁状态更新。</figcaption>
</figure>
<figure>
  <img src="/img/new-architecture/react18-renderer.gif" alt="一个视频演示应用根据 slider 输入渲染大量视图。slider 的值从 0 调整到 1000，UI 比前一个例子更快稳定到 1000 个视图，并且中间状态更少。" />
  <figcaption>使用 React 18 渲染器渲染频繁状态更新。</figcaption>
</figure>
</div>
</details>

新的并发特性，例如 [Transitions](https://react.dev/reference/react/useTransition)，让你能够表达 UI 更新的优先级。把某次更新标记为较低优先级，意味着 React 可以“中断”这次更新的渲染，转而处理更高优先级的更新，从而在关键场景保持响应性。

<details>
<summary>示例：使用 `startTransition`</summary>

我们可以在上一个例子的基础上展示 transitions 如何中断正在进行的渲染，以处理更新的状态变化。

我们用 `startTransition` 包裹 tile 数量的状态更新，表示渲染这些 tile 可以被中断。`startTransition` 还提供了 `isPending` 标志，用于告诉我们 transition 何时完成。

```jsx
function TileSlider({value, onValueChange}) {
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <View>
        <Text>
          Render {value} Tiles
        </Text>
        <ActivityIndicator animating={isPending} />
      </View>
      <Slider
        value={1}
        minimumValue={1}
        maximumValue={1000}
        step={1}
        onValueChange={newValue => {
          startTransition(() => {
            onValueChange(newValue);
          });
        }}
      />
    </>
  );
}

function ManyTiles() {
  const [value, setValue] = useState(1);
  const tiles = generateTileViews(value);
  return (
      <TileSlider onValueChange={setValue} value={value} />
      <View>
        {tiles}
      </View>
  )
}
```

你会注意到，当 transition 中有频繁更新时，React 渲染的中间状态更少，因为它一旦发现当前状态已经过期，就会放弃渲染。相比之下，不使用 transitions 时会渲染更多中间状态。这两个例子仍然都会使用自动批处理。不过，transitions 让开发者在批处理进行中的渲染时拥有更强控制力。

<div className="TwoColumns TwoFigures">
<figure>
  <img src="/img/new-architecture/with-transitions.gif" alt="一个视频演示应用根据 slider 输入渲染大量视图（tiles）。slider 从 0 快速调整到 1000 时，视图会分批渲染。与下一个视频相比，批量渲染次数更少。" />
  <figcaption>使用 transitions 渲染 tiles，从而中断已过期状态的进行中渲染。[查看代码](https://gist.github.com/lunaleaps/eac391bf3fe4c85953cefeb74031bab0/revisions)。</figcaption>
</figure>
<figure>
  <img src="/img/new-architecture/without-transitions.gif" alt="一个视频演示应用根据 slider 输入渲染大量视图（tiles）。slider 从 0 快速调整到 1000 时，视图会分批渲染。" />
  <figcaption>未把渲染标记为 transition 时渲染 tiles。[查看代码](https://gist.github.com/lunaleaps/eac391bf3fe4c85953cefeb74031bab0/revisions)。</figcaption>
</figure>
</div>
</details>

### 快速的 JavaScript/Native 接口

新架构移除了 JavaScript 和 native 之间的[异步 Bridge](https://reactnative.dev/blog/2018/06/14/state-of-react-native-2018#architecture)，并用 JavaScript Interface（JSI）取代它。JSI 是一种接口，允许 JavaScript 持有对 C++ 对象的引用，反过来也一样。有了内存引用，你就可以直接调用方法，而不需要承担序列化成本。

JSI 让 [VisionCamera](https://github.com/mrousavy/react-native-vision-camera) 这样的流行 React Native 相机库能够实时处理帧。典型帧缓冲区约为 30 MB，根据帧率不同，每秒数据量大约可达 2 GB。与 Bridge 的序列化成本相比，JSI 可以轻松处理这种规模的接口数据。JSI 还可以暴露其他复杂的、基于实例的类型，例如数据库、图片、音频采样等。

新架构采用 JSI 后，所有 native-JavaScript 互操作中这一类序列化工作都会被移除。这也包括初始化和重新渲染 `View`、`Text` 等原生核心组件。你可以阅读我们关于新架构[渲染性能调查](https://github.com/reactwg/react-native-new-architecture/discussions/123)以及我们测得的改进 benchmark。

## 启用新架构后我可以期待什么？

虽然新架构启用了这些特性和改进，但为你的应用或库启用新架构，不一定会立刻提升性能或用户体验。

例如，你的代码可能需要重构，才能利用同步布局 effects 或并发特性等新能力。虽然 JSI 会最小化 JavaScript 和 native 内存之间的开销，但数据序列化不一定曾经是你应用性能的瓶颈。

在你的应用或库中启用新架构，意味着选择进入 React Native 的未来方向。

团队正在积极研究和开发新架构解锁的新能力。例如，Web 对齐是 Meta 正在探索的一个活跃方向，并会发布到 React Native 开源生态中。

- [事件循环模型更新](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/0744-well-defined-event-loop.md)
- [Node 和布局 API](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/0607-dom-traversal-and-layout-apis.md)
- [样式和布局一致性](https://github.com/facebook/yoga/releases/tag/v2.0.0)

你可以在专门的 [discussions & proposals](https://github.com/react-native-community/discussions-and-proposals/discussions/651) 仓库中跟进并参与贡献。

## 今天应该使用新架构吗？

从 0.76 开始，新架构会在所有 React Native 项目中默认启用。

如果你发现任何表现不正常的地方，请使用[这个模板](https://github.com/facebook/react-native/issues/new?assignees=&labels=Needs%3A+Triage+%3Amag%3A%2CType%3A+New+Architecture&projects=&template=new_architecture_bug_report.yml)提交 issue。

如果因为任何原因你无法使用新架构，仍然可以选择关闭它：

### Android

1. 打开 `android/gradle.properties` 文件
2. 把 `newArchEnabled` 标志从 `true` 切换为 `false`

```diff title="gradle.properties"
# Use this property to enable support to the new architecture.
# This will allow you to use TurboModules and the Fabric render in
# your application. You should enable this flag either if you want
# to write custom TurboModules/Fabric components OR use libraries that
# are providing them.
-newArchEnabled=true
+newArchEnabled=false
```

### iOS

1. 打开 `ios/Podfile` 文件
2. 在 Podfile 的主作用域中添加 `ENV['RCT_NEW_ARCH_ENABLED'] = '0'`（模板中的[参考 Podfile](https://github.com/react-native-community/template/blob/0.76-stable/template/ios/Podfile)）

```diff
+ ENV['RCT_NEW_ARCH_ENABLED'] = '0'
# Resolve react_native_pods.rb with node to allow for hoisting
require Pod::Executable.execute_command('node', ['-p',
  'require.resolve(
```

3. 使用以下命令安装 CocoaPods 依赖：

```shell
bundle exec pod install
```
