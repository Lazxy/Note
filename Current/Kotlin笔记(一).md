

### Kotlin笔记

#### 1.Package：

同Java基本一致，不过新增了 as关键字为包取别名用于对包名进行简化。

```
示例：
import bar.Bar as bBar
```

___

#### 2.Basic Types

Kotlin中，所有类型都是对象（基本类型在被调用时都是其封装类型），可以大致将数据类型分为*numbers, characters, booleans , arrays*几种。

1. Numbers：

  其类型与Java类似，但并未将char归入，且不同类型间没有默认转换。但Kotlin允许在数字间加下划线使可读性更强，如 val bankCardNum: Long = 6220\_2321\_1567\_3587\_189L。另外，如Int、Long等类型虽然为对象，可以调用相应的成员方法，但其不能为null，而只能设置数字值，**可为空的数据类型在赋值时应声明为Int?**。由于上面提到的数字类型间没有默认自动转换的特点，Kotlin提供了一系列toXXX格式的类型转换函数(如toLong())，这些函数继承于其抽象基类Number，当然其使用情景只在于不同类型的直接赋值，**对于运算，它们的运算符都得到了重载，会自动向上转型**。最后，Int和Long还有一套对应类型的位运算运算符，包括

  ```
  shl(Int|Long) – 左移，补0

  shr(Int|Long) – 右移，补0

  ushr(Int|Long) – 无符号右移 (Java's >>>)

  and(Int|Long) – 与

  or(Int|Long) – 或

  xor(Int|Long) – 异或

  inv() – 按位取反，包括符号位

  （上述方法都以infix关键字修饰，表示可以像运算符一样置于两个变量间使用）
  ```

2. Characters：关于字符型，与Java不同的是其转义字符只剩下`\t`, `\b`, `\n`, `\r`, `\'`, `\"`, `\\` 和 `\$`

   等几种，其他的特殊字符都用Unicode表的对应码表示。并且同Numbers一样，其自身不能为空，除非声明时用Char?。

3. Boolean：和Java一致，同样不能为空，除非声明为Boolean?。

4. Arrays：Kotlin中数组的表示方法是一个带泛型的类(即Array<>)，泛型就是数组元素类型，其自带get\set\iterator等操作符类方法(当然实际使用时就是正常的a[i] = b的格式，语法糖的沿用)。此外对于原始数据类型Kotlin还另外封装了ByteArray, ShortArray, IntArray等几个数组，它们并不继承于Array类，但使用方式与Array一致。关于Array的类型创建，有以下几种方式：

   ```
   1. arrayOf(object1 ... objectN) 库函数，能够创建以参数为数组元素的数组。
   2. arrayOfNull(Int size) 库函数，能够创建指定长度的数组元素全为空的数组。注意，此时被赋值数组的泛型一定要加?,否则会提示类型不匹配
   3. Array(size, { i -> [Expressions] }) Array的静态构造函数，可以根据输入的size参数确定数组大小，并且依次迭代角标，将其值赋给参数列表中的接口函数作为参数，然后将表达式返回值作为数组元素。(如输入size = 3，设其表达式为 i+1，则该数组将被初始化为Array<Int>{1,2,3})
   ```

5. String：Kotlin中的String同Java一样不可变，但可直接通过循环迭代。其表示方式主要为两种：1. _Escaped Strings_(转义字符串) ，同Java一样用`"` 引起，其中可以包括上面讲到的可用转义字符，但两个`"`必须在同一行，跟Java的代码编写习惯不太一致；2._Raw Strings_ (原生字符串)，这种字符串需要用`""" `来引起(MarkDown语法既视感)，其内部不支持转义字符，但可以自定义换行标记，并用两种字符串通用的`$`符号进行对变量的格式化引用(相当于%s %d等符号，只不过这里变量名是直接跟在`$`后面的，且会自动调用toString()方法)，基本也不再需要转义字符了。值得一提的是，Raw String 的换行标记是通过调用`trimMargin(str)`方法设定的(默认参数为"|"，或者直接敲换行符也是一样的)，而这个方法的基础功能是去除开头的空格或换行。最后，上面提到的`$`符号，在Raw String 中因为没有转义字符，所以如果需要显示这个标记而不是错误得取其后名称的变量的值的话，得写成`${'$'}`(貌似有些Shell的影子)，如果需要放一个表达式的话，可以用`${[Expression]}`的形式来表达。

>Kotlin为对象间的比较设置了两种方式“==”和“===”，前者比较两者的值_(其实就相当于Java的equals)_，其反义为“!=”；而后者为引用比较，其反义为“!==”，只有当两个引用指向同一个地址时会返回true_(可空引用和非空引用一定不相等，被视为是两种类型)_，如：
>`val a: Int = 10`  ` val b: Int? = a` ` val c: Int? = a`  `val d: Int = 10`
>a === b //false    b === c //false    a === d//**true**_(这里的原理就是Java的常数池)_
>a == b//true b == c//true c == d//true



