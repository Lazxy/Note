## LearnForRust

### 随便整理些零星的点

- `cargo build` 编译项目，`cargo run` 编译并运行，`cargo check` 校验性编译， `cargo fmt` 格式化代码

- `let`、`const`、`let mut`，可以顾名思义。字段的遮蔽特性和 Kotlin 基本差不多。

- 元组（`truple`）和其解构过程和 Kotlin 也很类似，如果一个元组没有任何值，那就相当于 Kotlin 中的 Unit。

  ```rust
  let x:(i32, f64, u8) = (500, 6.4, 1);
  let unit = ();
  ```

- 数组的写法：

  ```
  // 声明类型和数量，标准写法
  let a:[i32; 5] = [1, 2, 3, 4, 5];
  // 推断类型
  let a = ["a", "b", "c"]
  // 初始化相同的值，左边是初始化值，右边是数量，所以这里是创建一个 [3, 3, 3, 3, 3] 数组
  let a = [3; 5]
  ```

- 赋值语句在 Rust 中不视为表达式，不会返回值，和 Kotlin 一样。当表达式作为函数的返回值时，末尾不带 `;`，否则会被视为是一个普通语句。

  ```
  fn getCount() -> u32 {
  	let x = 4;
  	x + 1 // 如果这里有分号就相当于默认返回 unit value 
  }
  ```

- `if`、`while`、`for` 的用法都和 Kotlin 差不多，不过表达式都不带括号

- 所有权规则：

  1. 一个**非原生类型实体对象**，在同一时刻只存在一个所有者（其实形式类似于其他语言中的引用量，但 Rust 的引用概念其实更接近 C 的指针），从一个所有者转移到另一个所有者的过程叫`move`，一般有两种形式：1. 直接在语句中进行赋值 2. 对象进入函数，有形参到实参的转换。

     **Rust 不存在浅拷贝。**

  2. 如果需要深拷贝，需要调用 `clone` 方法

  3. 对于原生类型，由于其数据都存储在栈上，所以不受`move` 的影响，每次引用赋值都不会使上一个所有者消失。

- 引用规则：

  1. 可以通过 `&` 声明一个引用，该引用不允许调用可更改对象的方法，这样引用的过程叫做**借用（borrowing）**

  2. `&mut`可以用来声明一个可变引用，但这个引用在同一作用域中只能存在一个，并且其与不可变引用的存在也互斥。

     > 可以认为这个机制是编译时生效的**读写锁**机制：允许同时读（不可变引用），但是写操作（可变引用）时不允许读，也不允许其他引用写。

  3. 一个引用不能离开其引用对象所有者的作用域（否则就会变成垂悬引用 *Dangling References*），该情况下编译会报错。

  4. 可以通过切片（slice）构造一个数组（包括 String，视为字节数组）的部分引用，通过 `&obj[a..b]`的方式进行表示。（类似 subString 操作，不过注意 a 和 b 都是包括的）

- 结构体主要有以下几种写法

  - 常规定义：

    ```
    struct User{
    	name: String,
    	age: u16,
    }
    
    // 加 mut 表示结构体数据可变，否则结构体所有字段都不可变
    let mut user = User{
    	name: String::from("Lazxy"),
    	age: 28,
    };
    ```

  - 变量名与字段名同名时的简写方式：

    ```
    fn build_user(name: String, age: u16) -> User{
    	User(name, age)
    }
    ```

  - 从其他结构体复制值：（*？？如果结构体里面的字段是对象，那么复制时复制的是引用还是内容？内容如何复制？*）

    ```
    // 这里继承了上面的 User 的 age 字段，新的结构体只更新了 name 字段
    let user2 = User{
    	name: String::from("Wcl"),
    	..user
    	};
    }
    ```

  - 元组结构体（*tuple struct*）：

    ```
    // 只定义类型，不定义名称，取值时按照数组取值的方式
    struct Color(i32, i32, i32)
    struct Point(i32, i32, i32)
    
    fn main(){
    	let black = Color(0, 0, 0)
    	let origin = Point(0, 0, 0)
    }
    ```

  - 类单元结构体（*unit-like structs*）

    ```
    // 只有名称没有字段，用来作为泛型标识
    struct TypeA
    
    // ...
    let subject = TypeA;
    ```

