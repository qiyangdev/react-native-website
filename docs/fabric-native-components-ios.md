---
id: fabric-native-components-ios
title: 'Fabric 原生组件：iOS'
---

import Tabs from '@theme/Tabs'; import TabItem from '@theme/TabItem'; import constants from '@site/core/TabsConstants';

现在该编写一些 iOS 平台代码，以便渲染 Web View。你需要完成以下步骤：

- 运行 Codegen。
- 编写 `RCTWebView` 的代码。
- 在应用中注册 `RCTWebView`。

### 1. 运行 Codegen

你可以[手动运行](the-new-architecture/codegen-cli) Codegen，不过更简单的方式是使用将要演示该组件的应用来帮你完成这一步。

```bash
cd ios
bundle install
bundle exec pod install
```

重要的是，你会看到 Codegen 的日志输出。接下来我们会在 Xcode 中使用它来构建 WebView 原生组件。

:::warning
将生成代码提交到仓库时需要谨慎。生成代码与具体的 React Native 版本相关。请使用 npm [peerDependencies](https://nodejs.org/en/blog/npm/peer-dependencies) 来限制兼容的 React Native 版本。
:::

### 3. 编写 `RCTWebView`

我们需要使用 Xcode 准备 iOS 项目，并完成下面 **5 个步骤**：

1. 打开 CocoaPods 生成的 Xcode Workspace：

```bash
cd ios
open Demo.xcworkspace
```

<img className="half-size" alt="打开 Xcode Workspace" src="/docs/assets/fabric-native-components/1.webp" />

2. 右键点击 app，选择 <code>New Group</code>，并将新 group 命名为 `WebView`。

<img className="half-size" alt="右键点击 app 并选择 New Group" src="/docs/assets/fabric-native-components/2.webp" />

3. 在 `WebView` group 中，创建 <code>New</code>→<code>File from Template</code>。

<img className="half-size" alt="使用 Cocoa Touch Class 模板创建新文件" src="/docs/assets/fabric-native-components/3.webp" />

4. 使用 <code>Objective-C File</code> 模板，并将其命名为 <code>RCTWebView</code>。

<img className="half-size" alt="创建 Objective-C RCTWebView 类" src="/docs/assets/fabric-native-components/4.webp" />

5. 重复第 4 步，创建一个名为 `RCTWebView.h` 的头文件。

6. 将 <code>RCTWebView.m</code> 重命名为 <code>RCTWebView.mm</code>，使它成为 Objective-C++ 文件。

```text title="Demo/ios"
Podfile
...
Demo
├── AppDelegate.swift
...
// highlight-start
├── RCTWebView.h
└── RCTWebView.mm
// highlight-end
```

创建头文件和实现文件后，就可以开始实现它们。

下面是 `RCTWebView.h` 文件的代码，它声明了组件接口。

```objc title="Demo/RCTWebView/RCTWebView.h"
#import <React/RCTViewComponentView.h>
#import <UIKit/UIKit.h>

NS_ASSUME_NONNULL_BEGIN

@interface RCTWebView : RCTViewComponentView

// You would declare native methods you'd want to access from the view here

@end

NS_ASSUME_NONNULL_END
```

这个类定义了一个 `RCTWebView`，它继承自 `RCTViewComponentView` 类。`RCTViewComponentView` 是所有原生组件的基类，由 React Native 提供。

实现文件（`RCTWebView.mm`）的代码如下：

```objc title="Demo/RCTWebView/RCTWebView.mm"
#import "RCTWebView.h"

#import <react/renderer/components/AppSpec/ComponentDescriptors.h>
#import <react/renderer/components/AppSpec/EventEmitters.h>
#import <react/renderer/components/AppSpec/Props.h>
#import <react/renderer/components/AppSpec/RCTComponentViewHelpers.h>
// highlight-next-line
#import <WebKit/WebKit.h>

using namespace facebook::react;

@interface RCTWebView () <RCTCustomWebViewViewProtocol, WKNavigationDelegate>
@end

@implementation RCTWebView {
  NSURL * _sourceURL;
  WKWebView * _webView;
}

-(instancetype)init
{
  if(self = [super init]) {
    // highlight-start
    _webView = [WKWebView new];
    _webView.navigationDelegate = self;
    [self addSubview:_webView];
    // highlight-end
  }
  return self;
}

- (void)updateProps:(Props::Shared const &)props oldProps:(Props::Shared const &)oldProps
{
  const auto &oldViewProps = *std::static_pointer_cast<CustomWebViewProps const>(_props);
  const auto &newViewProps = *std::static_pointer_cast<CustomWebViewProps const>(props);

  // Handle your props here
  if (oldViewProps.sourceURL != newViewProps.sourceURL) {
    NSString *urlString = [NSString stringWithCString:newViewProps.sourceURL.c_str() encoding:NSUTF8StringEncoding];
    _sourceURL = [NSURL URLWithString:urlString];
    // highlight-start
    if ([self urlIsValid:newViewProps.sourceURL]) {
      [_webView loadRequest:[NSURLRequest requestWithURL:_sourceURL]];
    }
    // highlight-end
  }

  [super updateProps:props oldProps:oldProps];
}

-(void)layoutSubviews
{
  [super layoutSubviews];
  _webView.frame = self.bounds;

}

#pragma mark - WKNavigationDelegate

// highlight-start
-(void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation
{
  CustomWebViewEventEmitter::OnScriptLoaded result = CustomWebViewEventEmitter::OnScriptLoaded{CustomWebViewEventEmitter::OnScriptLoadedResult::Success};
  self.eventEmitter.onScriptLoaded(result);
}

- (BOOL)urlIsValid:(std::string)propString
{
  if (propString.length() > 0 && !_sourceURL) {
    CustomWebViewEventEmitter::OnScriptLoaded result = CustomWebViewEventEmitter::OnScriptLoaded{CustomWebViewEventEmitter::OnScriptLoadedResult::Error};

    self.eventEmitter.onScriptLoaded(result);
    return NO;
  }
  return YES;
}

// Event emitter convenience method
- (const CustomWebViewEventEmitter &)eventEmitter
{
  return static_cast<const CustomWebViewEventEmitter &>(*_eventEmitter);
}
// highlight-end

+ (ComponentDescriptorProvider)componentDescriptorProvider
{
  return concreteComponentDescriptorProvider<CustomWebViewComponentDescriptor>();
}

@end
```

这段代码使用 Objective-C++ 编写，包含多个细节：

- `@interface` 实现了两个协议：
  - `RCTCustomWebViewViewProtocol`，由 Codegen 生成；
  - `WKNavigationDelegate`，由 WebKit framework 提供，用于处理 Web View 导航事件；
- `init` 方法会实例化 `WKWebView`，将它加入 subviews，并设置 `navigationDelegate`；
- `updateProps` 方法会在组件 props 变化时由 React Native 调用；
- `layoutSubviews` 方法描述自定义视图应该如何布局；
- `webView:didFinishNavigation:` 方法让你可以处理 `WKWebView` 完成页面加载后的行为；
- `urlIsValid:(std::string)propString` 方法用于检查作为 prop 接收的 URL 是否有效；
- `eventEmitter` 方法是一个工具方法，用于获取强类型的 `eventEmitter` 实例；
- `componentDescriptorProvider` 会返回 Codegen 生成的 `ComponentDescriptor`。

#### 添加 WebKit framework

:::note
只有在创建 Web View 时才需要这一步。iOS 上的 Web 组件需要链接 Apple 提供的 WebKit framework。如果你的组件不需要访问 Web 专属功能，可以跳过这一步。
:::

Web View 需要访问 Apple 通过 Xcode 和设备附带的 framework 提供的一些功能，也就是 WebKit。
你可以在原生代码中看到这一点：`RCTWebView.mm` 中添加了 `#import <WebKit/WebKit.h>`。

要在应用中链接 WebKit framework，请按以下步骤操作：

1. 在 Xcode 中点击你的项目
2. 选择 app target
3. 选择 General 标签页
4. 向下滚动，找到 _"Frameworks, Libraries, and Embedded Contents"_ 区域，然后点击 `+` 按钮

<img className="half-size" alt="将 WebKit framework 添加到应用 1" src="/docs/assets/AddWebKitFramework1.png" />

5. 在搜索栏中筛选 WebKit
6. 选择 WebKit framework
7. 点击 Add。

<img className="half-size" alt="将 WebKit framework 添加到应用 2" src="/docs/assets/AddWebKitFramework2.png" />
