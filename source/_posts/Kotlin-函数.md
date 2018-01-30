---
title: Kotlin-函数
date: 2018-01-30 17:18:22
tags: Kotlin
---

# 开始

本篇博客将介绍 Kotlin 中的函数。

下面的例子都来自于 Kotlin 的官方文档。

<!-- more -->

# 普通函数

函数是在编程中必不可少的一部分，Kotlin 吸取了一些现代语言的函数定义的优点，并将这些优点融入到 Java 的函数中，方便我们日常的开发。

## 函数的基本使用

Kotlin 中使用 `fun` 关键字声明函数，如下：

``` kotlin

fun double(x: Int): Int {
    return 2 * x
}

//调用函数
val result = double(2)

```

Kotlin 中运行给函数添加默认参数，在很多语言中都有这一特性，但是唯独 Java 中没有，原因可能是因为 Java 的设计者觉得 Java 中有方法重载，所以必须要默认参数了，因为方法重载和默认参数两者的效果是一样。

默认参数的定义如下：

``` kotlin

fun read(b: Array<Byte>, off: Int = 0, len: Int = b.size) {

}

//当重写父类方法的时候，如果被重写的方法有默认参数
open class A {
    open fun foo(i: Int = 10) { ... }
}

class B : A() {
    //不能有默认值
    override fun foo(i： Int) { ... }
}

//当默认参数在无默认参数的前面的时候
fun foo(bar: Int = 0,baz : Int) { ... }
//调用 foo() 方法
foo(baz = 1) //使用默认值 bar = 0


//当 lambda 表达作为参数的时候
fun foo(bar: Int = 0, baz: Int = 1, qux: () -> Unit) {
    ....
}

foo(1){ println("hello") } //使用默认参数 baz = 1
foo{ println("hello") } //使用默认参数 bar = 0 baz = 1

```

## 命名参数

当一个函数有大量的参数的时候，而且这些参数中又又默认参数，这个时候如果无需要调用这个函数，并不使用默认参数，就会变得很麻烦，为此，Kotlin 提供了名称参数这一特性，来改善这一情况，代码如下：

``` kotlin
```