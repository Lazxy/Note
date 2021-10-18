## ReactNative学习笔记

### 基本介绍和个人感受

ReactNative的实现思路基本上可以总结为：通过React框架API构造页面，再使用对应平台的原生组件进行实现，ReactNative库作为中间的翻译层，并完成打包工作。

在直观感受上，ReactNative的代码书写方式与Flutter十分接近，JSX可以使用和Dart很像的标签化语言，以及相当直观的属性配置（虽然Flutter的属性配置是Dart对象方法的一部分，而RN的UI属性只是一个装饰部分，更像CSS）。包括组件核心的状态（state）描述，以及更新方式，两者几乎如出一辙的编码方式，只能说Flutter应该算是个好学生。

### 基础配置和HelloWorld

```
//npm换淘宝源
npx nrm use taobao

//装yarn
npm install -g yarn

//要求的Android SDK和NDK版本
API29以及对应Image NDK:20.1.5948944

//配置环境变量
ANDROID_HOME -> Sdk目录
%ANDROID_HOME%\platform-tools
%ANDROID_HOME%\emulator
%ANDROID_HOME%\tools
%ANDROID_HOME%\tools\bin

//注意事项
目录和文件命名中不能有空格或者中文特殊符号 不能使用各种常见关键字作为项目名 最好直接在Windows自带窗口下面运行npx命令

//构造一个RN项目（在命令所在目录下）
npx react-native init [项目名]

//构造带TS模板的项目
npx react-native init [项目名] --template react-native-template-typescript

//运行RN项目（包括编译、JS打包、依赖下载等操作）
yarn android (这个脚本实际上是写在 package.json的scripts 配置中的，完整的命令为 react-native run-androdid，如果没有配置对应脚本，可能会报错“找不到这个命令”)
```

> 实践中注意到，npm有可能无法正常地拉下来一些github上的包，需要考虑npm仓库本地化建设。

### 组件

和Flutter一样，RN的代码编写也推荐将每一块布局提成一个个组件来进行组装，以此达成复用的目的（同时代码内的嵌套不会严重到没法看的地步）。

其组件的形式类似于HTML的标签，或者Android中XML化的布局配置方式，包括各种命名也很类似。类似TextView等一般通用组件，RN预置了`<Text>、<Image>`等**核心组件**作为基础使用。

而在基础组件之外，更多的组件来自社区贡献，基于RN的实现原理，称为**原生组件**。

> 很好奇原生组件是怎么自定义出来的，如果学会了这个，大概对RN的整个构成机制就能够有一个比较详细的了解了。

下面是一个基本组件的实例：

```jsx
import React from 'react'
import { Text } from 'react-native'

const SimpleText = () => {
	return (
		<Text> Hello World </Text>
	)
}
export default HelloWorld;
```

### 布局方式

关于布局方式，RN似乎没有给出特别多的选择，只提到了[Flex布局](https://reactnative.cn/docs/flexbox)，看上去和小程序的Flex布局形式和参数都差不多。

### 组件参数（props）

组件参数和Flutter的对象构造参数一样，基本上就是在组件的标签内用一个key-value传参就可以，并且由于基于React（JS），这里的对象不需要严格的声明，可以随便传。style属性也是其中的一员。

```Jsx
//...
const PropWidget = (props)=>{
	return (
		<Text> Call me {props.name} </Text>
	)
}
```



### Native与RN的交互

Native端与RN的交互，看起来和WebView的交互方案几乎没有差别（实际上可能也很难再想出什么完全不同的方案了吧）：

- RN调起Native使用的是**ReactContextBaseJavaModule**对象，这个对象会通过`@ReactMethod`注解的接口声明可被映射到JS的接口，然后在**ReactPackage**的子类中被实际注册声明（这不就是addJavaInterface吗！），注册的key来自于上面Module的`getName`方法。这种调用方式相比WebView有个缺点，即**无法完成一个同步操作**——Native的方法执行结束之后是不能将返回值直接丢给RN的，还是需要进行一次对RN的事件调用，或者通过调用参数中存在的Callback方法来完成回调。

- Native调用JS的方式也比较直接，通过`RCTDeviceEventEmitter`和`NativeEventEmitter`这一对对象就可以完成。

  > 这里的接口参数依然是Map，但没看到能不能用Callback，如果可以的话，到还算是方便。

- 剩下的交互与Activity跳转和生命周期有关，下次再补。



### 未完待续

### 踩坑

- 某些依赖如果版本太低或太新，会出现npm拉不到现成的包而自己去打包的情况，这种情况下本地环境得有Python和VS的配置环境，建议直接换版本。
- 某些依赖对react-native的版本号本身有要求，需要到对应的页面去探查仔细。
- 可能会出现一些组件在某些版本对于某些包会有强依赖的情况，需要看着报错把对应的包也手动npm添加进来。