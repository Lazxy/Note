---
title: Flutter学习笔记
date: 2020-03-12 17:34:02
tags: 笔记
---

> 当时为了赶时髦，同时也闲得无聊，于是学了一阵Flutter，最后想要实践个项目，然而无疾而终。画页面真的好难！！

#### 1. Widget

类React的响应式布局，通过对比算法确定Widget组件变化时的最小更改。

Widget的构建顺序是从上到下的，上层Widget决定控件在布局中的位置，底层Widget决定控件的具体形状（RenderObject）。

Widget的手势设置和设置布局属性类似，需要在外层裹上一层GestureDetector

StatefulWidget与StatelessWidget的继承用于声明一个控件状态可变/不可变，前者的实际控件声明都写在其State实现类中。

Widget的嵌套写法中，子Widget可以作为参数传入父Widget的构造方法， 且其命名与父Widget中一致，这样其位置也是固定的。

<!--more-->

---

关于无状态Widget的状态监听，状态的变更交由有状态的父Widget执行，此时下层控件状态可由上层控件掌握，并由上层发起对下层控件的重绘（通过上层的setState方法将子控件标记为脏视图）

StatefulWidget在首次加入到视图树中时，会通过createState创建一个状态实例，这个状态在Widget（采用相同的命名）重绘时，会进行重用。

>关于这种状态管理方式的好处，类似于DataBinding的状态绑定：只需要设置不同状态下视图的属性，就可以简单地完成视图的更新，并且得益于比对算法， 视图不需要完全重绘
>
>将StatefulWidget控件的实际build方法放在State中的用意是(来自State的build方法注释)
>
>1. 如果StatefulWidget中包含了带State的构造方法，这样其子类也不可避免地会持有其State的引用，可能影响父类的状态。
>2. 同理如果StatefulWidget的子类为StatelessWidget，其子类就不能不带状态，这违背设计的初衷。
>3. 再者如果在StatefulWidget中直接定义状态，则在buildState方法中使用闭包时获取现有属性就会是以`this.XX`的形式，而Dart的闭包内的引用是不可变的实体——于是在Widget重建时，如果不同时重建State，就会出现State依然持有被重建前的Widget内容，而不会因为重建得到更新，这影响了State恒定而Widget可变的设计初衷。

---

GlobalKey可以用来在整个视图树中表示唯一的控件。

#### 2.Diffrence

Flutter的每一帧都要重绘一次视图树，且Widget不可变

> 这里的不可变指的是，一方面，Widget对象除了初始化之外没有设置修改属性的方法，完全依赖StatefulWidget的状态对象存储，每次更新属性时重建一个对象；另一方面Widget在树中的位置不变

Flutter中的动画设置也是通过**AnimationController（控制动画绘制每一帧的属性值）及Interpolator（插值器，但是也继承自Animation）**来实现的。除非自己实现TickerProvider的接口，否则**含有动画的控件必须是StatefulWidget**（Ticker是触发每帧动画回调的对象）。

可以通过**CustomPainter和CustomPaint**来自定义绘制画布，和View重写onDraw方法类似的，重写paint方法，获得Canvas对象来进行绘制操作。

Flutter中的自定义控件修改一般是通过控件组合实现的，而非继承覆盖父类方法

> _这样做的好处在哪里？_

Flutter的页面定向方式与Native大为不同，其倾向于在main方法中通过**routes**参数声明所有可能跳转的页面，包括其**路径名**和页面对象构造方式，然后通过**Navigator**（类似页面栈的Widget）来切换各个页面（以路径名为路由方式），并且可以通过Navigator方法的返回值来获得页面跳转返回的数据（类似startActivityForResult）。或者也可以使用native集成方式调用Intent。

> _route方式切换页面，如何在页面间传递数据呢？_

Flutter可以通过MethodChannel来定义、调用原生应用与Flutter约定的接口，以此来进行数据传递。

Dart是一个单线程模型，_类似Android的Message事件循环模式，优先执行"主线程"的操作，然后依次将后续异步事件（**async**关键字标注）插入执行队列，以此实现伪异步执行的效果_。另外还有Isolate机制，用于处理计算量大的并发任务，类似Java的多线程，但是其任务间并不共享内存，仅通过约定调用方与被调用方的Port形式来进行通信。

