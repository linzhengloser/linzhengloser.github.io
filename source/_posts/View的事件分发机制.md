---
title: View的事件分发机制
date: 2016-12-12 10:36:45
tags: View
---
# 概述
Android的事件分发机制是一个很重要的知识体系,系统以及帮我们处理好了大部分的情况,但是在日常开发中总是会遇到一些奇葩的需求,这个时候就要对系统的事件分发机制有一定的了解,下面就来分析一下系统的事件分发机制.

<!-- more -->

# 什么是事件分发
顾名思义,就是事件(MotionEvent)在Android中传递的机制,什么时候该拦截事件,什么时候事件归子View处理,如果都不处理等等....很多情况,这个时候就需要对系统在处理事件的机制有一定的了解.


# 图解事件分发
![图一](http://ofdvg4c5w.bkt.clouddn.com/2016-12-12_105929.png "事件分发机制")

图片出自:<http://www.jianshu.com/p/e99b5e8bd67b>

>- 事件首先到Activity的dispatchTouchEvent方法,无论return true还是false都会消费掉这个事件,如果调用super,事件就会传递给ViewGroup的dispatchEvent方法来处理.
>- ViewGroup 的dispatchTouchEvent return true 表示消费事件,调用super 会调用ViewGroup的onInterceptTouchEvent.
>- ViewGroup 的onInterceptTouchEvent方法 return true 会调用ViewGroup的onTouchEvent方法,return false 或者调用super的onInterceptTouchEvent方法 会 调用View的dispatchTouchEvent方法.
>- View 的 dispatchTouchEvent return true 消费事件,return false 会调用ViewGroup的onTouchEvent 调用super会调用View的OnTouchEvent.
>- View 的onTouchEvent 方法return true 消费事件,return false或调用super会调用ViewGroup的onTouchEvent
>- ViewGroup 的onTouchEvent return true 消费事件,return false 或调用super 会调用Activity的onTouchEvent,到此处整个事件就结束了.

上面就是整个事件传递的流程,通过上面的分析,如果整个事件传递不被中断,整个事件流向是一个类U型图,如下图

![图2](http://ofdvg4c5w.bkt.clouddn.com/2016-12-12_140726.png "事件分发机制")
图片出自:<http://www.jianshu.com/p/e99b5e8bd67b>

如果没有对控件里面方法进行重写或更改返回值,而直接用super调用默认实现,那么整个事件的流程就是:Actvitiy ---> ViewGroup ---> View 从上往下调用dispatchTouchEvent,在有View ---> ViewGroup ---> Activity 从下往上调用onTouchEvent方法,形成了上面那个U型图.

## 事件的传递与分发

如图一所示,dispatchTouchEvent 和 onTouchEvent方法,如果返回true 表示拦截事件,那么如果事件一旦被拦截,就说明事件到达了终点,没有谁能在收到这个事件,dispatchEvent和onTouchEvent return false 的时候,事件就会回传给父控件的onTouchEvent.对于dispatchTouchEvent返回false,事件就会停止往子View传递和分发,同时开始往父控件回溯(父控件的onTouchEvent开始从下往上回传,直到某个onTouchEvent return true),事件的分发就像递归,return false的意义就是递归停止开始回溯.对于onTouchEvent返回false 就比较简单,他就不消费事件,并让事件继续往父控件从下往上流动.

dispatchTouchEvent,onTouchEvent,onIntercapterTouchEvent, View 和 ViewGroup的这些方法的默认实现,即使用return super.XXXX(),就会按照图2中的U型图走完整个流程,中间不做任何改动,不回溯,不终止,每个环节都走到.

## onInterceptTouchEvent 的作用
intercept的意识就是拦截,每个ViewGroup每次在做分发的时候,问一问拦截器,是否需要拦截这个事件,如果要自己拦截和这个事件 就 return true 就会交给自己的onTouchEvent方法处理,如果不拦截就是继续往子View传,默认是不会拦截的,即 return super.onInterceptTouchEvent() 的时候 跟 return false 是一样的,这个时候就会调用View的onDispatchTouchEvent方法.

## View dispatchTouchEvent 和 ViewGroup dispatchEvent返回super.dispatchEvent()的时候事件的走向

ViewGroup 的dispatchTouchEvent,上面说道return true 是终结传递,return false 是回溯到父View 的 onTouchEvent,那ViewGroup怎样才能通过dispatchTouchEvent方法把事件分发到自己的onTouchEvent中处理？只能通过onInterceptTouchEvent,所以在super.dispatchTouchEvent方法中默认调用了onIntercaptTouchEvent.

## 总结

对于dispatchTouchEvent,onInterceptTouchEvent,onTouchEvent, return true 表示事件终结,return false 表示 回溯到父View的onTouchEvent.
ViewGroup要想把事件分发给自己的onTouchEvent,那么需要在onInterceptTouchEvent方法return true 拦截事件.
ViewGroup的拦截器onInterceptTouchEvent默认是不拦截的,即return super.onInterceptTouchEvent = return false
View没有拦截器,为了让View把可以把事件分发给自己的onTouchEvent,View的dispatchTouchEvent即return super.dispatchTouchEvent() 默认会把事件分发给自己的onTouchEvent.

ViewGroup和View的dispatchTouchEvent方法是用来做事件分发的,那么这个事件可能分发出去的4个目标:

1. return true 事件终结 自己处理.
2. 调用super.dispatchTouchEvent()系统默认会调用自己onInterceptTouchEvent,在onInterceptTouchEvent return true 的时候会把事件分发给自己的onTouchEvent.
3. 调用super.dispatchTouchEvent().系统默认会调用自己onInterceptTouchEvent,在onInterceptTouchEvent return false 的时候,这个时候就会把事件传递给子View.
4. 如果在dispatchTouchEvent() return false,事件终止往下传递,并开始回溯,从父类的onTouchEvent开始,事件从下往上回归执行每个控件的onTouchEvent.

注:由于View没有子View,所有也就没有onInterceptTouchEvent方法,即在View调用super.dispatchTouchEvent()方法的时候,会把事件传递给自己的onTouchEvent方法,对比ViewGroup的dispatchTouchEvent事件分发,View的事件分发没有上面提到的第3点.

ViewGroup和View的onTouchEvent方法是用来做事件处理的,那么这个事件只能有2个处理方式:

1. return turn 自己消费 事件终结
2. 继续从下往上传,return false 或 return super.onTouchEvent,让父View的onTouchEvnent收到这个事件.

ViewGroup的onInterceptTouchEvent对事件有2中处理方式

1. return true 给自己的onTouchEvent 处理
2. reutnr false 或 return super.onInterceptTouchEvent ,将事件传给子View

## ACTION_MOVE 和 ACTION_UP 
上面的分析都是ACTION_DOWN这个事件,ACTION_MOVE 和 ACTION_UP这事件和ACTION_DOWN有点区别,在执行ACTION_DOWN的时候,return false,后面的一系列其他的action就都不会传递过来.简单的说,就说当dispatchTouchEvent在进行事件分发的时候,只有前一个事件如ACTION_DOWN,返回true,才会收到ACTION_MOVE和ACTION_UP的事件,下面举几个例子.

下图中,在View的onTouchEvent 中返回true, 红色箭头表示ACTION_DOWN的流向,蓝线表示ACTION_MOVE和ACTION_UP的流向.

![图3](http://ofdvg4c5w.bkt.clouddn.com/2016-12-14_145106.png "事件分发机制")

图片出自:<http://www.jianshu.com/p/e99b5e8bd67b>

下图中,在Activity的onTouchEvent 中返回true
红色箭头表示ACTION_DOWN的流向,蓝色表示ACTION_MOVE和ACTION_UP的流向.

![图4](http://ofdvg4c5w.bkt.clouddn.com/2016-12-14_150257.png "事件分发机制")

图片出自:<http://www.jianshu.com/p/e99b5e8bd67b>

下图中在ViewGroup2 的dispatchTouchEvent return true 表示拦截这个事件,但是在ViewGroup1的onTouchEvent中 return true ,红色箭头表示ACTION_DOWN的流向,蓝线表示ACTION_MOVE和ACTION_UP的流向.

![图5](http://ofdvg4c5w.bkt.clouddn.com/2016-12-14_150343.png "事件分发机制")

图片出自:<http://www.jianshu.com/p/e99b5e8bd67b>

下图中View的dispatchTouchEvent return false,这个时候回将事件回溯到父View的onTouch方法,然后在ViewGroup2的onTouch方法return true,红色箭头表示ACTION_DOWN的流向,蓝线表示ACTION_MOVE和ACTION_UP的流向.

![图6](http://ofdvg4c5w.bkt.clouddn.com/2016-12-14_150328.png "事件分发机制")

图片出自:<http://www.jianshu.com/p/e99b5e8bd67b>

总之,在哪个View的onTouchEvent方法返回true即return true,那么ACTION_MOVE和ACTION_UP这两个事件,就会从上往下,传递到这个View 的 onTouchEvent方法,然后就不会在继续传递,换句话说,ACTION_DOWN在哪个View消费了,那么ACTION_MOVE和ACTION_UP就会传递到消费的地方,而且不会在继续往下传,如果ACTION_DOWN在dispatchTouchEvent中消费了,那么事件到此为止就停止传递了,如果ACTION_DOWN在onTouchEvent消费了,那么ACTION_MOVE和ACTION_UP就会传递到消费事件的onTouchEvent,并结束传递.

