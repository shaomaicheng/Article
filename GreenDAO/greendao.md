Last login: Mon Apr  2 20:57:53 on ttys000
➜  ~ ls
Android-Project Applications    Documents       Library         Music           Public          go_pro
Android-demo    Desktop         Downloads       Movies          Pictures        go
➜  ~ mkdir https://github.com/shaomaicheng/Article.gi
➜  ~ mkdir article
➜  ~ cd a
cd: no such file or directory: a
➜  ~ cd article
➜  article ls
➜  article git clone https://github.com/shaomaicheng/Article.git
Cloning into 'Article'...
warning: You appear to have cloned an empty repository.
➜  article ls
Article
➜  article cd Article
➜  Article git:(master) ls
➜  Article git:(master) mkdir imgs
➜  Article git:(master) cd ..
➜  article ls
Article
➜  article cd ..
➜  ~ ls
Android-Project Applications    Documents       Library         Music           Public          go
Android-demo    Desktop         Downloads       Movies          Pictures        article         go_pro
➜  ~ cd Applications
➜  Applications cd //
➜  / cd ..
➜  / cd ~/article
➜  article l.s
zsh: command not found: l.s
➜  article ls
Article
➜  article cd Article
➜  Article git:(master) ls
imgs
➜  Article git:(master) cd ~/Desktop
➜  Desktop ls
android-hub golang-hub  snapshot.py sqlite
➜  Desktop ls -al
total 32
drwx------@  7 chenglei  staff    224  4  1 19:45 .
drwxr-xr-x+ 41 chenglei  staff   1312  4  3 20:21 ..
-rw-r--r--@  1 chenglei  staff  10244  4  1 16:28 .DS_Store
drwxr-xr-x   6 chenglei  staff    192  4  1 18:40 android-hub
drwxr-xr-x   3 chenglei  staff     96  4  1 21:58 golang-hub
-r-xr-xr-x@  1 chenglei  staff    542  8  9  2017 snapshot.py
drwxr-xr-x   3 chenglei  staff     96  3 31 18:18 sqlite
➜  Desktop cd ..
➜  ~ ls
Android-Project Applications    Documents       Library         Music           Public          go
Android-demo    Desktop         Downloads       Movies          Pictures        article         go_pro
➜  ~ cd ~
➜  ~ cd Desktop
➜  Desktop ls
```
android-hub golang-hub  snapshot.py sqlite
➜  Desktop cd ../Downloads/
➜  Downloads ls
GreenDAO.png                        IntelliJIDEALicenseServer           下载组件SDk.pdf                     白话深度学习与TensorFlow.pdf
➜  Downloads mv GreenDAO.png ~/article/Article/imgs
➜  Downloads cd ~/article/Article/imgs
➜  imgs git:(master) ✗ ls
GreenDAO.png
➜  imgs git:(master) ✗ cd ..
➜  Article git:(master) ✗ ls
imgs
➜  Article git:(master) ✗ git add .
➜  Article git:(master) ✗ git push
error: src refspec refs/heads/master does not match any.
error: failed to push some refs to 'https://github.com/shaomaicheng/Article.git'
➜  Article git:(master) ✗ git commit -m "添加文档"
[master (root-commit) 6537b9e] 添加文档
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 imgs/GreenDAO.png
➜  Article git:(master) git push
Username for 'https://github.com': shaomaicheng
Password for 'https://shaomaicheng@github.com':
Counting objects: 4, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (4/4), 56.07 KiB | 14.02 MiB/s, done.
Total 4 (delta 0), reused 0 (delta 0)
To https://github.com/shaomaicheng/Article.git
 * [new branch]      master -> master
➜  Article git:(master) ls
imgs
➜  Article git:(master) ls
imgs
➜  Article git:(master) cd ..
➜  article ls
Article
➜  article mkdir GreenDAO
➜  article ls
Article  GreenDAO
➜  article cd GreenDAO
➜  GreenDAO qls
zsh: command not found: qls
➜  GreenDAO ls
➜  GreenDAO ls
➜  GreenDAO touch greendao.md
➜  GreenDAO vim greendao.md
➜  GreenDAO git add .
fatal: Not a git repository (or any of the parent directories): .git
➜  GreenDAO lks
zsh: command not found: lks
➜  GreenDAO cd ..
➜  article cd ..
➜  ~ cd article
➜  article git add .
fatal: Not a git repository (or any of the parent directories): .git
➜  article cd Article
➜  Article git:(master) git add .
➜  Article git:(master) git commit -m "GreenDAO基础用法源码解析"
On branch master
Your branch is up-to-date with 'origin/master'.

nothing to commit, working tree clean
➜  Article git:(master) gp
Everything up-to-date
➜  Article git:(master) git pull
Already up-to-date.
➜  Article git:(master) git commit -m "GreenDAO基础用法源码解析"
On branch master
Your branch is up-to-date with 'origin/master'.

nothing to commit, working tree clean
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

* select 查询操作
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

可以发现GreenDao对于数据做了一次内存的缓存，每次查询的时候会从IdentityScope的map 中get一次数据。如果map中缓存拿到了，就使用缓存作为查询的结果。
* delete 删除的逻辑和增改查大致一样
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~

~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
~
                    identityScope.unlock();
        }
```

会走到loadCurrent方法
```java
 final protected T loadCurrent(Cursor cursor, int offset, boolean lock)
```

此处可以关于IdentityScope的代码
```javda
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

可以发现GreenDao对于数据做了一次内存的缓存，每次查询的时候会从IdentityScope的map 中get一次数据。如果map中缓存拿到了，就使用缓存作为查询的结果。

`关于数据库缓存的特性，我们需要视业务情况而定，如果需要拿到最新的数据库信息，可以使用DaoSession的clear方法删除缓存，源码如下`

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

* select 查询操作
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