- 可以通过 `dbg!`宏来打印符合结构体声明派生的 trait 的内容，输出结果带文件路径和行号。一个常用 trait 是 Debug，使用如下：

  ```rust
  #[derive(Debug)]
  struct Person {
  	name: String,
  	age: u16,
  }
  
  let me = Person{
  	name: String::from("Lazxy"),
  	age: 28,
  }
  println!("I'm {:?}", me) // 结果会为 I'm Person { name: Lazxy, age: 28 }
  println!("I'm {:#?}", me) // 结果会为 I'm Person{
                                          name: Lazxy,
                                          age: 28,
  									}
  dbg!(me.age + 1) // 结果会为 [文件路径/行号] me.age + 1 = 29
  ```

  注意 dbg! 会获取输入参数的控制权，并且在方法执行结束后返还，所以 dbg! 能包裹任何语句，而不影响其执行。

- 使用和 `struct`关键字并列的`impl`关键字，可以在其代码块里声明方法（*method*，与函数 *function* 相区分），使用起来和 Kotlin 没有什么区别，但要注意其和 JS 一样关注 `this`——即结构体对象本身的实例引用，在 Rust 里被声明为 `self`。

  如果需要在结构体方法里用到本身实例对象，就需要在方法里第一个参数声明一个 self 参数，实际是 `self:Self` 的简写形式。根据使用所有权的不同，可以将其定义为不可变引用、可变引用或者干脆传入结构体对象本身进行所有权转移。

  ```rust
  impl Person{
  	fn birthday(&self){
  		println!("{}'s birthday is today.'", self.name)
  	}
  }
  
  let me = Person{ 
  	//...
  };
  me.birthday();
  ```

  > 在 C++ 里，类方法的调用有 `.` 和 `->`两个符号，分别表示对象直接调用方法和类对象的指针调用方法，但 Rust 里有自动引用和解引用机制，因为引用的对象确定，有效性也确定，所以不需要区分，通过对象引用或对象本身来调用方法是一样的。

- 如果 `impl`代码块的方法没有 `self`字段，那它就不叫做方法（*method*），而是关联函数（*associated function*），调用时需要通过命名空间调用，即`::`语法，如：

  ```rust
  impl Person {
  	fn cry() -> String{
  		"wuwuwu~~"
  	}
  }
  
  let voice = Person::cry();
  ```

  > 与自身对象无关的函数，实际可以和 Kotlin 的 companion object 一样，视为静态函数。

- 允许通过多个 `impl`代码块为同一个结构体声明方法。 

- Rust 的枚举允许枚举的实际关联值是不同类型，类似一个有关联关系的类集合。

- `Option`枚举的设计思路和 Kotlin 的空安全很类似，如果一个值不放入 Option 类型，那么就会被认为是非空类型，否则，在使用前需要做空判断。

- `match`关键字类似 Kotlin 的 `switch`关键字，允许输入任意类型进行。如果是枚举类型，枚举的关联值可以被直接获取（**这似乎也是唯一一种能直接取到枚举关联值的方法**）。和 Kotlin 一样，这里的匹配项是需要有穷尽的，`other` 在这里等效于 Kotlin 的`else`，如果对于匹配的值无用，可以直接省略为 `_`。

- `if let` 可以用来简化只匹配单一项的`match`场景，这里的重点似乎不是`if`，而是`let`。实际上如果不要被比较的值，直接用 if 也是能完成一样的匹配的，但是如果要对应枚举的关联值，就需要通过这种方式用 let 将关联值赋给某个临时变量。

  ```rust
  #[derive(Debug)]
  enum Car {
      Fast(u8),
      Slow(String),
      Medium(u32),
  }
  
  let xiaomi = Car::Fast(158);
  let huawei = Car::Medium(34456);
  
  // 这里除了匹配，主要是为了拿 max 的值
  if let Car::Fast(max) = huawei {
        println!("max is {}", max)
    }else{
        println!("Not xiaomi, it's {:?}", huawei)
  }
  ```

