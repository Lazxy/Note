---
title: 关于Activity栈
date: 2017-05-10 16:15:30
tags: 笔记
---

> ​	前两天在写项目的时候突然碰到个问题，当用户进入应用时，先进入MainActivity，如果判断为非登陆状态，则应该直接跳转到登陆页面 ，如果在登陆界面放弃登陆，则应该直接回到主界面。这意味着在登陆界面打开时，MainActivity应该从回退栈中弹出，而不是下压。那个时候草草百度了一番，用Intent加Flag的方法匆匆混了过去，然而新栈建立的Transition动画着实让人难受(所以又回去补了一波动画的相关，然而还是没有补到Activity Transition的部分-_-)，今天完善各个界面的逻辑和细节的时候又想起来了这档子事，于是抛下写到一半的代码，准备温习一下Activity进栈出栈的具体过程。~~又逮着码字的机会了了！！~~

​	Activity的出入栈控制主要来自于两个地方Activity的声明处——Manifest文件定义的属性和Activity的开启者——Intent的Flag定义。下面分别看看它们都能定义哪些属性。

<!--more-->

#### Manifest

​	Manifest中关于Activity栈的控制，大多数人最先接触到的大概就是`android:launchMode`的四个属性：standard、singleTop、singleTask和singleInstance，它们的意义如下：

```
1.standard
	Activity的默认启动模式，在任何情况下都直接将一个新的Activity实例入栈。
2.singleTop
	常用的Activity启动模式，当要开启的Activity已在栈顶时，不创建新的实例，但调用onNewIntent方法；不在栈顶时，创建新的实例。
3.singleTask
	当Activity为该启动模式时，会先在栈中查找是否存在该Activity实例，没有的话会新建一个实例，这个实例所在的栈与taskAffinity属性有关；如果当前栈中存在该Activity实例的话，则会将该实例送到栈顶，并且将其之前所有的Activity全部弹出。
4.singleInstance
	当Activity为该模式时，该Activity实例会有一个独立的栈，并且该栈中只会有这一个实例，所有跳转到这个Activity的进程都会共享这一个实例。当从这个栈切换到别的Activity时，会将新的Activity压入Activity启动前的栈中，此时再返回就会跳过这个singleInstance的Activity而回到开始的栈。
```

有两个需要注意的点，一是被singleTask标注的Activity已在别的进程的Task中存在且处于栈顶时，就不会在Activity调用进程创建一个新实例，而是将后台进程的Task栈内容压入当前Task栈，如下图所示：

![singleTask工作原理示意图](关于Activity栈/pic1.png)

二是Android的栈弹出策略：在当前Task中的Activity实例已全部弹出，就会回到该Task的上一个栈，而不是直接退回到主界面（也是因为这个原因让我把LoginActivity设置为singleTask从而建一个新栈，在onBackPress时直接回到主界面的想法落空了）。

除了launchMode管控的Activity启动策略之外Activity还有下面几个与返回栈相关的属性：

```
android:taskAffinity = "string"
	其值默认为当前包名，用于标识无归属Activity的归属任务。这里的无归属Activity指以设置了Flag为FLAG_ACTIVITY_NEW_TASK的Intent打开的Activity ，或者是launchMode为singleTask而当前Task中无其实例需要新建一个实例的Activity。它能够指定上述Activity所属的Task，或者说设定其亲缘关系。这个属性设置可以达到创建多个栈并改变其退出顺序的效果（比如实现以为不可能实现的退出MainActivity而不退出应用）
	
android:allowTaskReparenting = [true|false]
	其值表示是否允许将标注的Activity在当前Task转入后台时改变其所处Task，而归属到taskAffinity标注的Task中去，仅对launchMode为standard与singleTop的Activity有效，默认为false。一个具体的例子就是当我们的应用调用了一个外部浏览器时，当该属性为true，则打开的web页面将归属于浏览器，此时退出浏览器后会直接回到主界面；反之，该属性为false时，退出浏览器界面后将会回到原来的应用界面。
	
android:clearTaskOnLaunch=["true" | "false"]
	其值表示home键返回桌面又从桌面再次打开应用时，是否清除栈底以外的所有Activity（注意，因为Launcher入口只有一个，所以这里的栈指的就是MainActivity栈，其他栈不受影响）。
	
android:alwaysRetainTaskState=["true" | "false"]
	其值表示是否允许始终保持Task的状态(系统一般会在应用无响应半小时后清除根Activity之外的所有Activity)。
	
android:finishOnTaskLaunch=["true" | "false"]
	当该值为true时，标记的Activity在栈顶，且应用此时从桌面被启动时，该Activity将被finish，（但当Activity为根Activity时失效）。
```

更多相关的属性，参见官方文档：[<activity>标签](https://developer.android.google.cn/guide/topics/manifest/activity-element.html)

---

**Intent**

​	Intent作为Android四大组件的沟通桥梁，用Action确定了响应对象，用Category规定了响应条件，用Data描述了请求数据类型，至于开启新组件的属性和模式，则交给了Flag。下面是列举出来的几个Flag：

```
FLAG_ACTIVITY_NEW_TASK
	根据亲缘关系开启一个Activity，若此时存在与其有亲缘关系的栈，则在该栈中开启Activity，开启方式受launchMode约束；否则创建一个新栈。与FLAG_ACTIVITY_MULTIPLE_TASK一起使用时，无论如何都会创建一个新栈。
	
FLAG_ACTIVITY_CLEAR_TOP
	类似singleTask模式，在开启已存在栈中的Activity实例时，将其推至栈顶，并将其上的Activity全部出栈，但此时“推至栈顶”这个过程实际上是连已存在的相关Activity实例也一并弹出栈，而重新创建一个Activity实例，而不是调用NewIntent.
	
FLAG_ACTIVITY_CLEAR_TASK
	在目标Activity打开前清除其所在栈内的Activity实例，必须与FLAG_ACTIVITY_NEW_TASK一起用。
	
FLAG_ACTIVITY_SINGLE_TOP
	效果同singleTop.
	
FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS 
	由此标志开启的Activity所处栈不会出现在Recent界面，该Activity需要是这个栈的根Activity，其效果与activity=excludeFromRecents = "true"一致。

FLAG_ACTIVITY_TASK_ON_HOME 
	由此标识开启的Activity所处栈将会直接置于Home所处栈之上，也就是onBackPress时会直接退出到主界面。此标识需要同FLAG_ACTIVITY_NEW_TASK一同使用，并且必须保证其开启了一个新栈。

FLAG_ACTIVITY_NO_ANIMATION
	取消Activity的Transition效果。

```

更多的Flag参见：[Intent](https://developer.android.google.cn/reference/android/content/Intent.html#FLAG_ACTIVITY_NEW_TASK)

#### 最后

讲道理其实Activity的栈操作还是挺麻烦的一件事，还有很多属性都没有列举出来，以及startActivityForResult的一些特殊情况也并没有总结，~~关爱懒癌晚期患者~~。不过上面的在正常情况下应该是够用了，一般的场景也不会太多的去关注onBackPress会返回到什么界面（至今看到同应用多栈操作的貌似也就系统的Setting和WPS的多任务阅读），实在不行反正链接就在那，随时可以回去翻，就这样吧~