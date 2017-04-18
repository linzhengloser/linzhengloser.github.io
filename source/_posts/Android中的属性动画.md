---
title: Android中的属性动画
date: 2016-12-07 12:30:00
tags: Animation
categories: Animation
---
# 属性动画 和 View动画的区别

属性动画在Android3.0中引入,主要是为了实现View动画不能实现的效果,如将一个颜色平滑的变成另外一种颜色.View动画只能用于View,而属性动画可以中用于不同场景,View动画对View的内部属性并没有改变,比如移动一个View 然后点击View原来的位置才能触发View的单击事件，属性动画却不会出现这种情况，因为属性动画是直接改变View的内部属性在达到动画效果的。

<!-- more -->
# ValueAnimator 和 ObjectAnimator

## ValueAnimator 的使用方法
ofInt(0,100): 2秒钟将0平滑的过度到100

``` java
ValueAnimator vaulueAnimator =  ValueAnimator.ofInt(0,100);
vaulueAnimator.setDuration(2000);
vaulueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animator) {
                System.out.println(animator.getAnimatedValue());
            }
        });
vaulueAnimator.start();
```
ofFloat(0,200,50,300): 先从0平滑到200在平滑到50最后在平滑到300,一共用了3秒

``` java
ValueAnimator valueAnimator = ValueAnimator.ofFloat(0,200,50,300);
valueAnimator.setDuration(3000);
valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animator) {
            System.out.println(animator.getAnimatedValue());
        }
    });
valueAnimator.start();
```
### ValueAnimator 常用函数
``` java

//设置动画时长,单位是毫秒
ValueAnimator setDuration(long duration);
//获取运动值
Object getAnimatedValue();
//开始动画
void start();
//设置循环次数,设置为INF INITE 表示无限循环
void setRepeatCount(int value);
//设置循环模式,RESTART 或 REVERSE
void setRepeatMode(int mode);
//取消动画
void cancel();

```
### ValueAnimator 监听器
前面的 addUpadteListenr 就是添加一个用来监听动画变化的监听器,除了这个监听器以外还有一个监听器.这个监听器可以监听动画的开始,结束,取消,重复状态,
``` java
        ValueAnimator valueAnimator = ValueAnimator.ofFloat(0,200,50,300);
        valueAnimator.setDuration(3000);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animator) {
                System.out.println(animator.getAnimatedValue());
            }
        });
        valueAnimator.addListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animator) {
                //动画开始
            }

            @Override
            public void onAnimationEnd(Animator animator) {
                //动画结束
            }

            @Override
            public void onAnimationCancel(Animator animator) {
                //动画被取消
            }

            @Override
            public void onAnimationRepeat(Animator animator) {
                //动画重复执行,如果设置动画重复执行 那么就只会走该方法 不会重复执行start
            }
        });
        valueAnimator.start();
```
### 插值器
插值器也叫加速器,比如上面0-200 的变化是按照一个速度变化的.这里可以使用插值器来改变这个速度,比如先快后慢,可以先慢后快.比如系统提供的BounceInterpolator(弹跳插值器)使用法如下:
``` java
        ValueAnimator animator = ValueAnimator.ofInt(0,600);
        //添加监听
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animator) {
                int curValue = (int) animator.getAnimatedValue();
                mTextView.layout(mTextView.getLeft(),curValue,mTextView.getRight(),curValue+mTextView.getHeight());
            }
        });
        //设置动画时间
        animator.setDuration(1000);
        //设置插值器
        animator.setInterpolator(new BounceInterpolator());
        //开始动画
        animator.start();
```
上面代码会让TextView 下落,到达底部之后会有一个回弹的效果
#### 自定义插值器
指定插值器只需要继承Interpolator,继承Interpolator只需要实现getInterpolation这个一个方法:
getInterpolation(float input) 参数input 取值返回时 0 - 1 表示动画执行的进度,0表示动画开始,1表示动画结束,0.5表示动画执行一半.其实值以此类推.input参数与任何我们设定的值没有关系,只与时间有关,随着时间的增长,动画的进度自然增加,而getInterpolator方法返回表示的是动画的当前数值执行的进度.

