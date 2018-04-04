# GreenDao源码

## 简述
DaoMaster、具体的Dao 和 DaoSession对象为greedao生成的代码
从平时的使用可以看出他们的作用
* DaoMaster
	`GreenDao的总入口，负责整个库的运行，实现了SqliteOpenHelper`
* DaoSession
	`会话层，操作Dao的具体对象，包括DAO对象的注册`
* xxEntity
	`实体类，和表内容一一对应`
* xxDao
	`生成的DAO对象，进行具体的数据库操作`
	
这几个类的关系如下UML图:
![](https://github.com/shaomaicheng/Article/blob/master/imgs/GreenDAO.png?raw=true)

Dao对象需要依赖DaoConfig对象

```java
public DaoConfig(Database db, Class<? extends AbstractDao<?, ?>> daoClass) 
```

DaoConfig对象需要传入具体的DaoClass类型

```java
reflectProperties(Class<? extends AbstractDao<?, ?>> daoClass))
```

获取DAO里面的Properties 所有static或者public字段
```java
pkProperty = pkColumns.length == 1 ? lastPkProperty : null;
            statements = new TableStatements(db, tablename, allColumns, pkColumns);

            if (pkProperty != null) {
                Class<?> type = pkProperty.type;
                keyIsNumeric = type.equals(long.class) || type.equals(Long.class) || type.equals(int.class)
                        || type.equals(Integer.class) || type.equals(short.class) || type.equals(Short.class)
                        || type.equals(byte.class) || type.equals(Byte.class);
            } else {
                keyIsNumeric = false;
            }
```

这里会获取所有的数据库字段，顺便判断表主键是否是数字类型


## 数据库加密
GreenDAO创建会话的时候我们一般会调用如下代码获取DaoSession对象：
```java
DevOpenHelper helper = new DevOpenHelper(this, ENCRYPTED ? "notes-db-encrypted" : "notes-db");
Database db = ENCRYPTED ? helper.getEncryptedWritableDb("super-secret") : helper.getWritableDb();
daoSession = new DaoMaster(db).newSession();
```

在AbstractDAO类中，有一个db字段，最终的数据库操作以及事务的开启都会通过这个对象开启。GreenDAO中存在Database和DatabaseStatement2个接口
这2个接口分别存在2个子类StandardDatabase、EncryptedDatabase  和 StandardDatabaseStatement 、EncryptedDatabaseStatement，这几个类其实都是一个代理模式
```java
public class StandardDatabaseStatement implements DatabaseStatement {
    private final SQLiteStatement delegate;
    public StandardDatabaseStatement(SQLiteStatement delegate) {
        this.delegate = delegate;
    }

    @Override
    public void execute() {
        delegate.execute();
    }
}
```
```java
public class EncryptedDatabase implements Database {
    private final SQLiteDatabase delegate;
    public EncryptedDatabase(SQLiteDatabase delegate) {
        this.delegate = delegate;
    }

    @Override
    public Cursor rawQuery(String sql, String[] selectionArgs) {
        return delegate.rawQuery(sql, selectionArgs);
    }
}
```
这里会发现GreenDAO调用了一个三方库叫做sqlcipher.，提供了Sqlite的数据库加密功能。所以GreenDAO在创建一次会话的时候可以指定数据库是否加密。如果没加密，会使用Android的Sqlite API去操作数据库，如果加密，则使用sqlcipher提供发API去操作数据库。


## GreenDAO的增删改查
GreenDAO通过AbstractDAO类实现数据库的增删改查逻辑，此处分析几个常用的方法
* **insert 插入数据**
```java
public long insert(T entity) {
        return executeInsert(entity, statements.getInsertStatement(), true);
    }
```

内部逻辑伪代码:
```java
if (db.isDbLockedByCurrentThread()) {
		// 数据库连接被其他占用
            rowId = insertInsideTx(entity, stmt);
        } else {
            // Do TX to acquire a connection before locking the stmt to avoid deadlocks (开启事务防止死锁)
            db.beginTransaction();
            try {
                rowId = insertInsideTx(entity, stmt);
                db.setTransactionSuccessful();
            } finally {
                db.endTransaction();
            }
        }
        if (setKeyAndAttach) {
            updateKeyAfterInsertAndAttach(entity, rowId, true);
        }
```

*  update 更新数据
```java
public void update(T entity) 
```

update的源码大概为
```java
assertSinglePk();  //判断是否只有一个主键
DatabaseStatement stmt = statements.getUpdateStatement();
if (db.isDbLockedByCurrentThread()) {
            synchronized (stmt) {
                if (isStandardSQLite) {
                    updateInsideSynchronized(entity, (SQLiteStatement) stmt.getRawStatement(), true);
                } else {
                    updateInsideSynchronized(entity, stmt, true);
                }
            }
        } else {
            // Do TX to acquire a connection before locking the stmt to avoid deadlocks
            db.beginTransaction();
            try {
                synchronized (stmt) {
                    updateInsideSynchronized(entity, stmt, true);
                }
                db.setTransactionSuccessful();
            } finally {
                db.endTransaction();
            }
        }
```

关注updateInsideSynchronized方法
```java
bindValues(stmt, entity);
        int index = config.allColumns.length + 1;
        K key = getKey(entity);
        if (key instanceof Long) {
            stmt.bindLong(index, (Long) key);
        } else if (key == null) {
            throw new DaoException("Cannot update entity without key - was it inserted before?");
        } else {
            stmt.bindString(index, key.toString());
        }
        stmt.execute();
        attachEntity(key, entity, lock);
```
和insert方法类似，添加了主键判断必须存在key的逻辑
其中添加了
```java
int index = config.allColumns.length + 1;
```
这个index bind的字段是update的条件语句，和id进行绑定
换成sql语句就是
```sql
where id = '?'
```

* **select 查询操作**
查询的代码和insert、update大致流程一致
会通过QueryBuilder构造查询条件，在QueryBuilder中list方法获取数据
```java
public List<T> list() {
	return build().list();
}
```

```java
public List<T> list() {
   checkThread();
   Cursor cursor = dao.getDatabase().rawQuery(sql, parameters);
   return daoAccess.loadAllAndCloseCursor(cursor);
}
```

会走到AbstractDao类的loadAllFromCursor方法
```java
if (cursor.moveToFirst()) {
            if (identityScope != null) {
                identityScope.lock();
                identityScope.reserveRoom(count);
            }

            try {
                if (!useFastCursor && window != null && identityScope != null) {
                    loadAllUnlockOnWindowBounds(cursor, window, list);
                } else {
                    do {
                        list.add(loadCurrent(cursor, 0, false));
                    } while (cursor.moveToNext());
                }
            } finally {
                if (identityScope != null) {
                    identityScope.unlock();
                }
            }
        }
```

会走到loadCurrent方法
```java
 final protected T loadCurrent(Cursor cursor, int offset, boolean lock) 
```

此处可以关于IdentityScope的代码
```java
T entity = lock ? identityScopeLong.get2(key) : identityScopeLong.get2NoLock(key);
            if (entity != null) {
                return entity;
            } else {
                entity = readEntity(cursor, offset);
                attachEntity(entity);
                if (lock) {
                    identityScopeLong.put2(key, entity);
                } else {
                    identityScopeLong.put2NoLock(key, entity);
                }
                return entity;
            }
```

* **delete 删除的逻辑和增改查大致一样**

## GreenDAO缓存
在上面query的源码中可以发现GreenDAO对于数据做了一次内存的缓存，每次写数据库的时候会从IdentityScope的map 中put一次数据。每次读数据库的时候会从IdentityScope的map中get一次数据。如果map中缓存拿到了，就使用缓存作为查询的结果。

在查询和更新数据的时候会执行
```java
attachEntity(K key, T entity, boolean lock)
```
方法，具体逻辑如下
```java
if (identityScope != null && key != null) {
            if (lock) {
                identityScope.put(key, entity);
            } else {
                identityScope.putNoLock(key, entity);
            }
        }
```

`关于数据库缓存的特性，我们需要视业务情况而定，在网上还是能搜到GreenDAO查询数据没有拿到最新结果的bug， 如果出现这个bug且需要拿到最新的数据库信息，可以使用DaoSession的clear方法删除缓存，源码如下`

```java
//DaoSession
public void clear() {
   noteDaoConfig.clearIdentityScope();
}
```

```java
//DaoConfig
public void clearIdentityScope() {
        IdentityScope<?, ?> identityScope = this.identityScope;
        if(identityScope != null) {
            identityScope.clear();
        }
    }
```

## GreenDAO的异步操作
有些时候，我们希望异步的操作数据库，GreenDAO给我们提供了异步的方法。
大致用法为：
```java
AsyncSession asyncSession = daoSession.startAsyncSession();
asyncSession.setListener(new AsyncOperationListener() {
   @Override
   public void onAsyncOperationCompleted(AsyncOperation operation) {
       AsyncOperation.OperationType type = operation.getType();
       Log.e(TAG, type.name());
   }
});
asyncSession.insert(note);
```

我们获取一个AsyncSession对象进行异步的数据库操作，并且可以设置AsyncOperation对异步操作进行监听。但是为什么所有的异步操作是同一个Listener回调呢？我们可以在源码中找到答案。

查看AsyncSession的insert方法

```java
public AsyncOperation insert(Object entity) {
    return insert(entity, 0);
}
```

最后可以走到enqueEntityOperation方法
```java
AbstractDao<?, ?> dao = daoSession.getDao(entityClass);
AsyncOperation operation = new AsyncOperation(type, dao, null, param, flags | sessionFlags);
executor.enqueue(operation);
return operation;
```

可以发现有一个异步操作的Executor，AsyncOperationExecutor
查看enqueue方法

```java
operation.sequenceNumber = ++lastSequenceNumber;
queue.add(operation);
countOperationsEnqueued++;
if (!executorRunning) {
   executorRunning = true;
   executorService.execute(this);
}
```
可以看到它把异步加入到一个队列里面排队等待。在线程池中执行这些操作。

查看run方法
```java
//线程池run方法代码
while (true) {
   AsyncOperation operation = queue.poll(1, TimeUnit.SECONDS);
   if (operation == null) {
      synchronized (this) {
	      // Check again, this time in synchronized to be in sync with enqueue(AsyncOperation)
	      operation = queue.poll();
	      if (operation == null) {
	          // set flag while still inside synchronized
	          executorRunning = false;
	          return;
	       }
	    }
	}
    if (operation.isMergeTx()) {
        // Wait some ms for another operation to merge because a TX is expensive
        AsyncOperation operation2 = queue.poll(waitForMergeMillis, TimeUnit.MILLISECONDS);
        if (operation2 != null) {
             if (operation.isMergeableWith(operation2)) {
                  mergeTxAndExecute(operation, operation2);
              } else {
                   // Cannot merge, execute both
	             executeOperationAndPostCompleted(operation);
                 executeOperationAndPostCompleted(operation2);
              }
              continue;
         }
   }
   executeOperationAndPostCompleted(operation);
}
```

这段代码我们可以看到，每次从队列中拿一个异步操作对象执行逻辑。这也解释了为什么外层只需要set一个Listener。GreenDAO的异步操作是所有的数据库操作在一个子线程中进行同步操作。

最终代码会走到executeOperation方法
```java
switch (operation.type) {
    case Delete:
        operation.dao.delete(operation.parameter);
        break;
    default:
	    break;
}
```
这里会执行具体的DAO对象的数据库操作方法。


## 和ReactiveX的结合
GreenDAO提供了API与rxjava结合使用，代码如下：
```java
RxDao<xxEntity, Long> xxDao = daoSession.getXXDao().rx();
xxDao.insert(xxEntity)
	.observerOn(AndroidSchedules.mainThread())
	.subscribe(new Action1<xxEntity>() {
		@Override
		public void call(xxEntity entity) {
			// insert success
		}
	})
```

我们来简单分析下源码看看RxJava是如何和GreenDAO结合的
查看AbstractDAO的rx()
```java
public RxDao<T, K> rx() {
	if (rxDao == null) {
	    rxDao = new RxDao<>(this, Schedulers.io());
	}
    return rxDao;
}
```

查看RxDAO的构造方法

```java
public RxDao(AbstractDao<T, K> dao, Scheduler scheduler) {
	super(scheduler);
    this.dao = dao;
}
```
包含了Dao对象和线程调度Scheduler对象，这里是io线程，即在异步线程执行
查看insert方法

```java
public Observable<T> insert(final T entity) {
        return wrap(new Callable<T>() {
            @Override
            public T call() throws Exception {
                dao.insert(entity);
                return entity;
            }
        });
    }
```
查看wrap方法， wrap方法里面也调用了一个重载的wrap方法。
参数为一个Callable对象，里面执行了数据库插入的逻辑，返回了实体类对象。
```java
protected <R> Observable<R> wrap(Callable<R> callable) {
	return wrap(RxUtils.fromCallable(callable));
}
```

```java
if (scheduler != null) {
	return observable.subscribeOn(scheduler);
} else {
    return observable;
}
```

这里的第二个wrap方法的参数observable是通过RxUtils.fromCallable获得的，查看这个方法的源码

```java
static <T> Observable<T> fromCallable(final Callable<T> callable) {
	return Observable.defer(new Func0<Observable<T>>() {
		@Override
		public Observable<T> call() {
			T result;
			try {
				result = callable.call();
			} catch (Exception e) {
				return Observable.error(e);
			}
			return Observable.just(result);
		}
	});
}
```

这里使用了defer操作符创建一个Observable对象。延迟创建，确保Observable被订阅后才执行。

以上就是rxjava和GreenDAO的结合使用的原理。和rx'java结合使用，会使GreenDAO尤其是异步操作写起来更加的优雅。


## GreenDAO代码生成
我们在GreenDAO使用的时候，会自动生成DaoMaster，DAO对象等等java文件。大致翻阅了DaoGenerator这个Module里面的代码。发现有模板引擎的库依赖：
```groovy
compile 'org.freemarker:freemarker:2.3.23'
```
并且发现了freemarker的模板文件
在DaoGenerator/src-template里面，有很多的.ftl文件。GreenDAO的文件就是通过freemarker模板生成的Java文件。具体的原理下次分析后会单独再写一篇博客。

## 总结
通过对GreenDAO的源码分析，可以发现GreenDAO号称自己是性能最好的ORM库也是有原因的，GreenDAO的特点总结为以下：

* 使用了静态代码生成，不通过反射运行时生成代理类，提升了性能
* 使用了SQLiteStatement 
* 同时提供了同步和异步的数据库操作方式
* 数据库提供内存缓存，更高效的查询
* 基于sqlcipher提供加密数据库
* 提供RxJava的API，异步操作更高效 