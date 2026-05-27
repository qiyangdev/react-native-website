---
id: bundled-hermes
title: Bundled Hermes
---

本页概述 Hermes 和 React Native 是**如何构建**的。

如果你想了解如何在应用中使用 Hermes，可以在另一页找到说明：[使用 Hermes](/docs/hermes)。

:::caution
请注意，本页是一篇技术深潜，目标读者是正在基于 Hermes 或 React Native 构建扩展的用户。普通 React Native 用户通常不需要深入理解 React Native 与 Hermes 如何交互。
:::

## 什么是 “Bundled Hermes”

从 React Native 0.69.0 开始，每个 React Native 版本都会与某个 Hermes 版本**一起构建**。我们把这种分发模型称为 **Bundled Hermes**。

从 0.69 起，你始终可以使用一个已经和每个 React Native 版本一起构建并测试过的 JS 引擎。

## 为什么迁移到 “Bundled Hermes”

过去，React Native 和 Hermes 遵循两个**独立的发布流程**，并且各自有独立的版本号。独立发布和不同版本号在 OSS 生态中造成了困惑，因为大家并不清楚某个特定 Hermes 版本是否兼容某个特定 React Native 版本。例如，你需要知道 Hermes 0.11.0 只兼容 React Native 0.68.0。

Hermes 和 React Native 都共享 JSI 代码（[Hermes 中的代码](https://github.com/facebook/hermes/tree/main/API/jsi/jsi)和 [React Native 中的代码](https://github.com/facebook/react-native/tree/main/packages/react-native/ReactCommon/jsi/jsi)）。如果两份 JSI 拷贝不同步，某个 Hermes 构建就无法兼容某个 React Native 构建。你可以在这里进一步了解这个 [ABI 不兼容问题](https://github.com/react-native-community/discussions-and-proposals/issues/257)。

为了解决这个问题，我们扩展了 React Native 发布流程，让它下载并构建 Hermes，同时确保构建 Hermes 时只使用一份 JSI。

这样一来，我们就可以在发布 React Native 版本时同时发布一个 Hermes 版本，并确保我们构建出的 Hermes 引擎与即将发布的 React Native 版本**完全兼容**。我们会把这个 Hermes 版本随 React Native 版本一起发布，因此称为 _Bundled Hermes_。

## 这会如何影响应用开发者

正如开头提到的，如果你是应用开发者，这个变更**不应该直接影响**你。

下面几段会描述我们在底层做了哪些变更，并解释其中的一些理由，以保持透明。

### iOS 用户

在 iOS 上，我们移动了你正在使用的 `hermes-engine`。

在 React Native 0.69 之前，用户会下载一个 pod（这里可以看到对应的 [podspec](https://github.com/CocoaPods/Specs/blob/master/Specs/5/d/0/hermes-engine/0.11.0/hermes-engine.podspec.json)）。

在 React Native 0.69 上，用户会改用一个定义在 `react-native` NPM 包中 `sdks/hermes-engine/hermes-engine.podspec` 文件里的 podspec。
这个 podspec 依赖一个 Hermes 预构建 tarball。作为 React Native 发布流程的一部分，我们会把这个 tarball 上传到 Maven 和 React Native GitHub Release（例如[这个 release 的 assets](https://github.com/facebook/react-native/releases/tag/v0.70.4)）。

### Android 用户

在 Android 上，我们会按下面的方式更新默认模板中的 [`android/app/build.gradle`](https://github.com/facebook/react-native/blob/main/template/android/app/build.gradle) 文件：

```diff
dependencies {
    // ...

    if (enableHermes) {
+       implementation("com.facebook.react:hermes-engine:+") {
+           exclude group:'com.facebook.fbjni'
+       }
-       def hermesPath = "../../node_modules/hermes-engine/android/";
-       debugImplementation files(hermesPath + "hermes-debug.aar")
-       releaseImplementation files(hermesPath + "hermes-release.aar")
    } else {
        implementation jscFlavor
    }
}
```

在 React Native 0.69 之前，用户会从 `hermes-engine` NPM 包中消费 `hermes-debug.aar` 和 `hermes-release.aar`。

在 React Native 0.69 上，用户会消费 `react-native` NPM 包中 `android/com/facebook/react/hermes-engine/` 文件夹里的 Android multi-variant artifacts。
另外请注意，我们会在未来某个 React Native 版本中完全[移除对 `hermes-engine` 的依赖](https://github.com/facebook/react-native/blob/c418bf4c8fe8bf97273e3a64211eaa38d836e0a0/package.json#L105)。

#### 使用新架构的 Android 用户

由于我们的 native code build setup 的性质，也就是我们使用 NDK 的方式，使用新架构的用户会**从源码构建 Hermes**。

这让使用新架构的用户在 React Native 和 Hermes 的构建机制上保持一致，也就是两者都会从源码构建。
这意味着这类 Android 用户在第一次构建时可能会遇到构建时间上的性能损失。

你可以在这个页面找到优化构建时间、降低构建影响的说明：[加快构建阶段](/docs/next/build-speed)。

#### 在 Windows 上使用新架构构建的 Android 用户

在 Windows 机器上使用新架构构建 React Native App 的用户，需要执行这些额外步骤，才能让构建正确工作：

- 确保[环境已经正确配置](https://reactnative.dev/docs/environment-setup)，包括 Android SDK 和 node。
- 使用 Chocolatey 安装 [cmake](https://community.chocolatey.org/packages/cmake)。
- 安装以下任意一项：
  - [Build Tools for Visual Studio 2022](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022)。
  - [Visual Studio 22 Community Edition](https://visualstudio.microsoft.com/vs/community/)。只选择 C++ desktop development 即可。
- 确保 [Visual Studio Command Prompt](https://docs.microsoft.com/en-us/visualstudio/ide/reference/command-prompt-powershell?view=vs-2022) 已正确配置。这是必需的，因为正确的 C++ 编译器环境变量是在这些命令提示符中配置的。
- 在 Visual Studio Command Prompt 中运行 `npx react-native run-android` 来启动应用。

### 用户还能使用其他引擎吗？

可以，用户可以自由启用或禁用 Hermes（Android 上使用 `enableHermes` 变量，iOS 上使用 `hermes_enabled`）。
“Bundled Hermes” 这个变更只会影响 Hermes **如何为你构建和打包**。

从 React Native 0.70 开始，`enableHermes`/`hermes_enabled` 的默认值是 `true`。

## 这会如何影响贡献者和扩展开发者

如果你是 React Native 贡献者，或者正在基于 React Native 或 Hermes 构建扩展，请继续阅读。下面会解释 Bundled Hermes 是如何工作的。

### Bundled Hermes 底层是如何工作的？

这套机制依赖在 `facebook/react-native` 仓库内，从 `facebook/hermes` 仓库**下载一个包含 Hermes 源码的 tarball**。我们对其他 native 依赖（Folly、Glog 等）也有类似机制，并让 Hermes 遵循同样的设置。

从 `main` 构建 React Native 时，我们会获取 facebook/hermes 的 `main` tarball，并把它作为 React Native 构建流程的一部分来构建。

从 release branch（例如 `0.69-stable`）构建 React Native 时，我们会改用 Hermes 仓库上的一个 **tag**，在两个仓库之间**同步代码**。具体使用的 tag 名称随后会存储在 release branch 中 React Native 内部的 `sdks/.hermesversion` 文件里（例如 0.69 release branch 上的[这个文件](https://github.com/facebook/react-native/blob/0.69-stable/sdks/.hermesversion)）。

从某种意义上说，你可以把这种方式理解为类似 **git submodule**。

如果你正在基于 Hermes 构建，可以依赖这些 tag 来理解构建 React Native 时使用了哪个 Hermes 版本，因为 React Native 的版本会写在 tag 名称中，例如 `hermes-2022-05-20-RNv0.69.0-ee8941b8874132b8f83e4486b63ed5c19fc3f111`。

#### Android 实现细节

为了在 Android 上实现这一点，我们在 React Native 的 `/ReactAndroid/hermes-engine` 中新增了一个 build，负责构建 Hermes 并打包供消费使用（[更多背景见这里](https://github.com/facebook/react-native/pull/33396)）。

现在你可以通过调用以下命令触发 Hermes engine 的构建：

```bash
# Build a debug version of Hermes
./gradlew :ReactAndroid:hermes-engine:assembleDebug
# Build a release version of Hermes
./gradlew :ReactAndroid:hermes-engine:assembleRelease
```

这些命令在 React Native 的 `main` 分支中执行。

你不需要在机器上安装额外工具，例如 `cmake`、`ninja` 或 `python3`，因为我们已经配置构建流程使用这些工具的 NDK 版本。

在 Gradle consumer 侧，我们也发布了一个小改进：从 `releaseImplementation` 和 `debugImplementation` 切换到了 `implementation`。这是可行的，因为更新后的 `hermes-engine` Android artifact 是 **variant aware** 的，能够把 debug 版本的 engine 与你的应用 debug 构建正确匹配。这里不需要任何自定义配置，即使你使用 `staging` 或其他 build types/flavors。

不过，这也让模板中必须出现这一行：

```groovy
exclude group:'com.facebook.fbjni'
```

这是因为 React Native 使用 non-prefab 方式消费 `fbjni`，也就是解压 `.aar` 并提取 `.so` 文件。Hermes-engine 和其他库则使用 prefab 来消费 fbjni。我们正在研究未来[解决这个问题](https://github.com/facebook/react-native/pull/33397)，让 Hermes import 可以变成一行。

#### iOS 实现细节

iOS 实现依赖一系列脚本，这些脚本位于以下位置：

- [`/scripts/hermes`](https://github.com/facebook/react-native/tree/main/scripts/hermes)。这些脚本包含下载 Hermes tarball、解压它并配置 iOS 构建的逻辑。如果你的 `hermes_enabled` 字段设置为 `true`，它们会在 `pod install` 时被调用。
- [`/sdks/hermes-engine`](https://github.com/facebook/react-native/tree/main/sdks/hermes-engine)。这些脚本包含真正构建 Hermes 的构建逻辑。它们从 `facebook/hermes` 仓库复制并调整而来，以便在 React Native 内部正确工作。具体来说，`utils` 文件夹中的脚本负责为所有 Mac 平台构建 Hermes。

Hermes 会作为 CircleCI 上 `build_hermes_macos` Job 的一部分进行构建。这个 job 会生成一个 tarball artifact；当使用已发布的 React Native release 时，`hermes-engine` podspec 会下载这个 tarball（这里是 React Native 0.69 中 `build_hermes_macos` 创建 artifact 的[一个例子](https://app.circleci.com/pipelines/github/facebook/react-native/13679/workflows/5172f8e4-6b02-4ccb-ab97-7cb954911fae/jobs/258701/artifacts)）。

##### 预构建 Hermes

如果当前使用的 React Native 版本没有预构建 artifact，例如你可能正在使用 `main` 分支中的 React Native，那么 Hermes 就需要从源码构建。首先，Hermes compiler `hermesc` 会在 `pod install` 期间为 macOS 构建；随后 Hermes 本身会作为 Xcode 构建流水线的一部分，通过 `build-hermes-xcode.sh` 脚本构建。

##### 从源码构建 Hermes

使用 React Native 的 `main` 分支时，Hermes 总是从源码构建。如果你使用的是稳定版 React Native，可以在使用 CocoaPods 时把 `CI` 环境变量设置为 `true`，强制从源码构建 Hermes：`CI=true pod install`。

##### 调试符号

Hermes 的预构建 artifact 默认不包含调试符号（dSYMs）。我们计划未来为每个 release 分发这些调试符号。在那之前，如果你需要 Hermes 的调试符号，就需要从源码构建 Hermes。每个 Hermes framework 旁边都会在构建目录中创建一个 `hermes.framework.dSYM`。

### 我担心这个变更会影响我

我们想强调的是，这本质上只是一个组织层面的变更：Hermes 在*哪里*构建，以及两个仓库之间的代码*如何*同步。这个变更对用户来说应该是完全透明的。

过去，我们会为某个特定 React Native 版本发布一个 Hermes 版本，例如 [`v0.11.0 for RN0.68.x`](https://github.com/facebook/hermes/releases/tag/v0.11.0)。

采用 “Bundled Hermes” 后，你可以改为依赖一个 tag。这个 tag 会表示某个特定 React Native 版本发布时使用的 Hermes 版本。
