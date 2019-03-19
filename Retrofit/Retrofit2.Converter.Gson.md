### 闲来无事看 Retrofit 源码

#### 前言
`Retrofit` 作为一个每天都要打交道的网络库，看看源码似乎也成了顺理成章的事情。考虑到日常使用中，还有配套的周边产品，现在来看看更简单的 `GsonConverter`。
本篇基于 converter-gson-2.4.0 中的源码。

#### 结构
本库极其简单，内部包含的类如下
* GsonConverterFactory
* GsonRequestBodyConverter
* GsonResponseBodyConverter

##### GsonConverterFactory
继承于 `Retrofit` 中的 `Converter.Factory`，根据是 `request` 还是 `response` 在回调中创建 `converter`。

##### GsonRequestBodyConverter
使用 `Gson` 将 `RequestBody` 转换成 `json`。

##### GsonResponseBodyConverter
使用 `Gson` 将 `ResponseBody` 转换成 `json`。

#### 最后
这，太简单了，跟个没写一下，就这样吧。
