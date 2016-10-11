---
layout: post
title: 探秘RxJava之线程调度
---


## RxJava简介

RxJava里的Rx是Reactive Extensions的缩写，起源于.NET。它是一种编程思想，巧妙地结合了观察者模式以及函数式编程，使得异步操作的代码写起来简单而清晰。现在有越来越多的语言实现了这种思想，比如RxJava、RxJS、RxSwift等等。这篇文章就基于RxJava，探究一下线程调度的实现。

<!-- more -->

RxJava中数据源被表示成`Observable`，它们会产生一个个数据或者说事件。你可以通过`Subscriber`来订阅或者说监听这些事件。一个简单的示例：

	Observable.from(new String[]{"a","b","c"})
	            .subscribe(new Subscriber<String>() {
	                @Override
	                public void onCompleted() {
	                    
	                }

	                @Override
	                public void onError(Throwable e) {

	                }

	                @Override
	                public void onNext(String s) {

	                }
	            });

在这个例子中，`Observable`会读取给定`String`数组中的每一个元素并调用`Subscriber`的`onNext`方法，最后调用`Subscriber`的`onCompleted`。 可以看到`Subscriber`有三个回调方法：`onNext`可以订阅`Observable`产生的每一个事件；如果一切顺利，当所有事件都产生完后，`onCompleted`会被调用；如果中途发生错误，则`onError`会被调用，并且结束此次订阅。`onCompleted`和`onError`不会同时被调用。

当然，我们可以自定义事件产生的方式和时机。`Observable`提供了一个静态方法`create(OnSubscribe<T>)`，参数`OnSubscribe`是一个接口，我们需要实现其中的`call`。例如把上面的例子改造一下：

	Observable.create(new Observable.OnSubscribe<String>() {
	            @Override
	            public void call(Subscriber<? super String> subscriber) {
	                subscriber.onNext("a");
	                subscriber.onNext("b");
	                subscriber.onNext("c");
	                subscriber.onCompleted();
	            }
	        }).subscribe(new Subscriber<String>() {
	            @Override
	            public void onCompleted() {

	            }

	            @Override
	            public void onError(Throwable e) {

	            }

	            @Override
	            public void onNext(String s) {

	            }
	        });


可以看到在`OnSubscribe`的`call`里调用了三次`subscriber.onNext`，分别传入"a"、"b"和"c"，最后调用`subscriber.onCompleted`。这个例子的效果和上面的例子是一样的。

## 一个使用RxJava的例子

假设现在有一个这样的需求：app要向服务器发送两个请求，服务器的返回的结果要么有效要么无效，而我们只需要取其中一个有效的结果就行。也就是说，如果第一个请求返回的结果是有效的，我们就可以直接使用它而忽略第二个请求的返回结果；如果第一个结果是无效的，那就需要等待第二个请求的结果。