>关于Kotlin符号的重载，其重载符号实际是以类成员方法的形式给出的，这些方法都以**operator**关键字为修饰符，可以由使用者自定义重载。值得注意的对应重载有下面几个：
>
>| 运算符表达式               | 对应运算方法                      |
>| -------------------- | --------------------------- |
>| **+a**               | **a.unaryPlus()**           |
>| __-a__               | __a.unaryMinus()__          |
>| __a in b__           | __b.contains(a)__           |
>| __a(i_1, ..., i_n)__ | __a.invoke(i_1, ..., i_n)__ |
>
> 一个看起来很诱人的例子：
>
>data class Point(val x: Int, val y: Int)
>
>operator fun Point.unaryMinus() = Point(-x, -y)
>
>val point = Point(10, 20)
>println(-point)  // prints "(-10, -20)"

至此基本数据类型算是告一段落。总体来说与Java无异，可能类型要求更严格了，但使用无疑更简单。

___

#### 3.Control Flow（控制流）

​	Kotlin中的流程控制同样是基于if、switch(在这里是when)、for、while等条件判断或者循环语句实现的，其使用思路和Java也是大体一致的，下面简单地列举一下：

1. if

   Kotlin中`if`保留了原来的用法，即作为判断语句，在其后的代码块或者下一行中写相应的表达式；但其增加了一种新的用法，或者说简化了其用法，即——将if语句作为赋值式，其示例如下：

   ```kotlin
   var a: Boolean = if(str == "Hello world") "Hello" else "world"
   //注意，当if为赋值语句时，必须要有else，否则编译器会报错。
   val a = if(str !== "Hello"){
           println("It's true")
           true //正确
       }else {
           false
           println("It's false") //错误
       }
   //这里与函数不同，函数的返回语句后不允许有可执行语句，编译会不通过，而这里的策略为返回最后一句所表示的值，编译只检查返回值是否与变量声明值一致，若不声明，则如上false的场景其并不会通过判断返回false，而是输出语句后返回Unit(即null)。
   ```

   显然，判断子句是事实上的赋值表达式，相当于将其`return`给了变量`a`，在代码块只有一句代码时，其效果与原来的三目运算符是一致的，而当if else的下面为多句逻辑代码时，便会依次执行赋值语句前的代码，然后完成赋值操作，与函数的实现方法一致。

2. when

   `when`关键字在Kotlin中用于代替`switch`，但其功能更加地完善。它的基本用法如下：

   ```kotlin
   when(x){
     1 -> println("It's first") //这里不需要break，只执行符合条件的语句
     2 -> println("It's second")
     else -> print("It's" NAN) //代替了default
   }
   ```

   同时判断值可以是表达式，包括in和is表达式：

   ```kotlin
   parseInt(s) -> println("It's s ")
   in 0..10 -> println("It's in 0-10") //相当于Java里的x>=0&&x=<10
   is String -> println("It's a string") //相当于Java里的Instanceof String
   //注意，当其符合多个语句的条件时，只会选择执行最前面的语句。
   ```

   最后，它甚至能够直接作为多if语句使用，比如：

   ```kotlin
   when {
       x.isOdd() -> print("x is odd")
       x.isEven() -> print("x is even")
       else -> print("x is funny")
   }
   ```

   此时不需要传入x的值，就能够直接进行逻辑判断。

   除此之外，上面所有的判断表达式后的表达式可以和if一样作为返回值输出，使用非常方便。

3. for

   Kotlin中的for循环与关键字`in`是不可分割的，其循环过程便是一个__迭代__过程，基本使用如下：

   ```kotlin
   for(item in collection){ //对集合的遍历
     //...
   }

   for(item: Int in ints){	//对数组的遍历
     //..
   }

   for(i in array.indices){ //根据数组或集合的角标进行遍历，等价于 i in 0..array.size-1
     //..
   }

   for((index,value) in array.withIndex()){//遍历过程中同时取得当前角标和其对应的值
     //..
   }
   ```

   可见由于其更高的集成化，更类似于Java中的for each，但使用起来总觉有些繁琐。

   > Kotlin定义，一个可以被迭代的对象必须有三个成员函数或者拓展函数：iterator(),返回一个迭代器;next(),返回一个类型确定的子类型对象;hasNext(),返回一个代表是否存在下一个值的Boolean变量。这三个函数都得用`operator`修饰。

4.  while

   Kotlin中的while语句与Java中完全一致，不再多说。

___

#### 4.Returns and Jumps

