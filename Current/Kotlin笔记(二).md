#### 十三.Functions

1. _声明方法_：特点是以**fun**关键字来表征函数，并且返回值类型写在参数列表之后，无返回值可省略，甚至返回值语句可以直接以赋值的形式表示返回。

   ```Kotlin
   示例：

      //正常的书写方式

      fun sum(a: Int, b: Int):Int{

        return a+b

      }

      //无返回值时的书写方式

      fun sum(a: Int, b: Int){

        println("a+b={a+b}") //这里的{}用法和shell类似，直接把a和b的值取出转化为了字符串

      }

      //简化有返回值的书写方式

      fun sum(a: Int, b: Int) = a+b
   ```

   >Kotlin中方法作用域同样取决于其定义域。由于可以直接**在包下定义方法**而不需要依赖于类，故可以直接导入相应的包实现调用方法，也可以通过导入相应的类调用它们的成员方法和声明的拓展方法。另外，Kotlin也允许**在方法内定义方法**，即本地方法，其代码块内允许调用外部方法的本地变量。

   ​

2. _调用方法_：可以跟Java使用一样的方式，使用对象来调用方法，或者采用`infix`修饰符直接调用（有点像MATLAB）：

   ```Kotlin
   //函数声明
      infix fun ClassName.handle(x: Int): Int{
        ...
      }
      //函数调用 |注意，只有单参数且用infix修饰的成员方法或者拓展方法才能够这样调用
      obj handle 1
      //其相当于
      obj.handle(1)
   ```

3. _函数参数_：Kotlin的函数参数允许默认值，这个特性类似Python，这可以减少重载，但需要注意的是，参数的顺序很重要，当参数列表前面有带默认值但未二次赋值的参数时，不允许直接跳过这些参数给后面的参数赋值：

   ```Kotlin
   //声明带默认参数的函数,注意这里的编译顺序是有序的，如果str在num之前，可以将num的默认参数声明为str.length,但现在这种情况则不行
   fun sayGoodbye(num: Int = 1, str: String = "GoodBye"){
   	...
   }
   //调用
   sayGoodbye()//允许
   sayGoodbye(2)//允许
   sayGoodbye("Hello")//不允许

   //另，函数的默认参数是固定的，当有函数被重写时，不允许再设置自己的默认值，而是默认使用父类函数的默认参数值
   ```

4. _命名参数_ ：Kotlin还支持给函数参数命名，但要求调用时的名字与声明时相同：

   ```Kotlin
   //以上面定义的函数为例，可以这样调用：
   sayGoodbye(num = 2, str = "Hello")
   sayGoodbye(num = 2)
   sayGoodbye(2, str = "Hello")
   //但不能这样调用：
   sayGoodbye(num = 2, "Hello")

   //另，官方标明该特性不能用在Java的函数中，理由是Java的字节码并不都保留参数名称
   ```

5. _可变参数_：与Java的可变参数相同，Kotlin的可变参数是以一个无上限的数组表示的，其类型可以为确定类型也可以为泛型，不过声明时不再是`var1...var2`的形式，而是用到了一个修饰符`vararg`，示例如下：

   ```Kotlin
   fun varargFun(vararg t: String): List<String>{
       var list = ArrayList<String>()
       for(s in t){ //可以看到这里的t是一个数组
           list.add(s)
       }
       return list
   }
   var list1 = varargFun("L","z","x","y") //以单个参数的形式一个个传入
   var list2 = varargFun(*array,"a") //传入已存在的数组，用"*"作为取值标记
   ```

6. _尾递归函数_：最后，Kotlin支持用简单的递归代替循环的编程风格。其提供了`tailrec`修饰符用于修饰需要递归的方法。只要该方法满足最后一个操作是调用自身的操作（即进入递归）这一条件，编译器就会自动对其进行优化，使其不会栈溢出。其目的是为了简化代码，并且不能用于`try-catch`语句中，目前只在JVM虚拟机支持下起作用。

___

#### 十四、Lambda表达式与高阶函数

Kotlin中，允许将一个函数作为参数的函数称之为高阶函数，其使用场景类似于直接传入一个不依赖于类的Callback。高阶函数的参数传入方式有下面几种：

