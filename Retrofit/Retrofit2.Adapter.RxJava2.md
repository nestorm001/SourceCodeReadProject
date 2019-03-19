### 闲来无事看 Retrofit 源码

#### 前言
`Retrofit` 作为一个每天都要打交道的网络库，看看源码似乎也成了顺理成章的事情。考虑到日常使用中，还有配套的周边产品，不如先从相对简单点的 `RxJava2CallAdapter` 开始入手。
本篇基于 adapter-rxjava2-2.4.0 中的源码。

#### 结构
本库相对简单，内部包含的类如下
* BodyObservable
* CallEnqueueObservable
* CallExecuteObservable
* HttpException
* Result
* ResultObservable
* RxJava2CallAdapter
* RxJava2CallAdapterFactory

##### HttpException
@deprecated Use {@link retrofit2.HttpException}
略过

##### RxJava2CallAdapterFactory
工厂类根据 `Retrofit` 调用创建 `RxJava2CallAdapter`,可设置 `Scheduler` 和是否为异步。

##### RxJava2CallAdapter
根据是否为异步调用创建 `CallEnqueueObservable` 或 `CallEnqueueObservable`，并将其根据需求转化为需要的 `Observable`。

##### Result
对请求返回结果的包装。

##### ResultObservable
根据请求结果分发 `Result` 中的内容。

##### BodyObservable
没细看，没用过，不想管，只知道 `rawObservableType` 不是 `Result` 或 `Response` 时会被认为是 `Body`。

##### CallEnqueueObservable
请求会被放入请求队列，不会自动调用 `onComplete`。

##### CallExecuteObservable
相对于 `CallEnqueueObservable` 请求立即执行，在 `onResponse` 后会自动调用 `onComplete`。

#### 最后
写得真是粗俗，粗鄙，就当记录一下，反正也不会有人看的。
