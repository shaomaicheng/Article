在调研 Flutter 动态化方案的时候，需要了解 Flutter 加载 dart 产物的流程，于是梳理了一遍 FLutter 的初始化流程

flutter的源码下载地址在 github 上可以找到，具体地址: [github-flutter/engine](https://github.com/flutter/engine)

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

这里我们可以看到实际加载了assets里面的flutter资源。并且会把资源 copy 到本地的路径。这里不做深究。

