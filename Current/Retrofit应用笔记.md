### Retrofit 应用笔记

#### 执行流程

- Retrofit对象调用`create`方法
- 通过反射invoke接口方法的方式获得method相关信息，主要是各种注解
- - 通过ServiceMethod.parseAnnotations获得可执行方法对象，并进行缓存
  - - 先通过__RequestFactory.parseAnnotaions__来构造一个Request工厂对象，里面包含了请求的类型、url及请求体等信息
    - 再用__HttpServiceMethod.parseAnnotations__方法进行实际的方法对象获得
    - - 先传入`ResponseT`（这里就是用户关心的数据实体类型），通过CallAdapterFactory构造出一个`CallAdapter<ResonseT,ReturnT>`，这里就是**RxJava2CallAdapterFactory**,`ReturnT`类型即Observable\<ResponseT>类型

      - 在这之后，通过ConverterFactory来构造一个`Converter<ResponseBody,ResponseT>`类型的Converter，前者即**GsonConverterFactory**

      - 接着，OkHttp需要一个`callFactory`，这里如果不额外设置，则Retrofit的callFactory就是默认的**OkHttpClient**

      - 最后会传入上述参数构造一个`CallAdapted`对象，它是HttpServiceMethod的一个静态内部子类，invoke实现继承父类实现，即构造一个**OkHttpCall**对象，并调用`adapt`方法，这是个抽象方法，CallAdapted的子类实现也很简单，就是通过上面得到的callAdapter对象调用了一次adapt方法。

      - 然后就是喜闻乐见的RxJava处理流程

      - - 首先RxJava根据调用类型构造了一个Observable对象，这里是同步调用，所以是**CallExecuteObservable**，然后把这个Observable对象包上一层**BodyObservable**包装，这层包装又给传入的Observer对象加了一层**BodyObserver**包装，它实现了这样一个逻辑：**如果这个网络请求被标记为请求成功（`response.isSuccessful()`），则将相应的Body传给onNext（此时这个Body就是我们的`Observable<RespnseT>`）;否则，调用onError方法，传给其一个HttpException，并且阻止调用onComplete**。

        - 所以经过上面的层层封装后，整个调用链逻辑是这样的：

          ```
          BodyObservable（在接口定义的返回值的实际实现类型） -> CallExecuteObservable -> BodyObserver -> Observer(实际在调用接口的时候定义的)
          ```

          于是可见OkHttp的操作封装在CallExecuteObservable内，其实也就是调用了一下`call.execute()`方法，然后在没有被调用`disposed()`方法的情况下，**按顺序调用onNext、onComplete，或者在前者出现异常时自动走向onError**，结束整个流程。

      - 在这里顺带描述一下`call.execute()`究竟做了什么：

      - - OkHttpCall中执行这个方法时，实际上走的是**RealCall**的execute方法，这个方法里又调用了OkHttpClient对象中dispatcher对象的executed方法，把这个请求加入异步请求队列（所以这个对象也可以用来控制队列长度）

        - 然后，Response对象是通过**getResponseWithInterceptorChain()**方法得到的，这个方法通过各种`interceptors`的添加描述了OkHttp整个网络请求的主要流程，包括前置的**失败重试设置、HTTP报文拼接（这里就包括了对Cookie的读取）、请求-响应缓存以及网络连接检查**，以及最重要的**请求发起-响应接收流程**（**CallServerInterceptor**）。

        - 在一次正常的请求中，response最终会走完上面的请求链，并返回交由RxJava处理，但在返回前会调用一次dispatcher对象的finish方法。

          >还有一个返回类型为空的`execute`方法在读代码的时候不幸被搅混了一次，这个方法应该是异步请求执行时调用的方法，其最后会通过responseCallback对象来进行结果的回调。