- Rust 的构建方式是以 `library crate` 或者 `binary crate` 作为 *root crate* 出发，根据根节点中 `mod`关键字声明的模块范围，以树的形式依次构建的。*root crate* 文件的存在类似于 C 中的头文件，声明了模块中用到的函数、常量或者结构体定义。

  这里有几种不同的目录索引方式：

  - 全部放在一个文件里。最简单的方式，没有文件索引过程，直接根据代码的嵌套形式进行模块分级。

    > 结构为 libs.rs -> mod A { mod B {mod C fn C} struct B }

  - 在不同的文件分别声明和定义模块，与引用 mod 的文件存在于相同的目录。这种情况下引用 mod 声明的文件同级目录下需要有一个与 mod 名相同的`.rs`文件，编译器会根据这种同名关系找到对应 mod 的定义文件。

    > 结构为 src/libs.rs -> use mod A
    >
    > src/A.rs -> pub mod A

  - 在不同的文件中分别声明和定义模块，与引用 mod 的文件存在于不同的目录。这种情况下引用 mod 声明的文件同级目录下需要有一个与 mod 同名的目录作为中间 mod，目录下存在一个 mod.rs 文件，其内按照上一条规则声明并定义相应的 mod，该 mod 可能与目录同名，但和目录 mod 是两个 mod。

    >结构为 src/libs.rs -> use mod alphabeta::A  // 引入对应声明
    >
    >src/alphabeta/mod.rs -> pub mod A // 跨目录的 mod 声明
    >
    >src/alphabeta/A.rs -> pub fn a() // 实际 mod 定义

  > *？？library crate 和 binary crate，除了名字一个叫 lib.rs，一个叫 main.rs ，还有什么其他区别呢？*
  >
  > 暂时的答案：一般认为 main.rs 负责参数接收以及 lib 中函数的功能调用（用于 cli），lib.rs 负责功能实现类的模块区分和实现。

- 路径访问规则：

  - 能够用绝对路径或者相对路径的方式访问某个模块中定义的方法，类似于文件路径，不过间隔符为`::`。
  - 或者如果需要访问的方法在上一级模块时，可以使用`super`关键字作为访问相对路径的根，直接写为`super::XXX()`
  - 可以使用 `use` 关键字，引入整个模块的绝对路径，类似 Kotlin 的 `import`

  

- 可见性规则：

  - 默认情况下，Rust 中的所有项都是私有的，子模块实现对父模块不可见，但是反之可以（类似接口和实现类的关系）。
  - 如果需要使某一项是公有可见的，那么包括该项在内的每一级模块或项，定义时都需要以 `pub` 关键字进行修饰。
  - 结构体中的字段默认也是对其他模块不可用的，除非在每个字段前加 `pub` 关键字。
  - 枚举的所有字段，只要枚举本身是 `pub` 修饰的，那么其所有枚举项也是公开可见的。

- 当使用集合 `vector`时，可以通过同时使用 `&`和`[]`获取对其元素的引用，或者通过 `get`获取一个 `Option`值，进行手动判空处理。

  需要注意的是，当获取到其中一个元素的引用时，就无法通过获取可变引用改变 vector 的内容，因为这个过程中可能会由于内存重新分配导致其元素引用失效。**所以获取 vector 元素的引用就等同于获取了 vector 本身的引用，其借用规则限制等效。**

- String 的 `+` 操作会夺取左边参数的所有权，因为其实现的 `add` 方法里传入的是对象本身而非引用。需要拼接时可以考虑使用 `format!`宏，这个调用不会获取参数的所有权。

  ```
  let s1 = "1".to_string();
  let s2 = String::from("2");
  let s3 = "3".to_string();
  let s = format!("{}-{}-{}", s1, s2, s3)
  ```

- HashMap 的插入值方法是 `insert` 而非 `put`，对应对象进入 HashMap 后，所有权就会失去，不可使用。如果放入的是对象的引用，那么就会有有效性检查， 在 Map 失效之前，这些对象都不允许失效。

  同 vector 类似的，通过 `get` 方法取到的表内值会是一个 `Option` 值，需要手动判空，且除了使用 for 循环遍历之外，没有其他能直接获取值的方法。

  也可以通过 `entry()` 方法获取到键值对项，然后以 `or_insert()` 方法获得当前值或插入默认值。

