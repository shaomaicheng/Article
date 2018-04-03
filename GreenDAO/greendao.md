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