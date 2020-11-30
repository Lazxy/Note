###Java多线程相关

####一些基本概念

1. 线程有创建-运行-临时堵塞-冻结-停止五种状态

2. 线程创建启动时会给其成员变量重新分配空间，除非定义static变量，否则它们的变量都是独立的。

3. 当一个线程被多次`start()`时，会发生线程异常错误。

4. `start()`和`run()`的区别是，前者会开始运行一个线程，而run()是直接以主线程运行线程内的`run()`函数。

5. 可以直接定义一个类实现Runnable重写`run()`方法，然后直接在构造Thread对象时将实现了Runnable接口定义类的对象为参数传入，这样开启线程时调用的`run()`方法为Runnable实现类中的`run()`方法。此时Runnable实现类种若有成员变量的话，它的成员变量是独立存在的（因为线程运行调用的Runnable对象是同一个），能同时被多个线程操作。由于Java的单继承方式，当其有自己的实际父类但又有一些代码需要在多线程操作时，就可以实现Runnable接口来达到目的。

6. `join()`方法和`yeild()`方法分别用于**等待调用该方法的线程结束**与**暂停当前线程**，其中join是通过wait机制进行循环等待的，在等待的线程结束后会调用notifyAll将其唤醒（这部分代码写在Jvm里）。

7. 用`synchronized`修饰符设定同步代码块或者函数，可以达到同步执行效果（即在持线程锁（或称监视器）的线程运行完成前无法其他线程执行`synchronized`内的内容），在其包括的代码内执行线程间共享数据来解决多线程安全问题。其所持的锁为任意一个对象，而同步函数的锁是调用这个函数的对象。当需要同步操作时，所有同步块用的锁必须一致，否则就会失去同步的效果。当同步嵌套并且所持锁都不一样时，会发生**死锁**现象，即两个线程都无法获得需要的锁，于是导致程序的停滞。其代码如下：

   ```java
   private static Object objectA = new Object();
       private static Object objectB = new Object();
   
       public static void main(String[] args) {
           new Thread(new TaskA()).start();
           new Thread(new TaskB()).start();
   
       }
   
       private static class TaskA implements Runnable {
           @Override
           public void run() {
               while (true) {
                   synchronized (objectA) {
                       LogUtil.println("Enter A");
                       synchronized (objectB) {
                           LogUtil.println("Enter B");
                       }
                   }
               }
           }
       }
   
       private static class TaskB implements Runnable {
           @Override
           public void run() {
               while (true) {
                   synchronized (objectB) {
                       LogUtil.println("Enter B");
                       synchronized (objectA) {
                           LogUtil.println("Enter A");
                       }
                   }
               }
           }
       }
   ```

   ​

8. 用`synchronized`同步操作的两个前提是一、同步代码块内的代码必须是两个或两个以上的线程都会操作到的内容。二、**同步锁必须一致**。

9. 当线程共享数据为静态变量时，静态同步函数的锁为`Class`对象。该对象先于本类的实例对象出现，是在static变量进内存时产生的**字节码文件**。而普通同步函数的锁是本类的实例对象，即**this**。

