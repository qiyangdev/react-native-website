---
id: turbo-native-modules-introduction
title: '原生模块：简介'
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import constants from '@site/core/TabsConstants';
import CodeBlock from '@theme/CodeBlock';
import {getCurrentVersion} from '@site/src/getCurrentVersion';
import {TurboNativeModulesAndroid, TurboNativeModulesIOS} from './\_turbo-native-modules-components';

# 原生模块

你的 React Native 应用代码可能需要与一些原生平台 API 交互，而这些 API 并未由 React Native 或现有库提供。你可以使用 **Turbo 原生模块（Turbo Native Module）** 自行编写集成代码。本指南将介绍如何编写一个 Turbo 原生模块。

基本步骤如下：

1. 使用 Flow 或 TypeScript 这类主流 JavaScript 类型标注语言，**定义一份带类型的 JavaScript 规范**；
2. **配置依赖管理系统来运行 Codegen**，它会把规范转换为原生语言接口；
3. 使用这份规范**编写应用代码**；以及
4. **使用生成的接口编写原生平台代码**，将你的原生代码接入 React Native 运行时环境。

接下来，我们通过构建一个 Turbo 原生模块示例，逐步完成这些步骤。本指南后续内容假设你已经运行以下命令创建了应用：

<CodeBlock language="bash" title="shell">
{`npx @react-native-community/cli@latest init TurboModuleExample --version ${getCurrentVersion()}`}
</CodeBlock>

## 原生持久化存储