```kotlin
//首先通过 参数->返回值类型 的方式声明函数参数应有的格式
fun <T> higherOrderFun(obj: Any, paramFun: () -> T ): T{
  //...
  return paramFun()
}

//传参方式一、通过 "::函数名"引用已有的函数
fun refrenceFun = doSomething()
var t = higherOrderFun(obj, ::refrenceFun)

//传参方式二、通过Lambda表达式直接将函数实体直接传入参数列表
var t = higherOrderFun(obj, { doSomething() })
```

关于Lambda表达式，还有下面这些特性：

```kotlin
//当需要传入函数是最后一个参数时，Lambda表达式可以直接写在括号外
var t = higerOrderFun(obj){
  //省略了" -> "
  doSomething() //这里不需要return
}

//当paramFun需要传入一个参数时，Lambda表达式可以用 参数名 -> doSomething() 的方式用自定义参数名调用传入参数，也可以直接用默认的"it"来表示这个参数。
var t = higerOrderFun(obj){
  str ->  //假设需要一个String为参数
  doSomething(str)
  //或直接写doSomething(it) 省略str ->
}

//当没有用到所用方法的参数时，可以直接用"_"代替参数名
var t = higerOrderFun(obj){
  _, str ->  //假设需要两个参数，第一个参数没有用到
  doSomething(str)
}
```

以上用Lambda表达式表示的函数，在不同的场景下属于不同的类型——作为参数在高阶函数中时，其参数类型为(type1 ... typeN) -> type，可以直接将其赋值给变量；而在高阶函数函数体中被使用时，其类型为一个参数为(type1 ... typeN)的函数。

与Lambda表达式类似，高阶函数还可以传入一个匿名函数作为参数，其与Lambda表达式不同的地方在于其能够声明返回值类型并且有`return`语句，不过不能将函数体写在参数括号外。示例如下：

```kotlin
higherOrderFun(obj, fun(s,i): String{
        return "s"
    })
//或直接用"="后接一个返回String的表达式
```

最后，Kotlin允许一种类似拓展函数的函数声明方式，叫做**接收者对象**_(receiver object)_。其允许对其他类进行方法拓展，并可将该函数作为变量给出：

```kotlin
//简单声明，同时也满足其他正常函数的声明方式
val sum = fun Int.(other: Int): Int = this + other
1.sum(2) //调用，类似中缀函数

//用Lambda表达式表示接受者对象
fun str(init: String.() -> Unit): String {
    val str = "test"  
    str.init()        
    return str
}
//调用
str{
  //... init函数的实际表达式过程，此时可以用this表示其调用对象，比如这里就是"test"字符串
}
```



___

#### 十五、Inline Function

前面提到的高阶函数有一个缺点：其维持的方法参数都被视作为一个对象，而这些方法内又包含着若干其他变量（包括外部方法的本地变量，即**闭包**_(closure)_），故过多高阶函数(或者说其函数参数)的存在会造成大量的内存消耗。为此Kotlin提出了**内联函数**的形式，即**将被标记为内联的函数代码块内语句全部生成转移到调用区域**。通过看文档理解这一点真的费了我好大的功夫，具体来说应该是这样的：

```Kotlin
//原来的高阶函数
fun higherOrderFun(i: Int, count: (i: Int) -> Int){
  prinln(count(i)) //当然这个语句没什么意义，完全没有必要用到高阶函数
}

//假设实现是这样的
higherOrderFun(1, {i -> i*2})

//如果将原来的高阶函数设为内联，则为
inline fun higherOrderFun(i: Int, count: (i: Int) -> Int){//...}

//对同样的实现，编译器此时会将代码生成为
//higherOrderFun(1, {i -> i*2})
println(1*2) //即直接将原有高阶函数的代码提出来作为表达式，而不去生成和调用函数
```

这样一来运行时内存中存放的就不是函数对象了，而是表达式的字节码，虽然增加了生成语句（不建议用在语句复杂的函数中），但可以省一些内存。

当然，如果高阶函数中有多个Lambda表达式传入，而使用者并不想每个都设为内联时，可以通过在Lambda式名前加`noinline`来标注其为非内联方法。

