<p align="center">
    <img alt="ovCompose Logo" src="img/ovCompose.svg" />
</p>

[English](./README.md) | 简体中文

## 项目介绍

ovCompose（online-video-compose）是腾讯大前端领域Oteam中，腾讯视频团队基于 Compose Multiplatform
生态推出的跨平台开发框架，旨在弥补Jetbrains Compose
Multiplatform不支持鸿蒙平台的遗憾与解决iOS平台混排受限的问题，便于业务构建全跨端App。

## 仓库说明

[compose-multiplatform](https://github.com/Tencent-TDS/KuiklyBase-platform/tree/main/compose-multiplatform)
：跨端 Compose 编译器插件源码

[compose-multiplatform-core ](https://github.com/Tencent-TDS/ovCompose-multiplatform-core)：跨端
Compose 源码

[ovCompose-sample ](https://github.com/Tencent-TDS/ovCompose-sample)：跨端 Compose 示例项目

## 使用说明

### ArkUI 接入 Compose

1. 创建 Compose 跨端项目

   使用 [Android Studio](https://developer.android.com/studio) 创建 Kotlin 跨端项目，并使用腾讯视频开源的支持鸿蒙平台的
   Kotlin 版本构建项目。

   `libs.version.toml` 参考如下

   ```toml
   [versions]
   agp = "8.5.2"
   compose-plugin = "1.6.1-KBA-002"
   kotlin = "2.0.21-KBA-004"
   kotlinx-coroutines = "1.8.0-KBA-001"
   
   [libraries]
   # Compose multiplatform
   compose-multiplatform-export = { module = "org.jetbrains.compose.export:export", version.ref = "compose-plugin" }
   
   kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinx-coroutines" }
   
   [plugins]
   androidApplication = { id = "com.android.application", version.ref = "agp" }
   androidLibrary = { id = "com.android.library", version.ref = "agp" }
   jetbrainsCompose = { id = "org.jetbrains.compose", version.ref = "compose-plugin" }
   kotlinAndroid = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
   kotlinMultiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
   cocoapods = { id = "org.jetbrains.kotlin.native.cocoapods", version.ref = "kotlin" }
   composeCompiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
   ```

   `build.gradle.kts` 参考如下

   其中链接 skia 库中的 `libskia.so` 从 `ovCompose-sample` 项目中获取。

   ```kotlin
   plugins {
       // 添加 kotlinMultiplatform jetbrainsCompose 和 composeCompiler 插件
       alias(libs.plugins.kotlinMultiplatform)
       alias(libs.plugins.jetbrainsCompose)
       alias(libs.plugins.composeCompiler)
   }
   
   kotlin {
       ohosArm64 {
           binaries.sharedLib {
               // 指定二进制产物名称
               baseName = "kn"
               // 链接 skia 库
               linkerOpts("-L${projectDir}/libs/", "-lskia")
               // 导出 compose.export
               export(libs.compose.multiplatform.export)
           }
       }
   
       sourceSets {
           commonMain.dependencies {
               implementation(compose.runtime)
               implementation(compose.foundation)
               implementation(compose.material3)
               implementation(compose.material)
               implementation(compose.ui)
   
               implementation(libs.kotlinx.coroutines.core)
           }
   
           val ohosArm64Main by getting {
               dependencies {
                   // 使用 api 方式依赖 compose.multiplatform.export 库，用于导出 C 函数
                   api(libs.compose.multiplatform.export)
               }
           }
       }
   }
   ```


2. 编写跨端 Compose 代码

   ```kotlin
   // 写在 commonMain 下，支持跨端复用
   @Composable
   internal fun Hello() {
       Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
           Text("Hello Compose!")
       }
   }
   
   // 写在 ohosArm64Main 下，创建接入鸿蒙 ArkUI 视图体系的 ArkUIViewController
   @CName("createHelloArkUIViewController")
   fun createHelloArkUIViewController(env: napi_env): napi_value {
       initMainHandler(env)
       return ComposeArkUIViewController(env) {
           Hello()
       }
   }
       
   ```


3. 编译跨端 Compose 代码，生成接入鸿蒙平台的二进制产物

   执行跨端模块的 `linkDebugSharedOhosArm64 ` 或 `linkReleaseSharedOhosArm64` 任务，

   在该模块的 `build/bin/ohosArm64` 目录下将会生成 `libkn.so` 和 `libkn_api.h`
   两个文件，这两个文件将被用于集成到鸿蒙项目中。


4. 创建鸿蒙 harmonyApp 项目

    - 创建项目

      在跨端项目下使用 [DevEco-Studio](https://developer.huawei.com/consumer/cn/deveco-studio/) 创建
      harmonyApp 项目，在 Create Project 选择 "Native C++" 创建带有 Native 代码的项目。

    - 添加 Compose 跨端二进制产物

      将前步骤 3 中生成两个文件复制到 harmonyApp 项目下，其中：

        - `libkn.so` 复制到 `entry/libs/arm64-v8a/` 目录下。
        - `libkn_api.h` 复制到 `entry/src/main/cpp/include/` 目录下。

      为了简化这个步骤，我们可以在跨端 Compose 项目中创建一个 Gradle Task 执行这个复制任务。这样只需执行
      `publishDebugBinariesToHarmonyApp` 或者 `publishReleaseBinariesToHarmonyApp` 即可编译 Compose
      跨端代码并复制产物到鸿蒙项目。

      ```kotlin
      kotlin { /* */ }
      arrayOf("debug", "release").forEach { type ->
          tasks.register<Copy>("publish${type.capitalizeUS()}BinariesToHarmonyApp") {
              group = "harmony"
              dependsOn("link${type.capitalizeUS()}SharedOhosArm64")
              into(rootProject.file("harmonyApp"))
              from("build/bin/ohosArm64/${type}Shared/libkn_api.h") {
                  into("entry/src/main/cpp/include/")
              }
              from(project.file("build/bin/ohosArm64/${type}Shared/libkn.so")) {
                  into("/entry/libs/arm64-v8a/")
              }
          }
      }
      ```

    - 添加 `skikobridge.har` 和 `compose.har` 依赖

        - 将 `skikobridge.har` 复制到 `entry/libs/` 目录下，其中：`skikobridge.har` 可以从
          `ovCompose-sample/harmonyApp` 项目下获取。
        - 将 `compose.har` 复制到 `entry/libs` 目录下，其中 `compose.har` 是从
          `compose-multiplatform-core/ui-arkui` 模块发布出来的，请参考文档中的编译发布板块里的第三部分内容。


5. 配置 harmonyApp 项目

   配置依赖：`entry/oh-package.json`

   ```json5
   {
     "name": "entry",
     "version": "1.0.0",
     "description": "Please describe the basic information.",
     "main": "",
     "author": "",
     "license": "",
     "dependencies": {
       "libentry.so": "file:./src/main/cpp/types/libentry",
       // 添加 compose.har 依赖
       "compose": "file:./libs/compose.har",
       // 添加 skikobridge.har 依赖
       "skikobridge": "file:./libs/skikobridge.har"
     }
   }
   ```

   配置 CMake 编译：`entry/src/main/cpp/CMakeLists.txt`

   ```cmake
   # the minimum version of CMake.
   cmake_minimum_required(VERSION 3.5.0)
   project(harmonyApp)
   
   set(NATIVERENDER_ROOT_PATH ${CMAKE_CURRENT_SOURCE_DIR})
   
   if(DEFINED PACKAGE_FIND_FILE)
       include(${PACKAGE_FIND_FILE})
   endif()
   
   add_definitions(-std=c++17)
   
   include_directories(${NATIVERENDER_ROOT_PATH}
                       ${NATIVERENDER_ROOT_PATH}/include)
   
   # 获取 skikobridge package
   find_package(skikobridge)
   
   add_library(entry SHARED napi_init.cpp)
   target_link_libraries(entry PUBLIC libace_napi.z.so)
   # 链接 libkn.so
   target_link_libraries(entry PUBLIC ${NATIVERENDER_ROOT_PATH}/../../../libs/arm64-v8a/libkn.so)
   # 链接 skikobridge 中的 skikobridge.so
   target_link_libraries(entry PUBLIC skikobridge::skikobridge)
   # 链接其他必要库
   target_link_libraries(entry PUBLIC ${EGL-lib} ${GLES-lib} ${hilog-lib} ${libace-lib} ${libnapi-lib} ${libuv-lib} libc++_shared.so)
   ```


6. 将 Compose 接入 harmonyApp 项目

   Compose ArkUI 初始化和 ArkUIViewController 的构建

   ```c++
   // entry/src/main/cpp/napi_init.cpp
   
   static napi_value CreateHelloArkUIViewController(napi_env env, napi_callback_info info) {
       // 调用 Compose 跨端项目构造 ArkUIViewController 的函数
       auto controller = createHelloArkUIViewController(env);
       return reinterpret_cast<napi_value>(controller);
   }
   
   static napi_value Init(napi_env env, napi_value exports) {
       // 初始化 compose arkui
       androidx_compose_ui_arkui_init(env, exports);
       // 使用 napi 注册一个 ArkTS 层的 createHelloArkUIViewController 函数
       napi_property_descriptor desc[] = {
           {"createHelloArkUIViewController", nullptr, CreateHelloArkUIViewController, nullptr, nullptr, nullptr, napi_default, nullptr}};
       napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
       return exports;
   }
   ```

   添加 `createHelloArkUIViewController()` 函数声明：

   ```typescript
   // entry/src/main/cpp/types/libentry/index.d.ets
   
   import { ArkUIViewController } from 'compose';
   
   // 声明 createHelloArkUIViewController 函数
   export const createHelloArkUIViewController: () => ArkUIViewController
   ```

   在 ArkUI 页面中接入 Compose

   ```typescript
   import { common } from '@kit.AbilityKit';
   import { ArkUIViewController, Compose } from 'compose';
   import { createHelloArkUIViewController } from 'libentry.so';
   
   @Entry
   @Component
   struct ComposePage {
     private controller: ArkUIViewController = createHelloArkUIViewController()
   
     // onPageShow 仅 @Entry 有效；页面每次显示时触发一次，包括路由过程、应用进入前台等场景
     onPageShow(): void {
       // 通知 controller onPageShow，用于处理生命周期
       this.controller.onPageShow()
     }
   
     // onPageHide 仅 @Entry 有效；页面每次隐藏时触发一次，包括路由过程、应用进入后台等场景
     onPageHide(): void {
       // 通知 controller onPageHide，用于处理生命周期
       this.controller.onPageHide()
     }
   
     // onBackPress 仅 @Entry 有效；
     onBackPress(): boolean | void {
       // 转发返回事件给 controller，由 controller 优先消费
       return this.controller.onBackPress()
     }
   
     build() {
       Stack({ alignContent: Alignment.Center }) {
         Compose({
           controller: this.controller,
           libraryName: "entry",
           onBackPressed: () => {
             // 未被 Compose 消费的返回事件，交由接入层兜底处理
             (getContext() as common.UIAbilityContext).windowStage.loadContent("pages/Index")
             return true
           }
         }).width('100%').height('100%')
       }
       .width('100%')
       .height('100%')
     }
   }
   ```

   在 ArkUI 导航节点中接入 Compose

   ```typescript
   import { ArkUIViewController, Compose } from 'compose';
   import { createHelloArkUIViewController } from 'libentry.so';
   
   @Component
   export struct ComposeDestination {
     private controller: ArkUIViewController = createHelloArkUIViewController()
     private navContext: NavDestinationContext | null = null;
   
     build() {
       NavDestination() {
         Stack({ alignContent: Alignment.Center }) {
           Compose({
             controller: this.controller,
             libraryName: "entry",
             onBackPressed: () => {
               // 未被 Compose 消费的返回事件，交由接入层兜底处理
               this.navContext?.pathStack.pop()
               return true
             }
           })
         }
         .width('100%')
         .height('100%')
       }
       .onReady((navContext) => {
         this.navContext = navContext
       })
       // 通知 controller onPageShow
       .onShown(() => {
         this.controller.onPageShow()
       })
       // 通知 controller onPageHide
       .onHidden(() => {
         this.controller.onPageHide()
       })
       // 转发返回事件给 controller，由 controller 优先消费
       .onBackPressed(() => this.controller.onBackPress())
     }
   }
   ```

### 在Compose中使用ArkUI

ovCompose可与ArkUI框架混排。您可以将 Compose Multiplatform 嵌入到 ArkUI 应用中，也可以将原生 ArkUI
组件嵌入到 Compose Multiplatform 中。
此部分提供了在Compose Multiplatform UI 中嵌入 ArkUI 组件的示例。

#### 在 Compose Multiplatform 中使用 ArkUI-TS

要在 Compose Multiplatform 中使用 ArkUI-TS 元素, 请将要使用的 ArkUI-TS 元素添加到Compose
Multiplatform 中的`ArkUIView`中

1. 创建您的ArkUI-TS native组件及其“Builder”

```typescript
@Builder
export function buttonBuilder(args: ButtonArgs) {
  button()
}

@Component
export struct button {
  @Consume compose_args: ButtonArgs

  build() {
    Column() {
      Button(this.compose_args.text).backgroundColor(this.compose_args.backgroundColor).width('100%').onClick(e => {
        console.log(`Button Clicked: ${this.compose_args.text}`)
      }).height('70%')

      Text(this.compose_args.text).backgroundColor(this.compose_args.backgroundColor).width('100%').onClick(e => {
        console.log(`Text Clicked: ${this.compose_args.text}`)
      }).height('20%')
      Stack().height('10%')
    }
  }
}

interface ButtonArgs {
  text: string
  backgroundColor: string
}
```

2. 使用`registerComposeInteropBuilder`注册ArkUI-TS 混排builder

```typescript
registerComposeInteropBuilder('button', buttonBuilder)
```

3. 请将要使用的 ArkUI-TS 元素添加到Compose Multiplatform 中的ArkUIView中

```kotlin
ArkUIView(
    name = "button",
    modifier = Modifier.width(250.dp).height(100.dp),
    parameter = js {
        "text"("ArkUI Button $index")
        "backgroundColor"("#FF0000FF")
    },
)
```

#### 在 Compose Multiplatform 中使用 ArkUI-C

要在 Compose Multiplatform 中使用 ArKUI-C 元素，请将要使用的 ArKUI-C 元素添加到Compose Multiplatform
中的`ArkUINodeHandle`中。
您可以纯用 Kotlin 编写此代码，也可以使用 `C/C++` 编写。

您可以使用 ArkUI-C 的API在 Compose Multiplatform 中实现Button视图。使用Compose Multiplatform
中的Modifier.size()或Modifier.fillMaxSize()函数为Button设置组件大小：

```kotlin
ArkUINodeHandle(
    factory = {
        ArkUI_NativeNodeAPI_1.createNode(ARKUI_NODE_BUTTON)
    },
    modifier = Modifier.size(300.dp)
)
```

### iOS 接入 Compose
Compose 项目公共配置部分可参考上述鸿蒙平台。
iOS 中提供了两种 Compose 容器方式接入。而每一种容器又分别可以选择 UIView 渲染模式或者 Skia 渲染模式

#### 使用 UIViewController 作为容器

```kotlin
fun MainViewController() = ComposeUIViewController(configure = {
    renderBackend = RenderBackend.UIView
}) {
    Box { ... }
}
```

我们提供了 `LocalLifecycleOwner`，可以在 Compose 函数内访问，并通过其感知页面的生命周期。

```kotlin
// 在 Compose 函数内可以使用 LocalLifecycleOwner 获取 LifecycleOwner，感知生命周期
val LocalLifecycleOwner = staticCompositionLocalOf<LifecycleOwner> { ... }
```

当 ComposeUIViewController 从视图栈中弹出后，内部会自动对 Compose 相关的数据结构进行 dispose，这可以在下一次 GC 到来的时候，释放相关内存。


#### 使用 UIView 作为 Compose 容器

UIView 作为容器是腾讯视频在官方基础上额外新增的一种方式。这更适合一些特殊的业务开发场景，比如 UIColletionViewCell 中嵌套 Compose 的 UIView。

```kotlin
fun MainComposeView(): UIView {
    // UIKit 渲染模式，此外还可以选择使用 Skia 渲染模式
    return ComposeUIView(configure = { renderBackend = RenderBackend.UIView }) {
        Box { ... }
    }
}
```

**注意点一**：由于 UIView 无法主动感知页面的 UIViewController 的生命周期。因此需要主动调用 ComposeUIView 的 `appear` 和 `disappear`，才能使驱动 Lifecycle 工作。

```kotlin
class ComposeUIView(
    private val configuration: ComposeUIViewConfiguration,
    private val content: @Composable () -> Unit,
) : UIView(...) {

    // view 曝光的时候调用
    fun appear() {  ... }
    
    // view 反曝光的时候调用
    fun disappear() { ... }
    
    // view 不再使用的时候需要调用，否则会出现内存泄漏
    fun dispose() { ... }

} 
```

**注意点二**：

函数返回类型为 UIView 或者 ComposeUIView 在导出的 OC 头文件中类型都是 UIView，因此是无法直接调用到`appear`、`disappear`、`dispose` 方法的。推荐使用自定义的类，包装 ComposeUIView。如下代码的`ComposeUIViewWrapper`
```kotlin
fun MainComposeView(): UIView { ... }

fun MainComposeView2(): ComposeUIView { ... }

class ComposeUIViewWrapper {

    private val view by lazy {
        ComposeUIView(configure = { renderBackend = RenderBackend.UIView }) {
            Box { ... }
        }
    }

    fun getUIView() : UIView = view
    
    fun dispose() = view.dispose()
    
    fun appear() = view.appear()

}
```

#### 有关说明

iOS应用程序提供了两种渲染模式：一种基于Skia，另一个基于UIKit。这两种模式可以在运行时同时存在。
UIKit渲染模式已在腾讯视频iOS中的许多核心页面中完全部署。
此模式旨在减少Skia模式引起的额外内存消耗，从而减轻使用多页面时低端iPhone的内存压力。

##### UIKit模式的跨平台一致性说明

| 维度            | 差异性                         | 说明                                                 |
|---------------|-----------------------------|----------------------------------------------------|
| Text          | 文本渲染保持与Skia模式完全相同的效果        | 无差异性                                               |
| Image         | 支持高斯模糊，着色等。目前仅部分支持其他特殊的过滤功能 | 和Skia存在一些差异                                        |
| GraphicsLayer | saveLayer操作尚不支持             | Marquee效果不支持.此功能在大多数业务开发方案中很少使用，并且可以使用UIKitView来解决 |

UIKit模式已在腾讯视频iOS中广泛采用，并且从业务反馈中没有报出UI不一致问题。如果您发现尚未支持重要功能，请随时提供反馈或提交拉动请求。

## License
ovCompose-sample 基于 Apache 2.0 协议发布，详见：[License](License.txt)