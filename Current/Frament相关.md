---
title: Fragment相关
date: 2018-07-09 14:14:02
tags: 笔记
---

> ​	项目中碰到了Fragment切换造成整个界面UI都混乱的情况，于是决定研究下Fragment的生命周期里具体都对UI做了什么，希望找到突破点。~~当然最后并没有折腾到最后~~

#### 从add()开始讲起

​	在对Fragment的使用中，除了对Fragment的对象构造，第一个动作应该就是用add()将其放到Activity准备的容器布局中，一般语句如下：

```java
getSupportFragmentManager().beginTransaction().add(container.getId(), fragment).commit();//这里用的v4包中的Fragment
```

首先，第一个方法，getSupportFragmentManager()，其获得的对象是一个FragmentManager抽象类的子类(也写在这个类的文件里，是一个final内部类)，即**FragmentManagerImpl**，这里要注意的是，**android.app**包里的Fragment调用getFragmentManager()也能得到一个FragmentManagerImpl对象，然而这个对象所属的类也在android.app包里，而不是上面的**v4**包，两者方法上有些差异，不能混为一谈。

<!--more-->

接着就是beginTransaction()，这个方法就就很简单，构造了一个**BackStackRecord**对象，它是**FragmentTransaction**的唯一官方实现子类，里面声明了各种对Fragment的操作方法，并定义了对Fragment栈存储所需要的一些数据结构。

然后重头戏来了，BackStackRecord 里的 add方法，它干了什么呢？上代码：

```java
@Override //重载方法一，主要用在DialogFragment的添加，没有制定相应的布局，而是直接加载在Window上
public FragmentTransaction add(Fragment fragment, String tag) {
        doAddOp(0, fragment, tag, OP_ADD);
        return this;
}

@Override //加载方式二，常用加载方式
public FragmentTransaction add(int containerViewId, Fragment fragment) {
        doAddOp(containerViewId, fragment, null, OP_ADD);
        return this;
}

@Override //加载方式三，利用tag作为Fragment的标记
public FragmentTransaction add(int containerViewId, Fragment fragment, String tag) {
        doAddOp(containerViewId, fragment, tag, OP_ADD);
        return this;
}    
```

而实际进行操作的doAddOp() 代码如下：

```java
private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
        final Class fragmentClass = fragment.getClass();
        final int modifiers = fragmentClass.getModifiers();
        //... 检查Fragment类是否为public独立类或者静态成员类，否则抛出相关异常

        fragment.mFragmentManager = mManager;

        //... 检查Tag是否变更，保证Fragment tag 标识的唯一性
  
        //... 当containerViewId不为0时，检查Fragment的容器是否存在，并不允许其变更容器

  		//构造新的Op数据，并加入栈的数据结构中
        Op op = new Op();
        op.cmd = opcmd;
        op.fragment = fragment;
        addOp(op); //具体代码见下面
}

void addOp(Op op) {
  		//一个典型的双向链表插入数据的过程
        if (mHead == null) {
            mHead = mTail = op;
        } else {
            op.prev = mTail;
            mTail.next = op;
            mTail = op;
        }
        //... 添加出入动画信息
        mNumOp++;//链表长度增加
}
```

事实上，Fragment的其他操作也大抵如此，只是链表节点对象的增删与**opcmd**不同而已。当所有的操作都变成了一个链，什么时候执行呢？当然是在commit()方法了，而它就是简单地调用了commitInternal方法，代码如下：

```java
int commitInternal(boolean allowStateLoss) {
        // ... 如果commit已经在本次事务中执行过了，则会抛出异常。没有的话，就给标志位置位
        if (mAddToBackStack) { //是否把Fragment保存在回退栈中，这个标志位只在调用了addToBackStack()后起效
            mIndex = mManager.allocBackStackIndex(this); //这里把Fragment放进了一个数组集合，并返回了集合的大小
        } else {
            mIndex = -1;
        }
        mManager.enqueueAction(this, allowStateLoss);//把整个动作链入列执行
        return mIndex;
}

```

在进行了一些必要的检验后，BackSackRecord就被送到了FragmentManager中进行操作了，于是把目光转回FragmentManager，下面的流程是这样的：