#### Evaluator 的用途
Evaluator 用来根据Interpolator返回的进度算出具体的动画值,加速器返回的小数值,表示当前动画的进度.无论是ofInt还是ofFloat定义的动画,动画的进度永远是从0 - 1,0表示开始,1表示结束,对于任何动画都是试用的,但是Evaluator不一样,如果使用的是ofInt那么返回的就是int类型的值,如果使用的是onFloat返回的就是float类型的值,那么就肯定有不同的Evaluator,系统对ofInt提供了IntEvaluator,ofFloat提供了FloatEvaluator,下面是IntEvaluator的默认实现.
``` java
    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }

```
使用一个很简单的公式 当前值= 开始值 + 当前经度 * (结束值 - 开始值) 

#### ArgbEvaluator 的用法
上面只说了2个比较简单的Evaluator,ArgbEvaluator用来实现颜色过渡的,下面代码可以实现从红色过渡到黄色.
``` java
        ValueAnimator valueAnimator =ValueAnimator.ofInt(Color.RED,Color.YELLOW);
        valueAnimator.setEvaluator(new ArgbEvaluator());
        valueAnimator.setDuration(3000);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animator) {
                mTextView.setBackgroundColor((Integer) animator.getAnimatedValue());
            }
        });
        valueAnimator.start();
```
ArgbEvaluator 的原理
``` java
public Object evaluate(float fraction, Object startValue, Object endValue) {
        int startInt = (Integer) startValue;
        int startA = (startInt >> 24) & 0xff;
        int startR = (startInt >> 16) & 0xff;
        int startG = (startInt >> 8) & 0xff;
        int startB = startInt & 0xff;

        int endInt = (Integer) endValue;
        int endA = (endInt >> 24) & 0xff;
        int endR = (endInt >> 16) & 0xff;
        int endG = (endInt >> 8) & 0xff;
        int endB = endInt & 0xff;

        return (int)((startA + (int)(fraction * (endA - startA))) << 24) |
                (int)((startR + (int)(fraction * (endR - startR))) << 16) |
                (int)((startG + (int)(fraction * (endG - startG))) << 8) |
                (int)((startB + (int)(fraction * (endB - startB))));
    }
```
整个变化过程分为3部分
1.首先求出开始的A R G B
2.然后求出结束的A R G B
3.通过进度求出变化中的A R G B
使用的公式跟IntEvaluator中的一样.

### ofObject 的用法
前面只使用了ofInt 和 ofFloat 这2个方法来指定动画的取值范围,这两种方法只能作用于int类型和float类型,如果想让ValueAnimator中用于其他类型的值,这个使用可能就要使用ofObject方法.

ofObject方法概述
参数: TypeEvaluator evaluator ,Object... values
简单使用方法
``` java
ValueAnimator valueAnimator = ValueAnimator.ofObject(new CharEvaluator(),new Character('A'),new Character('Z'));
        //设置插值器为 加速插值器,动画速度会越来越快
        valueAnimator.setInterpolator(new AccelerateInterpolator());
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mTextView.setText(String.valueOf(animation.getAnimatedValue()));
            }
        });
        valueAnimator.setDuration(10000);
        valueAnimator.start();
```

CharEvaluator 是一个自定义的Evaluator 具体实现如下

``` java
public class CharEvaluator implements TypeEvaluator<Character> {
    @Override
    public Character evaluate(float fraction, Character startValue, Character endValue) {
        int startInt = (int)startValue;
        int endInt = (int)endValue;
        int curInt = (int)(startInt + fraction * (endInt - startInt));
        char result = (char) curInt;
        return result;
    }
}
```
主要就是实现A-Z的一个变化,使用AccelerateInterpolator插值器,速度回睡着之间的推移而变快.

## ObjectAnimator 的使用方法
基本用法:将TextView 的背景透明度由1变成0,然后在从0变成1
参数说明 第一个参数:需要执行动画的View对象  第二参数:需要进行动画的属性名称, 第三个参数: 是一个可变参数,用来表示View的属性是从哪变到哪
``` java
        ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(mTextView,"alpha",1,0,1);
        objectAnimator.setDuration(2000);
        objectAnimator.start();
```