>官方示例中**Port**的约定方式是这样的：
>
>1. 主线程开启一个接收端口`receivePort1`（**ReceivePort**），通过**Isolate.spawn([methodName]，sendPort)**方法开启一个Isolate任务，并传递`receivePort1.sendPort`给任务线程。在这之后可以通过**await**关键字等待任务线程响应
>2. Isolate任务线程 也开启一个接收端口`iReceivePort`，并且把这个接收端口通过主线程传入的`receivePort1`调用send方法发给主线程，然后await等待接收任务的具体参数
>3. 主线程通过`receivePort1`的`first`参数获得了任务线程回调的接收端口`iReceivePort`，此时再通过`iReceivePort.sendPort`端口send任务参数以及主线程接收端口`receivePort2.sendPort`给任务线程
>4. 任务线程得到了参数及主线程回调端口，经过耗时任务操作之后，将结果通过`receivePort2.first`对象传递给主线程。
>
>这个例子可能略为有些复杂，实际上，如果Isolate是一个简单的固定任务的话，只需要主线程把sendPort对象传给任务线程，任务线程处理完任务后通过接收的sendPort对象将结果返回给主线程即可。
>
>但由这个例子可以看出，主线程和Isolate线程间的信息交互是通过ReceivePort实现的，并辅以await关键字的使用，就可以实现异步执行任务，同步等待结果（Future）的调度效果。

Flutter中图片的分辨率格式与iOS类似，采用1x、2x与3x的分辨率，置于images中，但不足的是需要在**pubspec.yaml**中进行声明才能引用（第三方库的依赖也在这个文件里）；而对于strings，Flutter中只能置于dart文件中，以静态对象的方式来引用

> 另外，对于strings文本的国际化，Flutter并没有语言资源文件分别放置这样的操作，如果要实现这个需求，就需要依赖flutter_localizations包，在APP创建时为它进行初始化配置，并构造相应的Localization文件来列举不同语言的文本，以键值对的方式存放，再由Location.of方式获取对应值。后半部分也可以通过采用**Flutter i18n**插件通过书写.arb文件的形式来进行简化，但调用时仍然具有侵入性（部分不在MaterialApp下的文本甚至并不能通过这种方法来进行国际化设置）。

Flutter中可通过WidgetsBinding观察监听didChangedAppLifecycleState来监听生命周期，通用事件主要有两种：

- resumed -对应onResume 
- paused - 对应onPause

而在onCreate中执行的动作一般会在initState中执行，onDestroy则对应dispose方法

ListView在Flutter中同时承担了ScrollView和ListView的功能，但这里的ListView没有 Adapter来进行子列构造以及数据更新提示，需要在每次更新数据时变更数据源List的引用或者采用ListView.builder来启动布局的更新。

> 本质上上述的两种方式应该都是为了Flutter对比新旧ListView时能得到一个不同的结果（不同的数据源引用对象）。

Text Widget的各种样式来自于TextStyle对象，TextField的hint文本则需要在布局下嵌套一个InputDecoration

Flutter的主题声明是在顶层widget中实现的。

Flutter的各种平台/硬件相关功能都通过**插件**来实现，如相机、传感器使用、持久化存储、调用数据库与实现Notification等等，其实现方式类似于Flutter发送相关请求与Platform层交互，然后Platform层完成请求任务后对Flutter进行通知。

