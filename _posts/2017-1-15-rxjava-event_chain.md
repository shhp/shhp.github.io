---
layout: post
title: 理解RxJava的事件传递链
---


我们使用RxJava时会将业务逻辑抽象成一条函数链。实际上RxJava中的各种事件（包括subscribe、onNext、onError等）也是沿着这条链条传递的。

<!-- more -->

假设有这么一段抽象的RxJava代码：

	O1
	.operator1(...) 
	.operator2(...) 
	.subscribe(S1)

其中`O1`是真正产生数据的`Observable`，然后将两个操作符`operator1`和`operator2`作用到`O1`上，最后用`S1`进行订阅。一般而言操作符有两种实现方式：1.
 实现`OnSubscribe`；2.
 通过`lift`函数进行变换。这里假定操作符`operator1`和`operator2`都是通过函数`lift`来实现。

表面看来这里只有一个`Observable`和一个`Subscriber`，但其实操作符会生成一些中间的`Observable`和`Subscriber`。下面这张图对此进行了阐释。

![]({{ site.url }}/lib/images/rxjava_encapsulation.png)

操作符对`Observable`的封装是从上到下的，而对`Subscriber`的封装正好相反。所以最后一句`.subscribe(S1)`实际上是用`S1`去subscribe`O3`。

接下来就看一下subscribe事件是如何传递的。仍然用一张图进行说明：

![]({{ site.url }}/lib/images/rxjava_subscribe_sequence.png)

subscribe事件是由下至上传递的。最后触发`S3` subscribe `O1`，进而引发数据产生。

在`O1`的`OnSubscribe`里调用`subscriber.onNext`、`subscriber.onError`以及`subscriber.onCompleted`实际上都是调用`S3`对应的回调。然后`S3`在接收到数据后会进行相关处理，这取决于`operator1`的功能;数据处理完后再传递给`S2`，`S2`对数据处理完后传给`S1`。示意图如下：

![]({{ site.url }}/lib/images/rxjava_data_sequence.png)

因此数据的传递是从上至下的。

理解了RxJava中的事件传递链，再去看RxJava相关的代码就会容易很多了。