Kotlin中的循环跳出和函数返回关键字仍然为`break` 、`continue`和`return`，且它们的作用范围也与Java相同：在默认情况下，break和continue会跳出当前所在层的循环，而return会直接返回最外层函数的值，即跳出整个循环；但在当前函数为**匿名函数**时，则为返回当前匿名函数的结果，而不中断整个函数体的运行。这三者可以由`@`关键字修饰的`label`标识来确定范围，示例如下：

```kotlin
loop@ for((index,value) in array.withIndex()){  //label名字可以自定义，
        for(i in 1..3){
            println("The Number is " + value)
            if(i == 2)
            	continue@loop; //这里也可以为break
        }
    }
    
fun foo() {
    ints.forEach lit@ {
        if (it == 0) return@lit value//对于函数来说，这里可以直接以lambda表达式的名字forEach来命名label，即return@forEach value也能指定相同的范围
        print(it)
    }
}

fun foo() {
    ints.forEach(fun(value: Int) { //匿名函数下，return语句的范围默认为最小范围
        if (value == 0) return
        print(value)
    })
}
```

>   在Java中，其实也有类似的标记，此时该标记写在for之前，且没有@符号，如：
>
>   out: for(...) {
>
>   ​	for(...){
>
>   ​	 break out; 
>
>   ​	}
>
>   }

---

#### 5.Classes and Inheritance

**Kotlin类的表示**同样是`class`，不同点主要在于其构造方法的表示。在这里一个class拥有有一个主构造函数和若干个副构造函数，并且主构造函数的参数列表是直接写在类的声明中的，其表示方法如下：

```kotlin
1.无参无内容的类
	class Test  //直接省略了"{}"
2.无有参构造的类
	class Test constructor(){ //这里的constructor可省略，除非这个构造是有指定范围的，默认范围为public，下同；由于无参，这里的"()"也可以省略
      //..
	}
3.以有参构造作为主构造函数的类
	class Test public constructor(val name: String,val age: Int){//这里的val可以省略，参数列表内默认为常数
	  var mName = name //可以作为成员变量的初始化元素,同时也可以直接将构造参数拿来使用
      init{
        println(mName) //也可以在init代码块中进行操作，这个代码块即相当于构造的函数体，也可以省略
      }
	}
```

而对于副构造函数，其特点为必须以`constructor`关键字修饰，且必须实现主构造函数，原因大概也是因为主构造方法的参数会用来初始化成员变量（如同子类构造必须继承父类构造函数一样），其表示如下：

```kotlin
class Test(name: String){
  constructor(name:String , age: Int) : this(name){
    println("Name = " + name + "Age = " + age)
  }
}
```

除此之外，其实可以直接省略主构造的写法，以一个正常Java类的结构来写构造函数（区别仅仅在于将类名换为constructor修饰符），这样副构造函数就会成为仅剩的可选构造，也不需要对主构造进行显式依赖了。

```kotlin
class Test{
  constructor(){
    //...
  }
  constructor(name: String, age: Int){
    //...
  }
  //...
}
```

还有一种特殊情况是：**当主构造方法的所有参数都有默认值时，编译器会额外生成一个无参构造供使用，其实现主构造的方式为直接的默认值填入。**

最后，对象的构造仍然是对构造方法的调用，但`new`关键字也被废弃不用了：`t: Test = Test(); `

**Kotlin的继承机制**也有些不太一样。首先，其基类不再是Java中的Object，而是**Any**，它只有`equals()`, `hashCode()` and `toString()`三个方法，其他Object有的方法需要进行Java化的处理~~，后面会讲到~~ ；其次，基于上面的构造方法表示，其继承的写法与Java不太相同，主要有以下两种：

```kotlin
1.直接用主构造方法继承父类的构造
class SubClass(name: String): SuperClass(name){
  //... 此时副构造只能继承主构造的方法了
}
2.用副构造方法来继承父类的构造方法
class SubClass: SuperClass{
  constructor(name: String): super(name){
    //...
  }
}
//由此可见，其继承顺序为子类副构造->子类主构造->父类构造（主、副对子类来说没有区分）
```

而且不同于Java，基于**明确、显式(explicit)**的原则，父类中的成员变量和方法不再是默认可继承的了，而是需要关键字`open`的修饰，否则其默认为`final`，于此同时子类**必须**要用`override`关键字声明对父类变量或方法的修饰，而不再选择性地用`@Override`注解。其中有两个需要注意的点：1.对于方法的继承，由于Kotlin中的接口是可以有方法具体实现的~~（三观尽毁）~~，所以Java中原本没有的**多继承问题**就凸显了出来(*顺带一提Java中的判定方式是直接将父类的实现作为接口的实现，子类甚至都不需要去实现该接口的方法*)，解决方法是：对于接口与父类中都有的同名方法，在继承父类实现时，用`<>`来作区分，如下：

