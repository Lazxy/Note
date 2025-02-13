## TechLogTrans

### Gradle 相关

#### Gradle 原生配置：

- Gradle VersionCatalog 会自动读取gradle下名为`libs.version.toml`的文件

- 当 Gradle 提示  JAVA_HOME is set to an invalid directory 时，有可能是由于配置的目录末尾缺少文件分割符。

- Gradle 初始化流程：

  >Groovy：
  >
  >Script 加载流程：
  >
  >1. CompileConfiguration 设置脚本的编译参数，如脚本基类（为脚本提供能力继承）、生成文件目录（类加载环境）
  >2. Grovy 脚本引擎对脚本代码进行解析，根据设定的基类生成对应的 class 文件放置在对应目录（所以一个 buildScript 代码块在 runtime 时其实是一个类）
  >3. GroovyClassLoader 根据对应脚本名称和目录，加载对应 class
  >4. 执行脚本中的 run 方法（继承自 groovy.lang.Script）
  >
  >Script 运行流程：
  >
  >1. Pass 1，编译并运行 buildscript/initscript 的 pluginManagement、plugins 部分，加载对应的插件。
  >
  >  > 编译过程执行在 `DefaultScriptCompilationHandler.compileToDir` 方法内，可以通过 val extractingTransformer:CompileOperation 携带的 Transformer 进行编译时 Hook
  >
  >2. Pass 2，应用插件及相关逻辑，如 apply plugin 、repositories 代码块，dependencies 代码块
  >
  >KTS：
  >
  >Script 执行流程：
  >
  >1.  stage 1，编译并执行 buildscript 和 plugins 部分，影响后续的 classpath 内容
  >
  >2.  stage 2，**eval** 脚本剩余部分
  >
  >  > 两个阶段的生成类产物都在 `$HOME/.gradle/caches/gradle-version/kotlin-dsl/scripts/hashcode/classes` 下
  >
  >Script 编译流程：
  >
  >1. **Lexer**，通过原生 kotlin 库进行脚本的词法分析，将执行语句解析成一个个 kotlin token 元符号（纯工具性质处理，不带业务逻辑）。
  >
  >2. ProgramParser，将 Lexer 处理完的结果通过特定逻辑进行检出和整理，区分出 stage 1 和 stage 2 的执行代码。
  >
  >3. PartialEvaluator，将上一步骤划分好的执行逻辑进行分类，Static 或者 Dynamic
  >
  >4. ResidualProgramCompiler，将上一步骤中输出的 ResidualProgram 通过 ASM 转成字节码。
  >
  >  区别在于 Static 的 Program 是完整编译的（这里的编译分成两个部分，一个部分为基于 ASM 的Program 类生成，带 execute 方法，作为初始化入口；另一个部分为基于 KotlinToJVMBytecodeCompiler 编译的 Builde_gradel 类，会在构造里执行具体代码），可以直接通过`execute`执行（apply plugin、buildscript代码块内容等）。
  >
  >  而 Dynamic 中除了有直接生成和可执行的字节码，还有一些需要运行时编译和解释执行的字节码内容（以 String 字面量形式存放，未编译；如果代码量超出了 64KB，会以 LoadScriptResource 形式存放）会在stage 1 过程中执行编译和类生成。
  >
  >5. KotlinCompiler，基于`kotlin-compiler-embeddable`库，对 kts 脚本内的代码进行 kotlin -> JVM Bytecode 编译，需要设定生成脚本的基类（KotlinScript 接口的子类），默认导包和代码依赖的 classpath。

- Gradle 插件的编写中，类似 Task 和 Extension 等会被注册到运行时脚本的类，都会被 Gradle 构造对应的动态生成子类，所以需要是 `open class`，且 Task 的构造有参的情况下，需要加入 @Inject 注解，以及在参数前注明 `Input`、`Output`、`Internal`，让 Gradle 构建时能够知晓是否需要监听对应的 Task 参数变化，以及在参数无变化时不执行对应任务。

- 看 Gradle Task 相关代码时，由于没有一个公共的接口方法，需要通过索引 `@TaskAction` 来找实际的执行方法。

