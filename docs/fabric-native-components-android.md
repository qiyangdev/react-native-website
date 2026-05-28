---
id: fabric-native-components-android
title: 'Fabric 原生组件：Android'
---

import Tabs from '@theme/Tabs'; import TabItem from '@theme/TabItem'; import constants from '@site/core/TabsConstants';

现在该编写一些 Android 平台代码，以便渲染 Web View。你需要完成以下步骤：

- 运行 Codegen
- 编写 `ReactWebView` 的代码
- 编写 `ReactWebViewManager` 的代码
- 编写 `ReactWebViewPackage` 的代码
- 在应用中注册 `ReactWebViewPackage`

### 1. 通过 Gradle 运行 Codegen

运行一次下面的命令，生成你选择的 IDE 可以使用的样板代码。

```bash title="Demo/"
cd android
./gradlew generateCodegenArtifactsFromSchema
```

Codegen 会为 Web View 生成你需要实现的 `ViewManager` 接口，以及 `ViewManager` delegate。

### 2. 编写 `ReactWebView`

`ReactWebView` 是一个组件，用来包装 React Native 在使用自定义组件时将要渲染的 Android 原生视图。

在 `android/src/main/java/com/webview` 文件夹中创建 `ReactWebView.java` 或 `ReactWebView.kt` 文件，并写入以下代码：

<Tabs groupId="android-language" queryString defaultValue={constants.defaultAndroidLanguage} values={constants.androidLanguages}>
<TabItem value="java">

```java title="Demo/android/src/main/java/com/webview/ReactWebView.java"
package com.webview;

import android.content.Context;
import android.util.AttributeSet;
import android.webkit.WebView;
import android.webkit.WebViewClient;

import com.facebook.react.bridge.Arguments;
import com.facebook.react.bridge.WritableMap;
import com.facebook.react.bridge.ReactContext;
import com.facebook.react.uimanager.UIManagerHelper;
import com.facebook.react.uimanager.events.Event;

public class ReactWebView extends WebView {
  public ReactWebView(Context context) {
    super(context);
    configureComponent();
  }

  public ReactWebView(Context context, AttributeSet attrs) {
    super(context, attrs);
    configureComponent();
  }

  public ReactWebView(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    configureComponent();
  }

  private void configureComponent() {
    this.setLayoutParams(new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
    this.setWebViewClient(new WebViewClient() {
      @Override
      public void onPageFinished(WebView view, String url) {
        emitOnScriptLoaded(OnScriptLoadedEventResult.success);
      }
    });
  }

  public void emitOnScriptLoaded(OnScriptLoadedEventResult result) {
    ReactContext reactContext = (ReactContext) context;
    int surfaceId = UIManagerHelper.getSurfaceId(reactContext);
    EventDispatcher eventDispatcher = UIManagerHelper.getEventDispatcherForReactTag(reactContext, getId());
    WritableMap payload = Arguments.createMap();
    payload.putString("result", result.name());

    OnScriptLoadedEvent event = new OnScriptLoadedEvent(surfaceId, getId(), payload);
    if (eventDispatcher != null) {
      eventDispatcher.dispatchEvent(event);
    }
  }

  public enum OnScriptLoadedEventResult {
    success,
    error
  }

  private class OnScriptLoadedEvent extends Event<OnScriptLoadedEvent> {
    private final WritableMap payload;

    OnScriptLoadedEvent(int surfaceId, int viewId, WritableMap payload) {
      super(surfaceId, viewId);
      this.payload = payload;
    }

    @Override
    public String getEventName() {
      return "onScriptLoaded";
    }

    @Override
    public WritableMap getEventData() {
      return payload;
    }
  }
}
```

</TabItem>
<TabItem value="kotlin">

