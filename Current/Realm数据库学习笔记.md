### Realm数据库学习笔记

> ​	SQLite 用着实在有些烦，手写语句的安全性也确实不够高，于是从众多数据库开源项目中找了个听说挺靠谱的来学习一波。

#### 依赖配置

```groovy
//Project级gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "io.realm:realm-gradle-plugin:3.5.0"
    }
}

//Module级gradle
apply plugin: 'realm-android'
...
```

由于Realm的文本库中已经包含了ProGuard文件配置，故不需要再对其做额外的防混淆设置了。

#### 基本使用

​	Realm的初始化和很多开源库类似，都是在Application(~~Application类满为患啊~~)中加上一句：

```Java
Realm.init(this);
```

​	接下来的事情就完全是对象操作了，区别于SQLite原来的命令式操作，Realm主要是各种围绕着RealmObject进行操作的函数，一个RealmObject类就是一张表：

```Java
//数据结构类,格式为标准的JavaBean 或者 直接将所有字段设为pulic，通过“.”调用，也允许出现有参构造。
class Person extends RealmObject{
	private String name;
  	private int age;
  	public String getAge() {return age;}
    public void setAge(String age) {this.age = age;}
    public String getName() {return name;}
    public void setName(String name) {this.name = name;}
}
//对数据库的调用
public void handleData(){
  Realm realm = Realm.getDefaultInstance();
  Person person = new Person();//只是一个普通类对象，没有数据关联功能
  //这里也可以直接用 realm.createObject(Person.class)来新建并添加进数据库
  person.setName("Lazxy"); person.setAge(21);
  realm.beginTransaction();//所有修改数据库的数据操作都必须开启事务
  Person managePerson = realm.copyToRealm(person);//插入一条数据到数据库中，获取到其存在数据库中的对象，其与数据库有直接联系，相当于一个被监听对象
  realm.commitTransaction();//提交事务
  RealmResults<Person> results = realm.where(Person.class).contain("name","Lazxy").findAll();//根据条件查询数据，从这里取得的对象也是能直接修改数据库的
  LogUtil.e(results.get(results.size()-1).toString()); //输出Age = 21,Name =Lazxy
  realm.beginTransaction();
  managePerson.setAge("22"); //这里可以直接通过Setter方法修改数据库的数据，但也需要在事务中进行。并且如果直接设置引用为null，并不会删除该数据
  //managePerson.deleteFromRealm() 删除该数据
  realm.commitTransaction();
  LogUtil.e(results.get(results.size()-1).toString());//输出Age = 22,Name =Lazxy
  LogUtil.logJudge(managePerson == p);//输出False
  LogUtil.logJudge(managePerson == results.get(results.size()-1));//输出False      
}
```

#### 一些细节

​	接下来就是根据文档一点点地解释上面用到的内容了（~~请叫我人形翻译机~~）：

**数据类型**

​	首先，Realm表中的数据支持的类型为`boolean`, `byte`, `short`, `int`, `long`, `float`, `double`, `String`, `Date`，其中数字类的类型最终都会被表达为同一种类型`long`，就像SQLite中会把数据类型都写成Integer一样。前面出现的RealmObject或者RealmList<? extends RealmObject>就是这些数据的载体。另外RealmObject也同样接受上述类型的封装类型，当这些封装类型的值没有设定时，存入数据库的值会为`null`。

**列值选择**

​	接上面，当表中列值可以为`null`时，可以通过类似`@NotNull`的注解`@Required`来禁止取值为空。该注解不能对非封装类型使用，否则会发生编译错误。同时原始类型和**RealmList类型默认为非空(当然可以取得其对象并设为null，但这只会清空其内容，使其size为0，但本体对象仍保留在数据库中)**，RealmObject类型默认为可空。另外，当RealmObject的某些字段不需要被存入数据库时，可以用`@Ignore`注解屏蔽，其效果同`transient`关键字。最后`static`变量默认不会被写入数据库。

**索引和主键**