- [Flutter Widgets List](https://flutterchina.club/widgets)  
- [Plugins](pub.dev)



#### 3. No man‘s sky



---

### Dart

Dart的所有变量都是对象实例，包括方法和null（和kotlin一样），允许显式的声明对象类型或者用`var`声明一个动态（dynamic）类型对象。

Dart 的运行入口是无参的main()方法。

Dart没有可见性描述符，变量名以`_`开头时，即表示它是一个私有变量。

所有变量在初始化之前都为**null**，包括数字。

Dart中同时存在**final和const**，final变量只能被赋值一次，而const表示一个常量，不可被改变（不能是实例变量，在类中定义时必须是静态的）。且const也可以用来修饰一个构造方法，这样构造出来的对象也是不可变的（但和C++一样，其引用可以变化，如果引用没有用const或final修饰的话）。

数字类型在Dart中仅有int和double两种，其中int型的范围没有限制，且两者都继承num类。

Dart中的String与Kotlin类似，也支持单引号、双引号、三引号（多行字符串），前两者不支持书写时换行，三者都支持转义字符和参数引用（用`$`符号引用）。且由于多行字符串也支持转义字符(Kotlin并不支持)，Dart的“生”字符串（raw string）使用小写字母r开头来表示的，如：

```dart
var rawStr = r"In a raw string, even \n isn't special.";
```

这里的raw string既不支持转义字符，也不支持变量引用。

字符串间的比较可以直接用`==`，连接时则可以选择用`+`，或者直接省略，仅把字符串放在一起来表达连接的字符串（这种情况下这些字符串不能是变量）。

Dart中的布尔类型类型标志为`bool`，这个类型可以被其他类型转换而来（类似C++的int转boolean，不过只在检查模式下生效），但除了其值等于true的情况，其他情况都为false。

Dart中的数组即`List`对象，其定义时类似这样：

```
var strList = ["a","b"]; //这个经过初始化的数组也是可增长的
or
var strList = new List();
```

总体来说这个类与Java中的List还是非常像的，方法基本上都可以共用。

`Map`对象也得到了一定的简化，写法类似JS：

```dart
var tab = {
    "manager":"Lazxy",
    "staff":"Lzxin"
}; //初始化
var tab = new Map();//初始化
tab[manager] = "Gready";//赋值与取值
```

List和Map初始化时都可以指定泛型，写法类似:

```
var tab = <num,String> {1:"Lazxy",2:"Lzxin"} //之类查找表key为num类型，value为String类型 
```

Dart中的方法也同时是对象，书写方式和Java没有很大差别，但可以不写返回值类型声明和参数类型声明，并且当方法**表达式只有一行**时，可以采用简写方法，如：

```dart
isValid（int num）=> num > 0; //这里省略了 bool这个返回值类型，且没有return 
```

同时，方法声明中允许**命名参数**，并且将其设为可选项，如：

```dart
//声明一个命名参数方法
bool checkFace(String eye,{String mouse,String ear}){ //部分可命名
    //...
};
//调用一个命名参数方法
checkFace(eye:"shining", mouse:"pretty");//调用这样的方法时，所有参数都需要指定名称，即使填入的参数已经与整个参数列表一一对应
```

方法声明中也允许通过定义**可选位置参数**来进行参数缺省，如：

```dart
//声明一个可选位置参数方法
bool checkFace(String eye,String mouse,[String wings]){
    //...
}

//调用一个可选参数方法
checkFace("shining","pretty");
```

这里可选参数的范围也可以拓展到多个参数，实际就是扩大`[]`的作用范围，但是必须是后几个参数。在多个参数为可选位置参数时，调用方法时传入的参数将按声明顺序对应到各个参数。

与Kotlin类似的，Dart的方法还能声明默认参数，但只有以上两种可选参数允许设置默认参数，调用时若参数缺省，则使用默认值。

```dart
//声明一个带默认值的可选参数方法
bool checkFace({String eye,String mouse,String ear = "cute"}){
    //...
};
```

Dart的程序入口是**main**方法，其参数可以为空也可以是一个List\<String\>。

Dart的方法可以作为一个变量被引用，并允许匿名定义，也就是熟悉的闭包形式，不过和常见的lambda表达式不同，有点像是去了名字的方法定义：

```dart
var funcObj = (bool someCondition){ //这里的参数类型也可以不写
    if(someCondition){
        //...
        return "success";
    }
}；
//or 
var funcObj = (someCondition) => someCondition?"success":"fail";
```

此时这个变量类型为`Function`或者更详细的`String Function(boolean)`。且匿名方法作为方法，也支持上述缺省参数的声明方式。

允许在一个方法内，以正常的定义方式定义另一个方法。如：

```dart
void oneFunction(){
    //...
    boolean judge(Condition con){
        //...
    }//这个时候这里不需要写分号
}
```



Dart的匿名方法内传参是对象传参，其不会因为外部引用的更改而更改，故也不像Java一样需要把外部参数设为常量参数，可以等同于匿名方法留存了一套入参的对象备份。

方法对象间的比较是引用比较，当两个方法引用指向的实例对象地址不同时，称这两个方法不相等。

> Dart中的方法是不存在重载的，因为每个方法的命名都需要不同，且方法的参数列表实际上是可以不设定具体类型的，也就是说，只要方法内能够处理不同类型的参数，重载这个行为实际上是在方法内实现的，而与声明关系不大。

Dart的操作符和Java大体类似，但有一些区别，比如当一个运算符是中缀操作符时，其采用的重载实现是其左边类型的实现，比如下面`+`运算符的例子：

```dart
var result = "No." + 1; //正确 采用了String的+符号重写，结果是No.1
var result = 1 + "No."; //错误 采用了num的+符号重写，不允许第二个参数为String
```

还有诸如类型判断与类型转换，Dart的语法更像Kotlin，前者使用了`is`和`is!`关键字，而后者则使用了`as`，且启用了智能类型推断，用法如：

```dart
if(coder is Man){
    coder.drink()；//这里自动将coder类型推断为了Man
}
(coder as Man).drink(); //这里主动将coder类型转化为了Man
```

在条件表达式中，Dart除了和Java一致的三元表达式之外，多了一个判空选择操作符`??`，用来简化三元表达式第一个参数为null的情况下取第二个参数的操作，即：

```dart
var a = b != null ? b : c;
//eqauls
var a = b ?? c;
```

类似的还有一个判空赋值操作符`??=`，当左边的对象为空时，才会将右边的对象赋值给左边。

除此之外，Dart还有两个非常好用的操作符，一个是类似Kotlin防空操作的`?.`操作符，当一个空对象使用这个操作符引用成员变量或者方法时，会默认返回一个空值；另一个是级联操作符`..`，这个操作符可以在语言层面实现Builder设计模式的链式调用

> 文档中说明只能用于成员变量的链式赋值，而不能用于返回void的方法调用，但实际尝试是可以的，留待观察

控制流与循环方面，Dart和Java的关键字几乎没什么区别，仅在两个地方有不同：一个是for循环语句，forEach的调用方式使用了Kotlin中使用的`in`关键字，并且可以简写为forEach方法的调用，如：

```dart
for(var str in strList){
    print(str);
}
//or
strList.forEach((str) => print(str));
```

另一个是swith语句，Dart中的swith语句不再是不终止匹配的情况就自然向下的逻辑，而是强制限定了每条非空匹配必须通过break/return/throw终止当前匹配行为，或者采用continue选择向一个已命名的匹配分支跳转，如：

```dart
var command = 'CLOSED';
switch (command) {
  case 'OPEND':
  	executeOpened();
    break; //这里不写break会抛出编译时异常
  case 'CLOSED':
    executeClosed();
    continue nowClosed;
    // Continues executing at the nowClosed label.

nowClosed:
  case 'NOW_CLOSED':
    // Runs for both CLOSED and NOW_CLOSED.
    executeNowClosed();
    break;
}
```

和Kotlin一样，Dart中的异常也都是非检察异常，不会在编译时要求程序一定捕获，且`throw`这个关键字可以用以抛出所有非空对象为异常，并不要求其是Exception对象或者Error对象。`rethrow`关键字允许在catch到异常之后，再次将原异常抛出（虽然感觉并没有什么用）。

Dart中，类的声明和Java区别不算很大，主要的差别在构造方法上。

在默认构造方法上，两者没有区别，同样是一个空参数构造方法，如果父类没有默认构造，则子类必须调用父类的非默认构造方法，调用法法与Kotlin类似，使用了`:`符号：

```dart
class Parent{
	String name;
    num age;
    
    //类似Kotlin中的data类型的构造,等价于将name和age赋值给成员变量，语法糖
    //但需要注意的是Dart只允许类有一个非命名构造方法，且不允许重载
    Parent(name,age);
}

class Child extends Parent{
    Child(name) : super(name , 13);
}

var father = Parent("Lazxy",23);
var child = Child("Lzxin");
```

类似调用父类方法的方式，子类调用自己的非命名构造方法时也需要用`:`符号，称为重定向构造，如：

```dart
class Person{
	String name;
	num age;

    Person(name,age);
    //重定向构造，这种构造方法没有方法体，且被重定向的对象必须是一个普通无名构造
    Person.baby(name) : this(name,0);
}
```

在此基础上，构造方法可以提供自己的初始化列表，声明在构造方法体之前，同时也执行在方法体之前，甚至早于调用父类的构造，在此基础上，可以进行一些条件设置，如：

```dart
class Parent{
	String name;
    num age;
    
    Parent(name,age){
        this.name = name;
        if(age != 0){
            //这里设置age不为0时才设置其值
        	this.age = age;
        }
    }
}

class Child extends Parent{
	num age; //如果不在这里声明一次，就会提示无法在这个范围内访问该参数
    //这里预先初始化age = 12,再执行父类构造方法， 由于此时age已不为0，故age的值不会更改
    Child(name) : age = 12,super(name , 13); //与父类构造声明用“,”连接
}
```

Dart也可以声明**命名构造方法**，即通过类似静态方法的声明调用方式构造类对象的构造方式，例如：

```dart
class Person{
    String name;
    num age;
    
    //命名构造方法
    Person.fromRecord(Map record){
        name = record["name"];
        age = num.parse(record["age"]);
    }
}

//构造调用
var person = new Person.fromRecord({"name":"Lazxy","age":"23"}); //这里的new当然也可以省略，就像调用一个静态方法
```

还有一种特别的构造方法，成为**常量构造函数**，这个构造方法会成为编译时常量，并且要求其成员变量都是final的，整个类都不可变（虽然并不知道这个特性有什么用）：

```dart
class ImmutablePoint {
  final num x;
  final num y;
  const ImmutablePoint(this.x, this.y);
  static final ImmutablePoint origin = const ImmutablePoint(0, 0); //这里的const也不是必须的
}
```

最后，在许多SDK预置的类中还可以看到的一种构造方法，**工厂方法构造函数**，这个构造并不一定会构造一个新的对象，而是采取自定义操作来获取一个对象（诸如从缓存中获取实例或者构造一个子对象实现）的动作，如：

```dart
class Parent{
    //无名工厂方法构造
    factory Parent(){
        return Child();
    }
    
    //命名工厂方法构造
    factory Parent.withName(String name){
        return Parent(name);
    }
}
```

在工厂方法构造中，不允许访问`this`实例，此时命名工厂方法就更像一个静态的getInstance方法了。

Dart中的成员变量自带getter和setter方法，且支持自定义（和Kotlin类似）。

Dart中的抽象类定义与Java中相同，使用`abstract`修饰符，抽象方法的声明则简单的不写方法体就可以了，不需要修饰符修饰。但不同的是普通类也是可以定义抽象方法的，所以不实现这个方法调用的结果就是运行时异常（实在没搞懂为什么要做这么不安全的设计）。

Dart中的**每个类都隐式地定义了包含所有实例成员的接口**，其他类可以通过`implement`继承实现，并需要重写所有接口方法与继承成员变量。

Dart中一个特别的继承机制，**mixin**，其作用相当于允许进行多继承，用`with`关键字声明。继承类可以使用mixin类的成员变量和方法，并允许重写这些实例成员。但mixin类本身不能有构造方法，在早期版本里也不能继承其他类与调用super方法。

> 这里仍然需要讨论下多继承情况下的方法实现冲突，经过实验其先后顺序应该是这样的：一个类必须先extends 再 with最后implements，如果这三者的实现对象都有同一个方法，子类不需要任何实现，该方法的实现由with的对象实现，如果多个with对象都有一个方法，则选择最后一个with的对象；如果没有with对象，即使extends的父类已有这个方法的实现，接口也会要求再次重写这个方法。

>Dart里的继承方式更具多样性，其定制程度层层递进——**mixin实现了完全的代码复用，子类可以什么都不实现；普通的extends需要实现父类的构造方法进行变量初始化；abstract规定了抽象方法的APi，并可以提供部分实现；implements接口提供了实例成员的API，需要开发者完全地进行实现。**

Dart库（Library）的引用同样使用`import`关键字，内置的库与自定义的库引用方式有所不同：

```dart
import 'dart:io'; //引用内置库
import 'package:mylib/mylib.dart'; //引用自定义的库
```

在两个库有冲突类型时，可以通过`as`指定库的别名来进行解决，类似Kotlin：

```dart
import 'package:lib/lib.dart' as lib;
```

同时Dart支持部分导入，这个“部分”单位是class，用 `hide`和`show`关键字实现：

```dart
// 只导入foo.class
import 'package:lib1/lib1.dart' show foo;

// 除了foo.class都导入
import 'package:lib2/lib2.dart' hide foo;
```

当需要延时导入相关的库以节省开销时，可以采用`defferred as`关键字来实现，同时在合适的时候`loadLibrary`，写法如下：

```dart
import 'package:deferred/hello.dart' deferred as hello; //延时加载一个库
greet() async {
  await hello.loadLibrary(); //通过库的别名启动loadLibrary方法
  hello.printGreeting();//调用库中的方法，在导入成功前是不可用的
}
```

Dart的异步支持算是和Java的一大区分点，其没有了Java中显式的线程概念，而多采用Future与Steam的对象表示一个延迟返回/即时读取状态的异步任务，用到的关键字是`async`和`await`。其中async关键字用于声明一个方法是异步方法，而await关键字修饰一个需要异步执行的操作，这个关键字必须使用在async方法中，如：

```dart
networkTask() async{ 
    var response = await postRequest();
    return response；
}
```

上述方法在加上了async关键字后，返回的结果会被自动包装为Future\<T\>对象来承载方法结果，且这个方法在方法体执行完之前就会返回结果。而await修饰的语句会一直阻塞至执行完成，返回一个相应的对象。

> Steam相关的暂不记录，没弄清楚这个的使用场景。

如果在Dart的类中定义`call`方法，则可以将类对象名视为方法名来调用这个方法，参数可以自定义。

可以通过`typedef`关键字来修饰方法，来定义方法的类型，这个类型包括方法的入参类型和返回值类型，如：

```dart
//定义返回值为bool，接收两个String入参的方法未Compare方法
typedef bool Compare(String ori,String src);
```



### 实战笔记

- 在开始之前：无论需要用到哪个控件，如果对它的属性不太熟悉的话，可以直接进去看它的构造方法，flutter的变量和注释还是比较容易看懂的。

- 关于布局的一些先行概念：Flutter的布局采用的是沙盒模型，控件通过约束（**Constraint**）来决定沙盒的尺寸（**Size**），同时父级控件会将约束条件传递给子集控件，从而影响子控件的绘制。在此基础上，需要注意**无界约束（Unbounded constrains）**与**柔性对象（Flex）**的概念——前者表示控件至少在某个方向上的约束设置是无限的，典型代表即作为柔性对象的**Column、Row**或者具有滚动条的**ListView**；而后者表示一类在某一方向上有序排列的布局，其特性为：在有界约束下，其会尽可能地占满空间（往延伸方向上，即match_parent），在无界约束下时，则会适配子控件的大小（即wrap_content）。

  > 就约束本身而言，虽然看起来是在设定控件本身的大小和尺寸，但实际上约束设定的是其子控件绘制的区域大小，而父控件往往只是在根据子控件的大小进行变化。
  >
  > 这种说法解释了为什么Container在直接作为Material的直接子对象时，自身设置的约束不起效，大小永远为占满全屏。

- 万物始于**Container**，如果需要设置布局的背景色、padding、**子控件整体宽高**等属性，就需要在最外面套一层Container,然而Container的宽高设置遵从了复杂的规则，并会受到父类约束规则的影响，列举如下：

  ```
  1.如果有约束配置（无论有无对齐方式），在无界约束下，Container会wrap_content，而在有界约束下，Container会match_parent。
  2.如果无约束配置，在有对齐方式的情况下，Container会将父控件的约束传给子控件，并wrap_content
  ```

  但是暂时不知道Material或者Container这类控件属于什么类型的约束，反正在这两者的约束下，Container的约束配置会失效。

- **GridView**作为网格布局，当正方向子项数量确定时，默认状态下其纵横比为1，基准为正方向子项的尺寸（横向为width，纵向为height），此时子项的边界容易溢出，故注意要设置`childAspectRatio`，保证子项的绘制空间足够。

- 动画效果而言，有**AnimationController**、**Animation**、**Animatable**、**AnimatedWidget**四个关键类。**AnimationController**类似ValueAnimator，将一个动画过程用double值表示，有下限值与上限值，能设置动画运行时间，并执行重复、停止等操作；**Animation**类则主要承担了动画状态监听设置和**AnimationController**值的动态处理工作；**Animatable**的实现基本就是各种**Tween**类，它的作用是对Animation的value值进行转化，从而变成各种AnimatedWidget能够直接使用的对象，如Offset、Color等；最后，**AnimatedWidget**，其子类是各种**Transition**类，作为视图树中的一部分，即动画执行目标的容器，会根据Tween类转化的各种对象来对当前视图进行动态渲染（也有类似FadeTransition这种类不属于AnimatedWidget，但同样可以通过Animation获得动画效果，其作用应该是直接获取value值作为渲染参数）。

- 关于页面跳转，之前看文档的时候提到过，相关的两个类是**Router**和**Navigator**。其中**Router的作用类似于Android中的Activity/Dialog/PopupWindow等基础页面单位 + Transition**，即一方面Router可以用来设定当前页面的展示形式（全屏占满还是部分中间悬浮），另一方面它可以决定当前页面是通过怎样的动画形式完成展示与隐藏（Android的上滑渐进或者iOS的左右切入等）；而Navigator的作用就更接近它的名字的意义了——即**通过新增、删除Router来完成当前页面的展示、隐藏，并通过记录Router的历史顺序来串联访问流程**，类似ActivityStack。目前看来Flutter的页面管理比Android更透明，观感上更像Fragment管理，可以直接通过Navigator方法来完成类似singleTask的效果，也方便调整页面的展示动画，但缺点是每个页面都需要手动从栈中弹出。

  >如果在main方法的APP中同时设置了home和initialRoute，则在打开APP的时候会同时打开两个页面。

- **Scaffold**是一个按MD风格首页绘制的Widget，其内容接近AS提供的Navigation Drawer Activity的那个页面模板，即包括一个Drawer、FloatActionButton和App Bar的页面。其中，AppBar是一个`PreferredSizeWidget`，这类控件定义了一个预期尺寸，而不固定大小，方便Scaffold对其进行调整（比如处于全局模式时加上一个状态栏的高度，这个模式由`primary`属性决定，默认为true）；

- Flutter页面的生命周期与数据的关联方式相当有意思，当一个Route被添加到窗口中（也就是调用了`push`）之后，会返回一个Future\<T\>对象，该对象是这个Route被关闭（也就是被`pop`）时的返回值。这种情况可以类比为：将一个Bundle对象通过Intent传给Activity，在该Activity完成任务之后向该Bundle放入需要的结果值，然后关闭页面时回调上个页面的onActivityResult。

  和Android不同，Flutter通过Future机制替代了回调形式，在上面类比的场景中，可以通过生命周期回调中处理Future的方式来传递数据；或者像`WillPopScope`所做的一样，通过await等待得到Future的结果（这个控件就通过Future标志位阻断了pop的过程，从而能够实现首页按两次返回退出的功能）。

- Flutter的常用网络请求框架是Dio，其用法搭配Dart的异步机制可以说相当简单，基本就是通过get、post等方法， 获得相应的Response，然后对其进行解析即可，大多数请求语句可能就一行代码。Dio的一大亮点是提供了并发请求统一处理的方法和可以锁定请求队列完成前置任务的Interceptor Lock设计。其扁平化的编码方式得益于Dart语言本身的特性，但的确节省了当前网络请求框架需要与RxJava等框架结合使用的开发成本。

- **Completer**是一个Future的生成器与控制器。很多时候Future的使用依赖于一个显示的对象构造，且只能预设其获得目标值之后的操作。但Completer允许通过complete、completeError等方法，来控制Future的设值与完成状态，从而起到与Java体系中回调相似的作用，不过其功能实现基于Future的监听机制。

  >这里关于Future需要详细说明一下：
  >
  >Future的各种构造入参都是FutureOr\<T\>，其声明了一个Future\<T\>或一个T类型的参数，这是一个综合类型，由编译器动态解释（语法糖之一）。
  >
  >当一个方法的返回值为Future时，这个方法的非Future返回值，若类型为T，则会被包装成Future\<T\>（语法糖之二）。
  >
  >Future任务的执行依靠isolate的事件队列循环进行调度，其中`microTask`(微任务)的执行优先级高于一般的`event`(事件)，可以通过Future的方法来构造这两种优先级不同的事件。
  >
  >Future的异步效果实际上是**“延迟执行任务”**的表现，一个Future任务也许会晚于当前方法的代码执行，但当几个Future任务在一个时段进行排队执行时，其执行效果永远是有序的，而明显区别于多线程工作。同时在Future任务执行时间较长时，会阻塞UI的交互。所以密集型计算任务，还是需要开启新的isolate来执行。

### Tips

- 当项目运行卡在`resolve dependences`时，可以在项目的`android`目录中用gradlew命令手动下载依赖文件。
- flutter对gradle的配置文件在`[flutter sdk 目录]\packages\flutter_tools\gradle\flutter.gradle`文件中