- 可以通过 `panic!` 宏抛出一个异常。可能带异常抛出的方法应该返回一个 `Result` 对象，可以通过 match 语句判断其是正常还是异常，让上层决定是否终止程序。

  返回 Result 不需要构造具体对象，直接返回 `Ok` 包裹目标正确值或者返回一个 `Err` 对象即可。如果本身内部调用的方法就会返回 Result ，可以直接在表达式后加 `?` 来在错误的时候直接向上传递错误。

  ```
  fn getError() -> Result<String, io::Error> {
  	// match 写法。于是最后一句表达式，不需要 return 关键字，自动返回 match 得到的值
  	match fs::read_to_string("hello.text"){
  		Ok(content) => content,
  		Err(error) => Err(error)
  	}
  	
  	// ? 写法。返回内容值，或者向上传递 Err
  	fs::read_to_string("hello.text")?
  }
  ```

  直接调用 `unwrap()` 方法可以在获得 Result.Err 时直接抛出异常，`expect`也能达到一样的效果，但是可以自定义异常的错误信息。

  > 但主要是一种简略写法，使程序不用处理一些不太可能发生的异常，类似 Kotlin 中的 `!!`

- Rust 的泛型基本使用和 Java 没太大区别（不包括逆变谐变之类有继承关系的语法），其编译期实现进行了名为单态化（`monomorphization`）的操作，本质上就是编译时根据代码引用填充对应泛型方法里的泛型类型使其变为具体类型，再用新方法的引用替换旧泛型方法。

  泛型可以有默认类型参数，只要在声明泛型时用`<T=[Type]>`的方式书写即可。

- `trait` 可以认为是 Rust 中的接口，不过其声明方式更接近于拓展函数或者某种具有泛型参数的方法， 而参数类型取决于它被什么类实现。

  trait 可以有默认实现。

  ```
  pub trait Closable{
  	fn close($self) -> String
  }
  
  pub struct A {
  	//... 
  }
  
  // for 关键字类似于 implementation 不过是倒过来的
  impl Closable for A {
  	fn close(&self) -> String {
  		// ...
  	}
  }
  
  let a = A {
  	// ...
  }
  // 结构体 A 于是就有了 Closable 的能力
  a.close()
  
  // 可以将实现了 trait 的对象传入
  pub fn finish(item: impl Closeable){
  	//...
  }
  
  // 也可以这样写
  pub fn finish<T: Closeable>(item: T)
  ```

  需要注意的是，由于 trait 实现的声明和结构体的声明是可以不在一起的，**所以只允许对应结构体的 crate 内为结构体添加 trait 实现（孤儿原则）**，否则就可能出现同一结构体多 trait 实现的问题（和 kotlin 拓展函数的覆盖实现不一样。）

  另一外一点需要注意，同一结构体是可以实现同一 trait 的多个不同泛型类型的，即下面的代码可以编译通过：

  ``` rust
  pub trait Generate<T>{
  	fn generate() -> T
  }
  
  struct Generator{
  	//...
  }
  
  impl Generate<String> for Generator{
  	fn generate() -> String{
          //...
      }
  }
  
  impl Generate<i32> for Generator{
  	fn generate() -> i32{
          //...
      }
  }
  ```

  在不需要有多类型泛型实现的情况下，可以用**关联类型**进行简化：

  ```
  pub trait Generate{
  	type Item;
  	fn generate() -> Self::Item
  }
  
  impl Generate for Generator{
  	type Item = String;
  	fn generate() -> Self::Item {
  		//...
  	}
  }

- 当限定某结构体需要实现多个 trait 时，可以通过`+`组合 trait 类型，还有`where`关键字可以在函数签名后声明 trait 的组合类型：

  ```
  fn some_function<T, U>(t: T, u: U) -> i32
      where T: Display + Clone,
            U: Clone + Debug
  {}
  ```

- 当返回值是 trait 类型时，需要用 `impl XX` 的语法声明返回值类型。

- 上面的 trait 类型表示方式，都可以拿来作为泛型的类型声明。

  > 不过声明的范围，如果类比 Java 或者 Kotlin 的话，只有 extends 或 out 的部分，没有 super 或 in 的部分，可能后者真的不常用吧。

- 当一个函数的返回值是一个引用，且该引用与作为函数入参的引用相关时，需要通过泛型标注对应引用的生命周期：

  ```
  // 'a 就是对应的生命周期标注，a 可以用任意字母替换。此处的标注含义为，取参数 x 和 y 的生命周期交——即两者中生命周期更短者的生命周期，作为返回值的生命周期
  fn longest<'a>(x: &'a str, y: &'a str) -> &'a str{
   	if(x.len() > y.len()){
   		x
   	}else{
   		y
   	}
  }
  
  let x = "ABCDE"
  let result;
  {
  	let y = "FG"
  	result = longest(x, y)
  }
  // 这里无法调用成功，因为 result 可能是 x 的引用，也可能是 y 的引用，后者在此处就已经被释放。生命周期就明确标注了这一点。
  println!(result)