如果不用RxJava，可能这样实现这个需求：

	boolean gotOne;
	new TestTask1().execute();
	new TestTask2().execute();


	class TestTask1 extends AsyncTask<Void, Void, String> {

	    @Override
	    protected String doInBackground(Void... params) {
	        try {
	            Thread.sleep(1500);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	        return "1";
	    }

	    @Override
	    protected void onPostExecute(String s) {
	        super.onPostExecute(s);
	        if (!gotOne && !TextUtils.isEmpty(s)) {
	            Log.i("Test", "get:"+s);
	            gotOne = true;
	        }
	    }
	}

	class TestTask2 extends AsyncTask<Void, Void, String> {

	    @Override
	    protected String doInBackground(Void... params) {
	        try {
	            Thread.sleep(2000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	        return "2";
	    }

	    @Override
	    protected void onPostExecute(String s) {
	        super.onPostExecute(s);
	        if (!gotOne && !TextUtils.isEmpty(s)) {
	            Log.i("Test", "get:"+s);
	            gotOne = true;
	        }
	    }
	}

这里使用两个`AsyncTask`来发送两个请求。在每个`doInBackground`方法里用`sleep`来表示网络请求的延迟。假设服务器会返回一个字符串，字符串非空表示结果有效。为了实现只取其中一个有效结果的需求，需要一个bool变量`gotOne`来记录是否已经得到了一个有效结果。

再看看用RxJava可以怎么实现：

	Observable<String> observable1 = Observable.create(new Observable.OnSubscribe<String>() {
	    @Override
	    public void call(Subscriber<? super String> subscriber) {
	        try {
	            Thread.sleep(1500);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	        subscriber.onNext("1");
	        subscriber.onCompleted();
	    }
	}).subscribeOn(Schedulers.io());
	Observable<String> observable2 = Observable.create(new Observable.OnSubscribe<String>() {
	    @Override
	    public void call(Subscriber<? super String> subscriber) {
	        try {
	            Thread.sleep(2000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	        subscriber.onNext("2");
	        subscriber.onCompleted();
	    }
	}).subscribeOn(Schedulers.io());

	Observable.merge(observable1, observable2)
	        .filter(new Func1<String, Boolean>() { //过滤掉无效结果
	            @Override
	            public Boolean call(String s) {
	                return s != null;
	            }
	        })
	        .take(1) // 只取第一个有效结果
	        .observeOn(AndroidSchedulers.mainThread())
	        .subscribe(new Subscriber<String>() {
	            @Override
	            public void onCompleted() {
	                Log.i("Test", "onCompleted");
	            }

	            @Override
	            public void onError(Throwable e) {
	                Log.i("Test", "onError:"+e.getMessage());
	            }

	            @Override
	            public void onNext(String s) {
	                Log.i("Test", "onNext"+s);
	            }
	        });


使用两个`Observable`来进行两个网络请求。在链式调用中使用了一些操作符来实现需求：首先`merge`两个`Observable`表示要发送两个网络请求，然后通过`filter`过滤掉无效结果，再用`take(1)`取出第一个有效结果。注意到这里还使用了后面将分析到的两个线程调度的方法：创建`Observable`的时候使用`subscribeOn(Schedulers.io())`表示在子线程中进行网络请求；后面调用`observeOn(AndroidSchedulers.mainThread())`表示在主线程中处理请求结果。

乍一看，两种方法的代码量似乎没有太大差别。当然使用RxJava并不是说一定能减少代码量，但是它可以使代码逻辑变得简洁清晰。而且在需求改变的时候，我们可以快速进行修改。比如说现在需求变成从n个请求里取出前k个有效的结果。使用`AsyncTask`的方法需要将bool变量改成一个计数器；而用RxJava的话只需把`take`的参数改成k。如果难度再次升级，现在需要所有请求都在同一个线程中进行。使用`AsyncTask`的方法就要把所有发送请求的代码放到同一个`doInBackground`里，而且每得到一个结果我们还要想办法去通知主线程，可想而知改动的代码很多；用RxJava只需把创建`Observable`的时候调用的`subscribeOn(Schedulers.io())`移到下面的链式调用里即可，一切就是这么简单！

## 线程调度

RxJava中有两个方法跟线程调度有关，分别是`subscribeOn`和`observeOn`. 在分析这两个方法之前，先来看一下最简单的一个事件产生－订阅的流程是怎样的。代码如下：

	Observable.create(new Observable.OnSubscribe<Object>(){/*...*/})
	          .subscribe(new Subscriber<Object>(){/*...*/});

`Observable.create`非常简单，直接返回一个`Observable`对象：

	public static <T> Observable<T> create(OnSubscribe<T> f) {
	     return new Observable<T>(hook.onCreate(f));
	}

这里的`hook`定义为：

	static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();

类`RxJavaObservableExecutionHook`的JavaDoc解释是

> Abstract ExecutionHook with invocations at different lifecycle points of Observable execution with a default no-op implementation.

也就是说在`Observable`执行期间的某些时间点会调用这个`ExecutionHook`的回调函数，你可以在这些回调函数里做一些事情。默认的实现是什么都不做。因此`hook.onCreate(f)`直接返回`f`. 

再看一下`Observable`的构造函数：

	protected Observable(OnSubscribe<T> f) {
	    this.onSubscribe = f;
	}

也很简单，将成员变量`onSubscribe`指向给定的`OnSubscribe`实现。

接着看一下`subscribe`函数做了什么事，这里将相关代码进行了简化，把最关键的几处提取了出来：

	subscriber.onStart();
	        
	if (!(subscriber instanceof SafeSubscriber)) {
	    subscriber = new SafeSubscriber<T>(subscriber);
	}
	
	try {
	    // allow the hook to intercept and/or decorate
	    hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
	    return hook.onSubscribeReturn(subscriber);
	} catch (Throwable e) {/*...*/}

首先调用`Subscriber`的`onStart`方法，你可以在这里做一些准备操作。然后将给定的`Subscriber`封装成`SafeSubscriber`以保证在整个订阅过程都能遵守Rx的规范。下面的`hook.onSubscribeStart`在默认情况下仍然是什么都不做而直接返回`observable.onSubscribe`，因此最后`observable.onSubscribe`的`call`被调用。前面已经看到了在`call`中可以决定什么时候产生事件并通知`Subscriber`。

### subscribeOn

`subscribeOn`可以决定事件在哪个线程产生。看一下函数的实现：

	public final Observable<T> subscribeOn(Scheduler scheduler) {
	    if (this instanceof ScalarSynchronousObservable) {
	        return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
	    }
	    return create(new OperatorSubscribeOn<T>(this, scheduler));
	}

函数接收一个参数`Scheduler`. 这个`Scheduler`就是实现线程调度的关键类。它里面有一个抽象方法：

	public abstract Worker createWorker();

这个方法返回一个`Worker`类对象。`Worker`的解释是

> Sequential Scheduler for executing actions on a single thread or event loop.

也就是说`Scheduler`将线程调度交给了具体的`Worker`。在`Worker`中有两个关键方法：

	public abstract Subscription schedule(Action0 action);
	public abstract Subscription schedule(final Action0 action, final long delayTime, final TimeUnit unit);

在这两个方法的具体实现中就可以决定在哪个线程去执行`action`。RxJava提供了一系列接口`ActionN`（N取值从0到9）。这些接口里都只有一个返回值为`void`的方法`call`，而`call`所接受的参数个数就是由`N`确定。比如这里的`Action0`就表示没有参数。与`ActionN`相对应有一个系列接口叫`FuncN<R>`，它也只有一个函数`call`，所不同的是`call`会返回类型为`R`的返回值。

RxJava提供了一个工厂类`Schedulers`，它里面有一些静态方法返回可以满足不同需求的`Scheduler`。在上面的例子中就用到了`Schedulers.io()`。这个方法返回的`Scheduler`维护了一个线程池。`Woker`提交的`Action`，有可能是在一个新的线程中执行，也有可能在线程池中某一个空闲线程中执行。

回到`subscribeOn`的函数体。通常的`Observable`都不会是`ScalarSynchronousObservable`，所以跳过这个`if`语句直接看接下来的返回语句。调用`create`函数产生一个新的`Observable`。之前已经看到`create`接受的参数是`OnSubscribe`接口的实现，在这里就是`OperatorSubscribeOn`。看一下它的源码：

	public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {
	
	    final Scheduler scheduler;
	    final Observable<T> source;
	
	    public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
	        this.scheduler = scheduler;
	        this.source = source;
	    }
	
	    @Override
	    public void call(final Subscriber<? super T> subscriber) {
	        final Worker inner = scheduler.createWorker();
	        subscriber.add(inner);
	        
	        inner.schedule(new Action0() {
	            @Override
	            public void call() {
	                final Thread t = Thread.currentThread();
	                
	                Subscriber<T> s = new Subscriber<T>(subscriber) {
	                    @Override
	                    public void onNext(T t) {
	                        subscriber.onNext(t);
	                    }
	                    
	                    @Override
	                    public void onError(Throwable e) {
	                        try {
	                            subscriber.onError(e);
	                        } finally {
	                            inner.unsubscribe();
	                        }
	                    }
	                    
	                    @Override
	                    public void onCompleted() {
	                        try {
	                            subscriber.onCompleted();
	                        } finally {
	                            inner.unsubscribe();
	                        }
	                    }
	                    
	                    @Override
	                    public void setProducer(final Producer p) {
	                        subscriber.setProducer(new Producer() {
	                            @Override
	                            public void request(final long n) {
	                                if (t == Thread.currentThread()) {
	                                    p.request(n);
	                                } else {
	                                    inner.schedule(new Action0() {
	                                        @Override
	                                        public void call() {
	                                            p.request(n);
	                                        }
	                                    });
	                                }
	                            }
	                        });
	                    }
	                };
	                
	                source.unsafeSubscribe(s);
	            }
	        });
	    }
	}

在构造函数里会记下传进来的`Scheduler`以及原来的`Observable`。然后是`call`方法。所做的事情很简单，通过给定的`Scheduler`的`Worker`提交一个`Action0`。在`Action0`里，把传入的`Subscriber`封装到一个新的`Subscriber`里，然后调用原`Observable`的`unsafeSubscribe`方法去产生事件并通知`Subscriber`。从代码中可以看到这个封装的`Subscriber`做的事情也很简单，就是调用原`Subscriber`相应的方法。有一个较为复杂的方法是`setProducer`，这个函数的作用涉及到一个概念叫`Backpressure`，不在这篇文章里详述。通过代码可以看到重写这个方法也是为了让操作能在指定的线程执行。

`subscribeOn`所做的事情可以用下面一张图进行概括：

![subscribeOn]({{ site.url }}/lib/images/Rxjava_subscribeOn.png)

### observeOn

`observeOn`可以决定在哪个线程消费事件。`Observable`重载了多个`observeOn`方法，最终都是调用下面这个：

    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));
    }

直接看return语句，这里调用了`lift`方法返回一个新的`Observable`。`lift`在RxJava中是一个比较关键的方法，很多操作符都是通过它实现。所以先看一下`lift`的实现。

    public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribeLift<T, R>(onSubscribe, operator));
    }

