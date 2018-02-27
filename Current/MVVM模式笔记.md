### MVVM模式笔记

#### 基本组成：

​	**View**：大部分是使用了_data-binding_的XML文件，通过`data`标签声明绑定的数据，在这里是一个(或多个)ViewModel，将控件的数据源（包括可见性、点击事件和文字等）通过指派ViewModel的各个成员变量完成绑定。除此之外还有**Activity**和**Fragment**等组件，用于对ViewModel和xml文件的初始化绑定以及处理一些较为复杂的、不可由data-binding直接代理的UI相关操作（实例中是对RecyclerView的配置和数据填入）。

​	**ViewModel**：作为View绑定数据的载体和Model类的调用方，其内包括对xml中需求变量和点击事件的处理，同时会将一些复杂操作交由Activity/Fragment处理。其内与View相关的数据都为**观察者模式**相关类型，为ObservableXXX(基本数据类型)或者ObservableField<T>(对象类型，多为String)

​	**Model**：包括实体数据类和网络请求、算法类等，主要用于由进行I/O与计算操作获得结果，并反馈到ViewModel，作为View的数据源依据。

#### 组件细节：

​	MVVM模式的实现基本都依赖于data-binding的双向绑定特性，故ViewModel类是整个框架的实现核心。

​	简单地扫了一下其Observable数据类的代码，基本就是标准的观察者模式，没什么好说的，但不得不提一下其数据的关联方式：在XML文件中，控件属性的数据源表达方式为@{viewModel.property}(假设此时定义的VM变量名为viewModel，属性名为property)，则在ViewModel中该属性的声明可以有两种方式——直接声明一个public修饰的同名成员变量property，变量类型为上面提到的两种观察者模式类型；或者声明一个public修饰的函数，函数名为getProperty()，返回值为其所需类型。

​	两边都完成了相应的声明后，在Activity中，通过DataBindingUtils进行setContentView得到了XXXBinding对象，对象名前缀和布局文件名一致（去下划线的帕斯卡式命名）。该函数中调用了activity的setContentView函数，并将得到的窗口根布局进行了监听建立。最后，XXXBinding对象通过setViewModel(这里的ViewModel也源自XML文件中定义的变量名)设置绑定的数据源，整个初始化流程结束。

#### 关于Data binding

#### 基础配置

在module级的gradle文件中配置：

```
android {
    ....
    dataBinding {
        enabled = true
    }
}
```

#### 基本用法

​	Data Binding的布局用XML文件与一般的布局文件不同，其分为`data` 和`view`两个区块，共同被`<layout>`标签包裹，其中`data`区块用`<data>`标签包裹，里面是关于控件数据源的声明，`view`区块即正常的布局代码，里面有对`data`中数据源变量的引用，其示例如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="viewModel"  
       type="com.example.ViewModel"/><!--数据源变量名及其类型-->
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView
       		android:layout_width="wrap_content"
           	android:layout_height="wrap_content"
           	android:text="@{viewModel.params1}"/><!--对变量的引用-->
   </LinearLayout>
</layout>
```

​	如上面写过的，在Activity中可以通过DataBindingUtil.setContentView(resourceId)取得一个XXXBinding类对象，该类由编译器根据布局文件生成。更一般地，可以用一些经过封装的LayoutInflater类方法获得相应类对象：

```Java
MainActivityBinding binding = MainActivityBinding.inflate(getLayoutInflater());

MainFragmentBinding binding = MainFragmentBinding.inflate(layoutInflater, viewGroup, false);

MainFragmentBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.main_fragment, viewGroup, false);
```

#### 点击事件设置

​	对于被绑定控件的事件响应，Data Binding给出了两种设置方式：一、设置静态监听事件，相当于原来的实现OnClickListener等接口，默认传值为控件本身，在绑定关系建立时起效，其写法是这样的：

```
//Java Code
public class ClickListener{
  public void onClick(View view){
    //...
  }
}
//XML
<data>
	<variable name = "listener"
		      type = "com.example.ClickListener"/>
</data>
<Button
	//...
	android:onClick="@{listener::onClick}"
	/>
```

二、设置监听绑定，即在事件被触发时构造一个监听器并响应事件，其内容来自XML文件中的Lambda表达式，可以包括当前的绑定数据源：

```xml
<data> 
	<variable name = "listener".../>
	<variable name = "data" .../>
<data/>
 <Button
 	//...
 	android:onClick = "@{() -> listener.onClick()}" //无参回调
 	//android:onClick = "@{(view) -> listener.onClick(view)}"//有参回调
 	//android:onClick = "@{(view) -> listener.onClick(data)}"//带绑定参数的回调
 	//android:onClick="@{(v) -> v.isVisible() ? listener.onClick() : void}"//通过断言确定回调方式，void为占位标记，表示什么都不做
 	/>
```

需要注意的是，Lambda表达式的参数，只能选择**全写或全不写**（针对CheckBox或者Dialog按钮等多参数的情况），并且其指定的回调返回值必须为回调所需要的返回值，否则为各数据类型的默认值。

#### 声明变量相关

​	承接上面的应用情况，当响应事件的Lambda表达式中用到断言判断时，可能会需要类似View等类的引用，此时应该在`data`区块中导入View的包，如：

```xml
<data>
	<import type="android.view.View"
			alias="AndrodiView"/><!--可以通过给类取别名避免同名类冲突-->