```kotlin title="Demo/android/src/main/java/com/webview/ReactWebView.kt"
package com.webview

import android.content.Context
import android.util.AttributeSet
import android.webkit.WebView
import android.webkit.WebViewClient
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.WritableMap
import com.facebook.react.bridge.ReactContext
import com.facebook.react.uimanager.UIManagerHelper
import com.facebook.react.uimanager.events.Event

class ReactWebView: WebView {
  constructor(context: Context) : super(context) {
    configureComponent()
  }

  constructor(context: Context, attrs: AttributeSet?) : super(context, attrs) {
    configureComponent()
  }

  constructor(context: Context, attrs: AttributeSet?, defStyleAttr: Int) : super(context, attrs, defStyleAttr) {
    configureComponent()
  }

  private fun configureComponent() {
    this.layoutParams = LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT)
    this.webViewClient = object : WebViewClient() {
      override fun onPageFinished(view: WebView, url: String) {
        emitOnScriptLoaded(OnScriptLoadedEventResult.success)
      }
    }
  }

  fun emitOnScriptLoaded(result: OnScriptLoadedEventResult) {
    val reactContext = context as ReactContext
    val surfaceId = UIManagerHelper.getSurfaceId(reactContext)
    val eventDispatcher = UIManagerHelper.getEventDispatcherForReactTag(reactContext, id)
    val payload =
        Arguments.createMap().apply {
          putString("result", result.name)
        }
    val event = OnScriptLoadedEvent(surfaceId, id, payload)

    eventDispatcher?.dispatchEvent(event)
  }

  enum class OnScriptLoadedEventResult {
    success,
    error;
  }

  inner class OnScriptLoadedEvent(
      surfaceId: Int,
      viewId: Int,
      private val payload: WritableMap
  ) : Event<OnScriptLoadedEvent>(surfaceId, viewId) {
    override fun getEventName() = "onScriptLoaded"

    override fun getEventData() = payload
  }
}
```

</TabItem>
</Tabs>

`ReactWebView` 继承自 Android `WebView`，因此你可以轻松复用平台已经定义好的所有属性。

这个类定义了三个 Android 构造函数，但把实际实现委托给私有的 `configureComponent` 函数。该函数负责初始化组件的所有专属属性：在这个例子中，你会设置 `WebView` 的布局，并定义用于自定义 `WebView` 行为的 `WebClient`。在这段代码里，`ReactWebView` 通过实现 `WebClient` 的 `onPageFinished` 方法，在页面加载完成时发出一个事件。

随后，代码定义了一个真正发出事件的辅助函数。要发出事件，你需要：

- 获取 `ReactContext` 的引用；
- 获取当前呈现视图的 `surfaceId`；
- 获取与该视图关联的 `eventDispatcher` 引用；
- 使用 `WritableMap` 对象构建事件 payload；
- 创建需要发送到 JavaScript 的事件对象；
- 调用 `eventDispatcher.dispatchEvent` 发送事件。

文件最后一部分包含发送事件所需数据类型的定义：

- `OnScriptLoadedEventResult`，包含 `OnScriptLoaded` 事件可能产生的结果。
- 实际的 `OnScriptLoadedEvent`，它需要继承 React Native 的 `Event` 类。

### 3. 编写 `WebViewManager`

`WebViewManager` 是连接 React Native 运行时和原生视图的类。

当 React 从应用接收到渲染某个特定组件的指令时，会使用已注册的 view manager 创建视图，并传入所有必需属性。

下面是 `ReactWebViewManager` 的代码。

<Tabs groupId="android-language" queryString defaultValue={constants.defaultAndroidLanguage} values={constants.androidLanguages}>
<TabItem value="java">

```java title="Demo/android/src/main/java/com/webview/ReactWebViewManager.java"
package com.webview;

import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.module.annotations.ReactModule;
import com.facebook.react.uimanager.SimpleViewManager;
import com.facebook.react.uimanager.ThemedReactContext;
import com.facebook.react.uimanager.ViewManagerDelegate;
import com.facebook.react.uimanager.annotations.ReactProp;
import com.facebook.react.viewmanagers.CustomWebViewManagerInterface;
import com.facebook.react.viewmanagers.CustomWebViewManagerDelegate;

import java.util.HashMap;
import java.util.Map;

@ReactModule(name = ReactWebViewManager.REACT_CLASS)
class ReactWebViewManager extends SimpleViewManager<ReactWebView> implements CustomWebViewManagerInterface<ReactWebView> {
  private final CustomWebViewManagerDelegate<ReactWebView, ReactWebViewManager> delegate =
          new CustomWebViewManagerDelegate<>(this);

  @Override
  public ViewManagerDelegate<ReactWebView> getDelegate() {
    return delegate;
  }

  @Override
  public String getName() {
    return REACT_CLASS;
  }

  @Override
  public ReactWebView createViewInstance(ThemedReactContext context) {
    return new ReactWebView(context);
  }

  @ReactProp(name = "sourceUrl")
  @Override
  public void setSourceURL(ReactWebView view, String sourceURL) {
    if (sourceURL == null) {
      view.emitOnScriptLoaded(ReactWebView.OnScriptLoadedEventResult.error);
      return;
    }
    view.loadUrl(sourceURL, new HashMap<>());
  }

  public static final String REACT_CLASS = "CustomWebView";

  @Override
  public Map<String, Object> getExportedCustomBubblingEventTypeConstants() {
    Map<String, Object> map = new HashMap<>();
    Map<String, Object> bubblingMap = new HashMap<>();
    bubblingMap.put("phasedRegistrationNames", new HashMap<String, String>() {{
      put("bubbled", "onScriptLoaded");
      put("captured", "onScriptLoadedCapture");
    }});
    map.put("onScriptLoaded", bubblingMap);
    return map;
  }
}
```