由于Lambda表达式中不能有`return`语句，故函数中循环流控制无法实现。但如果高阶函数本身是个内联函数，则Lambda中也能有`return`，且可以指定跳出范围（想想内联的作用结果就不难理解了）。不过如果此时Lambda表达式的作用域处于另一个嵌套方法或者匿名内部类中，内联的作用被破坏，也不能用`return`作流控制，并且必须在参数声明前加上`crossinline`表示禁止（想要禁止控制流时也可以用这个修饰词）。

另外，普通函数的泛型类型，正常情况下是无法直接作为一个可供比较的类使用的，需要传入相应类型的`class`参数，如：

```kotlin
fun <T> TreeNode.findParentOfType(clazz: Class<T>): T? { 
    var p = parent
    while (p != null && !clazz.isInstance(p)) { //这里需要利用传入的Class对象才能进行类型比较
        p = p.parent
    }
    @Suppress("UNCHECKED_CAST")
    return p as T?
}
```

但内联函数提供了`reified`关键字用于修饰泛型，使其可以作为一个正常的类对象被使用：

```kotlin
inline fun <reified T> TreeNode.findParentOfType(): T? {
    var p = parent
    while (p != null && p !is T) { //可以用is直接判断类型,甚至可以用"::"调用其类型和构造等参数
        p = p.parent
    }
    return p as T?
}
```

这种泛型称为**验证类型参数**_(Reified type parameters)_

最后，从Koltlin1.1开始，成员变量也能被`inline`修饰了，此时其setter和getter是内联的，与一般的内联函数相同。

___

#### 十六、Coroutines

**协程机制**是Kotlin试用中的一种异步任务控制方法，其主要思想是**自主控制任务的挂起操作来代替传统并发情景下的线程抢占式行为，即将多任务操作流程放在一个线程中，依靠事件驱动，并模仿CPU调度机制对多任务进行自主调度** 。从实现效果上来说，协程基本上就是综合了Java已有的Future和线程池的特性，其一个典型应用如下：

```kotlin
GlobalScope.lunch(Dispatchers.Main){
    val child = suspend{ getChild()} //假设这里的getChild是一个耗时网络请求
    val mom = async{ child.getMom() } //同上 这里的getMom也是个耗时方法
    val father = async{ child.getFather() }
    Toast.makeText("Child info: Mom is ${mom.await()}, Father is ${father.await()}")
}
```



上述代码中，launch方法是一个构造器方法，相当于开启一个协程，并返回一个`Job`对象（它的API形式相当于Java的Thread，可以开始或终止协程）。该方法的`start`参数默认为**CoroutineStart.DEFAULT**,表示立即开始执行协程（这里的立即执行指执行代码抢到CPU时间片的时候），如果这个参数是**LAZY**，则该协程只会被声明，需要通过Job对象的`start`或者`join`方法来开启——前者和DEFAULT一样，开始协程，并继续执行下面的代码，后者则是阻塞住当前线程，直到协程体内的任务全部完成。

>start参数还有**ATOMIC**和**UNDISPATCHED**，前者会忽略Job的cancelling标记位，一直执行到第一个挂起点才会停止；后者则直接忽略线程调度，直接在当前线程立即执行协程体。

Dispatchers.Main作为launch的构造器参数，表示下面的协程体运行在哪个线程，如果不填这个参数，默认的调度器会让它运行在一个子线程内；协程体中的`suspend`和`await`是断点方法，代码运行至这两个方法时会中断运行等待其内容运行结束后再返回；`async`是另一个协程开启构造器，表示异步地开启一个协程，不阻塞当前代码运行，其结果返回会终止await的中断状态。



___

#### 十七、Destructuring Declarations

关于解构声明，其实前面记录数据类相关的内容时就有所提及，简单来说，就是一种快速用对象中的成员变量创造引用的机制，其使用如下：

```Kotlin
//原数据类，默认实现了componentN()方法
data class Data(var value1: Int, var value2: String) 
//结构为单独的变量组
var(age,name) = Data(20,"Lazxy")
println("age = $age, name = $name") //输出age = 20, name = Lazxy

//解构Map
for((key, value) in map){ ... }

//作为Lambda表达式参数，此时可以与普通参数变量混用，不用的参数名一样可以取为"_"
{(name, age) -> ...}
```