```kotlin
open class A {
    open fun f() { print("A") }
}

interface B {
    fun f() { print("B") } //接口的方法默认都是open的
}

class C() : A(), B {
    // The compiler requires f() to be overridden:
    override fun f() {
        super<A>.f() // call to A.f()
        super<B>.f() // call to B.f()
    }
}
```

>修正，这里的多继承冲突问题其实Java8中已经出现了，其在两个父类/接口拥有同名方法而不重写的时候，会直接报编译时错误，强制要求重写。

2.对于成员变量的继承，**允许将父类的常量重写为变量**(实际就是给变量加了一个setter)

最后是关于抽象类的继承，这与Java没什么不同，open关键字在abstract修饰的方法上也不再需要显示写明，以前未注意到的一点是：**一个非abstract父类的子类可以将重写的成员函数用abstract修饰， 当然其本身也应该是个abstract类。**（Java里也可以这样重写。）

>对应于Java中的`private` `protected` `default` `public`，Kotlin中的可见性修饰符为`private` `protected` `internal`和`public`，基本与Java一致。但 internal与default有很大的不同，此时其作用范围不再是`package`，而是Android中的一个`module`或者Maven或Gradle中的一个`project`，并且Kotlin中的默认修饰符为`public`。除此之外，标明为`private`的内部类/嵌套类成员对外部类是不可见的，这与Java不同。
>
>顺便一提，Java中内部类的创建方式为先创建外部类对象，再通过外部类对象创建内部类对象，如下：
>
>`OutClass o = new OutClass(); OutClass.SubClass s = o.new SubClass(); `
>

___

#### 6.Properties and Fields

Kotlin类中的成员变量使用与Java一致，都是用"."表示调用变量(实际调用的是其getter和setter方法)，但其声明和表示则复杂许多，定义如下：

```
成员变量基本结构
var <propertyName>[: <PropertyType>] [= <property_initializer>] //从1.1版本开始，类型可推测时可将类型声明省略
    [<getter>] //可选，默认为获取该变量的值
    [<setter>] //可选，默认为将传入值赋予该变量
```

而根据变量的两种类型`val`和`var`，有如下区别：

```kotlin
对于var变量，当其存在非默认setter方法且getter和setter方法中都没有用到本体字段field时，其可以不进行初始化，如：
var age: Int
	get() = name.length
	set(value){
		value*2
	}
但当其不存在非默认setter方法或者其在getter和setter中用到了field时，其初始化必须在声明或者构造中完成。
var name: String = "Lazxy"
	//get() =	getName()
	set(value){
		field = value
	}

而对于val常量，由于其不能有setter方法，情况变得稍微简单了一些。当其只有默认getter方法，或者getter方法中用到了field字段时，必须在声明或构造时对其进行初始化，如：
val size: Int = 1
而当其存在非默认getter方法时，只能通过getter方法对其赋值，而无法自主进行初始化操作。并注意，由于这里的getter方法没有限制，所以实际上这个常量是可变的，不过其变化范围依赖于getter中涉及到的变量，且无法被二次赋值。

//上面提到的field字段指代的值为该变量/常量的实际值，当该字段被使用时，必须存在back field，即可用的初始备用值。
```

需要补充的是，所有变量的可见性与其getter/setter方法是一致的（或者说变量的可见性约束了它的方法），这样一来要如何实现JavaBean的结构呢？~~官方给出的方法是这样的：~~ 

```kotlin
private var _table: Map<String, Int>? = null
public val table: Map<String, Int>
    get() {
        if (_table == null) {
            _table = HashMap() // Type parameters are inferred
        }
        return _table ?: throw AssertionError("Set to null by another thread")
    }
```

~~即为实际要访问的变量设置一个可见性更高的封装，虽然实际调用值没有区别，但是没有办法直接调用原变量。这种做法略显繁琐，同时也暴露了一个没有必要可见的table封装，但由于设计理念差异，要遵循传统就只能这样做了。~~ （翻新的文档发现已经找不到这段了 = = 显式调getter和setter方法似乎已经彻底被放弃了）

