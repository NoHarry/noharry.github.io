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
> 本文的Room源码基于1.1.1版本

对于遇到的问题举个实际例子，有下面2种场景

* V1->V2： 增加字段
* V2->V3： 修改原有字段类型，删除原有字段

### 增加字段(V1 -> V2)

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
---

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

---

### 修改原有字段类型，删除原有字段(V2 -> V3)
#### 实现V3版本
现在来看看V3版本的Entity发生了什么变化:

```java
@Entity(tableName = "user")
public class User {
  @PrimaryKey(autoGenerate = true)
  private int id;
  @NonNull
  private String name;
  private int age;
  private String weight;
  private String sex;
  ...//setter,getter
}
```
对比V2版本的Entity，V3版本将**weight**字段的类型从int变为了String，删除了**timestamp**字段，同时给**name**字段加上了@NonNull的注解(这一改动产生的影响稍后说明),接下来我们来看看AppDatabase中的改动：

```java
@Database(entities = {User.class},version = 3,exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
  private static AppDatabase INSTANCE;
  private static final String DB_NAME="user_test.db";
  public static AppDatabase getINSTANCE(){
    if (INSTANCE==null){
      synchronized (AppDatabase.class){
        if (INSTANCE==null){
          INSTANCE=Room.databaseBuilder(App.context,AppDatabase.class,DB_NAME)
              // .addMigrations(MIGRATION_1_2)
              .addMigrations(MIGRATION_2_3)
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

  static final Migration MIGRATION_2_3=new Migration(2,3) {
    @Override
    public void migrate(@NonNull SupportSQLiteDatabase database) {
      //1.创建一个新的符合Entity字段的新表user_new
      database.execSQL("CREATE TABLE user_new (id INTEGER NOT NULL,"
          + "name TEXT NOT NULL,"       //关注点1
          + "age INTEGER NOT NULL,"     //关注点2
          + "weight TEXT,"              //关注点3
          + "sex TEXT,"
          + "PRIMARY KEY(id))");
      //2.将旧表user中的数据拷贝到新表user_new中
      database.execSQL("INSERT INTO user_new(id,name,age,weight,sex) "
          + "SELECT id,name,age,weight,sex FROM user");
      //3.删除旧表user
      database.execSQL("DROP TABLE user");
      //4.将新表user_new重命名为user,升级完毕
      database.execSQL("ALTER TABLE user_new RENAME TO user");
    }
  };
}
```
对于V2升V3的流程我们可以在上面的注释中了解,现在我们来看看SQL中注释的那三个关注点，对于这三个关注点你可能会提出下面几个疑问：
* 1.为什么TEXT类型的name字段加上了NOT NULL的约束?
* 2.为什么INTEGER类型的age字段加上了NOT NULL约束？就算加上了NOT NULL为什么不像V1升V2时那样再加上DEFAULT 0之类的初始值?
* 3.为什么从INTEGER改为TEXT类型的weight字段不用加上NOT NULL?

对于这三个疑问我们先不急着回答，我们先来看看Room的数据库升级的过程大致是怎么进行的。

### Room数据库迁移的大致流程
> Room中使用了大量的注解,由于我对于注解还不够了解，所以有错误望指正。

* Room在编译期间通过使用apt来获取注解信息并通过javapoet来生成实现类的代码,以本文中的代码为例,这些实现类的位置在./app/build/generated/source/apt/debug/package/ 中。对于本文中的AppDatabase这个类，Room会生成一个名为AppDatabase_Impl的实现类在上述文件夹中。