### ObjectAnimator原理
ObjectAnimator 通过调用View的setter 方法来实现动画,所以指定的要动画的值必须要有setter方法,如 setAlpha setRotation 等等.
下面是View 的一些常用的方法:
``` java

//1、透明度：alpha  
public void setAlpha(float alpha)  
  
//2、旋转度数：rotation、rotationX、rotationY  
public void setRotation(float rotation)  
public void setRotationX(float rotationX)  
public void setRotationY(float rotationY)  
  
//3、平移：translationX、translationY  
public void setTranslationX(float translationX)   
public void setTranslationY(float translationY)  
  
//缩放：scaleX、scaleY  
public void setScaleX(float scaleX)  
public void setScaleY(float scaleY)  

```
### ObjectAnimator 和 ValueAnimator 的区别
ObjectAnimator 是同过调用View 的 setter 方法来实现动画,而ValueAnimator并不用指定View,而是单纯的提供一个回调,返回动画执行中的值,需要手动设置具体要动画的值.
在ObjectAnimator中如果使用的是ofFloat函数构造动画,那么对应的set方法的参数也必须要是float类型,ObjectAnimator值负责将动画具体的值传给set方法,至于具体是怎么操作的,就取决于set方法里面的逻辑.
### 自定义View中ObjectAnimator的用法
如果要在自定义的View中使用ObjectAnimator需要声明对应值的set方法
比如:
``` java
public class MyView extends View{
    //省略部分代码
    private Point mPoint;

    public void setPointRadius(int radius){
        mPoint.setRadius(radius);
        invalidate();
    }

    //.....
    //省略onDraw方法
}


public class Point{
    private int mRadius;

    public int getRadius(){
        return mRadiusl
    }

    public void setRadius(int radius){
        this.mRadius = radius;
    }
}


//Activity中使用   因为上面的setPointRadius 的 参数类型是Int  所以这里要用ofInt
ObjetAnimator obejectAnimator = ObjectAnimator.ofInt(mMyView,"pointRadius",100);
objectAnimator.setDuration(2000);
objectAnimator.start();

```
### 在没有getter方法的时候,只传入一个参数
上面代码中ofInt方法只传入了一个变化值,也就是100,如果只传一个参数,而且在View中只有setter 没有getter  那么默认动画的开始值默认是从0开始,如果量指定一个默认的起始值,就需要声明一个getter方法,如 public int getPointRadius(){ return 50; } ，这个时候动画就会从 50 到 100 而不是0 到 100.
### ObjectAnimator 常用函数
``` java
//设置动画时间
setDuration();

//获取动画值
getAnimatedValue();

//开始动画
start();

//设置循环次数,设置INFINITE 表示无限循环
setRepeatCount(int value);

//设置循环模式
setRepeatMode(int mode);

//取消动画
cancel();


//监听器
public static interface AnimatorUpdateListener{
    void onAnimationUpdate(ValueAnimator animation);
}

public static interface AnimatorListener{
    //动画开始
    void onAnimationStart();
    //动画结束
    void onAnimationEnd();
    //动画取消
    void onAnimationCancel();
    //重复执行动画
    void onAnimationRepeat();
}

//添加监听器
addUpdateListener(AnimatorUpdateListener listener);

addListener(AnimatorListener listener);


//插值器 和 Evaluator

setInterpolator(TimeInterpolator value);

setEvaluator(TypeEvaluator value);

```
# PropertyValuesHolder 实现组合动画
上面的ValueAnimator 和 ObjectAnimator 值能实现单个动画比如:缩放,位移等....,如果既想实现缩放,又想在缩放的同时实现颜色的变化,这个时候就需要用PropertyValuesHolder,
## PropertyValuesHolder基本使用方法
PropertyValuesHolder 的使用和上面的ofInt 和 ofFloat 用法类似,就是将几个动画 都 添加到 声明的PropertyValuesHolder中,

