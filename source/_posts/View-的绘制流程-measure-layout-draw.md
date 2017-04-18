---
title: 'View 的绘制流程(measure,layout,draw)'
date: 2016-12-21 16:12:51
tags: View
---
# 概述
View的创建大致分为:measure(测量),layout(布局,一般是ViewGroup干的事）,draw（绘制）.在自定义View的时候,打交道最多的就是这个3个方法,但是系统的View是怎么实现这个方法的呢？接下来就来缝隙一下这个3个方法.

<!-- more -->
# measure 测量
在上面所说的3个方法中,measure方法的实现相对于复杂一点,android在测量View的时候会生成两个值,分别是测量模式,和测量的具体值,android将这两个值封装在一个32位int值中,高两位表示测量模式(Mode),后30位表示测量的大小(size).

MeasureSpec一共分为3种测量模式
> UPSPECIFIED :父容器对子容器没有任何限制,子容器想要多大就多大.
>  
> EXACTLY :父容器已经为子容器设置了尺寸,子容器应当服从这些边界,不论子容器想要多大空间
> 
> AT_MOST :子容器可以是声明大小内任意大小


## ViewGroup 的measure
从代码上来看,子View的测量模式是从父View传过来的,这也就说明子View的测量模式和测量大小是由父View完成的,但是并不完全是父View的要求,而是由父View的MeasureSpec和子View的LayoutParams共同决定的,具体实现在源码中:

``` java

    //代码出自:ViewGroup 
    //这个方法主要是测量所有子View 并且将margin算在里面,为什么这里强调margin呢？ 
    //因为在系统ViewGroup实现中有一个measureChild方法 
    //这个方式和此方法是实现大同小异,除了这个方法加入了margin
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {

        //获取子View的LayoutParams
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        //通过getChildMeasureSpec方法获得子View的width的MeasureSpec
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);

        //通过getChildMeasureSpec方法获得子View的height的MeasureSpec
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        //调用子View的measure
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

    // 代码出自ViewGroup
    //这个方法的具体实现就是通过ViewGoup自身的MeasureSpec + 子View的LayoutParams,
    //参数: 父View的MeasureSpec,子View的padding(如果此方法在measureChild中使用的话,只会传入padding,但是在measureChildWithMargins中使用就会传入padding 和 margin 具体可以看上面源码),子View的width or height
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        //ViewGroup的 mode 和 size
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        //确保size 大于等于0
        int size = Math.max(0, specSize - padding);

        //用来保存最终的计算结果
        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }



```

上面getChildMeasureSpec源码中,首先是根据父View的MesureMode+子View的LayoutParams一起计算出子View的MeasureMode和MeasureSize,分为如下9种情况:


1. 父View的Mode为EXACTLY,子View的width/height大于等于0,这个时候子View的Mode为EXACTLY,size 为 width/height.
2. 父View的Mode为EXACTLY,子View的width/height为match_parent,这个时候子View的mode为EXACTLY,size 为 父View的Size.
3. 父View的Mode为EXACTLY,子View的width/height为wrap_content,这个时候子View的mode为AT_MOST,size 为 父View的size.
4. 父View的Mode为AT_MOST,子View的width/height大于等于0,这个时候子View的mode为EXACTLY,size 为 width/height.
5. 父View的Mode为AT_MOST,子View的width/height为match_parent,这个时候子View的mode为AT_MOST,size 为 父View的Size.
6. 父View的Mode为AT_MOST,子View的width/height为wrap_content,这个时候子View的mode为AT_MOST,size 为 父View的Size
7. 父View的Mode为UNSPECIFIED,子View的width/height大于等于0,这个时候子View的mode为EXACTLY,size 为 width/height
8. 父View的Mode为UNSPECIFIED,子View的width/height为match_parent,这个时候子View的mode为UNSPECIFIED,size = 0 或者 父View的size
9. 父View的Mode为UNSPECIFIED,子View的width/height为warp_content,这个时候子View的mode为UNSPECIFIED,size = 0 或者 父View的size

## 总结 MeasureSpec的计算过程
从上面9种情况分析得出,EXACTLY这个模式表示的是有确切的值,即width和height大于等于0,父View的width/height有具体的值了,子View的width/height为match_parent就等于父View的width/height,所以此时子View的测量模式就是EXACTLY,size 就等于父View的width/height.如果子View的width/height 为 wrap_content,表示子View的大小是根据自身内容来决定的,但是子View就是子View不可能超过父View,所以这个时候子View的size 就是 父View的size,即表示子View的大小为自身内容,但是不要超过父View大小.这就是AT_MOST这个测量模式的含义,对应的也就是wrap_content.

## 子View 的measure
子View的测量分为onMeasure方法和measure方法,measure方法就是在上面源码中的measureChildWithMargins方法末尾调用的,在measure内部调用了onMeasure方法,measure方法是final类型的,这也就说明了这个方法是不能被重写的,所以在自定义View的时候一般都是重写父View的onMeasure.
下面是View的onMeasure 默认实现源码
``` java

    //系统默认的onMeasure 的是按
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }


    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

    //通过最小size 和 measureSpec 计算出最终的size
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        //如果specMode = UNSPECIFIED 那么size 就等于最小size,如果specMode 等于AT_MOST or EXACTLY size就等于specSize 即 父View measure后计算出的size
        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }

```

从上面个getDefaultSize 这个方法可以看出,如果我们在自定义View的时候,如果不是从写onMeasure方法,默认无论wrap_content,match_parent还是具体的值,都是默认是用父View测量的size.即默认实现中,wrap_content = match_parent.

## 整个match过程
分析了View的measure 的原理,下面来分析一下是谁发起的这个measure.
首先写了一个布局,观察整个程序的执行流程.布局代码如下
``` xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout  xmlns:android="http://schemas.android.com/apk/res/android"    
   android:id="@+id/linear"
   android:layout_width="match_parent"    
   android:layout_height="wrap_content"    
   android:layout_marginTop="50dp"    
   android:background="@android:color/holo_blue_dark"    
   android:paddingBottom="70dp"    
   android:orientation="vertical">    
   <TextView        
    android:id="@+id/text"       
    android:layout_width="match_parent"     
    android:layout_height="wrap_content"  
    android:background="@color/material_blue_grey_800"       
    android:text="TextView"        
    android:textColor="@android:color/white"        
    android:textSize="20sp" />    
</LinearLayout>

```

上面布局文的结构对应下面的图片

![图一](http://ofdvg4c5w.bkt.clouddn.com/2016-12-22_165516.png "View的创建过程")

图片出自:<http://www.jianshu.com/p/5a71014e7b1b>

整个图是一个DecorView,DecorView可以理解成整个页面的根View,DecorView是一个FrameLayout,包含两个子View,一个id=statusBarBackground的View和一个LinearLayout,id=statusBarBackgroud的View可以看成是状态栏,而这个LinearLayout比较重要,它包括一个title和content,title就是ActionBar,content就是平时在写Activity的时候 使用setContentView的那个布局.整体View布局如下

![图一](http://ofdvg4c5w.bkt.clouddn.com/2016-12-22_163604.png "View的创建过程")

图片出自:<http://www.jianshu.com/p/5a71014e7b1b>

这里要注意几个地方,header是一个ViewStub,是使用惰性加载ActionBar,平时开发中都是使用的NoActionBar这个主题,这里就可防止复加载不必要的View.
我们都知道每个Activity均会创建一个PhoneWindow对象,这个PhoneWindow对象是Activity和View交互的接口,每个Window都会对应一个View和一个ViewRootImpl,Window和View通过ViewRootImpl来建立联系,对于Activity来说,ViewRootImpl是连接WindowManager和DecorView的纽带,绘制的入口是从ViewRootImpl的performTraversals方法来发起measure,layout,draw等流程,具体代码如下:

``` java

    private void performTraversals() {
        //.....省略部分代码
        int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
        int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        //.....省略部分代码
        performLayout(lp, mWidth, mHeight);
        //.....省略部分代码
        performDraw();
    }

    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

    //在performTraversals 中调用了该方法通过 mWidth/mHeight 
    //和 WindowManager.LayoutParams.width/height 默认值是match_parent
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }

```
上面就是ViewRootImpl的源码,上面代码中的mView其实就是DecorView,VIew的绘制是从DecorView开始的,在mView的measure方法中调用getRootmeasureSpec
方法获取两个MeasureSpec做为参数,getRootMeasureSpec方法接收两个参数,第一个参数是mWidth/mHeight,第二个参数是WindowManager.LayoutParams,它的width和height的默认值是match_parent,所以getRootMeasureSpec返回的MeasureSpec的mode为EXCATLY,size为屏幕宽高.
因为DecorView是一个FrameLayout,那么接下来就会进入它的measure方法,measure方法的两个参数就是上面的getRootMeasureSpec的返回值,所以最终测量的结果为width mode = EXACTLY size = 1440,height mode = EXACTLY size = 2560 ,因为屏幕的分辨率是1440*2560,所以测量出来的size也就为 1440和2560,那么DecorView开始测量自己的子View,源代码如下:

``` java

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //省略部分代码
        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                //省略部分代码
            }
        }
        //省略部分代码
    }

```

上面代码就是FrameLayout的onMeasure的部分代码,从代码中看出是先先循环子View,并使用之前提到的measureChildWithMargins 方法测量,最终回测量出ViewRoot的MeasureSpec
width mode = EXACTLY size = 1440,height mode = EXACTLY size = 2560.
算出VewRoot的Measure之后,开始调用ViewRoot的measure,因为ViewRoot是一个LinearLayout,最终会调用LinearLayout的onMeasure方法,这个时候LinearLayout会开始通过调用上面的measureChildWithMargins方法来逐个测量它的子View,根据上面View的层级图片,显示测量Header,但是因为使用的NoActionBar的主题,Header是隐藏的,所有这里只会测量content,这时要测量content就要把ViewRoot的MeasureSpec传进去,计算出来的MeasureSpec,mode 为EXACTLY, width size = 1440,height size = 2460,这里ViewRoot有一个100px的padding,可能是跟状态栏有关,这个padding是系统给的，无法去掉.计算完之后会调用自己的onMeasure,重复上面的过程,content自身是FrameLayout,接下来就计算LinearLayout,width 的mode为EXACTLY,height mode 为 AT_MOST 因为上面的layout_height是wrap_content,width size = 1440 ,height size = 2260 因为有padding 120dp.最后测量textview,MeasureSpec的 width mode 为 EXACTLY size为1440,height mode 为 AT_MOST,size 为1980,到此真个Measure 过程走完。

# Layout 布局
在上面performTraversals()方法源码中,调用了performLayout()方法,performLayout跟performMeasure()方法的本质差不多,都是发起Layout,从最上层的View开始,一层一层的调用layout方法.源码如下:

``` java

    @Override
    public final void layout(int l, int t, int r, int b) {
        //判断是否有动画 或者动画是否在执行
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
            //调用super 的layout方法 也就是 View 的 layout方法,在View的layout方法中又调用了onLayout方法.
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }


    public void layout(int l, int t, int r, int b) {
        //.... 省略部分代码

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        //changed 表示 View的位置是否有发生变化,如果没有发生变,就没有必要调用onLayout
        //setFrame 方法本质就是设置 left right top botto 等属性
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            //发生变化 调用onLayout 重新布局
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }


```

最终还是会调用onLayout方法,在ViewGroup中onLayout是一个abstract方法,也就是说继承ViewGroup必须要实现onLayout方法,在onLayout方法必须要测量所有子View,也就是调用所有子View的Layout方法来实现大概代码如下
``` java

    int childCount = getChildCount() ; 
    for(int i=0 ;i<childCount ;i++){
       View child = getChildAt(i) ;
       //整个layout()过程就是个递归过程
       child.layout(l, t, r, b) ;
    }

```
 代码很简单,就是循环调用自己所有子View的layout方法,给子View通过setFrame方法确定位置,而这里的left,right,top,bottom就是通过我们之前measure出来的width,height,还有一些参数来决定的,具体实现可以看FrameLayout的onLayout方法的实现,每个不同的Layout实现的方法是不一样,但是这里measure出来的width和height只是一个参考值,并不是必须要和measure出来的width和height一样,一个View包括ViewGroup,位置是由Left,Right,Top,Bottom,这4个参数了决定位置的,所以如果强制不使用measure出来的width和height的话,那么getMeasureWidth()和getWidth(),这两个方法返回的值会完全不一样.因为这两个方法的实现完全不一样,一个返回的是mMeasureWidth 一个返回的是mRight-mLeft.具体看下面源代码.
 ``` java

    public final int getMeasuredWidth() {
        return mMeasuredWidth & MEASURED_SIZE_MASK;
    }

    public final int getLeft() {
        return mLeft;
    }

 ```

 # onDraw 绘制
 layout之后 就是draw,顾名思义,就是绘制View,在performDraw方法中调用了子View的draw()方法,官方是不提倡从写View的draw,所以一般自定义View的时候,重写的都是onDraw方法,View的draw方法源码如下:
 ``` java

//View.java 的draw方法
public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        //下面注释说的很清楚了,绘制步骤分为,1.绘制背景 2.
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background 绘制背景
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content 绘制内容 调用onDraw方法
         *      4. Draw children 如果是VIewGroup 则循环调用子View的draw方法
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        //..... 省略部分代码
        // Step 2, save the canvas' layers
        //..... 省略部分代码
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }

            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }

            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }

            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }
        //..... 省略部分代码

        canvas.restoreToCount(saveCount);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
    }

 ```

从上面代码中,可以发现draw一共分为6步.

1. drawBackground 绘制背景

``` java

    private void drawBackground(Canvas canvas) { 
        Drawable final Drawable background = mBackground; 
        //省略部分代码....
        //mRight - mLeft, mBottom - mTop layout确定的四个点来设置背景的绘制区域 
        if (mBackgroundSizeChanged) { 
            background.setBounds(0, 0, mRight - mLeft, mBottom - mTop);   
            mBackgroundSizeChanged = false; rebuildOutline(); 
        } 
        //...省略部分代码
        //调用Drawable的draw() 把背景图片画到画布上
        background.draw(canvas); 
        ...... 
}

```

2. 对View的的内容进行绘制.
onDraw()方法是View用来draw自己的,具体如何绘制,就需要具体子View来实现,View.java中的onDraw方法只是一个空实现,ViewGroup也没有实现,每个View要绘制的内容都是不一样的,如TextView绘制文本,EditText绘制文本框等等...

3. 对当前View的所有子View进行绘制
dispatchDraw()方法用来绘制子View的,在View.java中的dispatchDraw()方法是一个空方法,因为View是没有子View,但是在ViewGroup中这个方法是有实现的.

``` java
 @Override
 protected void dispatchDraw(Canvas canvas) {
        //省略部分代码...
        if ((flags & FLAG_USE_CHILD_DRAWING_ORDER) == 0) {
            for (int i = 0; i < count; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                    more |= drawChild(canvas, child, drawingTime);
                }
            }
        } else {
            for (int i = 0; i < count; i++) {
                final View child = children[getChildDrawingOrder(count, i)];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                    more |= drawChild(canvas, child, drawingTime);
                }
            }
        }
      //省略部分代码
    }


    //调用子View的draw方法
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }

```
代码一眼看出,就是遍历子View,然后调用drawChild方法,drawChild方法中调用child的draw方法,这里ViewGroup理解实现大部分代码,所以一般平时在自定义ViewGroup的时候不用担心drawChild,

一张图看下整个View的绘制过程
![图一](http://ofdvg4c5w.bkt.clouddn.com/2016-12-27_145315.png "View的创建过程")

图片出自:<http://www.jianshu.com/p/5a71014e7b1b>

# 总结
到这里,View的Measure,Layout和Draw 这3个方法的调用,执行过程上面进行了分析,中而言之系统已经帮我们做好了大部分事情,我们自己在平时开发中如果要使用自定义View来解决业务需求,就只需要实现onMeasure,onDraw和onLayout 这个3个方法即可.