​	可以通过`@Index`注解来声明RealmObject中的某一字段为该表的索引，字段类型必须为`Date` `String`和其他基础数据类型。而通过`@PrimaryKey`设置某个字段为主键时，其隐式地同时设置了`@Index`，此时字段类型必须为`String`及数字类型且**不支持自增**（包括它们的封装类型，这似乎又与索引的设置要求不符了）。主键必须唯一，且因其的存在，我们可以使用copyToRealmOrUpdate()方法，对表内数据进行更新，而不是单纯地将数据插入。但同时，在使用copyToRealm()方法时，必须注意表内是否存在相同主键的值，尤其是默认值0或null。

**自定义Model**

​	除了像上面一样通过继承RealmObject来构造表类，还可以通过实现RealmModel接口来完成目的，不过还要额外在类声明时加上`@RealmClass`注解。这样所有由RealmObject对象调用的函数都能转为RealmObject静态调用，而RealmModel类作为参数。PS:RealmObject也实现了RealmModel接口。

**表间联系**

​	在Realm中，RealmObject的字段类型也可以为另一个RealmObject，或者是一个RealmList。当需要复杂条件查询时，可以将表与表联系起来，如：

```java
//两张关联的表
public class Person extends RealmObject {
  private String id;
  private String name;
  private RealmList<Dog> dogs;
  // getters and setters
}

public class Dog extends RealmObject {
  private String id;
  private String name;
  private String color;
  // getters and setters
}
//以一张表的内容为另一张表的查询条件
public void queryMaster(){
  //...
  //寻找狗叫做"pop"的人
  RealmResult<Person> result = realm.where(Person.class).equalTo("dogs.name","pop").findAll();
}
```

需要注意的是，这种查询必须明确结果对象的类型，否则会出现逻辑错误：

```java
RealmResult<Person> result1 = realm.where(Person.class)
							.equalTo("dogs.name","pop")//找到养的狗叫"pop"的人
							.equalTo("dogs.color","black")//找到养黑色狗的人
							.findAll();//由于查找对象是“人”，而“狗”与“人”是多对一的关系，故上面关于狗的两个条件取并集，而不是交集
RealmResult<Person> result2 = realm.where(Person.class)
							.equalTo("name","lazxy")//找到叫"lazxy"的人
							.equalTo("age","21")//找到21岁的人
							.findAll();//此时应取两个条件的交集
```

另外，通过`@LinkObect`注解，还可以为两个关联类提供反向关联，从而实现双向查找，如：

```java
public class Dog extends RealmObject{
  //...
  @LinkObject
  private final RealmResults<Person> owners; //该字段必须以final为修饰符，且类型为RealmResults
  //...
}
```

**事务与写操作**

​	上面已经提到过，当对数据库中的数据进行写入或删除时，都需要开启一个事务，在事务结束前（包括提交和取消）不改变数据库元数据，以保证安全性。开启事务的方式除了上面的按序调用函数，还有executeTransaction()方法，它会自动进行提交事务操作或者在发生异常时取消事务，使用如下：

```java
realm.executeTransaction(new Realm.Transaction() {
            @Override
            public void execute(Realm realm) {
                //... 数据读写操作
            }
        });
```

另外**Realm中对数据库的写操作是互斥的**，如果同时在主线程和子线程中开启事务进行数据库写入，则会发生ANR错误（由于主线程阻塞），在有这样的需求时，可以调用executeTransactionAsync()方法（比executeTransaction多传入一个回调参数，以在子线程数据库操作结束后回主线程继续操作）。但由于Realm采用_MVCC_构架，故数据在写入时是可以并发进行读操作的。

> ​	MVCC的基本思路是当数据库的读比写要多得多的时候，在写入时先拷贝并只允许拷贝一份原数据（写入互斥），在拷贝数据上进行修改，而直接读原数据，在拷贝数据修改完毕后再替换原数据，从而实现读写的互不干扰。

**查询操作**

​	关于查询操作前面的例子已经很多，这里主要列举出文档里讲的查询涉及到的类以及功能函数：