```java
public void enqueueAction(Runnable action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            checkStateLoss(); //校验saveInstanceState()是否被执行过
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<Runnable>();
            }
            mPendingActions.add(action);
            if (mPendingActions.size() == 1) { //正常状态下这个集合的大小必定满足条件
                mHost.getHandler().removeCallbacks(mExecCommit);
                mHost.getHandler().post(mExecCommit); //执行这个Runnable
            }
        }
    }
```

mExecCommit实际上是一个固定的Runnable对象，而这个对象的run语句只有一句话，那就是调用execPendingActions()，于是再来看看这个方法的代码：

```java
public boolean execPendingActions() {
        //... 校验该任务是否正在执行中，并且确保该方法在主线程中运行
        boolean didSomething = false;

        while (true) {
            int numActions;
            //其实不是很懂既然只允许在主线程中执行这个方法为什么还要加一个同步锁
            synchronized (this) {
              	//循环终止条件
                if (mPendingActions == null || mPendingActions.size() == 0) {
                    break;
                }
                
                numActions = mPendingActions.size();
                if (mTmpActions == null || mTmpActions.length < numActions) {
                    mTmpActions = new Runnable[numActions];
                }
                mPendingActions.toArray(mTmpActions);
                mPendingActions.clear(); //清空整个集合，这样首先会在执行动作完毕后终止这个循环，并在下一次动作链生效的时候就又会进入这个循环（满足.size() == 1的条件）
                mHost.getHandler().removeCallbacks(mExecCommit);
            }
            
            mExecutingActions = true;//任务执行中的标记
            for (int i=0; i<numActions; i++) {
                mTmpActions[i].run(); //开始执行任务链，这个run方法写在BackStackRecord中
                mTmpActions[i] = null;
            }
            mExecutingActions = false; 
            didSomething = true; //执行完成的标记
        }
        
        doPendingDeferredStart();//暂时先略过

        return didSomething;
}
```

所以在FragmentManager中哐哐一顿操作，最后实际的动作还是回到了BackStackRecord，所以继续去看run的操作：

```java
public void run() {
        //...检查加入回退栈是否在commit之前，否则抛出异常

        bumpBackStackNesting(1); //当允许Fragment进回退栈时，对其在回退栈中的层级做标识

        TransitionState state = null;
        SparseArray<Fragment> firstOutFragments = null;
        SparseArray<Fragment> lastInFragments = null;
        if (SUPPORTS_TRANSITIONS && mManager.mCurState >= Fragment.CREATED) {//此时的档Fragment已经被创建且Build.SDK>= Android.L
            firstOutFragments = new SparseArray<Fragment>();//存储已经被remove或hide的Fragment的查找表
            lastInFragments = new SparseArray<Fragment>();//存储已经被add的Fragment的查找表
			
          	//这个方法依次遍历了整个事务中操作记录链表中的内容，把add或show的Op放入lastInFragment，把add或hide的Op放入firstOutFragment，当对这两个查找表其中的一个进行插入时，也同时对另一个查找表进行删除操作（就像加减运算），并对未初始化完成的Fragment标记进入onCreate操作。
            calculateFragments(firstOutFragments, lastInFragments);

            state = beginTransition(firstOutFragments, lastInFragments, false);//应用Fragment的进出动画和View分享转换动画
        }

        //...
  
        Op op = mHead;
        while (op != null) { //遍历整条操作链表
            int enterAnim = state != null ? 0 : op.enterAnim;
            int exitAnim = state != null ? 0 : op.exitAnim;
            switch (op.cmd) {
                case OP_ADD: {
                    Fragment f = op.fragment;
                    f.mNextAnim = enterAnim; //添加动画
                    mManager.addFragment(f, false); //把Fragment加入mAdd和mActive集合中
                } break;
                case OP_REPLACE: {
                    Fragment f = op.fragment;
                    int containerId = f.mContainerId;
                    if (mManager.mAdded != null) {
                        for (int i = mManager.mAdded.size() - 1; i >= 0; i--) {
                            Fragment old = mManager.mAdded.get(i);
                            if (old.mContainerId == containerId) {
                                if (old == f) {
                                    op.fragment = f = null; //如果要替换的目标与当前Fragment相同，															则放弃此次操作，不显示转化动画
                                } else {
                                    if (op.removed == null) {
                                        op.removed = new ArrayList<Fragment>();
                                    }
                                    op.removed.add(old); //否则将其放入移除队列
                                    old.mNextAnim = exitAnim;
                                    if (mAddToBackStack) {
                                        old.mBackStackNesting += 1; //回退栈层级+1
                                    }
                                    //Fragment不在回退栈中或处于attach状态时，将其从mAdd中移除，并
                                    //设置为INITIALIZE或CREATE状态
                                    mManager.removeFragment(old, transition, transitionStyle);
                                }
                            }
                        }
                    }
                    if (f != null) {
                        f.mNextAnim = enterAnim;
                        mManager.addFragment(f, false);//同ADD操作
                    }
                } break;
                case OP_REMOVE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = exitAnim;
                    mManager.removeFragment(f, transition, transitionStyle);
                } break;
                case OP_HIDE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = exitAnim;
                    //设置了fragment.mView.setVisibility(View.GONE),顺便调了一下回调
                    mManager.hideFragment(f, transition, transitionStyle);
                } break;
                case OP_SHOW: {
                    Fragment f = op.fragment;
                    f.mNextAnim = enterAnim;
                  	//设置了fragment.mView.setVisibility(View.VISIBLE),顺便调了一下回调
                    mManager.showFragment(f, transition, transitionStyle);
                } break;
                case OP_DETACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = exitAnim;
                  	//设置标志位，从mAdd里移除fragment，这个case虽然进入的场景没那么多，但是其他case						做判断时都会先判断mDetach标志位是否为true，其跟remove的区别在于其不考虑回退栈的情					 	况，被detach的对象固定为CREATE状态。
                    mManager.detachFragment(f, transition, transitionStyle);
                } break;
                case OP_ATTACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = enterAnim;
                  	//与detach相反，最后的状态同当前Fragment的状态
                    mManager.attachFragment(f, transition, transitionStyle);
                } break;
                default: 
                    throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
            }
            op = op.next;
        }
		//mCurState的状态是由其宿主Activity决定的，在Fragment被初始配置时，这个状态会为最后的RESUME，			于是在正向生命周期中会将当前可见的Fragment状态全部变更为RESUME，不可见的为STOP
        mManager.moveToState(mManager.mCurState, transition, transitionStyle, true);

        if (mAddToBackStack) {
            mManager.addBackStackState(this);
        }
}
```

