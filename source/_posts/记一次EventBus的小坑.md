---
title: 记一次EventBus的小坑
date: 2017-06-15 20:35:22
tags: 疑难杂症
---
# 起因

**此博客纯属扯淡，后来进过本人测试，发现是自己代码有问题，跟 EventBus 没有半毛钱关系。**

最近在做公司项目的时候，大量的使用了 EventBus 。用的不亦乐乎，但是今天突然遇到了一个很奇怪的问题。

<!-- more -->

在接入微支付的时候，微信的回调在一个 Activity 中，所以就在里面是用 EventBus 来通知界面更新，本来没有什么大问题。可是后面我在封装支付工具类的时候，发现 EventBus 发不出去，反复检查也没发现问题。刚开始以为是自己代码写的有问题，后来进入 EventBus 的源码一看，才知道原来跟 EventBus 有关。

# 在接收EventBus Event的方法中在发Event

``` java

//伪代码如下，在支付的工具类中接收 微信支付的Event

//接收微信支付成功回调
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessageEvent(String event){
    if(支付成功){
        //通知Ui更新
        EventBus.getDefault().post(object);
    }
}

@Subscribe(threadMode = ThreadMode.MAIN)
public void onRefreshUI(Object object){
    //更新UI
}

```

然而就上面这几行简单的代码却怎么也调用不了更新 Ui 的代码.于是我跟进源码，一探究竟。

# EventBus 中的 ThreadLocal 

``` java

public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    //问题就在这里
    if (!postingState.isPosting) {
        postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                 postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}

```

观察上面代码，currentPostingThreadState 其实就是一个 ThreadLocal，用来存储一些在多个线程中要分开的属性，其中有个属性叫 isPosting 是一个 boolean 类型的，如果这个值是 true 的话 就不会想下执行了，也就说 如果当前调用 post 方法的线程是被 post 调用的(接收 Event 的方法)，那么在里面就无法再使用EventBus.post。

# 解决方法

``` java

//接收微信支付成功回调
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessageEvent(String event){
    if(支付成功){
        new Thread(new Runnable() {
            @Override
            public void run() {
                //通知Ui更新
                EventBus.getDefault().post(object);
            }
        });
    }
}

// or ....

@Subscribe(threadMode = ThreadMode.BACKGROUND or ASYNC)
public void onMessageEvent(String event){
    if(支付成功){
        //通知Ui更新
        EventBus.getDefault().post(object);
    }
}

```

解决的办法就是在接收EventBus的方法中开一个线程去调用 post ，或者让接收EventBus的方法运行在其他的线程中。因为 ThreadLocal 的存在，所以必须要换一个线程。

那么，为什么 EventBus 要这么设计呢？ 通过源码分析。EventBus 的原理就是通过反射调用方法，如果 post 和 接收 Event 的方法在同一个线程中，那么就跟直接调用方法没什么区别(只不过EventBus帮我们调用了)。我们假设 如果 EventBus 不这么设计。我们在更新 UI 的地方有使用 EventBus post 一个 Event 那么这样就会让最开始 post 的地方一直处于柱塞状态(比如上面微信支付的回调)。如果遇见耗时的操作 这就会触发 ANR 异常。所以 EventBus 为了避免这样的问题于是就用 ThreadLocal 来标记当前接收 Event 的方法所在的线程是否和 发出 Event 的线程是一个线程。如果是一个线程就不允许 post 。(以上纯属个人看法)

# 总结

通过这个问题，反应出我们平时在使用第三方库的时候，可以多看看这些第三方库的实现。这样就可以防止在遇到一些"疑难杂症"的时候，不至于无从下手。比如这个EventBus 的细节，如果不看源码可能还真不知道原因是因为线程的问题。