10. `volateil`修饰符可以实现线程共享变量的可见性统一——即在每个线程调用该变量时都**必须访问内存得到它的最新值**，然后拷贝为线程的变量副本，在完成关于变量的操作后立刻将副本的值修改入内存（需要是原子性操作，类似自增之类的多步操作不能起效），以此让每个线程取得的变量值都是同一个。除此之外`volatile`能够保证被其修饰变量所在的语句一定会按照代码顺序执行（将前后分为两个部分，两个部分内的顺序不保证，但代码一定会在这两个部分之间执行）。[参考文章](http://www.cnblogs.com/dolphin0520/p/3920373.html) 

11. 公平锁和非公平锁指的是等待锁的线程是以请求锁的顺序排队等待的还是随机抢占式获取锁的，synchronized是非公平锁，而ReetrantLock可以通过初始构造参数来设置是否使用公平锁。

####单例模式

饿汉式单例：
```java
public class Single{
private static final Single s = new Single();
	private Single(){}
	public static Single getInstance(){
		return this.s;
	}
}
```
懒汉式单例：
```java
public class Single{
	private static Single s = null;
	private Single(){}
	public static synchronized Single getInstance(){
		if(s==null)
          s=new Single();
      	return s;
	}
}
```
DLC单例：
```java
public class Single{
	private static volatile Single s = null;
	private Single(){}
	public static Single getInstance(){
		if(this.s==null){
			synchronized(Single.class){
              //这里的判断是为了让在synchronized前挂起的语句不至于再进来创建一个新的对象
				if(this.s==null){
					s=new Single(); 
                  	return s;
				}
			}
		}
      return s;
	}
}
```
静态类单例：
```java
public class Single{
	private Single(){}
	public static Single getInstance(){
	return SingleHolder.s; 	
	}
private static class SingleHolder(){
	private static final Single s = new Single();
	}
}
```
####线程间通信

1. 输入与输出线程为了保证安全，应该对共同操作的数据加同步锁，当需要双方线程动作有序时，可以用到`wait()` ` notify()`方法实现等待唤醒机制。`notify()`唤醒的线程为线程池中**第一个**开始等待（调用了`wait()`）的线程。这些相关方法定义在Object类中，因为它们都需要在同步环境下调用，且调用方需要是同步锁对象。等待唤醒机制种需要一些标识来阻止输入或输出单方面不断循环（来标识此时应该输出或者输入），而这样的标记判断应该由**while**来做。因为有多个线程共同进行输出或者共同输入时，会出现notify唤醒了同一方的线程而没有唤醒对方的线程，导致输入输出多次，故在唤醒后还需要进行标识判断是否应该进行接下来的动作。而相对应的，应该将`notify()`换为`notifyAll()`，确保对方一定会有线程被唤醒，防止循环判断后所有线程都处于等待状态，示例如下：

   ```java
   //被注释的代码部分为同步锁示例
   //private static Lock lock = new ReentrantLock(); 
   //private static Condition objectA = lock.newCondition();
   private static class TaskA implements Runnable {
           @Override
           public void run() {
               while(true) {
                   //lock.lock();
                   synchronized (objectA) {
                       try {
                         //如果这里还是if判断的话，可能会在唤醒同方线程时不作判断便开始执行下面的逻辑，造成不同步错误。
                           while(product != null) {
                               objectA.wait(); //放弃持有的同步锁，等待唤醒，这里的objectA就是一个普通的对象。
                             	//objectA.await();
                           }
                               LogUtil.println("Productor");
                               product = "";
                         //这里保证能够唤醒对方线程
                               objectA.notifyAll();
                         		//objectA.signalAll();
                       } catch (Exception e) {LogUtil.println(e.getMessage());}
                     	//finally{lock.unlock;}
                   }
               }
           }
       }

       private static class TaskB implements Runnable {
           @Override
           public void run() {
               while (true) {
                   synchronized (objectA) {
                       try {
                           while(product == null) {
                               objectA.wait();
                           }
                           LogUtil.println("Consumer");
                           product = null;
                           objectA.notifyAll();
                       }catch(Exception e){LogUtil.println(e.getMessage());}
                   }
               }
           }
       }
   ```

   ​

2. 在JDK1.5以后，开始用Lock接口实现类的对象显式地表达锁的获取和释放（`lock()`和`unlock()`），用**Condition**对象来代替作为监视器的Object对象，以`await()`和`signal()`方法来代替`wait()`和`notify()`。这种同步方式分离了对象和锁，使锁不需要以对象来作为标识，故可以用不同的Condition对象来控制不同的线程（见上面的注释）。

3. 由于`stop()`方法已经过时，中止线程的方法只有将`run()`方法中的代码执行完毕，其中一个方法是将循环标记改变。但如果在`run()`中线程出现了线程利用`wait()`或者`sleep()`方法冻结的情况，线程就将无法结束，此时可以用`interrupt()`方法中断其冻结状态，但此方法会丢出Interrupt异常。

4. 可以用`setDaemon()`方法来设置守护线程，或者说是后台线程，该标记下的线程运行与正常线程没有区别，区别在所有前台线程都结束时，虚拟机会自动退出该线程，即守护线程有伴生的特性。

5. `join()`方法即一个可操控的抢夺CPU执行权的方法，执行该方法后，执行该方法的线程对象会取得线程执行权，而当前线程会等待其执行完毕之后才从冻结状态中被唤醒。该状态也能被`interrupt()`唤醒。

6. 可以用`setPriority()`方式设定线程优先级，1最低，10最高。

7. 可以用`yield()`暂停当前线程，即释放当前执行权。

#### 线程池

1. 关于线程池，最关键的一个类是**ThreadPoolExecutor**，关于线池程的事务提交、计数和状态管理都通过调用该类的对象实现，其构造如下：

   ```java
   //最常用的构造
   public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
                             TimeUnit unit, BlockingQueue<Runnable> workQueue) {
           this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
                Executors.defaultThreadFactory(), defaultHandler);
       }
   //完全参数构造
   public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
                             TimeUnit unit, BlockingQueue<Runnable> workQueue,
                             ThreadFactory threadFactory, RejectedExecutionHandler handler) {
     /*全是对成员变量的赋值*/}
   ```

2. 下面需要分别解释一下各参数的含义：

   1. corePoolSize：核心线程池容量，表征一个**较轻的任务执行压力下允许的线程存在数量**。当线程池所需线程大于该值时，则需要将任务加入缓存队列，甚至构造新的线程。

   2. maximumPoolSize：最大线程池容量，表征**线程池数量的极限值**。在核心线程数量不满足需求且缓存队列已满的情况下，允许构造新的线程，但线程总数不能超过该值，否则会被拒绝执行任务或者直接抛出异常。

   3. keepAliveTime：线程存活时间，表征当线程池内线程数量大于核心线程池容量时，多余的临时线程能够以空闲状态存在的最大时间。另当`allowCoreThreadTimeOut()`被置为true时，空闲的核心线程也会以相同的标准被结束。

   4. unit：线程存活时间的单位。

   5. workQueue：任务缓存队列，当任务数大于核心线程池容量时用于存储未执行的任务，主要有下面几种：

      ```
      1) ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；

      2) LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；

      3) SynchronousQueue：可以视为一个永远为空的队列（或者一个任务传输管道），所有插入的任务都必须有一个线程直接取走执行，而不进行任何存留。

      4) DelayedWorkQueue：用二叉堆算法排序的无界延时序列，在最小延时任务到时前会处于阻塞状态。
      ```

   6. threadFactory：线程构造器，默认实现为构造一个新线程，并为其设置一个固定的线程组、自增的线程名，无守护线程且线程优先级为5。

   7. handler：任务拒绝策略，表征当需求线程数大于最大线程数时，对被提交的任务使用的拒绝策略，主要有下面几种：

      ```
      1) AbortPolicy：直接抛出一个RejectedExecutionException异常表示拒绝接受任务。

      2) DiscardPolicy：静默地拒绝任务，不做任何操作。

      3) DiscardOldestPolicy：抛弃队列顶端的任务，并重试执行excute；但如果线程池直接进入shutdown状态，则直接放弃当前任务。

      4) CallerRunsPolicy：直接在excute方法执行的当前线程执行任务；但如果线程池直接进入shutdown状态，则直接放弃当前任务。
      ```

3. Java提供了几种默认的ThreadPoolExecutor构造方式，如下：

   ```
   1) Executors.newFixedThreadPool(int nThreads)：构造一个核心线程池容量和最大线程池容量都为nThreads的线程池，其没有最大生存时间，同时缓存队列为Integer.MAX_VALUE大小的LinkedBlockingQueue。所有活动线程将存活到shutdown被调用。

   2) Executors.newSingleThreadExecutor()：构造一个只存在一个核心线程的线程池，但其缓存队列同样为Integer.MAX_VALUE大小的LinkedBlockingQueue。

   3) Executors.newCachedThreadPool()：构造一个无核心线程但最大线程数为Integer.MAX_VALUE的线程池，每个线程的空闲存活时间为60s，其缓存队列为SynchronousQueue。

   4) Executors.newSingleThreadScheduledExecutor()：构造一个只存在一个核心线程的定时线程池，线程池最大容量为Integer.MAX_VALUE，可以实现延时、定时、间隔执行任务等操作，其缓存队列为DelayedWorkQueue。

   5) Executors.newScheduledThreadPool(int corePoolSize)：构造一个核心线程池容量为corePoolSize的定时线程池，其余基本同上。
   ```

   **另外还有ForkJoinPool与ScheduledExecutorService的各种默认构造，这里先放着。**

4. 线程池在工作时，主要有五种状态，其含义如下：

   ```
   1) RUNNING: 正常的工作状态，接受并执行任务。

   2) SHUTDOWN: 即将关闭线程池，不接受新的任务，但会继续执行队列中的任务。 可由RUNNING状态通过调用shutdown()或者finalize()方法进入该状态。

   3) STOP: 线程池彻底停止，不接受新的任务，并会中断执行中的任务。 由前两种状态调用shutdownNow()进入。

   4) TIDYING: 所有任务都被终止，正在工作的线程数为0。 SHUTDOWN/STOP状态下缓存队列和线程池都为空时进入。

   5) TERMINATED: 彻底完成线程的终止。 terminated()方法执行完毕时进入。
   ```

5. 具体工作流程见下图：

   ![线程池任务提交流程](img\线程池任务提交流程.png)




```
需求：
	将需要处理的任务按照时间顺序传入线程池，核心线程数为CPU数量，要求仍按照时间顺序输出任务结果。
	即：输入 1-10号任务，在1号任务未执行完毕前，不允许输出2-10号任务；反之，当1号任务执行完毕后，若2号任务结果存在，则立刻输出2号任务结果。
	想法：另外维护一个时间戳信息队列和一个以时间戳信息为键的查找表，当线程池内任务完成时，对比当前任务的时间戳是否为最新时间戳，是则直接输出结果，并检查下一结果是否能够输出；否则将结果作为值加入查找表，等待之前的任务结束后取值。
	另，检测人脸的任务加锁进行，在完成任务后将标志位置位同时锁释放，在检测到之前，放弃人眼探测的线程任务。
```

