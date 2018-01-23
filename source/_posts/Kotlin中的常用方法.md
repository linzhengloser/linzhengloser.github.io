---
title: Kotlin中的Standard.kt
date: 2017-11-10 10:57:31
tags: Kotlin
---

# 开始

Kotlin 中有很多自带的方法，可以省略很多模板代码。这些方法只是帮我们封装了很多常用的代码如 for if 等等之类的，对性能提升没有什么关系，但是可以提高代码的可读性，让代码逼格上升。

<!-- more -->

# Standard.kt

## Java 版本的代码

首先我们先看看在 Java 里面如果需要实现一个字符串的拼接并打印拼接后的字符串，代码如下：

``` java

StringBuilder stringBuilder = new StringBuilder();
stringBuilder.append("Content :")
stringBuilder.append(stringBuilder.getClass().getCanonicalName())
System.out.println(stringBuilder.toString())

```

上面代码中首先创建一个 StringBuilder 对象，然后添加了两个字符串，最后输出，下面看看在 Kotlin 是怎么实现的。

首先我们扩展一下 Any 的 print() 方法，方便调试。

``` kotlin

fun Any.print() = println(this)

```

## let() vs run()

let() 和 run() 功能类似，只不过 run() 不用使用 it 这个参数来调用被调用者的方法，在上面的例子中，被调用为 StringBuilder 对象的方法。具体代码如下:

``` kotlin

//let()

StringBuilder().let{
    it.append("Content:")
    it.append(it.javaClass.canonicalName)
}.print()

//run()
StringBuilder.run(){
    append("Content:")
    append(javaClass.canonicalName)
}.print()

```

可以看到， let() 与 run() 的区别在，在 run() 中不需要使用 it 就可以直接使用 StringBuilder 的方法，其他地方没有任何区别。

## apply() vs also()

apply() 和 also() 的作用和上面的 let() 和 run() 类似，只是在返回上有点区别，代码如下

``` kotlin

//also()
StringBuilder().also {
    it.append("content: ")
    it.append(it.javaClass.canonicalName)
}.print()

//apply()
StringBuilder().apply {
    append("content:")
    append(javaClass.canonicalName)
}.print()

```

乍一看好像和 let()/run() 没啥区别，其实区别在返回值地方，我们可以通过 Standard.kt 的源码中看到区别，代码如下：

``` kotlin

@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R = block()

@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R = block(this)

@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T { block(); return this }

@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.also(block: (T) -> Unit): T { block(this); return this }

```

通过上面的源码可以看到，run()和let() 返回的是 R 类型的即我们定义的代码块的返回值。而 apply() 和 also() 返回的是 this，即指的是调用者对象本身。


## takeIf() 和 takeIf()

takeIf() 和 takeIf() 可以帮助我们实现一个 if 判断，代码如下：

``` kotlin

//takeUnless() 只是 takeIf() 取反而已
val list = listOf<Int>(1,2,3,4,5)
list.takeIf{!it.isEmpty()}?.map{doSomeThing()}
list.takeUnless{it.isEmpty()}?.map{doSomeThing()}

//上面代码等价于，当然 map() 自身就会对 list 做非空判断，这里只是做个参考

if(!list.isEmpty()){
    list.map{doSomeThing()}
}

```

## with 方法

with() 和上面的有点区别，它是一个普通方法而不是一个扩展方法，具体用法如下:

``` java

with(StringBuilder()){
    append("content:")
    append(javaClass.canonicalName)
}

```

可以发现，with() 方法和上面的 let() 作用一样，只不过他是一个方法而已。

## repeat() 方法

使用 repeat() 可以快速实现一个循环，代码如下：

``` kotlin

repeat(10){
    print(it)
}

```

上面代码会输出 0 ~ 9。

