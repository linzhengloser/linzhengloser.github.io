---
title: Android中不能在子线程中更新UI?!
date: 2016-12-08 17:05:50
tags: Android View
---
# 概述

在刚开始学Android那会,书上都写这不能再非UI线程中更新UI,比如在Android中请求网络要在子线程中执行,当获取到结果后,默认回调是在子线程中执行的,如果不做处理,会抛一个叫android.view.ViewRoot$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.的异常.这个时候书上就会推荐使用Handler中来处理结果,将更新UI的操作放在Handler中,这样就不会出现上面那个异常,那这究竟是为什么?下面慢慢分析。

<!-- more -->
# Android系统是基于单线程模型设计的

整个Android系统所有的UI操作都是在一个线程中执行的,也就是上面说的UI线程既:ActivityThread类,UI线程主要负责UI相关的操作,如用户点击屏幕,按下按键,View的绘制等等...
## 单线程模型的好处

单线程模型事件队列的定义:采用一个专门的线程从队列中取出事件,并把它们转发给应用程序定义的事件处理器.简单的说就是Looper,Message,Handler这3个类的设计,其实现代的GUI框架就是采用类似这样的设计,模型创建一个专门的线程,事件派发线程来处理GUI事件.单线程模型并不单单存在Android中,Qt,XWindow等都是单线程模型,有单线程模型就有多线程模型,但是多线程模型中的死锁的稳定性和竞争条件等问题,不得不又回到单线程模型,单线程模型通过限制来达到线程安全,所有GUI中的对象,包括可视化组件,和数据模型,都只能被事件线程访问.这也就解释了为什么android要使用单线程模型.

## 为什么在子线程中更新UI是不安全的

Android中更新View 的方式有两种,invalidate和postInvalidate,前者在UI线程中使用,后者在子线程中使用,换句话说,android的UI操作不是线程安全可以表述为invalidate在子线程中调用会导致线程不安全,做一个假设,在子线程中使用invalidate更新UI,在UI线程中也使用invalidate更新UI,这样会不会导致刷新不同步？既然刷新不同步,那么invalidate就不能再子线程中使用,这就是invalidate不能再子线程中使用的原因.
那postInvalidate内部是怎么实现在子线程中更新UI同时不会出现线程安全问题的呢？
postInvalidate源码如下

``` java
    public void postInvalidateDelayed(long delayMilliseconds) {
        // We try only with the AttachInfo because there's no point in invalidating
        // if we are not attached to our window
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
        }
    }

    public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
        Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
        mHandler.sendMessageDelayed(msg, delayMilliseconds);
    }

```
说到底还是给Handler发消息来更新UI,除了使用postInvalidate  在子线程中更新UI,android中还可以使用一下方法在子线程中更新UI:
Activity.runOnUiThread(Runnable);

``` java
    public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }
```

View.post(Runnable);

``` java
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
        // Assume that post will succeed later
        ViewRootImpl.getRunQueue().post(action);
        return true;
    }
```

如果强行在子线程更新UI 会出现 Only the original thread that created a view hierarchy can touch its views. 这个异常,这个异常的出处如下:
代码路径:ViewRootImpl.java
``` java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }

    //mThread 在 ViewRootImpl 的构造函数中初始化,代码如下

     public ViewRootImpl(Context context, Display display) {
        ....省略部分代码
        mThread = Thread.currentThread();
        ....省略部分代码
     }

```

通过上面的代码也就表明了如果更新UI的线程 不是 UI线程就会抛异常.
在仔细看异常的堆栈信息,会发现异常是在下面代码触发的:

``` java

 @Override
    public void invalidateChild(View child, Rect dirty) {
        invalidateChildInParent(null, dirty);
    }

    @Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        checkThread();
        ....省略部分代码
    }

```

在invalidateChildInParent中会调用checkThread 来对当前线程进行检查,看是不是在UI线程中更新的,如果不是就会触发上面那个异常.
但是说到底是可以在非UI线程中,前提是拥有自己ViewRoot,但是ViewRoot是不能手动创建的,必须要借助于WindowManager,具体代码如下

``` java
 
    //自定义一个线程
    public class NonUIThread extends Thread {

        @Override
        public void run() {
            Looper.prepare();
            TextView textView = new TextView(Main3Activity.this);
            textView.setText("non-UIThread update view");
            WindowManager windowManager = Main3Activity.this.getWindowManager();
            WindowManager.LayoutParams params = new WindowManager.LayoutParams(
                    200, 200, 200, 200, WindowManager.LayoutParams.FIRST_SUB_WINDOW,
                    WindowManager.LayoutParams.TYPE_TOAST, PixelFormat.OPAQUE);
            windowManager.addView(textView, params);
            Looper.loop();
        }
        
    }

    //源代码如下
    //代码目录 : WindowManagerGlobal.java
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
                ....省略部分代码
                ViewRootImpl root;
                ....省略部分代码
                root = new ViewRootImpl(view.getContext(), display);

                ....省略部分代码
            }

```
所以得出结论,只要在子线程中创建ViewRootImpl就可以,那么Activity是在上面时候创建ViewRootImpl的呢？
代码目录:ActivityThread.java 这个类就是我们口中说的UI线程

``` java

    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
                .... 省略部分代码
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    // Normally the ViewRoot sets up callbacks with the Activity
                    // in addView->ViewRootImpl#setView. If we are instead reusing
                    // the decor view we have to notify the view root that the
                    // callbacks may have changed.
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
            }

```
# 总结
从上面代码中可以得出结论,ViewRootImpl是在onResume 方法中创建的,根据前面的代码,只有当ViewRootImpl创建时候才会有checkThread这个检查操作,也就是说 只要在onResume之前或者换句话说在ViewRootImpl创建之前,在子线程中更新UI就不会引发异常.