`lift`接受一个参数`Operator`，`Operator`的定义为：

    public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>> {}

它是一个接口继承自`Func1<Subscriber<? super R>, Subscriber<? super T>>`，也就是说它里面有一个`call`方法，参数是一个`Subscriber`，最后会返回一个`Subscriber`。

回到`lift`，方法体非常简单，直接返回一个新的`Observable`。新`Observable`的`onSubscribe`是一个`OnSubscribeLift`对象，并且原`Observable`的`onSubscribe`以及`lift`的参数`operator`成为了`OnSubscribeLift`的构造函数的参数。接下来就看一下`OnSubscribeLift`的实现。

    public final class OnSubscribeLift<T, R> implements OnSubscribe<R> {
        
        static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();
    
        final OnSubscribe<T> parent;
    
        final Operator<? extends R, ? super T> operator;
    
        public OnSubscribeLift(OnSubscribe<T> parent, Operator<? extends R, ? super T> operator) {
            this.parent = parent;
            this.operator = operator;
        }
    
        @Override
        public void call(Subscriber<? super R> o) {
            try {
                Subscriber<? super T> st = hook.onLift(operator).call(o);
                try {
                    st.onStart();
                    parent.call(st);
                } catch (Throwable e) {
                    revents onErrorResumeNext and other similar approaches to error handling
                    Exceptions.throwIfFatal(e);
                    st.onError(e);
                }
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                o.onError(e);
            }
        }
    }