针对于上面说到的常量可变问题，Kotlin也提供了一个专门的修饰符`const`，与C++一样，专门来修饰不可变的常数。其修饰对象只能是为String或基本数据类型的类的成员变量/**包级变量**，且不能有非默认的setter方法。

最后，为了解决测试或者对象注入时变量不应被初始化的问题，Kotlin提供了`lateinit`修饰符用于变量的延迟初始化，不过该修饰符也只能用于非空类型的非基本类型`var`变量，其setter和getter必须为默认值。若访问一个未初始化的延迟初始化变量，则会抛出一个异常提示该变量并未被初始化。

>这里的包级变量指的是直接定义在package XXX 之下，类之外的变量，Kotlin中允许这样的定义，并称之级别为_top-level_,这种变量的可见性不能为protected。

___

#### 7.Interface

Kotlin中的接口类似Java8中的接口，可以为`default`方法提供默认实现（不过Kotlin里没有default这一字段，而是像声明普通方法一样声明虚拟拓展函数），但这里的接口还可以实现另一功能：提供可重写的成员变量（论接口常数类的灭亡），其规则如下：

```kotlin
//当成员变量类型为var时，var不允许有初始化值，也不允许自定义的setter和getter方法，默认为抽象变量。
var a: String
//当成员变量类型为val时，可为正常的val定义，可以设置初始化值或者设置getter方法，也可以为抽象变量。
val b: String = "lazxy"
or
val b: String

//以上两种变量都必须被接口的实现类重写，重写后必须为非抽象变量，且val可以被重写为var类型。
```

___

#### 8.Extensions

对各种类成员的拓展是Kotlin的一大特点，也是非常好用的一个特性，其类似于设计模式中的装饰模式，支持对成员方法和成员变量的拓展，下面分别来看。

1. **functions**

   对成员方法的拓展实际上是为某个类创建一个类型与方法的临时对应表，使之能够根据类型以`.`的方式调用执行相应的语句，它的声明如下：

   ```
   fun [CLASS].[FUNTION](params...): [RETURN_TYPE]{
     //.. 
   }
   //其中CLASS为拓展函数的宿主，也就是被拓展的类；FUNCTION为拓展函数的函数名；params为函数的参数列表；RETURN_TYPE为函数的返回值
   ```

   在该拓展函数中，可用`this`来表示调用该方法的类对象，同时，也可以不带前缀地调用被拓展类与当前类的其他方法**（这是拓展函数被提出的主要原因：为了尽量少地标识类名且不需要一次性导入所有所需类的相关方法）**，如下:

   ```kotlin
   class D {
       fun bar() { ... }
       val d: String = "Class D"
   }

   class C {
   	fun bar() { ... }
       fun baz() { ... }
       fun D.foo() {
      		println(this.d); // 调用被拓展类的其他参数
           bar()   // 默认调用被拓展类的方法，若需要调用当前类的同名方法，则需要写this@C.bar()
           baz()   // 调用当前类的方法
       }
   }
   ```

   **注意，由于该方法是静态地附加在一个确定的类上的，故当被调用时，以类名为准，而与其实际类型无关（类似于Java成员变量，父类与子类的同名成员变量是根据类的当前类型取值的），这跟继承特性是不同的。**并且当拓展函数与类的原成员函数同名时，会调用原成员函数，示例如下：

   ```kotlin
   open class C{
     open fun foo() = println("C foo") //原成员方法
   }
   class C1: C()
   class D{
       fun C.self() = println("C") //父类的拓展方法

       fun C1.self() = println("C1") //子类的拓展方法

     	fun C.foo() = println("D foo") //重写原成员方法为拓展方法

       fun invokeC (c:C){
           if(c is C1)
               c.self() //这里由于Kotlin的编译优化，在确认了c的类型后，会将其转为C1
           else
               c.self()
         	c.foo()
       }
   }
   //-main中调用
   var c: C = C()
   var c1: C = C1()
   val d = D()
   d.invokeC(c)
   d.invokeC(c1)

   //结果
   C
   C foo
   C1
   C foo
   ```

   另外，拓展函数的宿主可以为空，此时需要用`this == null`进行一次判空，从而避免空指针异常。

2. **properties**

   成员变量的拓展与成员方法基本一致，但由于成员变量不具有备用字段（即backing field，可能是由于变量的状态不能被保存？），故其不允许初始化，只能通过getter或setter方法对其进行显式设置，如：

   ```kotlin
   val <T> List<T>.lastIndex: Int
       get() = size - 1
   ```

​        类成员的拓展范围取决于其声明所在的类/包，当导入相应的声明类/包后则拓展方法和变量起效，**注意如果两个被依赖的类/包都对同一个拓展方法或者拓展属性进行了定义，那么会发生定义冲突导致编译不通过，这个时候可以为冲突的导入选项进行重命名，例如：**

```kotlin
import knative.index
import knative.other.index as index2 //同名拓展属性，方法也一样

fun whatever(){
    val c = C()
    c.index
    c.index2 //避免了定义不明的冲突
}
```



___

#### 9.Different Type Of Classes

根据Java的使用惯例和一些通用写法，Kotlin设计了一些特定的功能类，完成了很多常用封装，下面一一列举：

1. Data Class

   这种类通过其名字就能发现，它的作用为保存数据，相当于JavaBean，或者MVC中的Model。其声明标识为`data`，要求必须有有参主构造，且主构造中的参数必须明确标明`val`或`var`类型；其类型不能为`abstract` `open` `sealed` 或者内部类，且不能继承接口以外的类（1.1版本后可以继承了）。数据类构造示例如下:

   ```kotlin
   data class DataClass(var value1: String, var value2: Int)
   //这里的构造也可以填入默认值，这样在调用构造时可以使用DataClass()这样的构造方式构造一个全默认值的对象
   ```

   可见只要一行代码就可以构造完成了，但在Kotlin中，一个正常的类也是可以这样被构造并使用的，数据类有什么不一样呢？其特点在于有一个默认成员方法：`copy`，该方法可以以部分复制的方式构造出一个新的数据类对象，使用如下：

   ```kotlin
   var d: DataClass = DataClass(value1,value2,...valueN)
   var dn: DataClass = d.copy(value2 = value)//这里将value2的值改为value，其余全部复制d的值
   //注意，这里的"value2 ="是成员变量名注解，使用与部分默认构造的使用一致
   ```

   最后，由于Data Class 默认创建了`componentN() functions`，所以可以用匿名构造的方式创建对象，如：

   ```kotlin
   var d: DataClass = DataClass()
   var(value1,value2) = d //一个匿名对象，只暴露出了两个成员变量，而变量值来自d的component1、component2方法的返回,这种行为也称解构
   println(value1 + "$value2")
   ```

2. Sealed Class

   密封类是一个固定的类集合，声明标识为`sealed`。其保存类的方式类似于枚举类，所有其保存类都必须作为其子类，并且在其类实体内进行声明，多用于`when`语句的列举情况（由于完全列举可以不需要`else`分支）。下面是一个Sealed Class的声明：

   ```kotlin
   sealed class SealedClass{
       data class D(val age: Int): SealedClass()
       data class E(val age: Int): SealedClass()
       //...
   }
   ```

3. Nested Classes

   嵌套类与Java的内部类相似，但不完全一样。其写法为无修饰地在外部类中直接声明一个类（其实就是Java内部类的写法），而构造对象时不需要先构造外部类，如：

   ```kotlin
   class OuterClass{
     //...
     class NestedClass{ //嵌套类
       //...
     }
   }
   
   var n: NestedClass = OuterClass.NestedClass() //直接声明，不需要构造新的外部类
   ```

   根据前面讲过的可见性规则，嵌套类的`private`成员也不能被外部类访问到，同时其不能访问外部类的`private`和`protected`变量，即**嵌套类与其外部类没有任何从属关系，是两个同等级的类。**这里的用法基本相当于Java的静态内部类。 

4. Inner Classes

   内部类，用`inner`修饰符进行声明，它和嵌套类基本是一个类别的，但它需要依附于外部类对象，即Java内部类的声明方法：

   ```kotlin
   class OuterClass{
     //...
     inner class InnerClass{ //嵌套类
       //...
     }
   }
   
   var n = OuterClass().InnerClass() //需要通过先构造外部类来构造内部类对象
   ```

   内部类一样无法将`private`成员变量暴露给外部类，但其可以访问到外部类的`private`变量，这与Java是部分一致的。另外，内部类一样可以以**匿名内部类**的方式进行声明，此时需要用到关键字`object`，或者直接用**lambda表达式**声明：

   ```kotlin
   //用object进行声明
   v.setOnClickListener(object: View.OnClickListener(){
     //...
   })
   
   //用lambda表达式声明
   v.setOnClickListener({v ->  println("A click")}) 
   
   //用这两种方式声明的匿名内部类除了写法上的区别，其实现原理也有所不同。
   //用object形式声明的匿名内部类和内部类基本是等价的，属于一个独立的类对象，其this指向内部类自身对象，所以在引用外部类时需要用this@[类名]的方式来进行指名；
   //而lambda表达式声明的匿名内部类更像内联进当前类的代码，this指向引用类。
   ```
   这里的匿名内部类同Java一样都能访问到所在范围的局部变量，但不需要将局部变量设置为final，可进行更改。

   >Java中匿名内部类中的局部变量需要设为final的原因是：匿名内部类中用到的局部变量值实际为调用的局部变量的复制体，其生命周期与原变量不一致，故为了保持复制体和局部变量值的一致性，将其设为final，保证基础数据类型的值相同，或两个引用一直指向同一个对象。

5. Emum Classes

   枚举类用于限定类型和选项，在这里与Java基本上一样，用`emum`关键字标识。其可以进行初始化、构造匿名类、转化为数组和通过泛型取值，示例如下：

   ```kotlin
   enum class Color(val rgb: Int) {
           RED(0xFF0000), //为每个枚举对象赋值
           GREEN(0x00FF00),
           BLUE(0x0000FF)
   }
   
   enum class ProtocolState {
       WAITING {
           override fun signal() = TALKING //创建一个枚举类的匿名子类并重写其方法
       },
   
       TALKING {
           override fun signal() = WAITING
       }; //注意不同枚举对象间间隔用","  枚举变量与类成员间用";"间隔
   
       abstract fun signal(): ProtocolState
   }
   
   //假定枚举类名称为EnumClass
   EnumClass.valueOf(value: String): EnumClass //取对应名称的枚举变量 
   EnumClass.values(): Array<EnumClass> //取枚举类枚举对象构成的数组
   
   inline fun <reified T : Enum<T>> printAllValues() {
       print(enumValues<T>().joinToString { it.name }) //通过enumValues方法取得相应枚举类名T的枚举对象数组
   }
   ```

   每个枚举对象都有两个固定字段：`name`和`ordinal`，分别表示了枚举对象名和其在枚举类中的排序。其中后者的位置确定是通过枚举对象间的自然排序（都默认实现了Comparable）实现的。

___

#### 10.Generics

Kotlin中的泛型和Java的使用方式基本一致，但Kotlin中没有通配符`?`的存在，于是泛型间类型兼容就成了问题。Java中的两种通配符使用方式分别为`? extends Object` 和`? super Object`，各自代表了_producer(生产者)_和_consumer(消费者)_的角色——前者只能对集合（架设泛型是用于集合）内的数据进行操作，而不能对其进行添加或插入；后者只能对集合内的数据进行添加和插入，而不能利用其进行操作。基于Java的通配符模型，Kotlin分别为生产者泛型和消费者泛型设定了两个修饰符`out`和`in`，其用法如下：

```kotlin
//producer
abstract class Source<out T> { //等同于 <? extends T>
    abstract fun nextT(): T
}

fun demo(strs: Source<String>) {
    val objects: Source<Any> = strs // 允许由子类泛型转化为父类泛型,因为子类的子类自然就是父类的子类
    // ...
}


//consumer
abstract class Comparable<in T> {
    abstract fun compareTo(other: T): Int
}

fun demo(x: Comparable<Number>) {
    x.compareTo(1.0) //将<? super Number>泛型类实际构造为<Double>
    val y: Comparable<Double> = x // 允许由父类泛型转化为子类泛型,因为父类的父类自然也是子类的父类
}
```

这两个修饰符用在泛型中时起了**类型映射**的作用，限定了泛型某一端的可控范围。而`*`符号可以用于限定泛型两端的取值范围：

```
Function<*, String> //泛型类型必须为Nothing的父类，String的子类
Function<Int, *> //泛型类型必须为Int的父类，Any的子类
Function<*, *> //泛型类型必须为Nothing的父类，Any的子类，即任意类型也可以直接写作Function<*>
```

另外，也可以为函数设置泛型，方法与Java也基本一致，但同样在通配符这一块有所区别。函数的泛型限制不允许用`in`或`out`来做限定了，但可以通过`:`为泛型标定上界，如：

```kotlin
fun <T : Comparable<T>> sort(list: List<T>){ //设置泛型T必须为Comparable<T>的子类
  //..
}
```

也可以使用默认的上界`Any?`，而用`where`关键字进行筛选：

```kotlin
fun <T> cloneWhenGreater(list: List<T>, threshold: T): List<T>
    where T : Comparable,
          T : Cloneable {
  return list.filter { it > threshold }.map { it.clone() }
}
```

另附这一节的文档链接，里面有写了很多关于Java对泛型的处理，希望没事的时候还能去看看：[Generics][http://kotlinlang.org/docs/reference/generics.html]

> 实际上文档里是提了很多关于逆变*(contravariance)*、协变*(covariant)*和不变*(invariant)*的关系的，这里对其定义做一个解释：设**A<=B**表示A继承于B，**f()**为一种相关的类型变换（或者说如泛型、数组类型定义等实际应用），则当A<=B，f(A)<=f(B)时，称该类型变换是协变的；当A<=B，f(B)<=f(A)时称其是逆变的；当A<=B，f(A)与f(B)无继承关系时，称其是不变的。例子可以参见[Java的逆变和协变](http://www.cnblogs.com/en-heng/p/5041124.html) 。

___

#### 11.Object

`object`一词在Kotlin中与Java大有不同，在这里它的意义不再是某个类的实例，而更偏重于表达其为一个有类型或无类型的可操作实体，其等级应在类之下，但又区别于普通变量。object的使用方式有两种，即前面匿名内部类中提到的作为表达式关键字，以及作为声明的修饰词。

1. object在表达式中的使用如下：

   ```kotlin
   //作为匿名内部类的关键字
   val listener = object : ClickListener{ //这里可以并行地实现其他接口或继承类（单继承），书写方式与正常的继承一致，如 object : View(context), ClickListener{...}
     override fun onClick(){
       //...
     }
   }

   //作为匿名自定义类的关键字，相当于对Any的匿名继承实现。但这里变量的实际类型与其可见性有关。当变量为本地变量或私有变量时，其类型为匿名内部类类型；当其为public类型时，其为原接口/父类类型（有类型名时）或者Any（无类型名时）
   var o = object{
     val i = 1 //自定义变量
     //...
   }
   ```

2. object作为声明的修饰词：

   ```kotlin
   //修饰一个类，使其变成一个单例类。该类不允许有构造方法，但可以有父类。其不能定义在方法中（必须为类成员或者包级变量）
   object Singleton{
     fun test(){
       //...
     }
     var index = 1
   }
   Singleton.test() //直接通过其类名调用方法和成员变量，类似静态类

   //结合companion关键字构造静态对象
   class CompanionClass{ 
     companion object Factory{//这里的静态对象实际上还是一个真实对象的实例成员，可以继承父类或者实现接口，但其对象的构造仍然是随其外部类的加载而加载的
       fun create(): CompanionClass = CompanionClass()
     }
   }
   var c = CompanionClass.create() //companion关键字使其可以直接省略静态对象的名称调用其方法或者变量
   //事实上，也可以在声明时直接省略静态对象的名称，需要对其进行引用时只需要调其默认名称,如：
   companion object {...}
   var c = CompanionClass.Companion //这里指代object对象
   ```

   ​

___

#### 12.Delegation

在一般的继承和实现之外，Kotlin还有一种本地实现**代理模式**的方式，即使用关键字`by`将接口的已有方法和成员变量实现复制到当前类中(相当于一个对原始接口的显式继承，而忽略代理类名，但实际继承了代理类的实现)，**只能通过该方法依赖一个代理类。**其使用方式如下：

```kotlin
//原接口
interface Ki{
    fun onClick()

    fun onMove()
}
//接口实现类
class KiImpl : Ki{
    override fun onMove() {
         println("onMove")
    }

    override fun onClick() {
         println("onClick")
    }
}
//接口实现代理类
class Derived(k: KiImpl) :  Ki by k{
	//这里对已实现的接口方法进行了重写
    override fun onClick(){
        println("Down and Up")
    }
}

var k = KiImpl()
var d = Derived(k)
d.onClick() // 输出Down and Up, 这里调用了代理类自身重写的操作
d.onMove() // 输出onMove, 这里调用了被代理类的操作
```

除了对接口实现的代理之外，Kotlin还支持对类成员变量的代理，其代理方式为赋予当前变量相应的getter和setter实现，如：

```kotlin
class Client{
  var value: String by Delegate()
}

class Delegate{ //下面两个方法可通过实现ReadWriteProperty<Client,String>来重写，而不需要自己构造
  operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
  		//thisRef 为被代理变量的引用，这里为Client的引用
  		//这里的property包括变量的各种属性，包括变量名、是否为open、是否为const，可见性等等
        //... 
    }
 
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
    	//这里的value为赋值语句中被传入的值
        //...
    }
}
```

Kotlin将一些常用的变量赋值模式封装成了类库，可以直接用作代理，主要有下面几种：

```kotlin
1.lazy()
	该类代理的特点是只有第一次取变量值时会进行计算（计算方法由使用者传入），之后每次取值都直接用第一次取的值（类似单例的构造方式）。其声明示例如下：
	var i: Int by lazy{
      //...初始化操作，实际为简化版的lambda表达式，取最后一个表达式返回值为返回值，不允许使用return
	}
在默认情况下lazy代理是同步的，但也可以传入LazyThreadSafetyMode.NONE等构造参数来改变其同步属性。

2.observable()
	该类代理可以在变量值发生变化时（值已经完成了改变）触发设定的回调，需要在构造时传入初始值。声明示例如下：
	var i: Int by Delegates.observable(0){ //这里的0是传入的初始化值
      prop,old,new -> //未经省略的lambda表达式，prop为原变量属性，old为改变前的旧值，new为改变后的值
      //... 这里是回调的具体语句
	}
3.vetoable()
	其构造与observable类似，但在变量值改变前被调用回调，并且需要返回一个Boolean型值，true表示允许这次值的改变，false表示保留原值。
```

另外，还可以直接将变量通过传入一个图来对应赋值，如：

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int     by map
    val sex: String  by map
}
var user = User(mapOf(
	"name" to "Lazxy",
	"age"  to "21"
	//这里没有为sex变量赋值，如果之后调用该变量会报NoSuchElementException，不过不调用时就能正常运行
))
```

最后，还可以通过operate fun provideDelegate()来构造ReadOnlyProperty或者ReadWriteProperty，然后调用其所在类作为`by`的右方对象。