```
RealmQuery类的函数：
or() -对条件进行或操作
not() -对条件进行非操作
findAll() -实施查找
findAllSorted() -实施查找并排序
findAllAsync() -在子线程内进行查找，只能在有Looper的线程中调用
equalTo() -匹配相等的值
notEqualTo() -匹配不等值
in() -匹配给定数组中的值

//下列方法只能在匹配数字或者Date对象时使用
between() -匹配一个闭区间
greaterThan() -匹配大于
lessThan() -匹配小于
greaterThanOrEqualTo() -匹配大于等于
lessThanOrEqualTo() -匹配小于等于

//下列方法只能在匹配String时使用
contains() -匹配包含给定值的字符串
beginsWith() -匹配给定开始值的字符串
endsWith() -匹配给定结束值的字符串
like() -匹配给定的子字符串，用通配符*表示0个或多个unicode字符，？表示单个unicode字符

//下列方法只能在匹配二进制数组、字符串、RealmList时使用
isEmpty() -匹配空内容
isNotEmpty() -匹配非空内容

//下列方法只能在匹配可空对象时使用
isNull() -匹配空类型
isNotNull() -匹配非空类型

//下列方法只能在匹配数字和字符串时使用
distinct() -匹配到的符合条件的每个值只允许出现一次
```

当匹配条件中有or()或not()时，为了声明它们或\非的范围，可以用beginGroup()和endGroup()来将几个条件划定为一个组，如：

```java
RealmResults<User> r = realm.where(User.class)
                            .greaterThan("age", 10)  // implicit AND
                            .beginGroup()
                                .equalTo("name", "Peter")
                                .or()
                                .contains("name", "Jo")
                            .endGroup()
                            .findAll();
```

可以用sort()方法对RealmResult进行排序，默认为正序，可以在其第二个参数中传入`Sort.DESCENDING`实现逆序排序。

可以用sum()  max() min()  average() 等函数对RealmResult进行计算和筛选。

**如果利用某条件查找出RealmResults后，在后续的过程中改变RealmObject对象该条件的值，则RealmResult也会随之改变。**故当RealmResults需要进行条件修改的迭代时，可以用SnapShot保证RealmResult迭代时顺序保持不变，其可以由RealmObject或者RealmList手动生成，也可以直接通过ForEach迭代自动生成，如下：

```java
RealmResults<Person> guests = realm.where(Person.class).equalTo("invited", false).findAll();

// Use an iterator to invite all guests
realm.beginTransaction();
for (Person guest : guests) {
    guest.setInvited(true);
}
realm.commitTransaction();

// Use a snapshot to invite all guests
realm.beginTransaction();
OrderedRealmCollectionSnapshot<Person> guestsSnapshot = guests.createSnapshot();
for (int i = 0; guestsSnapshot.size(); i++) {
    guestsSnapshot.get(i).setInvited(true);
}
realm.commitTransaction();
```

当查找不到匹配数据时，返回一个size = 0 的RealmResults。

当用异步方法进行Realm的查找时，其会不阻塞当前线程，并在查找结束后将数据回调给RealmResults，这个过程可以通过附加RealmChangeListener来实现（注意一定要在生命周期中把监听去除），或者用isLoaded()验证现在的RealmResult是否已经加载完毕。