- 类似的，当一个结构体需要以一个引用作为成员时，也需要通过泛型声明对应的生命周期标注，结构体的生命周期一定要比作为其成员的引用更短。

  在这种情况下，对该结构体的方法声明需要带生命周期标注：

  ``` 
  struct A<'a>{
  	part:&'a str,
  }
  // 这里要附带生命周期标注
  impl<'a> A<'a>{
    	// ...
  }

- 生命周期的标注有时可以省略，其省略行为可以遵循三条规则对方法签名修改校验得到：

  1. 输入生命周期规则：每一个作为参数的引用都有其自己的生命周期。（无论实际上是不是这样的）
  2. 输出生命周期规则之一：如果只有一个输入生命周期参数，则输出生命周期参数的生命周期与其一致。（只要多个引用参数的情况，那么该条规则就会不生效）
  3. 输出生命周期规则之二：如果输入生命周期参数中包括 self 参数，即该方法是一个结构体对象的内部方法，则其输出生命周期参数的生命周期与结构体本身一致。

  当输入参数的生命周期经过规则一的修改后，能够通过规则二或者规则三得到输出生命周期参数的生命周期时，就可以省略对应的生命周期标注。

- `'static` 标记可以指定静态生命周期，使对象引用在整个程序执行期间都可用。

- 可以通过 `env::args.collect()`获取一个命令行参数的集合，集合的第一个参数是函数入口的代码路径。

- Rust 的闭包形式类似于 Java 的 Lambda 表达式或者 Kotlin 的方法参数，声明方式如下：

  ```rust
  let closure_obj = |num, string| -> {
  	println!("It's a closure")
  	num
  };
  ```

  闭包的入参和返回值的类型声明是可以省略的，因为其使用场景和上下文关系，编译器可以靠推断得出类型。但其推断只会进行一次，后续对于同个闭包就会进行类型锁定，所以闭包的入参和返回值不会是动态类型。

- 闭包可以视为 `Fn`、`FnMut`或`FnOnce`等 trait 中某一个的实现，所以可以在结构体中定义上述类型的闭包成员变量，如：

  ```
  struct <T: Fn(u32) -> u32> Cacher<T>{
  	calculation: T,
  	value: Option<u32>,
  }
  ```

  这样就可以在构造 Cacher 时通过传入一个闭包，行程能够存储的闭包实现和缓存值。

- 闭包作为作用域有限的代码块，允许捕获其同一作用域（就是其上下文代码）声明的变量，而捕获这些变量的方式与函数入参的分类类似，也有转移所有权、可变借用、不可变借用这几种，依次对应 `FnOnce`、`FnMut`和`Fn`类型。

- Rust 的迭代器在编译成汇编后，性能基本与其手写逐个取值版本的代码一样快（在集合长度固定的情况下，其实就是手写代码），是 Rust 的**零成本抽象**（*zero-cost abstractions*）之一。

- Rust 的 `opt-level` 配置由 0-3 表示由低到高的编译优化级别

- Rust 用 `///`表示文档注释，`//!`用来做文档注释的注释。可以用 `cargo doc` 直接解析对应的注释内容生成对应的文档页面。

- 智能指针在 Rust 中是一类**拥有**其指向数据的，带有元数据和额外功能的结构体，与普通结构体的区别主要在于其实现了  `Deref` 和 `Drop` trait。

- 递归类型（*recursive type*）是一种成员的一部分值类型可以是其本身类型的数据结构类型，其典型的例子是一个 `cons list`，即一个成员包括一个值和另一个本类型对象的数据结构，在 Java 中，这样的数据结构的代表就是链表节点（存有当前节点的值和下一个节点的引用。）

  递归类型是无法计算大小的（因为理论上其对象引用链条是无穷无尽的），此时就需要将其对象引用封装为 `Box<T>` 类型的智能指针，使其大小固定（将不确定性转移到了指针的引用上）。

  > **将数据类型大小固定，就是智能指针的主要意义。**

