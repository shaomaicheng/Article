最近大致分析了一把 Activity 启动的流程，趁着今晚刚🏊完精神状态好，把之前记录的写成文章。

开门见山，我们直接点进去看 Activity 的 `startActivity` , 最终，我们都会走到 `startActivityForResult` 这个方法，我们可以发现关键的代码：

```java
Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(
this, mMainThread.getApplicationThread(), mToken, this,intent, requestCode, options);
```

我们会发现 Activity 启动其实都经过了一个中转站叫做 `Instrumentation`, 查看`Instrumentation` 的 `execStartActivity` 方法：

```java
/// 删除了我们不关心的部分
try {
	intent.migrateExtraStreamToClipData();
	intent.prepareToLeaveProcess(who);
	int result = ActivityManager.getService().startActivity(whoThread, who.getBasePackageName(), intent,
                     intent.resolveTypeIfNeeded(who.getContentResolver()),token, target != null ? target.mEmbeddedID : null,requestCode, 0, null, options);
	checkStartActivityResult(result, intent);
	} catch (RemoteException e) {
		throw new RuntimeException("Failure from system", e);
	}
```

我们会发现这里通过 `ActivityManager.getService` 在进行通信，进去查看，我们发现这个 service 其实是一个 `IActivityManager.aidl`， 说明这里我们进行了一次 Android  的 IPC。

全局搜索 ` extends IActivityManager ` 我们可以发现进行通信的就是 `ActivityManagerService` , 查看 `startActivity` 最终可以走到 `ActivityStart` 的 `startActivityMayWait` 方法。我们抽取它的关键代码：




![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/1.jpeg)

这部分我们可以看到根据 intent 解析除了需要的信息，并根据信息去获取了跳转 Activity 的系统权限。


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/2.jpeg)

这一部分代码，则对 intent 进行了处理和判断，我们基本可以省略这部分非关键逻辑

最终我们会走到 `startActivityLocked` 方法，并走到 `startActivity`

![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/3.jpeg)


这里我们会看到很多对于不同的 `ActivityManager` 的 状态进行逻辑判断和处理，这里不影响我们的关键流程，我们可以继续往下分析, 分析 `doPendingActivityLaunchesLocked` 方法

```java
startActivity(pal.r, pal.sourceRecord, null, null, pal.startFlags, resume, null,
                        null, null /*outRecords*/);
```

最终还是会走到另一个重载的 `startActivity` :

```java
mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
```


查看 ` startActivityUnchecked` : 这里代码逻辑比较长，我们查看 ` ActivityStackSupervisor`的`.resumeFocusedStackTopActivityLocked` 方法


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/4.jpeg)

继续查看 `resumeTopActivityUncheckedLocked` 方法, 跟踪到 `resumeTopActivityInnerLocked` 方法:


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/5.jpeg)

这边我们查看需要 restart 这个 Activity 的简单情况，会调用 `ActivityStackSupervisor` 的 `startSpecificActivityLocked` 方法


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/6.jpeg)

这里我们找到了逻辑的关键：如果 app的线程和进程都存在，我们会执行 `realStartActivityLocked` 方法。否则，会继续进行 IPC 通知 `ActivityManagerService` 去执行 `startProcessLocked`

这里我们差不多能猜到启动逻辑：
1. 如果启动的是我们自己 app 进程的 Activity， 那么直接去启动就好了
2. 如果我们启动的 Activity 所在的进程不存在，例如：我们把微信 kill 了，然后跳转微信分享的 Activity，或者我们点击launch 的微信图标，那么，我么就会走创建新进程的逻辑

那么我们分别来跟踪这2种情况：

####  启动自己的Activity



![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/7.jpeg)

我们可以找到这段代码的关键逻辑，我们先分析下 `app.thread` 是什么。跟踪进去会发现是一个 `IApplicationThread`,  可以发现这里又是一个 aidl， 最后我们可以找到 `ApplicationThread` ，

