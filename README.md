<p align="center">
    <img alt="ovCompose Logo" src="img/ovCompose.svg" />
</p>

English | [简体中文](./README-zh_CN.md)

## Introduction
ovCompose (online-video-compose) is a cross-platform development framework launched by the Tencent Video team within Oteam, the leading frontend group at Tencent. It is based on the Compose Multiplatform ecosystem and aims to address the limitations of Jetbrains Compose Multiplatform, specifically its lack of support for the HarmonyOS platform and the constraints on mixed layout rendering on iOS. ovCompose makes it easier for businesses to build fully cross-platform apps.

## Compose Repositories

[compose-multiplatform](https://github.com/Tencent-TDS/KuiklyBase-platform/tree/main/compose-multiplatform): Plugins for compose multiplatform with compose-gradle-plugin, resources, ui-tooling-preview.

[compose-multiplatform-core](https://github.com/Tencent-TDS/ovCompose-multiplatform-core): The source core of multiplatform compose with material, foundation, ui, runtime.

[ovCompose-sample](https://github.com/Tencent-TDS/ovCompose-sample): Sample of multiplatform compose about ui, layout, gesture.

## Get Started

### Import Compose in ArkUI

1. Create compose multiplatform project

   Create kotlin multiplatform project with [Android Studio](https://developer.android.com/studio), and build project via the specific kotlin with ohos target, backed by `Tencent OnlineVideo`. 

   `libs.version.toml` is as follows

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

   `build.gradle.kts` is as follows

   ```kotlin
   plugins {
       // apply kotlinMultiplatform jetbrainsCompose and composeCompiler plugins
       alias(libs.plugins.kotlinMultiplatform)
       alias(libs.plugins.jetbrainsCompose)
       alias(libs.plugins.composeCompiler)
   }
   
   kotlin {
       ohosArm64 {
           binaries.sharedLib {
               // specify the shared lib name
               baseName = "kn"
               // link skia lib
               linkerOpts("-L${projectDir}/libs/", "-lskia")
               // export `compose.export`
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
                   // api compose.multiplatform.export lib for export C API.
                   api(libs.compose.multiplatform.export)
               }
           }
       }
   }
   ```

   

2. Declare Compose Multiplatform Code

   ```kotlin
   // declare in commonMain for all targets.
   @Composable
   internal fun Hello() {
       Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
           Text("Hello Compose!")
       }
   }
   
   // declare in ohosArm64Main, returning ArkUIViewController imported into ArkUI.
   @CName("createHelloArkUIViewController")
   fun createHelloArkUIViewController(env: napi_env): napi_value {
       initMainHandler(env)
       return ComposeArkUIViewController(env) {
           Hello()
       }
   }
   ```

   

3. Build kotlin multiplatform project to output binary product imported into harmony platform.

   Run `linkDebugSharedOhosArm64 ` or `linkReleaseSharedOhosArm64` task in multiplatform module.
   `libkn.so` and `libkn_api.h` will be output to `build/bin/ohosArm64` in own module.

   

4. Creating harmonyApp project

   - Creating

     create harmonyApp project with [DevEco-Studio](https://developer.huawei.com/consumer/cn/deveco-studio/) in sub，select "Native C++" for getting native configurations in step `Create Project`

   - Import multiplatform harmony binary product
     - copy `libkn.so` to `entry/libs/arm64-v8a/`
     - copy `libkn_api.h` to `entry/src/main/cpp/include/`

     To simplify all of this, we create Gradle Tasks in multiplatform project, just run `publishDebugBinariesToHarmonyApp` or `publishReleaseBinariesToHarmonyApp` to build and output to harmony project。

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

   - Import `skikobridge.har` and `compose.har` dependencies

     - Copy `skikobridge.har` to `entry/libs/`, the `skikobridge.har` can get from `ovCompose-sample/harmonyApp`。
     - Copy `compose.har` to `entry/libs`, the `compose.har` can build from `compose-multiplatform-core/ui-arkui`。

   

5. Configure  harmonyApp project

   configure dependencies in `entry/oh-package.json`

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
       // import compose.har dependency
       "compose": "file:./libs/compose.har",
       // import skikobridge.har dependency
       "skikobridge": "file:./libs/skikobridge.har"
     }
   }
   ```

   Configure CMake compile in `entry/src/main/cpp/CMakeLists.txt`

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
   
   # find skikobridge package
   find_package(skikobridge)
   
   add_library(entry SHARED napi_init.cpp)
   target_link_libraries(entry PUBLIC libace_napi.z.so)
   # link libkn.so
   target_link_libraries(entry PUBLIC ${NATIVERENDER_ROOT_PATH}/../../../libs/arm64-v8a/libkn.so)
   # link skikobridge.so in skikobridge
   target_link_libraries(entry PUBLIC skikobridge::skikobridge)
   # link others
   target_link_libraries(entry PUBLIC ${EGL-lib} ${GLES-lib} ${hilog-lib} ${libace-lib} ${libnapi-lib} ${libuv-lib} libc++_shared.so)
   ```

   

6. Import Compose in harmonyApp

   Init Compose ArkUI and ArkUIViewController

   ```c++
   // entry/src/main/cpp/napi_init.cpp
   
   static napi_value CreateHelloArkUIViewController(napi_env env, napi_callback_info info) {
       // call ArkUIViewController declared in Compose
       auto controller = createHelloArkUIViewController(env);
       return reinterpret_cast<napi_value>(controller);
   }
   
   static napi_value Init(napi_env env, napi_value exports) {
       // init compose arkui
       androidx_compose_ui_arkui_init(env, exports);
       // register a createHelloArkUIViewController for ArkTS with napi
       napi_property_descriptor desc[] = {
           {"createHelloArkUIViewController", nullptr, CreateHelloArkUIViewController, nullptr, nullptr, nullptr, napi_default, nullptr}};
       napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
       return exports;
   }
   ```

   Define `createHelloArkUIViewController()` in ArkTS：

   ```typescript
   // entry/src/main/cpp/types/libentry/index.d.ets
   
   import { ArkUIViewController } from 'compose';
   
   // declare createHelloArkUIViewController
   export const createHelloArkUIViewController: () => ArkUIViewController
   ```

   Import Compose in ArkUI

   ```typescript
   import { common } from '@kit.AbilityKit';
   import { ArkUIViewController, Compose } from 'compose';
   import { createHelloArkUIViewController } from 'libentry.so';
   
   @Entry
   @Component
   struct ComposePage {
     private controller: ArkUIViewController = createHelloArkUIViewController()
   
     // onPageShow only in component with @Entry, invoked on page show
     onPageShow(): void {
       // notify controller onPageShow to handle lifecycle
       this.controller.onPageShow()
     }
   
     // onPageHide only in component with @Entry, invoked on page hide
     onPageHide(): void {
       // notify controller onPageHide to handle lifecycle
       this.controller.onPageHide()
     }
   
     // onBackPress only in component with @Entry
     onBackPress(): boolean | void {
       // propagate back event to controller
       return this.controller.onBackPress()
     }
   
     build() {
       Stack({ alignContent: Alignment.Center }) {
         Compose({
           controller: this.controller,
           libraryName: "entry",
           onBackPressed: () => {
             // handle the event not consumed by compose
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

   Import Compose in ArkUI Navigation

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
               // handle the event not consumed by compose
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
       // notify controller onPageShow
       .onShown(() => {
         this.controller.onPageShow()
       })
       // notify controller onPageHide
       .onHidden(() => {
         this.controller.onPageHide()
       })
       // propagate back event to controller
       .onBackPressed(() => this.controller.onBackPress())
     }
   }
   ```
   
### Use ArkUI inside Compose Multiplatform
ovCompose Multiplatform is interoperable with the ArkUI framework. You can embed Compose Multiplatform within a ArkUI application as well as embed native ArkUI components within Compose Multiplatform. 
This section provides examples for embedding ArkUI components inside Compose Multiplatform UI.

#### Use ArkUI-TS inside Compose Multiplatform
To use ArKUI-TS elements inside Compose Multiplatform, add the ArkUI elements that you want to use to a `ArkUIView` from Compose Multiplatform.
1. Create your ArkUI-TS native component and its `Builder`
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
2. Register ArkUI-TS interop builder with `registerComposeInteropBuilder`
```typescript
registerComposeInteropBuilder('button', buttonBuilder)
```

3. Interop the ArkUI elements that you registered to use to a `ArkUIView` from Compose Multiplatform.
```kotlin
ArkUIView(
    name = "button",
    modifier = Modifier.width(250.dp).height(100.dp),
    parameter = js {
        "text"("ArkUI Button")
        "backgroundColor"("#FF0000FF")
    },
)
```

#### Use ArkUI-C inside Compose Multiplatform
To use ArKUI-C elements inside Compose Multiplatform, add the ArkUI elements that you want to use to a `ArkUINodeHandle` from Compose Multiplatform.
You can write this code purely in Kotlin or use `C/C++` as well.

You can implement a `Button` view in Compose Multiplatform using ArkUI-C's `ArkUI_NodeHandle` component. 
This allows your application to display and interact with Button content within the UI. Set the component size by using the Modifier.size() or Modifier.fillMaxSize() functions from Compose Multiplatform:
```kotlin
ArkUINodeHandle(
    factory = {
        ArkUI_NativeNodeAPI_1.createNode(ARKUI_NODE_BUTTON)
    },
    modifier = Modifier.size(300.dp)
)
```

### Integrating Compose with iOS
The common configuration parts of the Compose project can refer to the HarmonyOS platform mentioned above.

iOS provides two ways to integrate Compose containers. Each container can choose either the UIView rendering mode or the Skia rendering mode.

#### Using UIViewController as the Container

``` kotlin
fun MainViewController() = ComposeUIViewController(configure = {  
    renderBackend = RenderBackend.UIView  
}) {  
    Box { ... }  
}  
```

We provide LocalLifecycleOwner, which can be accessed within Compose functions to observe the lifecycle of the page.

```kotlin
// Within a Compose function, you can use LocalLifecycleOwner to get the LifecycleOwner and observe the lifecycle.  
val LocalLifecycleOwner = staticCompositionLocalOf<LifecycleOwner> { ... }  
```

When the ComposeUIViewController is popped from the view stack, it automatically disposes of the Compose-related data structures, which releases the associated memory during the next GC cycle.

#### Using UIView as the Compose Container
Using UIView as a container is an additional method introduced by Tencent Video on top of the official approach. This is more suitable for specific business development scenarios, such as embedding a Compose UIView inside a UICollectionViewCell.

```kotlin
fun MainComposeView(): UIView {  
    // UIKit rendering mode (Skia rendering mode is also available)  
    return ComposeUIView(configure = { renderBackend = RenderBackend.UIView }) {  
        Box { ... }  
    }  
} 
```

**Note 1**: Since UIView cannot actively observe the lifecycle of the UIViewController, you must manually call the appear and disappear methods of ComposeUIView to ensure the Lifecycle works correctly.

```kotlin
class ComposeUIView(  
    private val configuration: ComposeUIViewConfiguration,  
    private val content: @Composable () -> Unit,  
) : UIView(...) {  

    // Call when the view is exposed  
    fun appear() { ... }  

    // Call when the view is no longer exposed  
    fun disappear() { ... }  

    // Call when the view is no longer in use to avoid memory leaks  
    fun dispose() { ... }  
}  
```

**Note 2**: Whether the function return type is UIView or ComposeUIView, the exported Objective-C header file will treat it as UIView. Therefore, you cannot directly call methods like appear, disappear, or dispose. It is recommended to use a custom class to wrap ComposeUIView, such as ComposeUIViewWrapper in the following code:

```kotlin
fun MainComposeView(): UIView { ... }  

fun MainComposeView2(): ComposeUIView { ... }  

class ComposeUIViewWrapper {  

    private val view by lazy {  
        ComposeUIView(configure = { renderBackend = RenderBackend.UIView }) {  
            Box { ... }  
        }  
    }  

    fun getUIView(): UIView = view  

    fun dispose() = view.dispose()  

    fun appear() = view.appear()  
}  
```

#### Related Notes
The iOS app provides two rendering modes: one based on Skia and the other based on UIKit. These two modes can operate simultaneously during runtime. The UIKit rendering mode has been fully deployed across many core pages in Tencent Video iOS. This mode aims to reduce the additional memory consumption caused by the Skia mode, thereby alleviating memory pressure on low-end iPhones when multiple pages use Compose.

##### Cross-Platform Consistency Notes for UIKit Mode
| Aspect  | Differences  | Notes  |
| ---------| -----------|---------|
| Text  | Text rendering maintains exactly the same effect as Skia mode | No difference |
| Image | Supports Gaussian blur, tinting, etc. Other special filter capabilities are only partially supported for now | Some differences exist compared to Skia mode |
| GraphicsLayer | saveLayer operations are not yet supported   | Marquee effects are not supported. This capability is rarely used in most business development scenarios and can be addressed using UIKitView|

The UIKit mode has been widely adopted in Tencent Video iOS, and there have been zero reported UI inconsistency issues from business feedback. If you find that an important feature is not yet supported, feel free to provide feedback or submit a Pull Request.

## License
ovCompose-sample is released under the Apache 2.0 License. For details, see: [License](License.txt)
