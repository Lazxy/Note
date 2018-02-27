### NDK基础使用

> 记一些相关的原始记录，日后再补充整理。

#### 先看看JNI

1. JNI是为了解决Java程序兼容性问题、本地代码复用和提升计算效率而被设计出来的跨语言接口。

2. 需要在调用本地方法前利用System.loadLibrary("lib_name")的方式加载本地库，不同的系统会根据该库名编译出不同的文件。

3. C/C++中方法名是由`Java_` +` Java类名` + `_` + `Java方法名`构成的。

4. C/C++方法中的参数列表构成是这样的：第一个参数为JNIEnv 指针，其表示一个指向Java中的方法所在内存指针数组的偏移量；第二个参数在native方法为静态时表示一个Java的class对象，非静态时表示调用该方法的类的对象；剩下的参数即C/C++中与Java数据类型对应的各种类型参数。

5. native实现代码会持有Java对象的引用，导致GC无法工作。

6. native实现内不允许创建额外的本地对象，并且在进行密集计算前应手动回收大内存对象，否则虚拟机将不回收这些对象。

7. 不允许在native方法中传入非当前线程中创建的对象。

8. JNI设计总体原则如下：

   ```
   1. 减少JNI层所使用的资源与使用频率，因为JNI的调用是需要不低的成本的。
   2. 尽量都使用主要代码（在这里是Java）进行异步通信，而不是JNI层直接在子线程回调主线程，以此更易维护JNI接口（第7点的限制）。
   3. 减少JNI所能接触的线程量。如果确实需要在Java和C ++语言中使用线程池，请尝试在池所有者之间保持JNI通信，而不是在各个工作线程之间进行通信（对分散的线程收束到它们的管理线程）。
   4. 尽量把JNI调用接口和C++方法聚合在少数几个类里，以方便将来的重构。考虑适当地使用JNI自动生成库。
   ```

9. 所有JNI方法的入参和返回值几乎都是**本地变量**，当方法结束时，其引用会失效（即使内存中仍存在着其值）。所以可以通过**NewGlobalRef**方法来构造一个全局变量，其用法如下：

   ```c++
   jclass localClass = env->FindClass("MyClass");
   jclass globalClass = reinterpret_cast<[jclass]>(env->NewGlobalRef(localClass));
   ```

   也因为这个方法的存在，**两个指向同一个对象的引用值可能不同**，故判断两个引用是否为同一个对象的引用时，需要通过**IsSameObject**方法判断，此时`==`是失效的。

10. 由于JNI 的UTF-8做了一些修改，所以在从文件或流中读取字符时，最好进行一次过滤与转换。

11. JNI会将创建的局部引用都存储在一个局部引用表中，如果这个表超过了最大容量限制（最多16个局部引用），就会造成局部引用表溢出，使程序崩溃。

12. 注意一个问题：在用GetXXXArrayElements获取到jIntArray的指针，使用完成后用ReleaseXXXArrayElements 释放指针时，要保证指针指向的位置仍然是得到时的位置，否则应该用另一个指针保存初始位置，否则会出现访问内存失败的错误。



#### 然后是NDK

1. Android Studio中，可以通过`javah [完整类名]` 语句构造出包含其相应native方法的头，但这个命令只能在有该类class文件的路径下生效，比如在`\app\build\intermediates\classes\debug`中。

2. 因为旧版的NDK已经快要被禁用了，这里尝试用**CMake**来进行关联。AS的CMake算是比较省心的一个工具了，它不需要创建`jni`文件夹和`.mk`文件，而是通过创建一个**CMakeLists**文件对所需的C++库进行配置，然后关联Gradle，就能自动在打包时把`.so`文件放入apk，非常方便。

3. 其具体使用流程：

   ```
   1. 准备好C++文件,新建cpp(名字任取)文件夹，其目录为\app\src\main\cpp.
   2. 在module根目录下创建CMakeLists.txt文件，写入相关配置（具体看下一条）.
   3. 右键点击module文件夹，选择"Link C++ Project With Gradle",并在ProjectPath中选择CMakeLists文件的路径。
   4. 在Java文件中需要导入本地库的地方调用System.loadLibrary("库名")。
   5. 然后就可以自由地调用native方法了。
   ```

4. 下面是CMakeList的一些可用配置方法

   ```
   # 控制NDK版本
   cmake_minimum_required(VERSION 3.4.1)

   # 设置自定义库的名称、类型以及源文件，该方法可多次调用以定义多个库。SHARED库会在gradle执行时自动打包进APK
   add_library( # Specifies the name of the library.
                ndk_test

                # Sets the library as a shared library.
                SHARED

                # Provides a relative path to your source file(s).
                src/main/cpp/ndk_test.cpp )

   # 设置头文件所在路径
   include_directories(src/main/cpp/include/)

   # 定位NDK预置库log的路径并将其赋给变量log-lib
   find_library( # Defines the name of the path variable that stores the
                 # location of the NDK library.
                 log-lib

                 # Specifies the name of the NDK library that
                 # CMake needs to locate.
                 log )

   # 将NDK预置库与自定义库关联，从而可以在自定义库中调用其方法，可以在这个方法里指定多个预置库
   target_link_libraries( # Specifies the target library.
                          native-lib

                          # Links the log library to the target library.
                          ${log-lib} )

   # 导入未编译的NDK源码库
   add_library( app-glue
                STATIC
                #ANDROID_NDK是一个预设变量，用以指向NDK目录以方便导入NDK预置库
                ${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c )

   # 导入非NDK的预置库
   add_library( imported-lib
                SHARED
                IMPORTED )

   # 为导入的其他预置库设置路径等属性
   set_target_properties( # Specifies the target library.
                          imported-lib

                          # Specifies the parameter you want to define.
                          PROPERTIES IMPORTED_LOCATION

                          # 声明预置库路径，ANDROID_ABI同样也是一个预置变量，其值为当前设备的ABI
                          imported-lib/src/${ANDROID_ABI}/libimported-lib.so )

   ```

5. 可以在module级的build.gradle中加上下列语句来限制`.so`的打包渠道(PS 这里限制的是所有native库的的打包渠道，包括依赖的开源库)：

   ```
   android {
     ...
     defaultConfig {
       ...
       externalNativeBuild {
         cmake {...}
         // or ndkBuild {...}
       }

       ndk {
         // Specifies the ABI configurations of your native
         // libraries Gradle should build and package with your APK.
         // 这里一般留下x86 armeabi armeabi-v7a就差不多了
         abiFilters 'x86', 'x86_64', 'armeabi', 'armeabi-v7a',
                      'arm64-v8a'
       }
     }
     buildTypes {...}
     externalNativeBuild {...}
   }
   ```

6. 只要SDK Tools中装上**LLDB**，就可以像Debug Java代码一样**Debug native代码**了，当不需要Debug native代码时，可以在Run/Edit Configurations /Debugger/Debug Type  auto -> java，来设置禁用LLDB。

7. [官方参考文档](https://developer.android.com/studio/projects/add-native-code.html)

8. [NDK预置库](https://developer.android.google.cn/ndk/guides/stable_apis.html)