</TabItem>
<TabItem value="kotlin">

```kotlin title="Demo/android/src/main/java/com/webview/ReactWebViewManager.kt"
package com.webview

import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.module.annotations.ReactModule;
import com.facebook.react.uimanager.SimpleViewManager;
import com.facebook.react.uimanager.ThemedReactContext;
import com.facebook.react.uimanager.ViewManagerDelegate;
import com.facebook.react.uimanager.annotations.ReactProp;
import com.facebook.react.viewmanagers.CustomWebViewManagerInterface;
import com.facebook.react.viewmanagers.CustomWebViewManagerDelegate;

@ReactModule(name = ReactWebViewManager.REACT_CLASS)
class ReactWebViewManager(context: ReactApplicationContext) : SimpleViewManager<ReactWebView>(), CustomWebViewManagerInterface<ReactWebView> {
  private val delegate: CustomWebViewManagerDelegate<ReactWebView, ReactWebViewManager> =
    CustomWebViewManagerDelegate(this)

  override fun getDelegate(): ViewManagerDelegate<ReactWebView> = delegate

  override fun getName(): String = REACT_CLASS

  override fun createViewInstance(context: ThemedReactContext): ReactWebView = ReactWebView(context)

  @ReactProp(name = "sourceUrl")
  override fun setSourceURL(view: ReactWebView, sourceURL: String?) {
    if (sourceURL == null) {
      view.emitOnScriptLoaded(ReactWebView.OnScriptLoadedEventResult.error)
      return;
    }
    view.loadUrl(sourceURL, emptyMap())
  }

  companion object {
    const val REACT_CLASS = "CustomWebView"
  }

  override fun getExportedCustomBubblingEventTypeConstants(): Map<String, Any> =
      mapOf(
          "onScriptLoaded" to
              mapOf(
                  "phasedRegistrationNames" to
                      mapOf(
                          "bubbled" to "onScriptLoaded",
                          "captured" to "onScriptLoadedCapture"
                      )))
}
```

</TabItem>
</Tabs>

`ReactWebViewManager` 继承自 React 的 `SimpleViewManager` 类，并实现 Codegen 生成的 `CustomWebViewManagerInterface`。

它持有 `CustomWebViewManagerDelegate` 的引用，后者也是由 Codegen 生成的元素。

随后它会重写 `getName` 函数，该函数必须返回与 spec 中 `codegenNativeComponent` 函数调用所使用的名称相同的值。

`createViewInstance` 函数负责实例化新的 `ReactWebView`。

接着，ViewManager 需要定义 React 组件的所有 props 如何更新原生视图。在这个示例中，你需要决定如何处理 React 会设置到 `WebView` 上的 `sourceURL` 属性。

最后，如果组件可以发出事件，你需要映射事件名称：冒泡事件需要重写 `getExportedCustomBubblingEventTypeConstants`，直接事件则需要重写 `getExportedCustomDirectEventTypeConstants`。

### 4. 编写 `ReactWebViewPackage`

和原生模块一样，原生组件也需要实现 `ReactPackage` 类。这个对象可用于在 React Native 运行时中注册组件。

下面是 `ReactWebViewPackage` 的代码：

<Tabs groupId="android-language" queryString defaultValue={constants.defaultAndroidLanguage} values={constants.androidLanguages}>
<TabItem value="java">