一大串代码下来，实际上做的动作都差不多：设置标志位，为集合添加或删除数据，设置动画，最后，都调用了一下**moveToState**（这里是每个case在*除了hide和show* 的操作的具体方法中都调用了，但是没有贴出来），然后再while循环结束后，又来了一次moveToState，所以这到底是个什么玩意儿？继续看：

```java
//看这个方法的实现之前，先明确Fragment状态常数的类型和值
static final int INITIALIZING = 0;     // Not yet created.
static final int CREATED = 1;          // Created.
static final int ACTIVITY_CREATED = 2; // The activity has finished its creation.
static final int STOPPED = 3;          // Fully created, not started.
static final int STARTED = 4;          // Created and started, not resumed.
static final int RESUMED = 5;          // Created started and resumed.

//以及各个方法跳转的状态
add:if (moveToStateNow) { moveToState(fragment); }//在正常情况不会触发
remove:moveToState(fragment, inactive ? Fragment.INITIALIZING : Fragment.CREATED,
                    transition, transitionStyle, false);
attach:moveToState(fragment, mCurState, transition, transitionStyle, false);
detach:moveToState(fragment, Fragment.CREATED, transition, transitionStyle, false);

void moveToState(Fragment f, int newState, int transit, int transitionStyle,
            boolean keepActive) {
  		// 未经过add的fragment只能为CREATE状态
        if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
            newState = Fragment.CREATED;
        }
        if (f.mRemoving && newState > f.mState) {
          	// remove一个Fragment，如果其新状态是CREATE，则其原始状态必须为INITIALIZING(因为只有这两			  种newState)
            newState = f.mState;
        }
  		// 在Fragment对用户不可见且其未启动过时，将其状态由STARTED变为STOPPED
        if (f.mDeferStart && f.mState < Fragment.STARTED && newState > Fragment.STOPPED) {
            newState = Fragment.STOPPED;
        }
  		//创建、重启、开始运行等Fragment的正向生命周期
        if (f.mState < newState) {
            //当Fragment是通过XML文件创建的时，除非其被reload，否则不允许启动
            if (f.mFromLayout && !f.mInLayout) {
                return;
            }  
            if (f.mAnimatingAway != null) {
                //该Fragment中的View正在运行动画，此时不等待动画执行完毕，直接跳到动画执行完应到的状态
                f.mAnimatingAway = null;
                moveToState(f, f.mStateAfterAnimating, 0, 0, true);
            }
            switch (f.mState) {
                //当前Fragment还未经过onCreate
                case Fragment.INITIALIZING:
                	//如果该Fragment的状态是销毁后重建，那么其保存的状态会由Bundle的形式在此复原
                    if (f.mSavedFragmentState != null) {
                        f.mSavedFragmentState.setClassLoader(mHost.getContext().getClassLoader());
                        f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                                FragmentManagerImpl.VIEW_STATE_TAG);
                        f.mTarget = getFragment(f.mSavedFragmentState,
                                FragmentManagerImpl.TARGET_STATE_TAG);
                        if (f.mTarget != null) { //这里的Target类似于startActivityForResult的发起对象
                            f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                    FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                        }
                        f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                                FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                        if (!f.mUserVisibleHint) {
                            f.mDeferStart = true;
                            if (newState > Fragment.STOPPED) {
                                newState = Fragment.STOPPED;
                            }
                        }
                    }
                    f.mHost = mHost;
                    f.mParentFragment = mParent;
                	//如果是Fragment嵌套结构，则由其父Fragment取得Manager对象
                    f.mFragmentManager = mParent != null
                            ? mParent.mChildFragmentManager : mHost.getFragmentManagerImpl();
                    f.mCalled = false;
                    f.onAttach(mHost.getContext()); //主动关联初始化的Fragment与当前Context，这个方法					 会对标志位进行置位
                    //... 发现上面的方法没置位时，会抛出异常
                    if (f.mParentFragment == null) {
                        mHost.onAttachFragment(f); //这里基本是一个空实现，什么都没做
                    } else {
                        f.mParentFragment.onAttachFragment(f); //同上
                    }
					//这里开始判断是否为状态复原
                    if (!f.mRetaining) {
                        f.performCreate(f.mSavedFragmentState);//onCreate的入口就在这里，在这个方法中						   mState也正式变成了CREATE
                    } else {
                        f.restoreChildFragmentState(f.mSavedFragmentState);//如果其有子Fragment的						 	话，也顺便将其复原，这个行为逻辑与View很像
                        f.mState = Fragment.CREATED;
                    }
                    f.mRetaining = false;
                    if (f.mFromLayout) {
                        //Fragment直接写在布局里的情况下，其mView是确定的，需要立刻导入
                        f.mView = f.performCreateView(f.getLayoutInflater(
                                f.mSavedFragmentState), null, f.mSavedFragmentState);
                        if (f.mView != null) {
                            f.mInnerView = f.mView;
                            if (Build.VERSION.SDK_INT >= 11) {
                              	//允许保存视图状态
                                ViewCompat.setSaveFromParentEnabled(f.mView, false);
                            } else {
                                f.mView = NoSaveStateFrameLayout.wrap(f.mView);
                            }
                            if (f.mHidden) f.mView.setVisibility(View.GONE);
                            f.onViewCreated(f.mView, f.mSavedFragmentState);//回调布局已经创建完毕
                        } else {
                            f.mInnerView = null;
                        }
                    } //这里没有break，所以上面初始化完毕后，后面会接下去执行！！
                //当Fragment已经初始化完毕
                case Fragment.CREATED:
                    if (newState > Fragment.CREATED) {
                        if (!f.mFromLayout) { //Fragment不来自XML文件时，在上一步并没有初始化布局，故在											  这一步做
                            ViewGroup container = null;
                            if (f.mContainerId != 0) {
                                //... 为NO_ID时抛出异常
                                container = (ViewGroup) mContainer.onFindViewById(f.mContainerId);
                                //... 导入布局失败时也抛出异常
                            }
                            f.mContainer = container;
                          	//继续调onCreateView
                            f.mView = f.performCreateView(f.getLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                            if (f.mView != null) {
                                f.mInnerView = f.mView;
                                //... 和上面一样，根据版本判断是否能保存数据
                                if (container != null) {
                                    //... 动画相关
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) f.mView.setVisibility(View.GONE);
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                            } else {
                                f.mInnerView = null;
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState); //回调onActivityCreated，							同时状态也变更为ACTIVITY_CREATED
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);//布局复位
                        }
                        f.mSavedFragmentState = null;
                    }
                case Fragment.ACTIVITY_CREATED:
                    if (newState > Fragment.ACTIVITY_CREATED) {
                        f.mState = Fragment.STOPPED; //这里会对启动进行进一步的拦截，使没有启动过的							Fragment终态为STOPPED 
                    }
                case Fragment.STOPPED:
                    if (newState > Fragment.STOPPED) {
                        f.performStart();//对于启动过的Fragment，在此状态下直接回掉onStart，同时状态变										更为START
                    }
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        f.performResume(); //最后一种情况，回调onResume，状态也变为RESUME
                        f.mSavedFragmentState = null;
                        f.mSavedViewState = null;
                    }
            }
        } else if (f.mState > newState) {
          	//Fragment的逆向生命周期
            switch (f.mState) {
                case Fragment.RESUMED:
                    if (newState < Fragment.RESUMED) {
                        f.performPause(); //回调onPause，状态变更为START
                    }
                case Fragment.STARTED:
                    if (newState < Fragment.STARTED) {
                        f.performStop(); //回调onStop，状态变更为STOP
                    }
                case Fragment.STOPPED:
                    if (newState < Fragment.STOPPED) {
                        f.performReallyStop();//真正调用LoaderManager的doStop方法的地方，将状态变更为						ACTIVITY_CRATE
                    }
                case Fragment.ACTIVITY_CREATED:
                    if (newState < Fragment.ACTIVITY_CREATED) {
                        if (f.mView != null) {
                            //如果Fragment没有保存状态，则将其状态保存
                            if (mHost.onShouldSaveFragmentState(f) && f.mSavedViewState == null) {
                                saveFragmentViewState(f);
                            }
                        }
                        f.performDestroyView();//回调onDestroyView，状态变更为CREATE
                        if (f.mView != null && f.mContainer != null) {
                            Animation anim = null;
                            if (mCurState > Fragment.INITIALIZING && !mDestroyed) {
                                anim = loadAnimation(f, transit, false,
                                        transitionStyle);
                            }
                            //... 如果动画设置了的话，就运行一下动画，并且跳到设定的动画后状态
                            f.mContainer.removeView(f.mView);
                        }
                        f.mContainer = null;
                        f.mView = null;
                        f.mInnerView = null;
                    }
                case Fragment.CREATED:
                    if (newState < Fragment.CREATED) {
                        if (mDestroyed) {
                            if (f.mAnimatingAway != null) {
                                //如果该Fragment的宿主已经被销毁，则停止动画的执行
                                View v = f.mAnimatingAway;
                                f.mAnimatingAway = null;
                                v.clearAnimation();
                            }
                        }
                        if (f.mAnimatingAway != null) {
                            //等待动画完成，并为其动画完成后置标志位
                            f.mStateAfterAnimating = newState;
                            newState = Fragment.CREATED;
                        } else {
                            //没有动画，如果需要存留，不需要的话就回调onDestroy()，最终状态都会为									INITIALIZING
                            if (!f.mRetaining) {
                                f.performDestroy();
                            } else {
                                f.mState = Fragment.INITIALIZING;
                            }

                            f.performDetach();//回调onDetach，同时再让子Fragment调一次onDestroy
                            if (!keepActive) {
                                if (!f.mRetaining) {
                                    makeInactive(f);
                                } else {
                                    f.mHost = null;
                                    f.mParentFragment = null;
                                    f.mFragmentManager = null;
                                }
                            }
                        }
                    }
            }
        }

        if (f.mState != newState) {
            f.mState = newState;
        }
    }
```

至此，Fragment的整个生命周期调用都在这里了，在每次Fragment被添加或删除，以及主Activity状态变更的时候，这个方法都会被调用，然后更新mAdd列表中的Fragmen的状态。

>​	然后第一章结束了，可能也只有这一章了，因为BUG还在那里，Fragment只是一个给View套了层生命周期外套的无辜者，继续DEBUG去了。