* AppDatabase_Impl实现了4个抽象方法，一个是AppDatabase中的抽象方法,另外三个是AppDatabase的父类[RoomDatabase](https://android.googlesource.com/platform/frameworks/support/+/master/room/runtime/src/main/java/android/arch/persistence/room/RoomDatabase.java?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)中的三个抽象方法：

```java
//RoomDatabase初始化的时候调用,用于创建数据库,打开连接
protected abstract SupportSQLiteOpenHelper createOpenHelper(DatabaseConfiguration config);

//同步内存中数据到数据库
protected abstract InvalidationTracker createInvalidationTracker();

//清空数据
public abstract void clearAllTables();
```
* 在AppDatabase_Impl实现的createOpenHelper方法中创建了[RoomOpenHelper](https://android.googlesource.com/platform/frameworks/support/+/master/room/runtime/src/main/java/android/arch/persistence/room/RoomOpenHelper.java?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F),并实现了其代理的5个抽象方法:

```java
public abstract static class Delegate {
        ......
        //删除表,在RoomDatabase.Builder中使用.fallbackToDestructiveMigration()
        //这种升级策略的时候调用
        protected abstract void dropAllTables(SupportSQLiteDatabase database);

        //创建表
        protected abstract void createAllTables(SupportSQLiteDatabase database);

        protected abstract void onOpen(SupportSQLiteDatabase database);

        protected abstract void onCreate(SupportSQLiteDatabase database);

        //验证数据迁移
        protected abstract void validateMigration(SupportSQLiteDatabase db);
    }
```
* 在createAllTables的实现方法中,从第一条执行的语句我们可以看到**id、name、age**这三个字段都被设定了NOT NULL的约束：

```java
public void createAllTables(SupportSQLiteDatabase _db) {
        _db.execSQL("CREATE TABLE IF NOT EXISTS `user` (`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `name` TEXT NOT NULL, `age` INTEGER NOT NULL, `weight` TEXT, `sex` TEXT)");
        _db.execSQL("CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY,identity_hash TEXT)");
        _db.execSQL("INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, \"671ab1418436cf1c9b78819718f37062\")");
      }
```
* Room在检测到数据库版本有更新时会调用[RoomOpenHelper](https://android.googlesource.com/platform/frameworks/support/+/master/room/runtime/src/main/java/android/arch/persistence/room/RoomOpenHelper.java?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)中的onUpgrade方法:

```java
@Override
    public void onUpgrade(SupportSQLiteDatabase db, int oldVersion, int newVersion) {
        boolean migrated = false;
        if (mConfiguration != null) {
            List<Migration> migrations = mConfiguration.migrationContainer.findMigrationPath(
                    oldVersion, newVersion);
            if (migrations != null) {
                for (Migration migration : migrations) {
                    migration.migrate(db);
                }
                //验证数据迁移
                mDelegate.validateMigration(db);
                updateIdentity(db);
                migrated = true;
            }
        }
        ......
    }
```
* 在onUpgrade方法中又会调用AppDatabase_Impl中实现的validateMigration方法,在这个方法中验证了当前数据库的表结构与根据Entity生成的表结构是否相同,如果校验通过那升级的大致流程就结束了:
```java
protected void validateMigration(SupportSQLiteDatabase _db) {
      //将根据Entity生成的Column对象存入map集合
       final HashMap<String, TableInfo.Column> _columnsUser = new HashMap<String, TableInfo.Column>(6);
       _columnsUser.put("id", new TableInfo.Column("id", "INTEGER", true, 1));
       _columnsUser.put("name", new TableInfo.Column("name", "TEXT", true, 0));
       _columnsUser.put("age", new TableInfo.Column("age", "INTEGER", true, 0));
       _columnsUser.put("sex", new TableInfo.Column("sex", "TEXT", true, 0));
       _columnsUser.put("timestamp", new TableInfo.Column("timestamp", "INTEGER", true, 0));
       _columnsUser.put("weight", new TableInfo.Column("weight", "INTEGER", true, 0));
       final HashSet<TableInfo.ForeignKey> _foreignKeysUser = new HashSet<TableInfo.ForeignKey>(0);
       final HashSet<TableInfo.Index> _indicesUser = new HashSet<TableInfo.Index>(0);
       //上方创建的TableInfo
       final TableInfo _infoUser = new TableInfo("user", _columnsUser, _foreignKeysUser, _indicesUser);
       //读取当前数据库中的TableInfo
       final TableInfo _existingUser = TableInfo.read(_db, "user");
       //判断这两个TableInfo是否相同，是不是感觉有点眼熟~_~!
       if (! _infoUser.equals(_existingUser)) {
         throw new IllegalStateException("Migration didn't properly handle user(cc.noharry.dbtest.User).\n"
                 + " Expected:\n" + _infoUser + "\n"
                 + " Found:\n" + _existingUser);
       }
     }