本指南将展示如何编写 [Web Storage API](https://html.spec.whatwg.org/multipage/webstorage.html#dom-localstorage-dev) 中 `localStorage` 的实现。这个 API 对于在项目中编写应用代码的 React 开发者来说会比较熟悉。

要让它在移动端工作，我们需要使用 Android 和 iOS API：

- Android：[SharedPreferences](https://developer.android.com/reference/android/content/SharedPreferences)，以及
- iOS：[NSUserDefaults](https://developer.apple.com/documentation/foundation/nsuserdefaults)。

### 1. 声明带类型的规范

React Native 提供了一个名为 [Codegen](/docs/the-new-architecture/what-is-codegen) 的工具，它会接收用 TypeScript 或 Flow 编写的规范，并为 Android 和 iOS 生成平台专属代码。规范会声明在原生代码与 React Native JavaScript 运行时之间来回传递的方法和数据类型。一个 Turbo 原生模块同时包含你的规范、你编写的原生代码，以及 Codegen 根据规范生成的接口。

创建 specs 文件：

1. 在应用的根目录中创建一个名为 `specs` 的新文件夹。
2. 创建一个名为 `NativeLocalStorage.ts` 的新文件。

:::info
你可以在[附录](/docs/appendix)文档中查看规范里可使用的所有类型，以及生成后的原生类型。
:::

:::info
如果你想修改模块名称和相关 specs 文件名，请务必始终使用 `Native` 作为前缀（例如 `NativeStorage` 或 `NativeUsersDefault`）。
:::

下面是 `localStorage` 规范的一种实现：

<Tabs groupId="language" queryString defaultValue={constants.defaultJavaScriptSpecLanguage} values={constants.javaScriptSpecLanguages}>
<TabItem value="typescript">

```typescript title="specs/NativeLocalStorage.ts"
import type {TurboModule} from 'react-native';
import {TurboModuleRegistry} from 'react-native';

export interface Spec extends TurboModule {
  setItem(value: string, key: string): void;
  getItem(key: string): string | null;
  removeItem(key: string): void;
  clear(): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>(
  'NativeLocalStorage',
);
```

</TabItem>
<TabItem value="flow">

```flow title="NativeLocalStorage.js"
import type {TurboModule} from 'react-native';
import {TurboModule, TurboModuleRegistry} from 'react-native';

export interface Spec extends TurboModule {
  setItem(value: string, key: string): void;
  getItem(key: string): ?string;
  removeItem(key: string): void;
  clear(): void;
}
```

</TabItem>
</Tabs>

### 2. 配置 Codegen 运行

React Native Codegen 工具会使用这份规范，为我们生成平台专属接口和样板代码。为此，Codegen 需要知道在哪里找到规范，以及应该如何处理它。更新你的 `package.json`，加入以下内容：

```json title="package.json"
     "start": "react-native start",
     "test": "jest"
   },
   // highlight-add-start
   "codegenConfig": {
     "name": "NativeLocalStorageSpec",
     "type": "modules",
     "jsSrcsDir": "specs",
     "android": {
       "javaPackageName": "com.nativelocalstorage"
     }
   },
   // highlight-add-end
   "dependencies": {
```

Codegen 配置完成后，我们需要准备原生代码，将它接入生成的代码。

<Tabs groupId="platforms" queryString defaultValue={constants.defaultPlatform}>
<TabItem value="android" label="Android">
Codegen 通过 `generateCodegenArtifactsFromSchema` Gradle 任务执行：

```bash
cd android
./gradlew generateCodegenArtifactsFromSchema

BUILD SUCCESSFUL in 837ms
14 actionable tasks: 3 executed, 11 up-to-date
```

构建 Android 应用时会自动运行这项任务。
</TabItem>
<TabItem value="ios" label="iOS">
Codegen 会作为脚本阶段的一部分运行，该脚本阶段会自动添加到 CocoaPods 生成的项目中。

```bash
cd ios
bundle install
bundle exec pod install
```

输出大致如下：

```shell
...
Framework build type is static library
[Codegen] Adding script_phases to ReactCodegen.
[Codegen] Generating ./build/generated/ios/ReactCodegen.podspec.json
[Codegen] Analyzing /Users/me/src/TurboModuleExample/package.json
[Codegen] Searching for codegen-enabled libraries in the app.
[Codegen] Found TurboModuleExample
[Codegen] Searching for codegen-enabled libraries in the project dependencies.
[Codegen] Found react-native
...
```

</TabItem>
</Tabs>

### 3. 使用 Turbo 原生模块编写应用代码

下面是使用 `NativeLocalStorage` 修改后的 `App.tsx`，其中包含一段需要持久化的文本、一个输入框，以及几个用于更新这个值的按钮。

`TurboModuleRegistry` 支持通过两种模式获取 Turbo 原生模块：

- `get<T>(name: string): T | null`：如果 Turbo 原生模块不可用，则返回 `null`。
- `getEnforcing<T>(name: string): T`：如果 Turbo 原生模块不可用，则抛出异常。这种方式假设模块始终可用。

```tsx title="App.tsx"
import React from 'react';
import {
  SafeAreaView,
  StyleSheet,
  Text,
  TextInput,
  Button,
} from 'react-native';

import NativeLocalStorage from './specs/NativeLocalStorage';

const EMPTY = '<empty>';

function App(): React.JSX.Element {
  const [value, setValue] = React.useState<string | null>(null);

  const [editingValue, setEditingValue] = React.useState<
    string | null
  >(null);

  React.useEffect(() => {
    const storedValue = NativeLocalStorage?.getItem('myKey');
    setValue(storedValue ?? '');
  }, []);

  function saveValue() {
    NativeLocalStorage?.setItem(editingValue ?? EMPTY, 'myKey');
    setValue(editingValue);
  }

  function clearAll() {
    NativeLocalStorage?.clear();
    setValue('');
  }

  function deleteValue() {
    NativeLocalStorage?.removeItem('myKey');
    setValue('');
  }

  return (
    <SafeAreaView style={{flex: 1}}>
      <Text style={styles.text}>
        Current stored value is: {value ?? 'No Value'}
      </Text>
      <TextInput
        placeholder="Enter the text you want to store"
        style={styles.textInput}
        onChangeText={setEditingValue}
      />
      <Button title="Save" onPress={saveValue} />
      <Button title="Delete" onPress={deleteValue} />
      <Button title="Clear" onPress={clearAll} />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  text: {
    margin: 10,
    fontSize: 20,
  },
  textInput: {
    margin: 10,
    height: 40,
    borderColor: 'black',
    borderWidth: 1,
    paddingLeft: 5,
    paddingRight: 5,
    borderRadius: 5,
  },
});

export default App;
```

### 4. 编写原生平台代码

准备工作完成后，我们将开始编写原生平台代码。这个过程分为两部分：

:::note
本指南展示的是如何创建一个仅适用于新架构的 Turbo 原生模块。如果你需要同时支持新架构和旧架构，请参考我们的[向后兼容指南](https://github.com/reactwg/react-native-new-architecture/blob/main/docs/backwards-compat.md)。
:::

<Tabs groupId="platforms" queryString defaultValue={constants.defaultPlatform}>
    <TabItem value="android" label="Android">
        <TurboNativeModulesAndroid />
    </TabItem>
    <TabItem value="ios" label="iOS">
        <TurboNativeModulesIOS/>
    </TabItem>
</Tabs>