需要说明的是，解构声明的基础建立在被解构类定义了`componentN()`类方法的基础上，且该类方法必须被`operator`修饰。

___

#### 十八、Collections

Kotlin中的集合明确地区分了可变与不可变的类型，与此相关的接口主要是List<out T>(不可变)和MutableList<T>(可变)，以及类比的Set和Map等类。构造这些集合的方式主要还是利用`listOf()` `mutableListOf()`等库函数，它们对应地可以传入不定参数和Lambda表达式等，前面已提及，也不再多述。下面照抄一些文档里有的集合相关拓展方法：

```kotlin
val items = listOf(1, 2, 3, 4)
items.first() == 1
items.last() == 4
items.filter { it % 2 == 0 }   // returns [2, 4]

val rwList = mutableListOf(1, 2, 3)
rwList.requireNoNulls()        // returns [1, 2, 3]
if (rwList.none { it > 6 }) println("No items above 6")  // prints "No items above 6"
val item = rwList.firstOrNull()

val readWriteMap = hashMapOf("foo" to 1, "bar" to 2)
println(readWriteMap["foo"])  // prints "1"
val snapshot: Map<String, Int> = HashMap(readWriteMap)
```

___

#### 十九、Range

表示范围的函数为`rangeTo`，其作为一个运算符函数，常用`..`表示，与`in/!in`一同使用，前面在for循环中已经见到过。它只能用在有限的一个集合中。相关的函数还有`downTo`——逆序范围，`until`——设置右值为开区间，`step`——控制范围遍历步长，`reversed`——使遍历顺序倒置，相当于正序排列的`downTo`。它们的使用形式如下：

```kotlin
for(i in 5 downTo 1) //[5,4,3,2,1]
for(i in 1 until 5) //[1,2,3,4]
for(i in 1..5 step 2) //[1,3,5] step必须为正
for(i in (1..5).reversed()) //[5,4,3,2,1]
```

___

#### 二十、Type Checks and Casts

如同前面提到类型验证参数时写过的，Kotlin可以通过`is/!is`关键字来检查类型是否匹配，相当于Java里的`instanceof`，并随其衍生的，Kotlin提供了一个非常方便的功能，即**在类型检查后不需要通过强转就可以直接将被检查的变量视为已检查过的类型，调用相关的成员变量或者函数**，前提条件是保证该变量不会在检查过后被修改。另外，Kotlin中的强转也不再是`(TYPE)`的形式了，而是使用了关键字`as`。并且强转过程被分为了安全与不安全两种类型，其区别如下：

```kotlin
//不安全强转
val i: String = j as String //此时如果j不是String类型或其子类，就会像Java一样抛出运行时异常ClassCastException
//安全强转
val i: Sting? = j as? String //此时如果j强转失败，则会直接给i赋值为null，避免了运行时异常的抛出
```

___

#### 二十一、Null Safety

对于上述安全强转的表示方式其实也同样可以防止空指针异常，只要在变量后加"?"即可默认引用为空时返回空值。如：`var i = str?.length`。下面是更多空指针异常相关的操作： 

```kotlin
//如果str为空，则会返回null；如果需要在引用非空时进行一些操作，可以用let关键字代替非空判断
var i = str?.let{
  //...
}
  
//如果需要在引用为空的地方进行备选操作，则可以用?:符号来承接操作表达式
var i = str.length ?: -1 //这里也可以直接写return 或者抛出一个异常
  
//最后，如果有意要在空引用存在时抛出空指针异常，可以用!!来调用其相关成员，此时若引用为空，则会抛出空指针异常
var i = str!!.length
```



___

#### 二十二、Exception

Kotlin的异常机制其实和Java表面上看起来差不多，都是`try`-`catch`-`finaly`以及`throw`等操作，但其中还是有一些不同：1. Kotlin中`try`语句是一个表达式，可以直接返回值，返回规则和`if`基本一致；2.Kotlin**取消了Java的异常检查机制**，其既不能在定义函数时`throws`出一个异常，对于原先已经声明抛异常的Java函数，也不再强制要求一定要try-catch其抛出的异常。另外，`throw  Exception`这个动作也是一个表达式，其返回值为泛型中提到的下限类型`Nothing`，当一个变量或者一个函数返回该类型时，即代表语句将不再进行下去。