```java title="Demo/android/src/main/java/com/webview/ReactWebViewPackage.java"
package com.webview;

import com.facebook.react.BaseReactPackage;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.module.model.ReactModuleInfo;
import com.facebook.react.module.model.ReactModuleInfoProvider;
import com.facebook.react.uimanager.ViewManager;

import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class ReactWebViewPackage extends BaseReactPackage {
  @Override
  public List<ViewManager<?, ?>> createViewManagers(ReactApplicationContext reactContext) {
    return Collections.singletonList(new ReactWebViewManager(reactContext));
  }

  @Override
  public NativeModule getModule(String s, ReactApplicationContext reactApplicationContext) {
    if (ReactWebViewManager.REACT_CLASS.equals(s)) {
      return new ReactWebViewManager(reactApplicationContext);
    }
    return null;
  }

  @Override
  public ReactModuleInfoProvider getReactModuleInfoProvider() {
    return new ReactModuleInfoProvider() {
      @Override
      public Map<String, ReactModuleInfo> getReactModuleInfos() {
        Map<String, ReactModuleInfo> map = new HashMap<>();
        map.put(ReactWebViewManager.REACT_CLASS, new ReactModuleInfo(
                ReactWebViewManager.REACT_CLASS, // name
                ReactWebViewManager.REACT_CLASS, // className
                false,                           // canOverrideExistingModule
                false,                           // needsEagerInit
                false,                           // isCxxModule
                true                             // isTurboModule
        ));
        return map;
      }
    };
  }
}
```

</TabItem>
<TabItem value="kotlin">

```kotlin title="Demo/android/src/main/java/com/webview/ReactWebViewPackage.kt"
package com.webview

import com.facebook.react.BaseReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.model.ReactModuleInfo
import com.facebook.react.module.model.ReactModuleInfoProvider
import com.facebook.react.uimanager.ViewManager

class ReactWebViewPackage : BaseReactPackage() {
  override fun createViewManagers(reactContext: ReactApplicationContext): List<ViewManager<*, *>> {
    return listOf(ReactWebViewManager(reactContext))
  }

  override fun getModule(s: String, reactApplicationContext: ReactApplicationContext): NativeModule? {
    when (s) {
      ReactWebViewManager.REACT_CLASS -> ReactWebViewManager(reactApplicationContext)
    }
    return null
  }

  override fun getReactModuleInfoProvider(): ReactModuleInfoProvider = ReactModuleInfoProvider {
    mapOf(ReactWebViewManager.REACT_CLASS to ReactModuleInfo(
      name = ReactWebViewManager.REACT_CLASS,
      className = ReactWebViewManager.REACT_CLASS,
      canOverrideExistingModule = false,
      needsEagerInit = false,
      isCxxModule = false,
      isTurboModule = true,
    )
    )
  }
}
```

</TabItem>
</Tabs>

`ReactWebViewPackage` 继承自 `BaseReactPackage`，并实现正确注册组件所需的所有方法。

- `createViewManagers` 方法是一个工厂方法，用于创建管理自定义视图的 `ViewManager`。
- `getModule` 方法会根据 React Native 需要渲染的 View 返回正确的 ViewManager。
- `getReactModuleInfoProvider` 会提供在运行时注册模块所需的全部信息。

### 5. 在应用中注册 `ReactWebViewPackage`

最后，你需要在应用中注册 `ReactWebViewPackage`。为此，我们要修改 `MainApplication` 文件，将 `ReactWebViewPackage` 添加到 `getPackages` 函数返回的 packages 列表中。

```kotlin title="Demo/app/src/main/java/com/demo/MainApplication.kt"
package com.demo

import android.app.Application
import com.facebook.react.PackageList
import com.facebook.react.ReactApplication
import com.facebook.react.ReactHost
import com.facebook.react.ReactNativeHost
import com.facebook.react.ReactPackage
import com.facebook.react.defaults.DefaultNewArchitectureEntryPoint.load
import com.facebook.react.defaults.DefaultReactHost.getDefaultReactHost
import com.facebook.react.defaults.DefaultReactNativeHost
import com.facebook.react.soloader.OpenSourceMergedSoMapping
import com.facebook.soloader.SoLoader
// highlight-next-line
import com.webview.ReactWebViewPackage

class MainApplication : Application(), ReactApplication {

  override val reactNativeHost: ReactNativeHost =
      object : DefaultReactNativeHost(this) {
        override fun getPackages(): List<ReactPackage> =
            PackageList(this).packages.apply {
              // highlight-next-line
              add(ReactWebViewPackage())
            }

        override fun getJSMainModuleName(): String = "index"

        override fun getUseDeveloperSupport(): Boolean = BuildConfig.DEBUG

        override val isNewArchEnabled: Boolean = BuildConfig.IS_NEW_ARCHITECTURE_ENABLED
        override val isHermesEnabled: Boolean = BuildConfig.IS_HERMES_ENABLED
      }

  override val reactHost: ReactHost
    get() = getDefaultReactHost(applicationContext, reactNativeHost)

  override fun onCreate() {
    super.onCreate()
    SoLoader.init(this, OpenSourceMergedSoMapping)
    if (BuildConfig.IS_NEW_ARCHITECTURE_ENABLED) {
      load()
    }
  }
}

```