```java
private class ApplicationThread extends IApplicationThread.Stub
```

这是 `ActivityThread` 的一个静态内部类，ActivtyThread和启动Activity 相关，那么这个类就应该是和 Application 启动相关。


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/8.jpeg)

我们会发现最后其实发了一个message 到消息队列中，找到 `H` 这个 handler 的 `handleMessage` 方法

```java
case LAUNCH_ACTIVITY: {
	final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
	r.packageInfo = getPackageInfoNoCheck(
	r.activityInfo.applicationInfo, r.compatInfo);
	handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
} break;
```

查看 `handleLaunchActivity` 方法

```java
Activity a = performLaunchActivity(r, customIntent);
```

在`performLaunchActivity`方法中可以看到

```java
java.lang.ClassLoader cl = appContext.getClassLoader();
activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
```
这里，我们发现这里通过 `Insteumentation` new 了一个 Activity


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/9.jpeg)

![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/10.jpeg)

通过以上代码，我们还可以发现 new 出 Activity 后的几个步骤
1. attach Activity， 目测会有初始化 window 的流程
2. 设置 theme
3. Activity 的 `onCreate` 流程
4. Activity 如果已经销毁，会去执行 `onRestoreInstance` ，我们可以在这里做数据恢复的操作
5. Activity 在 `onCreate` 完成后的一些操作

到这里，我们的 Activity 就启动成功了

#### 启动新的进程

下面来分析我们的第二种情况，我们可以跟踪到 `ActivityManagerService` 的 `startProcessLocked 方法， 这个方法最终会走到自己的重载方法:


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/11.jpeg)

如果我们启动的是一个 webview service,  则会走到 `startWebView` ，这里我们不考虑，所以我们分析的是 `Process.start` 这种初始化一个普通进程的情况。


这个方法最后调用了 `ZygoteProcess` 的 `start` 方法

![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/12.jpeg)

这里我们也可以大致分析出来，这里就是在通过 socket 通信请求 `Zygote` 进程 fork 一个子进程，作为新的 APP 进程，具体流程本篇文章暂时不做深究。


最终我们会启动 `ActivityThread` 的 `main` 方法，继续走到 `attach` 方法

这里我们能看到启动主线程的 Looper， 创建系统 Context 等工作，最终我们走到 `ApplicationThread` 的 `bindApplication` , 代码这里就不贴了，这里负责了 Application 在初始化的时候的各种工作。包括 `LoadedAPK` 的 `makeApplication` 过程。

```java
if (normalMode) {
	try {
		if (mStackSupervisor.attachApplicationLocked(app)) {
			didSomething = true;
		}
	} catch (Exception e) {
		Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
		badApp = true;
	}
}
```

这里会发现，正常模式下，我们走到了 `ActivityStackSupervisor` 的 `attachApplicationLocked` 方法，后面就又会和第一部分介绍的一样，走到 `realStartActivityLocked` 方法，去创建并执行 Activity 的生命周期。

#### 总结
到这里，Activity 的启动流程就大致梳理出来了。基本就是，`Instrumentation` 负责 Activity 的创建和中转， `ActivityStackSupervisor` 负责 Activity的 栈管理。Activity 都通过了 `ActviityServerManager`   来进行管理。

大概的关系如下图所示：


![](https://raw.githubusercontent.com/shaomaicheng/Article/master/imgs/activitylaunch/13.jpeg)


#### 后续

这里我只是对Activity的启动流程做了一个简单的梳理。我们会发现每个模块和细节都有几百几百行的代码。非常的复杂。下面的内容有兴趣大家也可以细细探究。

* 存在的 Activity 是怎么管理的，怎么走 onResume 去恢复的
* Activity 不同的 launch mode是怎么处理的
* zygote fork 新的app进程的细节
* LoadedApk 是怎么加载 apk 的内容的
* Activity 初始化完成后，内部是 window 又是怎么初始化并且渲染上 UI 内容的











