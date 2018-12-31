在调研 Flutter 动态化方案的时候，需要了解 Flutter 加载 dart 产物的流程，于是梳理了一遍 FLutter 的初始化流程

flutter的源码下载地址在 github 上可以找到，具体地址: [github-flutter/engine](https://github.com/flutter/engine)

### FLutterMain的初始化

先从 Android 的入口开始看

在 `FlutterAppliation` 的 `onCreate` 中调用了

```java
FlutterMain.startInitialization(this);
```

跟进去我们会看到调用了 `startInitialization` 方法，最后会顺序调用这几个方法

```java
initConfig(applicationContext);
initAot(applicationContext);
initResources(applicationContext);
```

我们查看 `initResources` 方法如图

![](https://github.com/shaomaicheng/Article/blob/master/imgs/flutter/img1.png?raw=true)

这里我们可以看到实际加载了assets里面的flutter资源。并且会把资源 copy 到本地的路径。这里不做深究。`FlutterMan` 的初始化基本包括了
* 初始化配置
* 初始化 AOT 编译
* 初始化资源

3 个部分

继续看 `Flutter` 的 `View` 的初始化:

### FLutterView的初始化

以 `FlutterActivity` 为例，在 `onCreate` 中会调用到 `FlutterActivityDelegate` 的对应方法，最终调用 `FlutterView` 的 `runFromBundle` 方法

```java
public void runFromBundle(FlutterRunArguments args) {
    this.assertAttached();
    this.preRun();
    this.mNativeView.runFromBundle(args);
    this.postRun();
}
```
跟踪这段代码，会调用 `FlutterNativeView` 的 `nativeRunBundleAndSnapshotFromLibrary`方法。

这里会继续进行 `jni` 层的调用，查看 `platform_view_android_jni.cc`

```cpp
{
    .name = "nativeRunBundleAndSnapshotFromLibrary",
    .signature = "(J[Ljava/lang/String; Ljava/lang/String;"
    "Ljava/lang/String;Landroid/content/res/AssetManager;)V",
    .fnPtr = reinterpret_cast<void*>          (shell::RunBundleAndSnapshotFromLibrary),
},
```
查看 `RunBundleAndSnapshotFromLibrary`，这里删除了一些我们不关心的逻辑

```c++
static void RunBundleAndSnapshotFromLibrary(JNIEnv* env,
                                            jobject jcaller,
                                            jlong shell_holder,
                                            jobjectArray jbundlepaths,
                                            jstring jEntrypoint,
                                            jstring jLibraryUrl,
                                            jobject jAssetManager) {
    auto asset_manager = std::make_shared<blink::AssetManager>();      
       for (const auto& bundlepath :
       fml::jni::StringArrayToVector(env, jbundlepaths)) {
    const auto file_ext_index = bundlepath.rfind(".");
    if (bundlepath.substr(file_ext_index) == ".zip") {
      asset_manager->PushBack(
          std::make_unique<blink::ZipAssetStore>(bundlepath));
    } else {
      asset_manager->PushBack(
          std::make_unique<blink::DirectoryAssetBundle>(fml::OpenDirectory(
              bundlepath.c_str(), false, fml::FilePermission::kRead)));
      const auto last_slash_index = bundlepath.rfind("/", bundlepath.size());
      if (last_slash_index != std::string::npos) {
        auto apk_asset_dir = bundlepath.substr(
            last_slash_index + 1, bundlepath.size() - last_slash_index);

        asset_manager->PushBack(std::make_unique<blink::APKAssetProvider>(
            env,                       // jni environment
            jAssetManager,             // asset manager
            std::move(apk_asset_dir))  // apk asset dir
        );
      }
    }
  }      
  auto isolate_configuration = CreateIsolateConfiguration(*asset_manager);    
  RunConfiguration config(std::move(isolate_configuration),
                          std::move(asset_manager));  
    ANDROID_SHELL_HOLDER->Launch(std::move(config)); 
```

 首先会对资源路径进行处理
 会分为 `zip`包或者文件夹进行分别处理。最终会调用常量`ANDROID_SHELL_HOLDER` 的 `Launch` 函数.

 ![](https://github.com/shaomaicheng/Article/blob/master/imgs/flutter/2.png?raw=true)

 最终走到 `engine` 的 `Run` 函数。

![](https://github.com/shaomaicheng/Article/blob/master/imgs/flutter/3.png?raw=true)

这里有 2 个函数比较重要，先是 `IsolateConfiguration::PrepareIsolate`, 然后是 `RunFromLibrary` 或者 `Run` 函数

跟到 `PrepareAndLaunchIsolate` 函数,查看源码

```cpp
bool IsolateConfiguration::PrepareIsolate(blink::DartIsolate& isolate) {
  if (isolate.GetPhase() != blink::DartIsolate::Phase::LibrariesSetup) {
    FML_DLOG(ERROR)
        << "Isolate was in incorrect phase to be prepared for running.";
    return false;
  }

  return DoPrepareIsolate(isolate);
}
```

而有 `DoPrepareIsolate` 函数的类 `Configuration` 类有3个

* AppSnapshotIsolateConfiguration
* KernelIsolateConfiguration
* KernelListIsolateConfiguration

他们分别会调用 `DartIsolate` 的
* PrepareForRunningFromPrecompiledCode
* PrepareForRunningFromKernel

这2个方法的一个，可以看到这里的`prepare`操作分成了 **预先加载的代码** 和 **从内核获取** 2种

至于 `RunFromLibrary` 函数和 `Run` 函数

我们能看到他们最终都会调用 `dart:isolate` 和 `_startMainIsolate` 的逻辑:

```c++
Dart_Handle isolate_lib = Dart_LookupLibrary(tonic::ToDart("dart:isolate"));
if (tonic::LogIfError(Dart_Invoke(
          isolate_lib, tonic::ToDart("_startMainIsolate"),
          sizeof(isolate_args) / sizeof(isolate_args[0]), isolate_args))) {
    return false;
  }
```
这里说明我们正在执行调用 `Dart` 的入口方法。而 `Run` 和 `RunFromLibrary` 的区别，则是如果我们传入了 `entrypoint` 参数去进行 Flutter 的 bundle 初始化的时候，则会去加载我们制定的 library。

### 小结
到这里， Flutter 的初始化流程就就简单的分析了一遍。大致可以总结成三个部分

1. 初始化 FlutterMain
2. 初始化 FlutterView，开始加载 bundle
3. 初始化Flutter Bundle，这里获取了 Flutter 的入口方法、Flutter 的 library, 以及对 Flutter 入口方法的调用。

初始化的逻辑比较复杂，对后续一些初始化相关的性能优化应该也会有不小的启发。`FlutterMain` 中对资源的处理和写入本地的逻辑也给 Android 端研究 Flutter 动态化提供了基础。

很多 bundle 加载和后续初始化的逻辑也还没有完全弄清楚和深入研究。有兴趣的朋友可以一起研究，共同探讨。


