>-> 表示被嵌套或者调用顺序
>
> --> 表示中间参数
>
> [1]up[2] / [1]down[2]表示当前类型为[1]的Observer的上游/下游为[2]

Observable**Origin** -> Observable**SubscibeOn** -> Observable**ObserveOn** 

Observable**ObserveOn** 

- -> `subscribeActual`**[OriginObserbver]**

- - Observable**SubscibeOn**

  - ->`subscribeActual`**[ObserveOn**Observer]_down_[**Origin**Observer]

  - - **[ObserveOn**Observer]_down_[**Origin**Observer]

  - - ->`onSubscribe`**[SubscriberOn**Observer]_down_**[ObserveOn**Observer]

    - - [**Origin**Observer] 
    - - -> `onSubscribe`**[ObserveOn**Observer]_down_[**Origin**Observer]up**[SubscriberOn**Observer]

  - - **[SubscriberOn**Observer]_down_**[ObserveOn**Observer]_down_[**Origin**Observer]
    - -> `setDisposable`

    - ScheduledRunnbale->`run`

    - - DisposeTask -> `run`
    - - - SubscribeTask -> `run`
    - - - - [Observable**Origin]** 
    - - - - ->`subscribe` **[SubscriberOn**Observer]_down_**[ObserveOn**Observer]_down_[**Origin**Observer]
    - ...
    - - DisposeTask -> dispose()
    - EventLoopWorker ->dispose()
    - ScheduledRunnable-> dispose()



SubscribeTask(**[SubscriberOn**Observer]_down_**[ObserveOn**Observer])

-> DisposeTask

->ScheduledRunnable