___

#### 二十三、Reflection

这里的反射指的是通过名称来获取各种类、变量和函数的对象，它的操作符便是在Lambda表达式中见过的`::`。以下是各种反射的用法示例：

```kotlin
1.类的反射
  val k = KotlinClass::class //获取一个KClass实例，大概和Java的.class字节码文件差不多？
  val j = JavaClass::Class.java //对Java类获取KClass实例 在Kotlin1.1后可以直接通过.javaClass调用
2.函数的反射
  fun reflectFun(){
    //...
  } //原函数
  val func = ::reflectFun //对本类函数的引用，类型为KFunction，可能相当于Java的Method
  val funOut = String::get //对其他类函数的引用
  //这些函数的引用可以作为高阶函数的参数或者直接调用，参数仍跟原函数一致，可以视作是一种别名机制。
3.已知变量的反射
  var x = 1//原变量
  fun main(args: Array<String>){
    println(::x.get()) //获取变量的反射值，类型为KProperty，可以调用该变量的getter和setter
  }
4.未知变量的反射
  data class DataClass(value1: Int, value2: String)
  fun getData(): Int{
    var value = DataClass::value1 //将变量引用确定为类的成员变量，但在其被调用前，变量值是不确定的
    return value.get(DataClass(20,"Lazxy")) //返回20
  }
```



___

#### 二十四、Type Alias

像最开始的Package重命名一样，类名也可以进行重命名，其表达式也很简单，就是`typealias [OLDTYPE] = [NEWTYPE]`。

___

#### 二十五、JAVA与Kotlin混用

一般情况下，在Kotlin中用Java的类是没有问题的，但是需要注意以下情况：

1. 调用成员变量时

   ​	在Kotlin中以`.`的方式调用成员变量时实际调用的是getter和setter方法，如果被调成员的类是Java类，则该类必须有getter或setter方法（无参且开头为get，有一个参数且开头为set，参考FastJson的解析方式），且不允许只有setter方法。

2. 当Java类的函数返回`null`时，将自动转化为`Unit`。

3. 如果Java类的函数名与Kotlin的新增关键字冲突（如`in` `is` `object`等），则调用时应该用`''`将函数名引起。

4. Java类的变量类型称为平台类型，其可以为匹配的非空或可空变量赋值，也不会被Kotlin编译器进行非空验证，但在空指针异常发生或者传输过程中被检空的时候可能被抛异常。而平台类型并不能显式声明，故Kotlin设置了`!`作为助记符，表示该类型为可空类型或为非空类型，如：`Array<(out) T>!` //means "Java array of `T` (or a subtype of `T`), nullable or not"。

5. 非空类型注解所在包在各IDE上是不一样的，只列举相关的两个：**JetBrains** (`@Nullable` and `@NotNull` from the `org.jetbrains.annotations` package)

   **Android** (`com.android.annotations` and `android.support.annotations`)

6. Kotlin的泛型采用的`out`和`in`修饰符，代替了`Java的extends` 和 `super`，并将Java的无泛型记为`<*>`即\<out Any?\>! 但其如Java一样运行时不保留泛型信息，不允许判断泛型类型，即`is`关键字不能用于`a is List<Int>`之类的操作，但可以写`a is List<*>`（相当于没有设置泛型，只是判断其是否为List类）

7. Kotlin的基本数据类型数组在编译时会转化为Java的基本类型数组，以提高性能。~~文档中提到Kotlin的子类数组不能转化为父类数组类型，而Java的数组在Kotlin中可以，这跟泛型中讲到的不一样，还需要日后实验。~~

8. Kotlin的Any没有关于线程的方法，故如果需要调用`notify()`和`wait()`，则需要强转对象为Object`(foo as java.lang.Object).wait()`

9. 如果需要重写`clone()`方法，则要实现`kotlin.Cloneable`接口

10. 如果需要`finalize()`方法，只要直接在类中定义即可，不过其可见性不能为`private`

11. 如果需要在Kotlin中调用JNI，则可以用`external`修饰方法，相当于Java的`native`：`external fun foo(x: Int): Double`。