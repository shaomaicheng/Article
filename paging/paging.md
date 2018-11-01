来新公司半年多，最近一直在参与 Andorid  团队的架构升级工作。最近在图片选择库中使用了 paging 作为分页加载框架。顺便阅读了一下paging的源码。在这里记录一下。

初次接除 paging， 可能会一脸懵逼，感觉出来了很多 API， 不知道从哪里下手。我们先对 paging 的组成部分进行一个了解。

首先，我们按照 `列表分页加载` 这个行为进行一个基本的划分，分为 2 个部分， `数据` 和 `UI`， paging 就是按照这个来进行划分的

###### 数据

数据部分 paging 包括
* `PagedList` 一个继承了 `AbstractList` 的 `List` 子类， 包括了数据源获取的数据
* `DataSource` 数据源的概念，分别提供了 [PageKeyedDataSource](https://developer.android.google.cn/reference/android/arch/paging/PageKeyedDataSource)、[ItemKeyedDataSource](https://developer.android.google.cn/reference/android/arch/paging/ItemKeyedDataSource)、[PositionalDataSource](https://developer.android.google.cn/reference/android/arch/paging/PositionalDataSource)， 在数据源中，我们可以定义我们自己的数据加载逻辑。


###### UI
UI 部分 paging 提供了一个新的 `PagedListAdapter`, 在实例化这个 `Adapter` 的时候，我们需要提供一个自己实现的 `DiffUtil.ItemCallback` 或者 `AsyncDifferConfig`


##### 入门

以分页数据源 `PageKeyedDataSource` 为例

创建一个数据源， 其中 Language 为 demo 中的实体对象

```kotlin
class LanguageDataSource: PageKeyedDataSource<Int, Language>()
```

实现三个 override 方法

```kotlin
override fun loadInitial(params: LoadInitialParams<Int>, callback: LoadInitialCallback<Int, Language>) {
}
```

```koltin
override fun loadAfter(params: LoadParams<Int>, callback: LoadCallback<Int, Language>) {
}
```

```kotlin
override fun loadBefore(params: LoadParams<Int>, callback: LoadCallback<Int, Language>) {
}
```

着 3 个方法，依次解释为

* 初次加载
* 后面一页加载
* 前一页加载


我们给第一页数据填充逻辑

```kotlin
LanguageRepository.requestLanguages({datas->
    if (datas.code == 200) {
        val languages = datas.data
        Handler(Looper.getMainLooper()).post {
           callback.onResult(languages, null, 1)
        }
    } else {
    }
 }, {t->
     Log.e(javaClass.simpleName, "${t.message}")
})
```

其中 `LanguageRepository` 是利用 `retrofit` 请求了一个 Language 对象的列表。 我们调用
`callback.onResult` 就会刷新 RecyclerView 的视图

`loadAfter` 的实现大致与 `loadInitial` 一致，这里不做赘述。

我们再来看一下 UI 层，我们定义一个 `PagedListAdapter`

```kotlin
class LanguageAdapter(private val context: Context) : PagedListAdapter<Language, ViewHolder>(languageDiff)
```

这里我们需要 override 2个方法

```kotlin
 override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder
```

```kotlin
override fun onBindViewHolder(holder: ViewHolder, position: Int)
```

在 `onBindViewHolder` 中， 我们可以通过 `getItem(position)` 获取相对于的数据实例去进行 UI 的展示。


接下来是一个比较关键的部分，那就是如何连接 DATA 和 UI 这两部分。

```kotlin
val config = PagedList.Config.Builder()
    .setPageSize(15)
    .setPrefetchDistance(2)
    .setInitialLoadSizeHint(15)
    .setEnablePlaceholders(false)
    .build()

val pageList = PagedList.Builder(LanguageDataSource(), config)
    .setNotifyExecutor {
         Handler(Looper.getMainLooper()).post {it.run()}
    }
    .setFetchExecutor(Executors.newFixedThreadPool(2))
    .build()

 adapter.submitList(pageList)
```

在这里， pageList 的 `NotifyExecutor` 和  `FetchExecutor` 也是必须设置的。在 `Android arch componet` 完整的架构中，更推荐使用构建一个 `PageList` 的 `LiveData` 的方式。但是不使用也没有关系，`arch compoent` 的完整内容在这里不做过多的描述。具体的详细使用可以查看[google的实例源码](https://github.com/googlesamples/android-architecture-components/tree/master/PagingWithNetworkSample)


在大致了解了 paging 的组成部分后，我们会开始好奇，那我们到底为什么需要 paging 呢， 他和我们之前普通的使用方式有什么区别呢，我们可以在源码中寻找到答案。

我们可以在 2 个部分的真正对接处作为切入点进行分析，查看 `PagedList.Builder#build()` 的源码：

```java
return PagedList.create(
    mDataSource,
    mNotifyExecutor,
    mFetchExecutor,
    mBoundaryCallback,
    mConfig,
    mInitialKey);
```

继续查看

```java
return new ContiguousPagedList<>(contigDataSource,
    notifyExecutor,
    fetchExecutor,
    boundaryCallback,
    config,
    key,
    lastLoad);
```

跟到这个类的构造方法，可以看到如下逻辑
```java
mDataSource.dispatchLoadInitial(key,
    mConfig.initialLoadSizeHint,
    mConfig.pageSize,
    mConfig.enablePlaceholders,
    mMainThreadExecutor,
    mReceiver);
```

这里以 `PageKeyedDataSource` 为例， 其他的 `DataSource` 对象同理

查看 `dispatchLoadInital` 方法

```java
LoadInitialCallbackImpl<Key, Value> callback =
                new LoadInitialCallbackImpl<>(this, enablePlaceholders, receiver);
loadInitial(new LoadInitialParams<Key>(initialLoadSize, enablePlaceholders), callback);

callback.mCallbackHelper.setPostExecutor(mainThreadExecutor);
```

这里我们可以看到， `loadInitial` 就是我们需要在 override 的方法之一。那我们里面调用 callback 的 onResult 方法到底发生了什么呢？

查看 `LoadInitialCallbackImpl#onResult()` 的源码，关键逻辑如下

```java
mDataSource.initKeys(previousPageKey, nextPageKey);
int trailingUnloadedCount = totalCount - position - data.size();
if (mCountingEnabled) {
    mCallbackHelper.dispatchResultToReceiver(new PageResult<>(
    data, position, trailingUnloadedCount, 0));
} else {
    mCallbackHelper.dispatchResultToReceiver(new PageResult<>(data, position));
}
```

查看 `dispatchResultToReceiver`

![image](http://upload-images.jianshu.io/upload_images/1523772-4c2c96a6a49a05b3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

继续查看 `onPageResult` 方法

![image](http://upload-images.jianshu.io/upload_images/1523772-fa18ac3648e50d15?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们关注一下 `init` 时候的逻辑

```java
mStorage.init(pageResult.leadingNulls, page, pageResult.trailingNulls,
                        pageResult.positionOffset, ContiguousPagedList.this);
```

`init` 的逻辑很简单，只有 2 行

```java
init(leadingNulls, page, trailingNulls, positionOffset);
callback.onInitialized(size());
```

![image](http://upload-images.jianshu.io/upload_images/1523772-0260a6e1015f5ef3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这里， 我们可以看见关键的逻辑

```java
mPages.clear();
mPages.add(page);
```

这里，和 `PageList` 绑定的数据就发生了变化。之后我们把 `PageList` submit 给了 `adapter` 那么，数据就发生了更新。

初始加载我们看完了，那么，剩下的数据是如何加载的呢


我们反过来看 `RecyclerView`， 如果我们滑动列表或者其他操作的时候，很自然会调用 adapter 的 bind 方法。那么，我们去查看 `PagedListAdapter#getItem` 的源码。

```java
return mDiffer.getItem(position);
```

![image](http://upload-images.jianshu.io/upload_images/1523772-2fe02912d9f45e88?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

查看 `PageList` 的 `loadAround`

```java
loadAroundInternal(index);
```

继续，

![image](http://upload-images.jianshu.io/upload_images/1523772-4e6c31deedf40e54?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```java
if (mAppendItemsRequested > 0) {
    scheduleAppend();
}
```

查看 `scheduleAppend` 的实现

```java
mBackgroundThreadExecutor.execute(new Runnable() {
            @Override
            public void run() {
                if (isDetached()) {
                    return;
                }
                if (mDataSource.isInvalid()) {
                    detach();
                } else {
                    mDataSource.dispatchLoadAfter(position, item, mConfig.pageSize,
                            mMainThreadExecutor, mReceiver);
                }
            }
        });
```

这里，我们看到了 `dispatchLoadAfter` 方法的调用，之后的逻辑和之前的 `dispathLoadInitial` 就非常的类似了。


最终，会调用到如下逻辑
![image](http://upload-images.jianshu.io/upload_images/1523772-b8f65ff6a14148b8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里会走 `AsyncPagedListDiffer` 的 `PagedList.Callback` 的回调

![image](http://upload-images.jianshu.io/upload_images/1523772-2142830b94143dab?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里，callback 是和 adapter 关联起来的。所以会在这里刷新列表。

最后，我们看一下 `Adapter` 的 submit 方法，最后可以看到这样的逻辑
![image](http://upload-images.jianshu.io/upload_images/1523772-acc7aacfc4bb6b78?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到 paging 是利用了 `DiffUtils` 对 RecyclerView 进行刷新的。这样我们也无需担心 paging 会存在性能问题。


##### 理解

最后谈一下对 paging 的理解。 一般情况下，我们最原始的方式，列表 UI 所在的部分，是需要知道数据的来源等逻辑部分，我们在常见的 mvp 模式中，会对数据和 UI 进行分层。 而 paging 就利用一系列的封装， 提供了更加通用的 API 调用来做这些事情。更通俗点说，就是实现了分页加载结构中的 Presenter 层及 Presenter层的下游处理部分。

这种模式，业务的编写者，可以把 UI 部分的代码模板化， 只需要关心业务逻辑，并且把业务逻辑中的数据获取写在 DataSource 中，使分页加载的操作解耦程度更高。