```

* 之前在V1升V2的例子中因为缺少NOT NULL的约束导致在这一步中断了升级,那究竟应该如何判断是否应该加上这个约束呢？下面我们从源码中看看能不能发现问题的根源。

### Room源码

> 由于这次我们看的源码在[compiler](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)包中，官方用kotlin作为编码语言，鉴于我的kotlin水平还处于听过的水平，那我们试试联系上下文来完成这次的“阅读理解”吧，有错误的地方欢迎大家批评指正。

* 首先来到入口类[RoomProcessor.kt](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/RoomProcessor.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)中的初始化方法:
```kotlin
override fun initSteps(): MutableIterable<ProcessingStep>? {
        val context = Context(processingEnv)
        return arrayListOf(DatabaseProcessingStep(context))
    }
```

* 接着来到DatabaseProcessingStep类中的process方法:

```
override fun process(elementsByAnnotation: SetMultimap<Class<out Annotation>, Element>)
               : MutableSet<Element> {
           // 遍历注解获取所有的信息
           val databases = elementsByAnnotation[Database::class.java]
                   ?.map {
                       DatabaseProcessor(context, MoreElements.asType(it)).process()
                   }
          // 获取Dao注解中的所有方法
           val allDaoMethods = databases?.flatMap { it.daoMethods }
           // 用获取到的Dao中的方法生成实现类
           allDaoMethods?.let {
               prepareDaosForWriting(databases, it)
               it.forEach {
                   DaoWriter(it.dao, context.processingEnv).write(context.processingEnv)
               }
           }
           //用获取到的database中的信息生成实现类，也就是上面例子中的AppDatabase_Impl
           databases?.forEach { db ->
               DatabaseWriter(db).write(context.processingEnv)
          //根据database中注解的exportSchema的值决定是否生成schema文件
               if (db.exportSchema) {
                   val schemaOutFolder = context.schemaOutFolder
                   if (schemaOutFolder == null) {
                       context.logger.w(Warning.MISSING_SCHEMA_LOCATION, db.element,
                               ProcessorErrors.MISSING_SCHEMA_EXPORT_DIRECTORY)
                   } else {
                       if (!schemaOutFolder.exists()) {
                           schemaOutFolder.mkdirs()
                       }
                       val qName = db.element.qualifiedName.toString()
                       val dbSchemaFolder = File(schemaOutFolder, qName)
                       if (!dbSchemaFolder.exists()) {
                           dbSchemaFolder.mkdirs()
                       }
                       db.exportSchema(File(dbSchemaFolder, "${db.version}.json"))
                   }
               }
           }
           return mutableSetOf()
       }
```
* 从上面的注释中我们可以看到databases中的元素通过[DatabaseWriter](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/writer/DatabaseWriter.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)的父类[ClassWriter](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/writer/ClassWriter.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)中的write方法生成实现类,下面我们进入[DatabaseWriter.kt](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/writer/DatabaseWriter.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)看看生成createOpenHelper方法的语句：

```
private fun createCreateOpenHelper() : MethodSpec {
       val scope = CodeGenScope(this)
       return MethodSpec.methodBuilder("createOpenHelper").apply {
           addModifiers(Modifier.PROTECTED)
           returns(SupportDbTypeNames.SQLITE_OPEN_HELPER)

           val configParam = ParameterSpec.builder(RoomTypeNames.ROOM_DB_CONFIG,
                   "configuration").build()
           addParameter(configParam)

           val openHelperVar = scope.getTmpVar("_helper")
           val openHelperCode = scope.fork()
           SQLiteOpenHelperWriter(database)
                   .write(openHelperVar, configParam, openHelperCode)
           addCode(openHelperCode.builder().build())
           addStatement("return $L", openHelperVar)
       }.build()
   }
```
* 找到生成openHelper的方法SQLiteOpenHelperWriter(database).write(openHelperVar, configParam, openHelperCode),[SQLiteOpenHelperWriter.kt](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/writer/SQLiteOpenHelperWriter.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F):

```
private fun createValidateMigration(scope: CodeGenScope): MethodSpec {
        return MethodSpec.methodBuilder("validateMigration").apply {
            addModifiers(PROTECTED)
            val dbParam = ParameterSpec.builder(SupportDbTypeNames.DB, "_db").build()
            addParameter(dbParam)
            //遍历从database中获取到的关于Entity注解的信息，生成相应TableInfo.Column的代码
            database.entities.forEach { entity ->
                val methodScope = scope.fork()
                TableInfoValidationWriter(entity).write(dbParam, methodScope)
                addCode(methodScope.builder().build())
            }
        }.build()
    }