</data>
```

​	上面已经提到过，`variable`标签内的变量需要制定变量类型，并且在Java文件中用ObsevableXXX类型声明，这里不再重复。但有一点，即所有XML文件都存在一个隐式变量`context`，当自定义变量也叫这个名字时，它会被重写。

​	另外，当我们不想要让编译器根据布局文件名生成相应的数据类时，可以在`<data>`标签下用`class`属性设置其生成的类名和所在包的位置，如：

```
<data class="com.example.data.DataClass">
...
```

#### 布局嵌套相关

​	当布局中需要用到`<include>`标签时，可以通过`bind`属性将当前布局文件指定的数据源传递给子布局，但子布局中一定要有同名变量：

```xml
<include layout="@layout/child_layout"
		 bind="@{data}"/>
```

​	data binding 不支持`merge`标签的数据传递。

#### 运算符相关

​	数据源对属性的赋值可用表达式来表示，于是就可以用各种在Java中使用的运算符，但不包括`this` `super`和`new`。同时DataBinding增加了新运算符`??`，相当于判空三目运算符——当运算符前的对象为空时，则取运算符后的对象。

#### 变量相关

​	如前所言，可以用取对象成员变量的表示形式直接或调用`getter`方法取数据源所能提供的值，当该值为空时，会取其数据类型的默认值。

​	当需要取一个集合内的某个值时，可以用`[]`包含集合的角标或者键来表达取其位置的值，如：

```
<TextView
	...
	android:text="@{list[index]}"
	//当map的键为一个与变量重名的字符串时，可以用下列方式取值
	//android:text='@{map["key"]}'
	//android:text="@{map['key']}"
	//android:text="@{map[`key`]}"
/>
```

​	当属性的值为Resource值时，可以将其作为一般变量写入表达式；对于Format String和**Plurals**特定的表达方式，则可以直接将格式化参数或者复选项参数以函数参数的形式填入对应文件名的函数式中。如：

```xml
<!--一般表达式-->
android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
<!--格式化或复选表达式-->
android:text="@{@string/name(firstName,lastName)}"
android:text="@{@plurals/item(count)}"
//android:text="@{@plurals/item(count,count)}" 第二个count为格式化语句的参数
<!--这里的name和item分别为格式化字符串和数字选择字符串的name属性值-->
```

>Plural标签的作用类似与selector，可以在输入quntity不同的条件下选择不同的字符串作为返回使用，其多用于多语言适配，特别是对于0、1、2等数字有歧义的一些语种。在中文中较少用到。可以用getQuantityString(resourceId,quantity)方法来获取其对应的字符串，quantity参数在中文地区只有1与非1的区别，分别指向“one”和“other属性”（如果匹配不到对应的属性声明会报运行时异常），当需要传入格式化参数时可以将其作为第三个参数传入。



除此之外，还有各种复杂属性类型的更名：

| 属性                | 一般表达      | 更名后表达              |
| ----------------- | --------- | ------------------ |
| String[]          | @array    | @stringArray       |
| int[]             | @array    | @intArray          |
| TypedArray        | @array    | @typedArray        |
| Animator          | @animator | @animator          |
| StateListAnimator | @animator | @stateListAnimator |
| color `int`       | @color    | @color             |
| ColorStateList    | @color    | @colorStateList    |

#### 绑定数据类型

​	前面讲ViewModel代码的时候说过，控件所需属性可以用ObservableXXX(基本数据类型)或者ObservableField<T>(引用类型)来声明。但也可以自己继承BaseObservable类，构造一个可以随数据改变而重绘控件的数据类，如下：

```Java
private static class User extends BaseObservable {
   private String name;
   @Bindable  //这里的注解配合下面的notifyPropertyChanged实现了数据和控件的绑定
   public String getName() {
       return this.name;
   }
   public void setName(String name) {
       this.name = name;
       notifyPropertyChanged(BR.name);//这里的BR是一个自动生成的类似R文件的类，里面有XML中引用到的变量及其序号，需要注意的是，如上面非Observable类的变量需要有@Bindable注释才能被写入BR中
   }
}
```

#### 自定义属性方法

​	Data Binding允许在对应数据类中定义自定义方法，通过取控件属性值作为参数来进行一些设置，使用场景如图片的加载和一些android框架中未定义的单项属性控制方法（如TextView边距的设置）：

```
JavaCode:
@BindingAdapter("img:imageUrl","img:imageSize") //这个注解用于标明其调用的参数及命名空间，甚至可以重写android:...类属性
public static void loadImage(View view,String imageUrl,int imageSize){
	//...
}
XML:
<ImageView
	...
	app:imageUrl="@{...}"
	app:imageSize="@{@dimen/...}"
	/>
```

​	当所设定的属性涉及到事件响应时，则需要将原接口回调方法作为参数赋值，且当该接口回调不止一个时，需要分别为每个方法写一个自定义方法，最后再同意写一个自定义方法（如TextWatcher的几个方法就需要设置不同的BindingAdapter，然后在最后一个BindingAdapter中传入所有参数）。*这里的使用略复杂，还待进一步研究。*

#### 对象类型转换

Data Binding文件中支持android原生的对象类型适配和转换，如在android:background里会自动把color资源值转化为ColorDrawable，但该转换只会在初次设值时进行，故不允许用结果可能为两种不同类型的选择表达式。

#### IDE支持

AndroidStudio中允许用`default`字段来设定数据源不存在时的默认值。