- Rust 中的解引用运算符（*dereference operator*）`*` 实现的功能类似于 C 的指针解运算符，能够通过一个引用获得引用对象的值。通过实现 `Deref trait`可以使结构体具备被解引用的能力，向外提供一个内部数据的引用（即实际被解引用的对象）

- 对应于解引用操作的，Rust 提供另一个语法特性，称为解引用强制转换（*deref coercions*），即对于一个实现了 Deref 类型的结构体，可以通过 `type Target =`声明其关联类型，然后在解引用时直接提供关联类型的引用而非自身成员变量类型的引用，如 String 的实现：

  ```rust
  #[stable(feature = "rust1", since = "1.0.0")]
  impl ops::Deref for String {
      // String 在被解引用时会直接提供 str 的引用
      type Target = str;
  
      #[inline]
      fn deref(&self) -> &str {
          unsafe { str::from_utf8_unchecked(&self.vec) }
      }
  }
  ```

  由于强制转换过程的存在，就省去了对 &String 解引用、生成 slice、再生成该 slice 的引用的过程，直接得到 &str 引用。

  如果需要得到一个可变引用，可以实现 DerefMut trait，其方法实现和 Deref 类似。

  解引用强制转换允许一个可变引用解引用时提供一个不可变引用进行实际解引用，而不允许反向操作（因为当可变引用存在时，其对象不会存在其他引用，所以转成不可变引用也不会发生冲突；反之不能保证转变为可变引用时没有其他引用存在）。

- Drop trait 定义了指针在离开作用域时释放其指向的内存的行为，即 `drop`方法，类似于 C++ 中的析构函数。由于 Rust 会自动添加 drop 的调用，所以显式调用智能指针的 drop 方法是不允许的，但可以通过 `std:mem:drop`函数调用来触发智能指针的提前释放（编译器会将这个调用考虑在内，所以不会重复释放内存）。

- 当一个对象需要被多个指针指向时，可以用 `Rc<T>`进行包装，通过`Rc::clone(&value)`的形式复制对应的引用。由于其实际是引用复制，所以 clone 的过程实际不会进行数据的深拷贝，不会有性能损失。

  注意由于允许多引用，所以 Rc 只能是以**不可变**的方式指向对应的对象。

  Rc 指针指向对象回收的时机在 Rc 的引用计数归 0 以后（类似于 Java 的 GC 原理）。

- RefCell\<T\>  智能指针可以允许动态获取其指向对象的可变或不可变引用，而不受指针本身可变性的性质（如果是结构体，不可变引用指向的结构体对象，其内成员也是不可变的）。

  其功能相当于中断了 Rust 基础语法保持的内外部引用一致性，**即允许外部对象不可变，而内部成员变量可变**，其实更接近于 Java 的使用方式。

  这里的可变引用获取也是要遵循 Rust 的借用规则的，如果同时获取多个可变引用对象或者可变引用和不可变引用同时存在，会直接触发 panic。

- 如果同时使用 Rc 允许多引用、又使用 RefCell 允许可变性修改，可能导致循环引用，让代码离开作用域之后，由于引用计数不能归零，导致内存泄漏。

  所以可能导致引用循环的双向引用场景，可以用 `Rc::downgrade` 生成一个 `Weak<T>`引用，当需要获取引用值的时候，使用`Weak::upgrade()`尝试获取原引用，如果此时还存在对应对象的强引用，就可以获取成功，否则会得到一个 None。弱引用本身不影响对象被内存回收。

- Rust 为了减少运行时代码，更好地同 C 兼容，保持高性能，线程模型是 OS 线程 ： 语言线程 = 1:1