``` java

        PropertyValuesHolder rotationHolder = PropertyValuesHolder.ofFloat("Rotation",60f,-60f,40f,-40f,-20f,20f,10f,-10f,0f);
        PropertyValuesHolder colorHolder = PropertyValuesHolder.ofInt("BackgroundColor",0xffffffff,0xffff00ff,0xffffff00,0xffffffff);
        ObjectAnimator objectAnimator = ObjectAnimator.ofPropertyValuesHolder(mTextView,rotationHolder,colorHolder);
        objectAnimator.setDuration(3000);
        objectAnimator.setInterpolator(new AccelerateInterpolator());
        objectAnimator.start();

```
# KeyFrame 关键帧
关键帧,在制作动画的时候,比如一个矩形从0,0 移动到 100,100 我们不需要 把移动中所有坐标都设置一点,往往只需要设置一下开始的位置,和结束的位置,那么这两个位置就称之为关键帧。
Android 声明关键帧的方式如下
Keyframe keyframe = Keyframe.ofFloat(0,0);
KeyFrame keyframe = Keyframe.ofFloat(1f,0);
参数说明: ofFloat(float fraction,int float value); fraction:动画进度,取值范围为0-1 value:指定进度所对应的值

下面使用KeyFrame 实现一个左右摇摆+放大的动画
``` java
        /**
         * 左右震动效果
         */
        Keyframe frame0 = Keyframe.ofFloat(0f, 0);
        Keyframe frame1 = Keyframe.ofFloat(0.1f, -20f);
        Keyframe frame2 = Keyframe.ofFloat(0.2f, 20f);
        Keyframe frame3 = Keyframe.ofFloat(0.3f, -20f);
        Keyframe frame4 = Keyframe.ofFloat(0.4f, 20f);
        Keyframe frame5 = Keyframe.ofFloat(0.5f, -20f);
        Keyframe frame6 = Keyframe.ofFloat(0.6f, 20f);
        Keyframe frame7 = Keyframe.ofFloat(0.7f, -20f);
        Keyframe frame8 = Keyframe.ofFloat(0.8f, 20f);
        Keyframe frame9 = Keyframe.ofFloat(0.9f, -20f);
        Keyframe frame10 = Keyframe.ofFloat(1, 0);
        PropertyValuesHolder frameHolder1 = PropertyValuesHolder.ofKeyframe("rotation", frame0, frame1, frame2, frame3, frame4, frame5, frame6, frame7, frame8, frame9, frame10);
        /**
         * scaleX放大1.1倍
         */
        Keyframe scaleXframe0 = Keyframe.ofFloat(0f, 1);
        Keyframe scaleXframe1 = Keyframe.ofFloat(0.1f, 1.1f);
        Keyframe scaleXframe2 = Keyframe.ofFloat(0.2f, 1.1f);
        Keyframe scaleXframe3 = Keyframe.ofFloat(0.3f, 1.1f);
        Keyframe scaleXframe4 = Keyframe.ofFloat(0.4f, 1.1f);
        Keyframe scaleXframe5 = Keyframe.ofFloat(0.5f, 1.1f);
        Keyframe scaleXframe6 = Keyframe.ofFloat(0.6f, 1.1f);
        Keyframe scaleXframe7 = Keyframe.ofFloat(0.7f, 1.1f);
        Keyframe scaleXframe8 = Keyframe.ofFloat(0.8f, 1.1f);
        Keyframe scaleXframe9 = Keyframe.ofFloat(0.9f, 1.1f);
        Keyframe scaleXframe10 = Keyframe.ofFloat(1, 1);
        PropertyValuesHolder frameHolder2 = PropertyValuesHolder.ofKeyframe("ScaleX", scaleXframe0, scaleXframe1, scaleXframe2, scaleXframe3, scaleXframe4, scaleXframe5, scaleXframe6, scaleXframe7, scaleXframe8, scaleXframe9, scaleXframe10);
        /**
         * scaleY放大1.1倍
         */
        Keyframe scaleYframe0 = Keyframe.ofFloat(0f, 1);
        Keyframe scaleYframe1 = Keyframe.ofFloat(0.1f, 1.1f);
        Keyframe scaleYframe2 = Keyframe.ofFloat(0.2f, 1.1f);
        Keyframe scaleYframe3 = Keyframe.ofFloat(0.3f, 1.1f);
        Keyframe scaleYframe4 = Keyframe.ofFloat(0.4f, 1.1f);
        Keyframe scaleYframe5 = Keyframe.ofFloat(0.5f, 1.1f);
        Keyframe scaleYframe6 = Keyframe.ofFloat(0.6f, 1.1f);
        Keyframe scaleYframe7 = Keyframe.ofFloat(0.7f, 1.1f);
        Keyframe scaleYframe8 = Keyframe.ofFloat(0.8f, 1.1f);
        Keyframe scaleYframe9 = Keyframe.ofFloat(0.9f, 1.1f);
        Keyframe scaleYframe10 = Keyframe.ofFloat(1, 1);
        PropertyValuesHolder frameHolder3 = PropertyValuesHolder.ofKeyframe("ScaleY", scaleYframe0, scaleYframe1, scaleYframe2, scaleYframe3, scaleYframe4, scaleYframe5, scaleYframe6, scaleYframe7, scaleYframe8, scaleYframe9, scaleYframe10);

        ObjectAnimator objectAnimator = ObjectAnimator.ofPropertyValuesHolder(mTextView, frameHolder1, frameHolder2, frameHolder3);
        objectAnimator.setDuration(1000);
        objectAnimator.start();
```
动画主要由3个Keyframe组成,缩放x,缩放y,和左右摇晃,让后用PropertyValuesHolder 一起执行这几个动画.

