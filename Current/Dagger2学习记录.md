### Dagger2学习记录

​	Dagger2框架目的是使**依赖注入过程解耦**，即构造对象的过程与对象调用分离，**调用方只传入必须的构造依赖，而不关心其具体的构造过程**，这样在其构造的参数列表改变且该改变与当前调用类无关时，可以不改变其被调用处的代码，而只需要修改其本身的构造方法。

#### 依赖准备

```
classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'

apply plugin: 'com.neenbedankt.android-apt'//应用apt插件，在AndroidStudio2.3之后 apt就不再与Gradle兼容了，所以原来用到apt的地方要改为annotationProcessor

compile 'com.google.dagger:dagger:2.9'

apt 'com.google.dagger:dagger-compiler:2.9'

//java注解

compile 'org.glassfish:javax.annotation:10.0-b28'
```

#### 对象划分 

​	需要被注入的类与对象（即被构造和持有的对象） **Target(在官方文档中被称为binding)**：_@inject_ 注解，同时注解在 需要被注入的对象 和 构造该对象的构造函数 上。
​	提供构造函数所需参数的类 **Module** ：_@Module_ 注解，标明其为参数提供类。
​	提供构造函数所需参数的具体方法 **Provider** ：_@Provides_ 注解，函数名一般为provideXXX() ，简单地返回Module获得的参数，一般需要几个参数就有几个这样的方法。
​	Module 与 target的接口 **Component**：*@Component(modules = XXXModule.class)* 注解，有一个inject(Object object)函数，参数为被注入对象的持有者。

#### 调用方式

```Java
DaggerXXXComponent.builder().xXXModule(new XXXModule(args)).build().inject(this);
//其中DaggerXXXComponent为根据注释和接口名动态生成的类； args为构造函数所需参数； this为当前持有注入对象的类对象。
//当Component有依赖其他类型的Module时，其会多一个xXXModule方法，同样用于传入所需的依赖，为链式调用中的一环。
```

PS:如果DaggerXXXComponent类没有及时生成，就Ctrl+F9编译一下。

#### 内部实现

apt通过注解，动态生成了几个功能类，分别为：

```
//以下两个类都实现了Factory接口，并且分别以依赖类与被注入类为泛型，Factory接口是Provider的子类
XXXModule_ProvideYYYFactory :在该类的构造函数中获得@Module注解类中获得的构造参数，通过get()方法获取provideYYY返回的参数对象。

XXX_Factory: XXX为被注入的类名（即构造函数被@Inject注解的类），在构造函数中获得了上面那个Factory类的的对象，在get()方法中通过Factory类获取了相应的构造参数并返回实例化注入类。

//如果Component类是一个内部接口时，该生成类的名称将加上其外部类名称为前缀，并且加上下划线
DaggerXXXComponent :该类即被@Component注解类的子类，其包含一个内部类Builder，Builder中存在两个函数，分别是xXXModule(Module module)与build(）方法。前者用于注入为成员变量注入Builder.XXXModule对象，后者以Builder为参数构造了一个DaggerXXXComponent对象，而其构造函数中，依次创建了上面两个Factory类，至此被注入类的实例化过程已经完成。

XXX_MembersInjector ：XXX为注入对象持有者类名，该类持有被注入类的工厂类，在inject(Object object)被调用时，会转到该类中执行injectMembers函数调用工厂类的get()函数将实例化完成的被注入对象传给持有对象，整个注入过程结束。
```

#### 基础小结

​	所以整个过程是这样的：一、用@Inject注释需要被注入的对象和其构造方法；二、创造@Module注释的参数提供类，其构造中是需要传入的依赖对象（对于没有参数的构造可以不要Module部分，但是这样就失去了其注入的意义）；三、创造@Component注释接口，注释参数中注明用到的@Module类的组件类（当没有Module时，则不需要参数），提供inject方法，用于传入@Inject成员持有对象；四、最后通过生成的辅助类进行Module的构造（实际就是为了注入依赖）与@Inject对象的实际构造。

>Module的@Provides方法中如果也有参数，则必须在其绑定的Component绑定的任意一个Module中有一个相应返回类型的@Provides方法，即在一个Component关系网中，Module的参数是可以互相传递的。



#### 进阶用法

​	_@Singleton_注解的使用：当需要在一个类中对多个同类对象进行注入时，只需要调用一次inject方法，其默认构造时每次构造一个新的对象。如果需要让其构造的对象为单例，即**多个引用指向同一对象时** ，则可以在注入类和XXXComponent类处加上@Singleton注解（和@Component注解在一起，如果只加在注入类上的话会有编译错误），即可注入单例对象，需要注意的是全局使用这种单例方式时**Component必须为同一个对象**。关于这个注解还有一种用法是：对@Provides注解的方法进行注解（对应地也必须对其相关的Component进行注解），这样**当该方法返回过一次构造参数后，后面每一次返回的都会是第一次返回参数的缓存**，两个对象完全相同。

​	_@Scope_注解的使用：这个注解其实可以视作@Singleton的父接口，或者说@Singleton是其注解下的注解，其形式大概如下：

```
@Scope
@Retention(AnnotationRetention.RUNTIME)
public interface ActivityScope{}
```