- 可以通过 `thread::spawn`方法传入一个闭包作为线程体，开启一个线程并执行。该方法存在一个`JoinHandle`返回值，通过调用返回值的 `join` 方法，能够阻塞等待对应线程执行完毕。如：

  ```rust
  fn main() {
      let handle = thread::spawn(|| {
          for i in 1..10 {
              println!("hi number {} from the spawned thread!", i);
              thread::sleep(Duration::from_millis(1));
          }
      });
  	// 这里会等前面子线程结束才执行下面的代码，如果不进行干预，可能子线程还没有执行完，主线程执行完毕，就会连带子线程一同终止
      handle.join().unwrap();
  	
      for i in 1..5 {
          println!("hi number {} from the main thread!", i);
          thread::sleep(Duration::from_millis(1));
      }
  }
  ```

  如果需要在线程体闭包中使用上下文的引用，需要在闭包声明前加上 `move` 关键字，这样在线程体里用到的上下文对象所有权就会被转移入闭包内。

- 线程间通信的一种方式是使用 `mpsc::channel()` 构造一个进程间通道，该通道会返回 tx/rx 两个 vector，两个不同的线程可以分别持有它们之一，通过 `send` 和 `recv` 方法分别发送和获取对应的值。

  需要注意，当一个值被发送到另一个线程后，其所有权就会移交给另一个线程。

  channel 的生产者-消费者模式是多对一的，允许有多个生产者向消费者发送消息，可以通过 `tx.clone()` 复制多个生产者。

- 使用 `Mutex<T>`可以得到一个带锁的线程安全变量引用。每次使用对应的值时，需要通过 `lock()`获取锁并返回对应的引用值，并在离开 lock 返回值的作用域后自动释放锁。

  Mutex 本质上是一个智能指针，并且允许像 RefCell 一样突破可变性规则。

  如果一个持有锁的子线程产生了 panic，那么所有用到这个锁的线程在获取锁时都会 panic。

  如果要在多个线程间分享锁，需要用 `Arc<T>`进行引用计数持有，它基本上就是一个支持并发版本的 `Rc<T>`。

- Rust 中语言层面内嵌支持的并发概念有两个：

  1. `Send`，标记一个类型的**所有权**是允许线程间传递的。大部分类型都有该标记，只有少数特殊功能的类型，比如 `Rc<T>` 是不允许直接使用在多线程的。
  2. `Sync`，标记一个类型可以在多线程中安全持有其**引用**。与 Send 的定义很类似，区别在于主体。`Rc<T>`、`Cell<T>`一类的类型都是非 Sync 的。

  需要注意的是，这两个标记和 Java 中的可序列化标记类似，是具有传递性的。当一个结构体的所有成员变量都是 `Send`标记的时，那么该结构体本身也是 `Send` 的。

- Rust 在面向对象特性上，唯一不支持的是继承特性，或者说，只有接口继承而不能实现类继承。

  Rust 支持的部分多态特性基于**接口对象（事实上叫 trait object）**，一个接口对象的类型声明为 `dyn [traitName]`。

  能声明为接口对象的 trait 类型存在限制：即只有**对象安全**的 trait 才能组成 trait 对象。对于 trait 的方法，对象安全的要求有两条：

  - 返回值类型不为 `Self` （所以 Clone 这类 trait 就不是对象安全的）
  - 方法没有任何泛型参数

  可以看出，上述要求的目的，是**保证运行时能够通过 trait 实现完全推断出方法签名的具体类型**。

- let、match、for 相关的一些奇奇怪怪的语法，可以参考 https://rustwiki.org/zh-CN/book/ch18-03-pattern-syntax.html 这一章节。

