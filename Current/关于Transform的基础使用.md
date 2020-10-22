## 关于Transform的基础使用

- 首先，Transform是Gradle打包工作流的一环（或者说任务之一），负责处理项目文件从`.class`文件到打包成`dex`的预处理过程。

- 然后Transform本身被包含在插件中，通过插件的注册机制来实现gradlew时的自动调用。

- 插件的项目结构本身也很简单，和AndroidLib module差不多，不同的是本身需要依赖Maven和Groovy，并需要声明仓库信息，差不多长这样：

  ```gradle
  apply plugin: 'groovy'
  apply plugin: 'maven'
  dependencies {
      implementation gradleApi() //gradle sdk
      implementation localGroovy() //groovy sdk
  
      implementation 'com.android.tools.build:gradle:3.2.1'
  }
  repositories {
      jcenter()
  }
  
  uploadArchives {
      repositories.mavenDeployer {
          //本地仓库路径，以放到项目根目录下的 repo 的文件夹为例
          repository(url: uri('../repo'))
          //以下内容决定了最后打出来的插件包的目录与文件名 需要在声明classPath时用到
          //groupId ，自行定义，组织名或公司名
          pom.groupId = 'com.lazxy'
          //artifactId，自行定义，项目名或模块名
          pom.artifactId = 'plugin.demo'
          //插件版本号
          pom.version = '1.0.0'
      }
  }
  ```

- 然后，插件目录中的`src/main`目录依然是主要文件目录，不过此时`java`目录需要改为`groovy`目录，随便建适当层级的包，就可以开始新建groovy文件了。

- 需要注意的是AS对groovy文件的新建并不友好，需要自己输后缀，声明`package`和`class`，不过倒是还是能自动联想和import，自定义的Transform类继承Transform基类就好，然后实现它的抽象方法和`transform`方法，下面是一个简单的示例：

  ```groovy
  package com.lazxy.demo.plugin
  
  import com.android.build.api.transform.QualifiedContent
  import com.android.build.api.transform.Transform
  import com.android.build.api.transform.TransformException
  import com.android.build.api.transform.TransformInvocation
  import com.android.build.gradle.internal.pipeline.TransformManager
  
  class SecondTransform extends Transform{
  
      @Override
      String getName() {
          //这个名字决定了在gradlew运行时输出的transform名字叫什么，比如在aseembleDebug任务中，它就会是transformClassesWithLazxySecondTransformForDebug
          return "LazxySecondTransform"
      }
  
      @Override
      Set<QualifiedContent.ContentType> getInputTypes() {
      	//声明输入源是代码源文件，另外也能选资源源文件、jar包或者so文件什么的，其他定义可以去看TransformManager的代码
          return TransformManager.CONTENT_CLASS
      }
  
      @Override
      Set<? super QualifiedContent.Scope> getScopes() {
      	//声明处理上述类型文件的范围是仅原创的项目文件代码，不包括引入的jar包，其他定义也可以去看下TransformManager的代码，很好懂
          return TransformManager.PROJECT_ONLY
      }
  
      @Override
      boolean isIncremental() {
      	//是否允许增量编译时运行
          return false
      }
  
      @Override
      void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
          super.transform(transformInvocation)
          //真正有处理逻辑的方法，这里就是简单地打了个日志，可以通过其他.class文件编辑框架实现一些对代码的魔改
          println("Lazxy Second Transform invoke")
          //将input的文件放到output，如果少了这一步 会导致打包失败
      }
  }
  ```

- 接着需要新建一个Plugin类文件，在它的apply方法中注册这个Transform，简单代码如下：

  ```groovy
  package com.lazxy.demo.plugin
  
  import com.android.build.gradle.AppExtension
  import org.gradle.api.Plugin
  import org.gradle.api.Project
  
  class DemoPlugin implements Plugin<Project>{
  
      @Override
      void apply(Project project) {
          AppExtension appExtension = project.extensions.findByType(AppExtension.class)
          appExtension.registerTransform(new SecondTransform())
      }
  }
  ```

- 然后，在main目录下新建`resources/META-INF/gradle-plugins/`目录，在其下新建[name].properties文件，其中这个[name]最后会是事实上的插件名，它的内容很简单：

  ```
  implementation-class=com.lazxy.demo.plugin.DemoPlugin
  ```

  就是声明了一下这个插件的源类是哪一个。

- 于是，插件的编写就告一段落，在gradle中执行`uploadArchives`任务，就能打出对应的插件包，按照上面的配置，会出现在项目根目录的repo目录中。

- 在根目录的build.gradle文件中添加这个插件的classPath，按照上面的配置，就应该是：

  ```
  classpath "com.lazxy:plugin.demo:1.0.0"
  ```

  并且要注意，为了能在本地目录进行索引，需要在buildScript的`repositories`中添加对于本地目录的maven地址声明：

  ```
  maven(){ url uri('repo') }
  ```

- 最后，在需要使用这个插件的build.gradle中声明使用，按照上面的properties文件(com.lazxy.android.properties)的名字，应该写为：

  ```
  apply plugin: 'com.lazxy.android'
  ```

- Over

