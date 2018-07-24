---
title: Room版本迁移遇到的问题
tags:
  - Room
  - android-architecture-components
  - Android
date: 2018/7/23 20:30:00
update: 2018/7/23 21:01:00
---


## Room版本迁移遇到的问题

> 举个例子，有下面2种场景

* V1->V2： 增加字段
* V2->V3： 修改原有字段类型，删除原有字段

### 增加字段(V1->V2)

#### 先实现个V1版本


首先实现一个Entity类
```java
@Entity(tableName = "user")
public class User {
  @PrimaryKey(autoGenerate = true)
  private int id;
  private String name;
  private int age;
  private long timestamp;
  ...//setter,getter
}
```
该类一共有4个字段,其中id为自增长的主键。
接下来实现一个简单的DAO层接口:
```java
@Dao
public interface UserDao {

  @Insert(onConflict = OnConflictStrategy.REPLACE)
  void addUser(User user);

  @Query("SELECT * FROM user")
  List<User> getAllUsers();

}
```
这里该接口就简单的增加和查询，

再下来实现一个Database类：
```java
@Database(entities = {User.class},version = 1,exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
  private static AppDatabase INSTANCE;
  private static final String DB_NAME="user_test.db";
  public static AppDatabase getINSTANCE(){
    if (INSTANCE==null){
      synchronized (AppDatabase.class){
        if (INSTANCE==null){
          INSTANCE=Room.databaseBuilder(App.context,AppDatabase.class,DB_NAME)
              .build();
        }
      }
    }
    return INSTANCE;
  }
  //上面实现的DAO接口
  public abstract UserDao getUserDao();
}
```
最后,往数据库中写入一条数据就完成本次的V1版本

```java
new Thread(new Runnable() {
      @Override
      public void run() {
        User user=new User();
        user.setName(UUID.randomUUID().toString());
        user.setAge((int) (100*Math.random()));
        user.setTimestamp(System.currentTimeMillis());
        AppDatabase.getINSTANCE().getUserDao().addUser(user);
      }
    }).start();
```

#### 实现V2版本

现在我们要实现V2版本，先来看看Entity类中有何变化:
```java
@Entity(tableName = "user")
public class User {
  @PrimaryKey(autoGenerate = true)
  private int id;
  private String name;
  private int age;
  private long timestamp;
  //V2添加字段
  private int weight;
  private String sex;
  ...//setter,getter
}
```
从上面代码中可以发现新增了weight和sex两个字段，接下来需要提供migration以使数据库版本变化时数据可以保存下来：

```java
@Database(entities = {User.class},version = 2,exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
  private static AppDatabase INSTANCE;
  private static final String DB_NAME="user_test.db";
  public static AppDatabase getINSTANCE(){
    if (INSTANCE==null){
      synchronized (AppDatabase.class){
        if (INSTANCE==null){
          INSTANCE=Room.databaseBuilder(App.context,AppDatabase.class,DB_NAME)
              .addMigrations(MIGRATION_1_2)
              .build();
        }
      }
    }
    return INSTANCE;
  }
  //上面实现的DAO接口
  public abstract UserDao getUserDao();

  static final Migration MIGRATION_1_2=new Migration(1,2) {
    @Override
    public void migrate(@NonNull SupportSQLiteDatabase database) {
      database.execSQL("ALTER TABLE user ADD COLUMN weight INTEGER ");
      database.execSQL("ALTER TABLE user ADD COLUMN sex TEXT");
    }
  };
}
```
这里对AppDatabase中进行了一些改变，首先将**version**变为了2,同时调用了addMigrations方法,在migrate方法中我们执行了2条SQL语句,增加INTEGER类型的weight和TEXT类型的sex这两个字段,现在升级数据库的准备工作做完了,我们打包运行一下。

> 程序出现崩溃-_-!

```java
java.lang.IllegalStateException: Migration didn't properly handle user(cc.noharry.dbtest.User).
     Expected:
    TableInfo{name='user', columns={name=Column{name='name', type='TEXT', affinity='2', notNull=false, primaryKeyPosition=0}, timestamp=Column{name='timestamp', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=0}, age=Column{name='age', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=0}, sex=Column{name='sex', type='TEXT', affinity='2', notNull=false, primaryKeyPosition=0}, weight=Column{name='weight', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=0}, id=Column{name='id', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=1}}, foreignKeys=[], indices=[]}
     Found:
    TableInfo{name='user', columns={name=Column{name='name', type='TEXT', affinity='2', notNull=false, primaryKeyPosition=0}, timestamp=Column{name='timestamp', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=0}, age=Column{name='age', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=0}, sex=Column{name='sex', type='TEXT', affinity='2', notNull=false, primaryKeyPosition=0}, weight=Column{name='weight', type='INTEGER', affinity='3', notNull=false, primaryKeyPosition=0}, id=Column{name='id', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=1}}, foreignKeys=[], indices=[]}
```
细看可以发现可以发现这两者有一处不同：

Expected：

> weight=Column{name='weight', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=0}

Found:

> weight=Column{name='weight', type='INTEGER', affinity='3', notNull=false, primaryKeyPosition=0}

不同点就在于**notNull**这项,因此问题就出在之前migrate中的第一条SQL语句,现在我们修改那条语句:

```java
static final Migration MIGRATION_1_2=new Migration(1,2) {
    @Override
    public void migrate(@NonNull SupportSQLiteDatabase database) {
      database.execSQL("ALTER TABLE user ADD COLUMN weight INTEGER NOT NULL DEFAULT 0");
      database.execSQL("ALTER TABLE user ADD COLUMN sex TEXT");
    }
  };
```
> 为什么除了加入NOT NULL 之外还要加上DEFAULT 0 ?

> 因为加入了NOT NULL不能为空，所以要给它赋上初始值，具体初始值可根据你的实际情况考虑，在此默认写0.

![](v2.png)

现在重新打包运行,程序不崩了,导出数据库一看，字段也成功加了上去，现在从v1到v2升级成功。

#### 实现V3版本
