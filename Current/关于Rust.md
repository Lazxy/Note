## 关于Rust

- Rust 的产物是二进制文件，不依赖运行时。

- Rust 的稳定版本互相之间是兼容的

- Rust 的缩进是四个空格而非 Tab

- Cargo 命令一览：
  
  ```
  // 更新 Rust 版本
  rustup update
  
  // 构建项目
  cargo new [项目名]
  
  // 编译项目（如果加上 release 生成产物会运行地更快，但打包更慢）
  cargo build [--release]
  
  // 编译并运行项目（无修改就直接运行）
  cargo run
  
  // 编译但不生成可执行文件
  cargo check
  ```

- const 标记的常量必须声明类型

- let 声明的不可变量可以在作用域内被覆盖，并且允许不同类型的覆盖。

- 数字类型允许用`_`进行分割表示，不影响其真实值。

- int 类型默认为 `i32`；同时`isize`和`usize`支持根据当前处理器架构自动选择 32 位或 64 位取值；运行时如果发生位溢出，`release`版本会进行取余截断。

- 可以用`(type1,type2,type3)` 来声明一个元组`truple`，其允许对每个元素声明不同的类型，并且用角标获取对应值，以及有和 kotlin 类似的解构赋值的能力。**其长度不可变**

- Rust 的数组长度是不可变的，显式类型声明类似 `let a: [i32;5] = [1,2,3,4,5]`。也提供默认的初始化填充操作`let a = [3;5]`等价于`let a = [3,3,3,3,3]`

- Rust 的函数使用下划线式命名，声明顺序没有影响。

- 代码块内表达式返回值时，语句末不要加`;`，否则其会变成无值返回状态。

- 有返回值的函数声明：
  
  ```
  fn function1(num:i32)-> i32{
      // 返回结果值不需要 ;
      5 + num
  }
  ```

- Rust 的 if 后不跟括号，允许直接用控制流结果值作为表达式赋值内容。

- `loop`关键字等于`while(true)`，可以用`break`或者`continue`进行控制。甚至可以作为表达式赋值内容（通过 break 后带表达式，等价于其他语言的 return）。loop 可以取名作为跳转选择节点。

- `for`的用法有点像 Python：
  
  ```
  for i in nums {
      //...
  }
  
  for i in (1..4).rev(){
   //...
  }
  ```

- 栈的大小是固定的，堆的大小是不定的，可以根据现有内存占用情况选择入堆的位置。所以入栈比入堆效率要高，因为省略了寻找足够连续内存以分配空间的过程

- Rust 遵循内存使用权仅限于作用域范围，离开作用域自动回收对应内存的原则，其实质是在对应对象离开作用域时，调用对象的`drop`方法。

- 但由于同一个对象不能`drop`两次，所以当一个对象声明被赋值给另一个声明时，为了避免这两个声明在离开作用域时分别进行`drop`操作，前一个声明在赋完值后就会被废弃，**这个过程称作`move`**。和 Java 一样，原生类型会直接进行值拷贝，不会有声明迁移。
  
  ```
  let s1 = String::from("str")
  let s2 = s1 // 此时 s1 就会失效 
  ```

- 方法调用时的参数传递同样会发生`move`，传入方法后，对应的声明会直接被废弃，不再允许使用。但方法能把对应的声明转移到接受其返回值的接受作用域，并且支持多参数返回。
  
  ```
  fn main() {
      let s1 = String::from("hello");
  
      let (s2, len) = calculate_length(s1); // 多返回值接收
  
      println!("The length of '{}' is {}.", s2, len);
  }
  
  fn calculate_length(s: String) -> (String, usize) {
      let length = s.len(); // len() returns the length of a String
  
      (s, length) // 多返回值返回
  }
  ```

- 为了解决每次调用参数都要原样将入参作用域迁回过于繁琐的问题，Rust 提供了 `reference borrowing`机制——即可以通过生成引用对象临时指向实际值对象，代替其作为方法入参。这样在读取值的时候，可以访问到实际值，但是在离开作用域时，又不会实际影响到实际对象。其语法为：
  
  ```
  let s = String::form("origin");
  let length = calc_length(&s) // 此处进行引用代替作用域转移，s 本身的声明仍可用
  
  fu calc_length(str: &String)->usize{
      str.length()
  }
  ```
  
  用上面的声明提供的引用是只读的，不允许对引用对象进行修改，如果需要进行支持修改，需要将`&`替换为`&mut`。

- 允许多个不可变引用同时对一个对象声明进行引用，但不允许多个可变引用或者可变引用和不可变引用同时对一个对象进行引用，避免代码执行过程不可预期的对象变化。

- 数组切片（slice）类似子数组引用的概念，在 String 中是对字符的子数组截取，在基本类型数组中是对基本类型的子数组截取，语法写作：
  
  ```
  let s = String::form("Hello World");
  let str_slice = &s[0..5] // 内容相当于 "Hello",类似于 Java 的 subString 方法，方括号内参数是[inclde,exclude]的
  // 简写形式也有几种
  let str_all = &s[..] // 全取整个 "Hello World"
  let str_start = &s[..5] // 取 "Hello"
  let str_end = &s[6..] // 取 "World"
  ```
  
  需要注意的是，切片实际上是对原数组的一个不可变引用，所以根据上面的作用域规则，当不可变引用存在时，不允许其他可变引用的获取，即类似 `clear()`等会通过获取可变引用改变原值的都会引起编译错误。
  
  另外，也引出实际上类似 `let s = "Hello World"` 的赋值方式是获取了一个字符串的切片，所以它是一个不可变引用，类型为 `&str`。

- Rust 的结构体概念类似于 C，构造方式则更像 Dart：
  
  ```
  struct User{
      active: bool,
      username: String,
      email: String,
      sign_in_count: u64
  }
  
  let user = User{
      active:true,
      username:String::from("Lazxy),
      email:String::from("lazxy7zh@gmail.com"),
      sign_in_count:1
  };
  ```

- xiayiye