在`call`方法里，首先调用`operator.call`将传入的`Subscriber`对象`o`转换成一个新的`Subscriber`对象`st`。然后调用`parent.call(st)`，这条语句的效果相当于让新的`Subscriber`订阅原`Observable`。

`lift`的主要功能可以总结如下：

1. 生成一个新`Observable`，记下原`Observable`的`onSubscribe`。
2. 当一个`Subscriber`订阅这个新`Observable`时，先通过给定的`Operator`将`Subscriber`转换成一个新的`Subscriber`。
3. 用新的`Subscriber`去订阅原`Observable`。

现在回到`observeOn`，看一下它的return语句

    return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));

这里的`OperatorObserveOn`就是`Operator`的具体实现，它的`call`方法如下：

    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }

如果提供的`Scheduler`既不是`ImmediateScheduler`也不是`TrampolineScheduler`的话，则返回`ObserveOnSubscriber`对象，否则直接返回原`Subscriber`。

再来看一下`ObserveOnSubscriber`的实现，这里只摘取关键代码。

    private static final class ObserveOnSubscriber<T> extends Subscriber<T> implements Action0 {
        final Subscriber<? super T> child;
        final Scheduler.Worker recursiveScheduler;
        ......
    
        public ObserveOnSubscriber(Scheduler scheduler, Subscriber<? super T> child, boolean delayError, int bufferSize) {
            this.child = child;
            this.recursiveScheduler = scheduler.createWorker();
            ......
        }
    
        @Override
        public void onNext(final T t) {
            if (isUnsubscribed() || finished) {
                return;
            }
            if (!queue.offer(on.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
            schedule();
        }
    
        @Override
        public void onCompleted() {
            if (isUnsubscribed() || finished) {
                return;
            }
            finished = true;
            schedule();
        }
    
        @Override
        public void onError(final Throwable e) {
            if (isUnsubscribed() || finished) {
                RxJavaPlugins.getInstance().getErrorHandler().handleError(e);
                return;
            }
            error = e;
            finished = true;
            schedule();
        }
    
        protected void schedule() {
            if (counter.getAndIncrement() == 0) {
                recursiveScheduler.schedule(this);
            }
        }
    
        ......
    }

首先看一下构造函数。成员变量`child`指向原`Subscriber`，`recursiveScheduler`指向给定`Scheduler`的`Worker`。
再看一下`onNext`、`onError`以及`onComplete`这三个函数，注意到最后它们都调用了`schedule`方法。而`schedule`也很简单，
通过`recursiveScheduler`提交一个`Action0`。由于`OperatorObserveOn`实现了`Action0`，所以这里就是`OperatorObserveOn`对象本身。
在它的`call`方法里会调用原`Subscriber`的`onNext`、`onError`或者`onComplete`，具体代码在这就不作详述。
这个时候原`Subscriber`相应的那些方法已经是在`Worker`指定的线程里执行了，因此实现了线程调度的效果。

## 总结

这篇文章通过分析`subscribeOn`和`observeOn`两个主要函数的源码讲解了RxJava的线程调度。其实RxJava还是很博大精深的，可讲解的内容有很多，例如各种操作符的使用、backpressure策略、并发处理等等。我也只是刚刚入门，还需要花时间继续研究。希望在后续的文章中可以带来更多RxJava的分析。
