### Groovy简单笔记

>   也不知道为什么看着看着就看到Groovy来了，反正看了就记点东西吧。

​	Groovy是Java平台上设计的面向对象编程语言。这门动态语言拥有类似Python、Ruby和Smalltalk中的一些特性，可以作为Java平台的脚本语言使用。Groovy的语法与Java非常相似，以至于多数的Java代码也是正确的Groovy代码。Groovy代码动态的被编译器转换成Java字节码。由于其运行在JVM上的特性，Groovy可以使用其他Java语言编写的库。——来自Wiki

#### 快速入门

1.  Groovy的bean不需要写getter和setter方法，只需要定义**无访问修饰符**的成员变量，编译器会隐式地提供默认的getter和setter方法，当然也允许对其进行自定义。

2.  定义变量用到的关键字是`def` ，此时只需要声明变量名，编译器可以根据上下文推断变量的类型。

3.  允许用`$` 来在字符串中表示对变量的引用。

4.  Groovy声明断言时允许不带或者带一或多个参数，参数的默认引用为`it` ，其写法类似：

    ```groovy
    def niceHello = {firstName, lastName -> "Hello, $firstName $lastName!"} //声明断言
    assert niceHello('Chris', 'Bennett') == 'Hello, Chris Bennett!' //验证断言
    ```

5.  Groovy的List基本上和数组差不多，包括一些被简化的方法，如：

    ```groovy
    //get a range of elements
    list[1..2] == ['Two', 'Three'] //这里..表示一个连续范围

    // iterates the list
    def emptyList = []
    list.each {emptyList << "$it!"}
    assert emptyList == ['3!', '1!', '2!']

    // iterates the list and transforms each entry into a new value
    // using the closure
    assert list.collect {it * 2} == [6, 2, 4]

    // sorts using the closure as a comparator
    assert list.sort {it1, it2 -> it1 <=> it2} == [1, 2, 3]

    // gets min or max using closure as comparator
    assert list.min {it1, it2 -> it1 <=> it2} == 1
    ```

    map。。其实也就是一般的键值对，用起来和Java的Map没什么不同，就是风格更简洁了：

    ```groovy
    // predefined map
    def map = [John: 10, Mark: 20, Peter: 'Not defined']

    //bean型用法
    def map = [John: 10, Mark: 20, Peter: 'Not defined']

    // the array style
    assert map['Peter'] == 'Not defined'

    // the bean style
    assert map.Mark == 20

    // also you can preset default value that will be returned by
    // the get method if key does not exist
    assert map.get('Michael', 100) == 100

    // 键值对用法
    def emptyMap = [:]
    def map = [John: 10, Mark: 20, Peter: 'Not defined']
    map.each { key, value ->
        emptyMap.put key, "$key: $value" as String
    }
    assert emptyMap == [John: 'John: 10', Mark: 'Mark: 20', 
            Peter: 'Peter: Not defined']

    // iterates the map and transforms each entry using
    // the closure, returns a list of transformed values
    assert map.collect { key, value ->
        "$key: $value"
    } == ['John: 10', 'Mark: 20', 'Peter: Not defined']

    // sorts map elements using the closure as a comparator
    map.put 'Chris', 15 //连括号都没有了，真是丧病
    assert map.sort { e1, e2 ->
        e1.key <=> e2.key
    } == [John: 10, Chris: 15, Mark: 20, Peter: 'Not defined']
    ```

6.  Groovy里的`==`等于Java里的`equals`，然后`is()` 相当于Java里的`==` 。

7.  Groovy的方法可以没有返回语句，直接把返回值写在最后就行。