# AnimatorSet 的使用方法
顾名思义,AnimatorSet 用来执行一组动画,简单用法如下
``` java
        ObjectAnimator objectAnimator1 = ObjectAnimator.ofInt(mTextView,"BackgroundColor",0xffff00ff,0xffffff00,0xffff00ff);
        ObjectAnimator objectAnimator2 = ObjectAnimator.ofFloat(mTextView,"translationX",0,300,0);
        ObjectAnimator objectAnimator3 = ObjectAnimator.ofFloat(mTextView,"translationY",0,300,0);
        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.playSequentially(objectAnimator1,objectAnimator2,objectAnimator3);
        animatorSet.setDuration(3000);
        animatorSet.start();
```
方法解析: playSequentially方法,将动画添加到AnimatorSet中并按添加顺序执行动画,除了这个方法外,还有一个方法叫playTogether,此方法也是将动画添加到AnimatorSet中,不同的是在执行的是一起执行添加的动画,换句话说,playSequentially和playTogether只负责开始动画,只是开始的方式不一样,一个是一个一个开始,一个是开始所有,但是有一个共同点,就是不干涉动画的执行。

## 自由设置动画的顺序
如果想手动的指定哪个动画先执行,哪个动画后行,这时候就需要用到AnimatorSet.Builder用法如下:
``` java
        ObjectAnimator objectAnimator3 = ObjectAnimator.ofFloat(mTextView,"translationY",0,300,0);
        AnimatorSet animatorSet = new AnimatorSet();
        ObjectAnimator objectAnimator1 = ObjectAnimator.ofInt(mTextView,"BackgroundColor", Color.BLACK,Color.RED);
        objectAnimator1.setEvaluator(new ArgbEvaluator());
        AnimatorSet.Builder builder = animatorSet.play(objectAnimator1);
        builder.with(objectAnimator3);
        animatorSet.setDuration(3000);
        animatorSet.start();
```
上面首先使用Play方法生成一个AnimatorSet.Builder对象,然后调用with对象指定一个和前面play的动画对象一起执行的动画对象.这里除了with方法以外还有其他方法如下:
``` java
//和前面的动画一起执行
with(Animator anim)
//在前面动画前面执行
before(Animator anim);
//在前面动画后面执行
after(Animator anim);
//在前面动画延迟N秒后执行
after(long n);

//上面4个方法返回值都是AnimatorSet.Builder 所有这里支持链式调用

```
## AnimatorSet 的监听器
和ObjectAnimator或ValueAnimator相似,AnimatorSet也可以添加监听器,但是不同于之前的监听器,AnimatorSet的监听器值监听AnimatorSet 并不会受到添加到AnimatorSet中的Animator的影响.监听器代码如下:
``` java

public static interface AnimatorListener{
    //动画开始
    void onAnimationStart();
    //动画结束
    void onAnimationEnd();
    //动画取消
    void onAnimationCancel();
    //重复执行动画 因为Animator没有设置repeat的函数,所以这个函数永远都不会被调用
    void onAnimationRepeat();
}

```