在官方说明中，该注解的作用是划定注入的具体范围，结合Singleton的作用来讲，其实就是实现**局部单例**。dagger规定，**Component与各组件必须处于一个Scope中**（这也是为什么上面提到Component必须与Module和Target共同设置@Singleton），而当该Component对象唯一时，其注入的变量对象也就是唯一的，**所以Component对象的持有范围决定了单例的有效范围，且Component中inject()方法的类型决定了单例能被哪些类型中引用。**因此@Scope注解其实并没有真正地限定单例的作用范围和生命周期，它只干了两件事：**以对象缓存的形式实现单例和用注解名提醒开发者该单例的范围和生命周期。**

​	_@Reusable_注解的使用：这个注解的作用是将注解的构造参数或者类对象缓存起来再利用，而不是每次构造新的对象。乍一看它的作用和@Singleton一样，但是它们的使用场景不同——由于一个Component无法依赖一个scope为@Singleton的类，故若需要保持被依赖的Component构造出的Target也能只有一个实例的话，就可以在其Target上注解@Reusable。此时也**要求所有共享实例的子Component依赖同一个父Component**，否则它们仍然会构造出不同的实例。当然它也可以在直接使用时取得单例的效果，不过此时只能在Target类注解，而不需要在每一个组件都注解一次，这个注解也不起范围标定作用。

​	_@Component_间的依赖：一个Component可以依赖另一个Component，并能够调用到父Component暴露出来的取参方法（这里的父子关系和它们对应的注入对象的父子关系正好是相反的）。依赖方式主要有两种，如下：

```Java
//直接用@Component声明依赖
@Component(dependencies = AppComponent.class,modules = MainModule.class)
interface MainComponent{
  //...
}
@Component(modules = AppModule.class)
interface AppComponent{
	Context getContext(); //暴露给子组件的方法，只有这些方法可以被子组件调用
  	//...
}
//采用@SubComponent声明依赖
//子组件声明与原来基本一致，除了Component换成了SubComponent
@SubComponent(modules = MainModule.class)
interface MainComponent{
  //...
}
@Component(modules = AppModule.class)
interface AppComponent{
  //...
  MainComponent getMianComponent(...); //父组件要声明获取子Component的方法，来表示可以被继承
}
```

这两者的区别是前者只能如上所说调用父Component暴露出来的取参方法，而后者能够调用全部父Component的可用参数。**但注意两个Component不能用同一个@Scope类标识。**

>看到这里才意识到一个问题，dagger取构造参数的方法(provideXXX)以及子Component取依赖的参数的方法(可能是getXXX)都**不是通过参数名来明确调用时刻的，真正让其被调用的判断依据是返回值类型！**所以当某个被注入类的构造函数参数列表有同类型参数时，dagger就无法判断应该调用哪一个取参函数了，这个问题大概还得好好看下文档（似乎可以通过自定义注解解决）

​	*引用与内存回收*：当Target使用了@Scope类注解时，在同一注解范围内的Component会持有Target的引用（用于实现单例），故可能出现内存泄露的情况。为防止这种情况发生，可以在定义注解时加入*@CanReleaseRefrence*注解，并手动地在需要内存回收时调用相关方法使它们之间的引用变成弱引用，在内存充足时切换回强引用，其方法如下：

```Java
//注解声明
@Scope
@Retention(RUNTIME)
@CanReleaseRefrence
public @interface ActivityScope {}

//调用注解对象
@Inject@ForReleasableReferences(ActivityScope.class)
ReleasableReferences mScopeReferences;

private void release(){
  mScopeReferences.releaseStrongRefrences();//将所有引用转化为弱引用
}

private void refer(){
  mScopeReferences.restoreStrongRefrences();//将所有引用转化为强引用
}

```

​	_延迟初始化_：当需要进行延迟注入时，可以用Lazy<T>类型变量代替需要被初始化的变量进行注入，<T>为原变量类型，在需要进行注入时再通过get()方法即可获得原变量对象，其使用如下：

```java
@Inject
Lazy<Target> lazyTarget;
...
public void lazyInit(){
  Target target = lazyTarget.get();
}
```

当Target为单例时，每次get()获得的对象也都会是同一个。

​	_Provider<T>_：当需要多个注入对象时，可以用Provider<T>来代替<T>作为注入对象，每次通过Provider对象的get()方法就可以获得一个新的相同对象，相当于调用一次clone()。

​	_@Qualifier_：当被注入对象需要选择不同的实际构造值时，可以采用该注解下的自定义注解对其进行区分，具体使用如下：

```java
//假设自定义注解叫Named
@Qualifier
@Retention(RUNTIME)
public @interface Named {
  String value() default "";
}
//在Module中
@Provides
@Named("small")
int provideSmallSize(){return 14}
@Provides
@Named("large")
int provideLargeSize(){return 20}
//在实际注解情景下
@Inject
@Named("small")
Size smallSize;
@Inject
@Named("large")
Size largeSize;
```

依赖Component不允许有多重qualifier注解(文档所述，不解其意)。

​	_@BindsInstance_：该注解用于Component.Builder类中，可以将构造参数动态地赋给构造对象，相当于Module的一个外置@Provides方法，该方法必须被调用，并传入一个非空的值。当其参数被设为@Nullable时，在Target的构造中也一样要设为Nullable，此时可以在创建Component时忽略调用该值，自动认为其为null。

```java
//在Component中声明动态传参的方法
@Component.Builder
  interface Builder {
    @BindsInstance Builder userName(@UserName String userName); //并不是很明白@UserName这个注解在这里的作用
    AppComponent build();
  }
}
//在注入情景使用
DaggerAppComponent component = DaggerAppComponent.builder().userName(args[0]).build();
```

​	