- 8.x 后，插件的依赖从`配置 classpath -> 从 classpath 中索引对应的插件配置文件 -> 应用插件`，变成了`插件配置基本信息脚本 -> 发布时按单个插件发布包 -> 单独配置插件名和版本号 -> 索引 repository/[plugin id]/[version]/[plugin id].gradle.plugin-[version].pom，找到对应 jar -> 索引 jar 中的插件声明配置文件 -> 应用插件`，简化了 classpath 的声明过程，不过依赖插件发布插件自动生成 jar 内的 properties 文件，以及向 maven 发布单独的索引文件 

#### Android 工程相关

- AndroidManifest里的`android:extractNativeLibs`属性，用于设置原生库的存储方式：

  - 为true，在打包时会对so库进行压缩后打入APK，同时安装时从APK解压到`data/app/`对应目录，会减少APK的大小，但在安装后相当于存在两个native库文件备份，会增大占用体积，且不利于进行差分打包

  - 为false（AGP 3.6.0版本后的默认选项），会将so库原样打入APK包。相对来说打出来的APK会较大，但是加载so时能够直接从APK文件中加载，速度较快。

  > 从 AGP 4.2.0 开始，`extractNativeLibs` 清单属性已被 DSL 选项 [`useLegacyPackaging`](https://developer.android.com/reference/tools/gradle-api/7.1/com/android/build/api/dsl/JniLibsPackagingOptions?hl=zh-cn#uselegacypackaging) 取代。 您应该使用应用的 `build.gradle` 文件中的 `useLegacyPackaging`（而非清单文件中的 `extractNativeLibs`）来配置原生库压缩行为。如需了解详情，请参阅版本说明[使用 DSL 打包压缩的原生库](https://developer.android.com/studio/releases/gradle-plugin?buildsystem=ndk-build&hl=zh-cn#compress-native-libs-dsl)。

- AS Dolphin版本 原来在 `.iml`中的内容，被移动到了AS的用户数据目录，每个项目有一个对应编号，例如：`C:\Users\Lazxy\AppData\Local\Google\AndroidStudio2021.3\external_build_system\tkmainservice.cbd3f616\modules`

  其下包含着各个模块的配置信息。

- 把包设置成`debuggable`时，会增加包体大小，因为`dex`文件会被多拆分。

- 直接将IDEA工程打成jar包时，建`Artifacts`时，jar要选择`From modules with denpencies`，将依赖的Jar包 extract出来，否则会出现运行时索引错误。

- Kotlin 不支持识别单层 `xx.xx.xx`的目录结构下的 Java 文件，需要有切实嵌套的多个目录，但 Java 支持。

- gradle 8.0 在编译字节码时，就会考虑当前的  minSDK 属性，将代码中版本判断相关的无用分支删除

- 当多个源中都存在对应的资源时，资源合并的优先级如下：

  > build variant > build type > product flavor > main source set > library dependencies

  当多个 module 存在对应资源时，合并顺序应该和声明依赖的顺序有关？

- public.xml 中的 `<public type=”string“ name="Lazxy" id="0x7f040001">`写法能够固定某个属性的 ID，避免被其他应用引入时被重新编译编号为别的 ID。

#### Android 相关知识

- 只有Activity、ContentProvider、Service中可以进行bindService操作。

- 非指定的系统签名应用不能够直接通过`bindService`直接被拉起服务进程。

  > 特别的，由于Android O以后对后台服务的限制，如果使用startService进行进程拉起，则无论是不是系统应用都不可行。但添加`persistent`关键字后就可以正常拉起。
  >
  > 不过`persistent`应用无法通过直接安装的方式进行更新，必须推送到系统目录重启运行。

- Binder 的缓冲区大小为 1M，进程内共用。

- 窗口层级排列在 WindowManagerPolicy. getWindowLayerFromTypeLw

- AS Profiler dump 下来的内存信息，shallow size 指内存对象本身的大小，retained size 指该对象引用到的对象的内存占用大小，单位都是 KB

- 内存信息指标：

  RSS（Resident Set Size），实际使用物理内存（包括全部的共享库占用内存）

  PSS（Proportional Set Size），实际使用的物理内存，如果有共享库被多个进程共享，则等比例划分对应库内存占用

  USS（Unique Set Size），进程独自占用的物理内存（不包含共享库占用的内存）

- `fitSystemWindow`生效的原理来自于 `setSystemUiVisibility` 方法的设置，允许布局延伸至 SystemUI 之下。但该属性允许根布局延展的同时，可能会主动对其内控件设置偏移，使其与 SystemUI 不重叠展示。

- Fragment 的页面切换使用解释器模式实现，其中主要文法为 FragmentTransaction.OP_XXX指令：

  ```
  1. Fragment 展示或者入栈时，如果限定了需要 addToBackStack，那么 FragmentTransaction 会记录下所有执行的 OP 指令，一般为 ADD、REMOVE、REPLACE，也包括 HIDE（针对上一个页面），存入BackStackRecord 对象中。
  具体执行在 FragmentManager.executeOpsTogether,在 isPop 的判断流程中，会走到 record.expandOps 方法，对照指令执行对应的正向流程
  2. 调用 popBackStack 时，获取到栈顶的 BackStackRecord 之后，也会执行到 FragmentManager.executeOpsTogether，但是 isPop 流程会走 record.trackAddedFragmentsInPop 方法，对照指令执行对应的反向流程（add 对应 remove，show 对应 hide，以及 OP_SET_PRIMARY_NAV 等与当前基页面相关的操作）
  ```

- targetSDK < 28 时，ViewRootImpl 和 View 都会设置 sAlwaysAssignFocus = true。前者会导致第一次绘制页面时，主动查找 Focusable 的 View，并调用 requestFocus 方法；后者会导致在 View clearFocus 时，重新寻找 Focusable 的 View，在页面内没有其他符合条件的 View 存在时，可能会使 clearFocus 一直失败。

- 输入法键盘展示的条件为：

  - 触发条件：存在一个获取焦点的 EditText
  - 存在条件：触发后存在任一获取到焦点的布局（没有 detachToWindow）
  - 关闭条件：通过 InputMethodManager.hideSoftInputFromWindow(windowToken)，该token需要与触发键盘的WindowToken一致。

- ContentResolver 方式读写数据库时，获取到的 Cursor 可以不手动 close，因为其实现类继承了 `ContentResolver.CursorWrapperInner`，在 `finalize`调用时会主动进行 close。即只要不存在 Cursor 相关的内存泄露，就不会有数据库相关的资源占用。

- 当两个进程 A、B 发生通信过程， A->B 请求发生时，A 处于阻塞等待结果状态，此时 B 发起对 A 的请求，那么会复用 A-B 已经建立的 Binder 通道，所以 B->A 的请求会执行在 A->B 请求执行时的线程里，**可以是主线程，也可以是任意子线程**。

  而如果这两个请求不是同步发生的，那么 B->A 的请求会唤醒 A 进程的 ProcessThread 线程池中某个挂起等待中的线程，执行对应的方法，也就是通常情况下的 **Binder 线程**。

- Linux 文件系统中，单个文件占用的最小存储空间，由 `block size` 决定，它们是磁盘空间分配的最小单位。可以用 `stat -f [path]`来查看块大小和文件实际占用空间。

#### 语言语法相关

- 当一个 Lambda 表达式不依赖外部引用时，会自动生成静态对象以重用。

- Kotlin 中类声明的成员变量实例化，在实际的字节码中可能不会先于构造方法执行（除非构造中也直接用到了对应的变量），而是被排在构造方法中进行。

- 不应该在构造方法中提供外部调用匿名内部类的机会（比如作为 Thread 的执行体或者异步调用回调），因为在对应方法执行时，会需要等待内部类的宿主类构造加载结束，从而发生不按照预期时序执行代码的问题。

  示例如下：

  ```kotlin
  val runnable = Thread({
  	println("Run thread event")
  },"ThreadA")
  
  init{
      // 直觉上来讲，这行代码执行完之后，就会立刻启动 ThreadA，但实际上 ThreadA 一定会在 “Construct finished” 打印之后才会启动。
  	runnable.start()
  	// ...
  	println("Construct finished")
  }
  ```

- Native 代码中，如果创建新线程，在子线程里无法正常加载应用的类，因为没有继承应用本身的 ClassLoader，应用代码的 classpath 不在范围内。此时需要用在主线程提前预加载类对象的方式进行规避。

- Kotlin 的接口默认方法，如果没有被实现类重写，**在编译时也会自动添加到对应实现类的字节码中**；但 Java 的接口默认方法，不重写的情况下就不会出现在 class 文件里。

#### 工具相关 TIP

- 如果有文件被进程占用，可以通过 任务管理器 - CPU - 打开资源监视器 - 关联的句柄 来索引到对应进程，然后进行进程结束。