## AnimatorSet 中的全局设置方法
在AnimatorSet中还有几个比较诡异的函数:
//设置动画执行时长
setDuration(long duration);
//设置插值器
setInterpolator(TimeInterpolator interpolator);
//设置动作作用于哪个View
setTarget(Object target);
上面这几个方法在ObjectAnimator和ValueAnimator 中都有,如果没有动画对象都设置动画事件,动画的插值器和动画作用于哪个View,那么这个时候,AnimatorSet没有设置上面这几个方法的话,就默认使用没有动画对象中设置的属性,如果调用了上面几个方法设置了这些属性,就会替代掉原本动画对象中设置的属性,类似一个全局的设置.但是有一个意外,就是AnimatorSet的setStartDelay(int time),这个方法并不会替换掉原本动画对象中设置的属性值,而是会延时启动AnimatorSet,所有这个方法只会对AnimatorSet 有效,里面的Animator对象无效.

# 在Xml中定义Animator 和 AnimatorSet
除了可以在代码中创建Animaotr 和 AnimatorSet以外,还可以通过xml创建Animator和AnimatorSet,
创建ValueAnimator的方法如下,在res/animator目录下新建animator.xml 内容如下
``` xml
<?xml version="1.0" encoding="utf-8"?>
<animator xmlns:android="http://schemas.android.com/apk/res/android"
          android:valueFrom="0"
          android:valueTo="300"
          android:duration="1000"
          android:valueType="intType"
          android:interpolator="@android:anim/bounce_interpolator"/>
```
``` java
        ValueAnimator valueAnimator = (ValueAnimator) AnimatorInflater.loadAnimator(this,R.animator.animator);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                int offset = (int) animation.getAnimatedValue();
                mTextView1.layout(mTextView1.getLeft(), offset, mTextView1.getRight(), offset+mTextView1.getHeight());
            }
        });
        valueAnimator.start();
```
因为animator.xml中设置的valueType的值是intType 所以valueFrom 和 valueTo 必须是int类型,使用AnimatorInflater来讲xml转换成Animator对象.下面在使用xml创建ObjectAnimator对象,
``` xml
<?xml version="1.0" encoding="utf-8"?>
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
                android:duration="1000"
                android:valueTo="200"
                android:valueType="floatType"
                android:propertyName="y"
                android:repeatCount="1"
                android:repeatMode="reverse"/>
```
``` java
        ObjectAnimator objectAnimator = (ObjectAnimator) AnimatorInflater.loadAnimator(this,R.animator.object_animator);
        objectAnimator.setTarget(mTextView1);
        objectAnimator.start();
```
用法和上面的ValueAnimator 一样,首先通过AnimatorInflater将xml转换成ObjectAniamtor 对象,然后通过setTarget方法指定动画作用于哪个View,最后调用start方法开始动画.
还可以在xml中定义AnimatorSet,具体定义方式如下
``` java
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
        android:ordering="together">

    <objectAnimator
            android:propertyName="x"
            android:duration="500"
            android:valueFrom="0"
            android:valueTo="400"
            android:valueType="floatType"/>

    <objectAnimator
            android:propertyName="y"
            android:duration="500"
            android:valueFrom="0"
            android:valueTo="300"
            android:valueType="floatType"/>

</set>
```
``` java
        AnimatorSet animatorSet = (AnimatorSet) AnimatorInflater.loadAnimator(this,R.animator.set_animator);
        animatorSet.setTarget(mTextView1);
        animatorSet.start();
```
用方跟上面差不多,不做过多解释.

# 总结 
属性动画差不多就这么多,主要报扩了ValueAnimator,ObjectAnimator和AnimatorSet,使用方法都差不多,并且上面几个动画都可在xml中定义,使用AnimatorInflater 创建,而且都而已设置动画监听,对动画的执行状态进行监听.