- 裸指针（*raw pointers*）指没有任何附加功能的，指向对象的指针，实现类似于 C 的指针，不保证内存安全、没有可变不可变引用的数量限制，不会自动释放。声明方式为：

  ``` rust
  let mut num = 5;
  // 不可变指针，指向引用
  let r1 = &num as *const i32;
  // 可变指针，指向引用
  let r2 = &mut num as *mut i32;
  
  let address = 0x012345usize;
  // 不可变指针，直接指向某个地址
  let r = address as *const i32;
  ```

  解引用裸指针时，需要在 unsafe 块内操作：

  ```rust
  unsafe{
  	println!("r1 is: {}", *r1);
      println!("r2 is: {}", *r2);
  }

- 不安全函数是加了`unsafe` 关键字的函数，其函数体就是一个有效的 unsafe 代码块，可以集中在其内进行不安全操作。调用不安全函数也需要在 unsafe 代码块内执行。

  类似的，可以用 `unsafe` 来标记 trait。

  > 但 unsafe trait 到底用在哪呢？

- 可以通过 `extern` 关键字声明对其他语言函数的依赖（类似 C++ 依赖 C ），调用这些函数需要在 unsafe 块中执行。

  ```rust
  extern "C" {
      fn abs(input: i32) -> i32;
  }
  
  fn main() {
      unsafe {
          println!("Absolute value of -3 according to C: {}", abs(-3));
      }
  }
  ```

  如果需要导出接口给其他语言调用，也可以通过 extern 声明，不过要额外标注 `#[no_mangle]` 以免编译器进行了名称混淆。示例如下：

  ```rust
  #[no_mangle]
  pub extern "C" fn call_from_c() {
      println!("Just called a Rust function from C!");
  }

- 可以用 `static`或者`static mut`关键字来声明全局静态变量。使用**可变**静态变量需要在 unsafe 代码块内进行。

- Rust 允许一个结构体实现多个 trait 的同名方法，包括结构体自身也能声明对应的同名方法，在这种情况下，结构体自身的方法声明优先，对于接口的方法调用：

  - 如果是成员方法，需要用 `[traitName]::[methoName](&[obj])`的语法进行调用，可以根据参数类型自动推断调用者
  - 如果是关联函数，由于无法推断类型需要使用完全限定语法`<[Type] as [Traint]>::[functionName]()`

- trait 能够通过依赖 supertrait 调用其他 trait 的方法，语法类似：

  ```rust
  pub trait View: Display{
  	//...
  }
  ```

  当一个类实现 View 这个 trait 时，也要同时实现 Display 这个 trait，否则会编译错误。

- 如果需要代理一个不在 crate 范围的类型，可以用新建类型，目标类型作为成员变量的方式进行封装，然后用 `self.[成员变量角标]`的形式进行调用目标类型对象。但这样做需要手动在代理类型内声明需要用到的目标类型方法。

- 可以用 `type` 关键字声明类型的别名

- Rust 中有一个特殊类型 `!`，称作 `nerver type`，在函数从不返回时作为返回值。从不返回的函数称为发散函数（*diverging functions*）。

  这种类型的作用在于，需要时它可以强转为其他任何类型（因为没有值，所以不用值校验），在 match 语句中，例如 `continue`、`panic!`这样的语句，实际就是 ! 类型，将它们放在语句中不会因为类型校验报错。

- str 和 trait 这一类只能在运行时确定占用空间大小的类型称作**动态大小类型**（*dynamically sized types*），他们使用时需要某种指针进行引用，而不能直接作为变量。

  大小在编译时可知的类型实际实现了 `Size` trait，当需要定义一个编译时大小不可知的泛型时，可以使用`<T: ?Sized>`语法进行泛型声明。

- 可以用 `fn(paramType) -> returnType`声明函数类型的参数，`fn` 被称为函数指针，它是一种类型，而不像闭包是一种`Fn trait` 的实现。一般它和闭包是能互相代替的，但在像与 C 语言这种没有闭包的代码交互时，需要用函数指针而不是闭包。

  函数的返回值也可以是闭包，但由于闭包是动态大小类型（trait 对象），所以需要用指针进行封装。



- cargo 下载的库缓存在 `D:\Rust\.cargo\registry\src\index.crates.io-6f17d22bba15001f`

- cargo 代理配置在 .cargo 的 config 文件里

- Rust 本身的更新的地址在 `RUSTUP_UPDATE_ROOT`环境变量里

- 需要关联 Git 私有库时，需要先配置

  ```toml
  [net]
  git-fetch-with-cli = true
  ```

- Vec 转 数组，可以直接用切片语法 `vec[..]`

- AES 里设置 NoPadding 会使输出字节数是 16 的倍数，否则会是实际字节数。

- JNI 可以用 `JString::from` 将 `JObject` 转成 JString

- From 这个 trait 可以在其他 crate 进行实现（比如在自己的工程里实现 String 的 From 定义），为什么？

- 宏定义展开可以用 cargo-expand，一个指令示例如下:

  ```
  // 这里 -p 可能不是必须的，后面的参数值会有命令行提示；--lib 后要跟包名，而不是模块名
  cargo expand -p jni --lib wrapper::java_vm::vm

- 



>“不要通过共享内存来通讯；而是通过通讯来共享内存。”（“Do not communicate by sharing memory; instead, share memory by communicating.”） - 《Go 编程语言文档》