```
* 在[SQLiteOpenHelperWriter.kt](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/writer/SQLiteOpenHelperWriter.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)中找到了生成validateMigration的方法,现在我们离最后的真相又进了一步，现在进入[TableInfoValidationWriter.kt](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/writer/TableInfoValidationWriter.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)看看：

```
class TableInfoValidationWriter(val entity : Entity) {
    fun write(dbParam : ParameterSpec, scope : CodeGenScope) {
        val suffix = entity.tableName.stripNonJava().capitalize()
        val expectedInfoVar = scope.getTmpVar("_info$suffix")
        scope.builder().apply {
            val columnListVar = scope.getTmpVar("_columns$suffix")
            val columnListType = ParameterizedTypeName.get(HashMap::class.typeName(),
                    CommonTypeNames.STRING, RoomTypeNames.TABLE_INFO_COLUMN)

            addStatement("final $T $L = new $T($L)", columnListType, columnListVar,
                    columnListType, entity.fields.size)
            //遍历entity中的field获取属性，生成相应代码
            entity.fields.forEach { field ->
                addStatement("$L.put($S, new $T($S, $S, $L, $L))",
                        columnListVar, field.columnName, RoomTypeNames.TABLE_INFO_COLUMN,
                        /*name*/ field.columnName,
                        /*type*/ field.affinity?.name ?: SQLTypeAffinity.TEXT.name,
            // 注意：我们现在发现了NOT NULL
                        /*nonNull*/ field.nonNull,
                        /*pkeyPos*/ entity.primaryKey.fields.indexOf(field) + 1)

                        ....
            }

          }
}
```
* 现在我们发现了NOT NULL的约束，它看起来是field的属性，我们到[Field.kt](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/vo/Field.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)中看一下:

```
data class Field(val element: Element, val name: String, val type: TypeMirror,
                 var affinity: SQLTypeAffinity?,
                 val collate: Collate? = null,
                 val columnName: String = name,
                 /* means that this field does not belong to parent, instead, it belongs to a
                 * embedded child of the main Pojo*/
                 val parent: EmbeddedField? = null,
                 // index might be removed when being merged into an Entity
                 var indexed : Boolean = false) {

                   /** Whether the table column for this field should be NOT NULL */
                       val nonNull = element.isNonNull() && (parent == null || parent.isNonNullRecursively())

                       .....
                   }
```
* 现在我们进入[element_ext.kt](https://android.googlesource.com/platform/frameworks/support/+/master/room/compiler/src/main/kotlin/android/arch/persistence/room/ext/element_ext.kt?autodive=0%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F)看下isNonNull()方法:

```
fun Element.isNonNull() =
        asType().kind.isPrimitive
                || hasAnnotation(android.support.annotation.NonNull::class)
                || hasAnnotation(org.jetbrains.annotations.NotNull::class)
```

* **真相终于找到了！**一共三个条件判断其类型是不是Primitive，是否包含Android中的NonNull注解，是否包含kotlin中的NonNull注解，而[isPrimitive](http://www.docjar.com/html/api/javax/lang/model/type/TypeKind.java.html)是OpenJdk中的方法，也就是判断其类型是不是基础类型：

```java
public boolean isPrimitive() {
             switch(this) {
             case BOOLEAN:
             case BYTE:
             case SHORT:
             case INT:
             case LONG:
             case CHAR:
             case FLOAT:
             case DOUBLE:
                 return true;

             default:
                 return false;
             }
         }
```
> 现在可以回答V2->V3中的三个疑问了

> 1.name 字段加上NOT NULL约束是因为Entity中加上了@NonNull注解

> 2.age 字段加上NOT NULL约束是因为int是基本类型，同时因为非空所以要加上初始值

> 3.weight 字段虽然之前在数据库是INTEGER类型的需要NOT NULL约束，但在变成TEXT类型后Room已经不会自动给他加上NOT NULL约束，因此不需要加上NOT NULL


## 总结
现在我们知道了在Room中所有**基本类型**的字段，和**@NonNull注解**的字段都会加上NOT NULL约束，在数据库升级时我们在写migrate方法时尤其需要注意