上面函数的完整[API](https://realm.io/docs/java/latest/api/io/realm/RealmQuery.html) 。

**Reaml的类别和设置**

​	除了用上面说到的`Realm.getDefaultInstance()`获得Realm对象实例外，还可以自定义Realm对象的各种配置，这就要通过`RealmConfiguration`类来实现了，具体用法和OkHttp等框架的配置形式很接近，都是Builder链式调用，如下：

```java
//一般的Realm构造
RealmConfiguration config = new RealmConfiguration.Builder()
  .name("myrealm.realm") //设定数据库文件名，文件路径在应用私有目录的file文件夹下
  .encryptionKey(getKey()) //用64字节数据进行数据库加密和解密
  .schemaVersion(42) //设置该数据库的版本号，如果当前版本低于这个设定的数字，则必须设置migration进行迁移，否则会报异常
  .modules(Realm.getDefaultModule(),new MySchemaModule()) //用传入Module的schema替换当前Module的schema。第一个参数为基础Module，必填，后面是附加Module，为可变参数，若传入类型无@RealmModule注解，则报异常。
  .migration(new MyMigration()) //控制数据库版本迁移的中间操作
  .build();

//同步Realm构造，详见文档的同步章节
SyncCredentials myCredentials = SyncCredentials.usernamePassword("bob", "greatpassword", true);
SyncUser user = SyncUser.login(myCredentials, serverUrl());
SyncConfiguration config = new SyncConfiguration.Builder(user, realmUrl())
    .schemaVersion(SCHEMA_VERSION)
    .build();
 
// Use the config
Realm realm = Realm.getInstance(config);
```

> 关于上面出现的module，这里需要做一个补充说明。这里的module其实指的是加了@RealmModule注解的类，注解内容为该数据库所包含的类（或者说表）。默认的module是包含所有类的，可以自定义module以决定数据库的数据范围。



当Realm对象的构造也是一个耗时任务时（数据库版本变更数据迁移），可以选用**Realm.getInstanceAsync()**方法进行实际构造，在其回调内得到Realm对象。

当Realm需要等待从远程服务获取完整的数据库再进行下面的动作时（针对同步构造），可以在配置构造链中加入    waitForRemoteInitialData()方法。它会在后台线程自动下载远程数据库，并在完成后通过异步回调（getInstanceAsync内的回调）提供Realm对象。

当Realm内的数据需要设置为只读模式时，可以在配置构造链中加入**readOnly()**，其使用如下：

```java
//设置本地数据库为只读
RealmConfiguration config = new RealmConfiguration.Builder()
    .assetFile("my.realm") //从本地文件中获得数据库
    .readOnly() 
    .modules(new BundledRealmModule())//当设置当前数据库为只读时，
    				//其他的进程或者设备也可能调用同一个数据库，
    				//此时若在一个事务内对该数据库进行写操作（包括版本迁移）则会抛出异常，
    				//故建议设定一个通用的版本模块，避免出错。
    .build();
//设置远端数据库为只读
SyncUser user = getUser();
String url = getUrl();
SyncConfiguration config = new SyncConfiguration.Builder(user, url)
    .waitForRemoteInitialData();
    .readOnly()
    .modules(new BundledRealmModule())
    .build();

RealmAsyncTask task = Realm.getInstanceAsync(config, new Realm.Callback() {
    @Override
    public void onSuccess(Realm realm) {
        // Realm is now downloaded and ready. It is readonly locally but will 
        // still see new changes coming from the server.
    }
});
```

如果不需要将数据库保存在磁盘中，可以选择在构造链中添加**inMemory()**方法来设置其仅在内存中保存。这种数据库名称不能与保存在磁盘中的数据库相同，在内存低时仍会将数据存入磁盘文件，但当引用不存在后会自动将数据都删除。

​	**DynamicRealm**区别于传统的Realm，其为了追求灵活性，不要求数据实体再继承RealmObject类，而是能自定义所需要的字段和表名录入数据库，用类似Intent的getExtra的方式根据变量名和类型获得相应的条目。其所用的配置类与传统Realm都一致，但是会无视版本号等配置，使用如下：

```java
RealmConfiguration realmConfig = new RealmConfiguration.Builder().build();
DynamicRealm realm = DynamicRealm.getInstance(realmConfig);
RealmSchema schema = realm.getSchema();
schema.create("Person").addField("name",String.class).addField("age",int.class);//定义结构名和表名，不允许重复定义，否则会抛出异常
DynamicRealmObject person = realm.createObject("Person"); 
String name = person.getString("name");
int age = person.getInt("age");
person.getString("I don't exist");//对于不存在的字段名，会直接抛出异常 
```

Realm对象与SQLiteDatabase对象一样，都需要在不用时及时释放，且其实行引用计数，通过getInstance构造了多少个Realm对象，就需要close多少次。

**线程相关**

​	关于Realm数据库的线程调度基本用上sync和async的相关方法就行，数据的更新也都是自动的。但注意：**不能将Realm、RealmObject、RealmResult等受监听的类在线程间传递。**当在多个线程都需要这些类的实例时，可以直接构造新的对象，它们的内容都一致。

**JSON直接转换**

​	可以通过createObjectFromJson()或者createAllFromJson()直接将Json字符串或者文件流直接转化为对象，忽略RealmObject类不包含的字段，并当Json的内容不满足RealmObject的非空要求时，则抛出异常。

**数据状态监听**

​	Realm中监听数据变化状态 的接口大致有三种，分别用于监听整个数据库的变化、数据集合的变化和单个RealmObject的变化，它们都需要在有**Looper**的线程中（不包括IntentService）运行基本如下：

```java
//监听整个数据库的变化
realmListener = new RealmChangeListener() {
        @Override
        public void onChange(Realm realm) {
            // ... do something with the updates (UI, etc.) ...
        }};
      realm.addChangeListener(realmListener);
    }
//监听某个集合的变化
private final OrderedRealmCollectionChangeListener<RealmResults<Person>> changeListener = new OrderedRealmCollectionChangeListener<>() {
    @Override
    public void onChange(RealmResults<Person> collection, OrderedCollectionChangeSet changeSet) {
        // `null`  means the async query returns the first time.
        if (changeSet == null) {
            notifyDataSetChanged();
            return;
        }
        // For deletions, the adapter has to be notified in reverse order.
        OrderedCollectionChangeSet.Range[] deletions = changeSet.getDeletionRanges();
        for (int i = deletions.length - 1; i >= 0; i--) {
            OrderedCollectionChangeSet.Range range = deletions[i];
            notifyItemRangeRemoved(range.startIndex, range.length);
        }

        OrderedCollectionChangeSet.Range[] insertions = changeSet.getInsertionRanges();
        for (OrderedCollectionChangeSet.Range range : insertions) {
            notifyItemRangeInserted(range.startIndex, range.length);
        }

        OrderedCollectionChangeSet.Range[] modifications = changeSet.getChangeRanges();
        for (OrderedCollectionChangeSet.Range range : modifications) {
            notifyItemRangeChanged(range.startIndex, range.length);
        }
    }
};
//监听某个RealmObject的变化
private final RealmObjectChangeListener<Dog> listener = new RealmObjectChangeListener<Dog>() {
    @Override
    public void onChange(Dog dog, ObjectChangeSet changeSet) {
        if (changeSet.isDeleted()) {
            Log.i(TAG, "The dog was deleted");
            return;
        }

        for (String fieldName : changeSet.getChangedFields()) {
            Log.i(TAG, "Field " + fieldName + " was changed.");
        }
    }
};
```

**数据迁移**

​	这里又回到了Realm配置时讲到的migration，它的作用相当于SQLite的版本更新函数，可以根据不同的版本号进行字段与表的修改，如下：

```java
RealmMigration migration = new RealmMigration() {
  @Override
  public void migrate(DynamicRealm realm, long oldVersion, long newVersion) {
     RealmSchema schema = realm.getSchema();
     if (oldVersion == 0) {
        schema.create("Person")
            .addField("name", String.class)
            .addField("age", int.class);
        oldVersion++;
     }
     
     if (oldVersion == 1) {
        schema.get("Person")
            .addField("id", long.class, FieldAttribute.PRIMARY_KEY)
            .addRealmObjectField("favoriteDog", schema.get("Dog"))
            .addRealmListField("dogs", schema.get("Dog"));
        oldVersion++;
     }
  }
}
```

若想直接抛弃前面的所有数据结构，直接创建新的.realm文件，则可以在配置链中加入`deleteRealmIfMigrationNeeded()`直接删除前面保存的.realm文件。

**与其他开源库的使用**

​	RealmObject的**序列化**过程不能由**GSON**的默认方式来完成，因为GSON在序列化对象时会直接调用它们的值而不是Getter和Setter方法。此外，RealmObject也不支持JSON解析时直接返回原始类型的数组，并且在RealmObject类中存在非数据库字段而GSON尝试去解析时，需要加上：

```java
public boolean shouldSkipField(FieldAttributes f) {
  return f.getDeclaringClass().equals(RealmObject.class) || f.getDeclaringClass().equals(Drawable.class); //字段的类型名
}
```

​	在与**Kotlin**一同使用时，需要注意的是model类的可见性必须为open，并有时需要@RealmClass注解。

​	在与**Retrofit**一同使用时，需要手动将数据加入Realm，而不会自动完成。

___

文档地址：[Realm文档](https://realm.io/docs/java/latest/)

> 前面提到的关于同步和远端服务器，详见文档的Sync部分。这里为什么不写了呢？因为我懒（其实是翻译得太痛苦，弃坑了）。

