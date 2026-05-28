---
id: fabric-native-components-introduction
title: Fabric 原生组件简介
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import constants from '@site/core/TabsConstants';
import {FabricNativeComponentsAndroid,FabricNativeComponentsIOS} from './\_fabric-native-components';

# 原生组件

如果你想构建*新的* React Native 组件，用来包装某个[宿主组件（Host Component）](https://reactnative.dev/architecture/glossary#host-view-tree-and-host-view)，例如 Android 上某种独特的 [CheckBox](https://developer.android.com/reference/androidx/appcompat/widget/AppCompatCheckBox)，或 iOS 上的 [UIButton](https://developer.apple.com/documentation/uikit/uibutton?language=objc)，那么应该使用 Fabric 原生组件（Fabric Native Component）。

本指南将通过实现一个 Web View 组件，展示如何构建 Fabric 原生组件。步骤如下：

1. 使用 Flow 或 TypeScript 定义一份 JavaScript 规范。
2. 配置依赖管理系统，让它根据提供的规范生成代码，并完成 autolink。
3. 实现原生代码。
4. 在应用中使用该组件。

你需要一个由普通模板生成的应用来使用这个组件：

```bash
npx @react-native-community/cli@latest init Demo --install-pods false
```

## 创建 WebView 组件

本指南将展示如何创建一个 Web View 组件。我们会使用 Android 的 [`WebView`](https://developer.android.com/reference/android/webkit/WebView) 组件，以及 iOS 的 [`WKWebView`](https://developer.apple.com/documentation/webkit/wkwebview?language=objc) 组件来创建它。

先创建用于存放组件代码的文件夹结构：

```bash
mkdir -p Demo/{specs,android/app/src/main/java/com/webview}
```

这会得到如下工作目录结构：

```
Demo
├── android/app/src/main/java/com/webview
└── ios
└── specs
```

- `android/app/src/main/java/com/webview` 文件夹用于存放 Android 代码。
- `ios` 文件夹用于存放 iOS 代码。
- `specs` 文件夹用于存放 Codegen 的规范文件。

## 1. 为 Codegen 定义规范

你的规范必须使用 [TypeScript](https://www.typescriptlang.org/) 或 [Flow](https://flow.org/) 定义（更多细节请参阅 [Codegen](the-new-architecture/what-is-codegen) 文档）。Codegen 会使用这份规范生成 C++、Objective-C++ 和 Java 代码，用来把平台代码连接到运行 React 的 JavaScript 运行时。

规范文件必须命名为 `<MODULE_NAME>NativeComponent.{ts|js}` 才能与 Codegen 配合使用。`NativeComponent` 后缀不仅是一种约定，Codegen 实际上也会用它来检测 spec 文件。

为我们的 WebView 组件使用下面这份规范：

<Tabs groupId="language" queryString defaultValue={constants.defaultJavaScriptSpecLanguage} values={constants.javaScriptSpecLanguages}>
<TabItem value="typescript">

```typescript title="Demo/specs/WebViewNativeComponent.ts"
import type {
  CodegenTypes,
  HostComponent,
  ViewProps,
} from 'react-native';
import {codegenNativeComponent} from 'react-native';

type WebViewScriptLoadedEvent = {
  result: 'success' | 'error';
};

export interface NativeProps extends ViewProps {
  sourceURL?: string;
  onScriptLoaded?: CodegenTypes.BubblingEventHandler<WebViewScriptLoadedEvent> | null;
}

export default codegenNativeComponent<NativeProps>(
  'CustomWebView',
) as HostComponent<NativeProps>;
```

</TabItem>
<TabItem value="flow">

```ts title="Demo/RCTWebView/js/RCTWebViewNativeComponent.js":
// @flow strict-local

import type {CodegenTypes, HostComponent, ViewProps} from 'react-native';
import {codegenNativeComponent} from 'react-native';

type WebViewScriptLoadedEvent = $ReadOnly<{|
  result: "success" | "error",
|}>;

type NativeProps = $ReadOnly<{|
  ...ViewProps,
  sourceURL?: string;
  onScriptLoaded?: CodegenTypes.BubblingEventHandler<WebViewScriptLoadedEvent>?;
|}>;

export default (codegenNativeComponent<NativeProps>(
  'CustomWebView',
): HostComponent<NativeProps>);

```

</TabItem>
</Tabs>

除 imports 以外，这份规范主要由三个部分组成：

- `WebViewScriptLoadedEvent` 是一个辅助数据类型，用于描述事件需要从原生侧传递到 JavaScript 的数据。
- `NativeProps` 定义了可以在组件上设置的 props。
- `codegenNativeComponent` 语句让我们可以为自定义组件生成代码，并定义组件名称，用于匹配原生实现。

与原生模块一样，你可以在 `specs/` 目录中放置多个规范文件。关于可用类型以及它们映射到的平台类型，请参阅[附录](appendix.md#codegen-typings)。

## 2. 配置 Codegen 运行

React Native 的 Codegen 工具会使用这份规范，为我们生成平台专属接口和样板代码。为此，Codegen 需要知道在哪里找到规范，以及应该如何处理它。更新你的 `package.json`，加入以下内容：

```json package.json
    "start": "react-native start",
    "test": "jest"
  },
  // highlight-start
  "codegenConfig": {
    "name": "AppSpec",
    "type": "components",
    "jsSrcsDir": "specs",
    "android": {
      "javaPackageName": "com.webview"
    },
    "ios": {
      "componentProvider": {
        "CustomWebView": "RCTWebView"
      }
    }
  },
  // highlight-end
  "dependencies": {
```

Codegen 配置完成后，我们需要准备原生代码，将它接入生成的代码。

注意，对于 iOS，我们会以声明式方式，将 spec 导出的 JS 组件名称（`CustomWebView`）映射到原生实现该组件的 iOS 类。

## 2. 构建原生代码

现在该编写原生平台代码了。这样，当 React 需要渲染一个视图时，平台就能创建正确的原生视图，并将它渲染到屏幕上。

你需要同时完成 Android 和 iOS 两个平台的实现。

:::note
本指南展示的是如何创建一个仅适用于新架构的原生组件。如果你需要同时支持新架构和旧架构，请参考我们的[向后兼容指南](https://github.com/reactwg/react-native-new-architecture/blob/main/docs/backwards-compat.md)。

:::

<Tabs groupId="platforms" queryString defaultValue={constants.defaultPlatform}>
    <TabItem value="android" label="Android">
        <FabricNativeComponentsAndroid />
    </TabItem>
    <TabItem value="ios" label="iOS">
        <FabricNativeComponentsIOS />
    </TabItem>
</Tabs>

## 3. 使用原生组件

最后，你可以在应用中使用这个新组件。将生成的 `App.tsx` 更新为：

```javascript title="Demo/App.tsx"
import React from 'react';
import {Alert, StyleSheet, View} from 'react-native';
import WebView from './specs/WebViewNativeComponent';

function App(): React.JSX.Element {
  return (
    <View style={styles.container}>
      <WebView
        sourceURL="https://react.dev/"
        style={styles.webview}
        onScriptLoaded={() => {
          Alert.alert('Page Loaded');
        }}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    alignContent: 'center',
  },
  webview: {
    width: '100%',
    height: '100%',
  },
});

export default App;
```

这段代码会创建一个应用，使用我们新建的 `WebView` 组件加载 `react.dev` 网站。

网页加载完成后，应用还会显示一个 alert。

## 4. 使用 WebView 组件运行应用

<Tabs groupId="platforms" queryString defaultValue={constants.defaultPlatform}>
<TabItem value="android" label="Android">
```bash
yarn run android
```
</TabItem>
<TabItem value="ios" label="iOS">
```bash
yarn run ios
```
</TabItem>
</Tabs>

|                                      Android                                      |                                     iOS                                      |
| :-------------------------------------------------------------------------------: | :--------------------------------------------------------------------------: |
| <img style={{ "max-height": "600px" }} src="/docs/assets/webview-android.webp" /> | <img style={{"max-height": "600px" }} src="/docs/assets/webview-ios.webp" /> |
