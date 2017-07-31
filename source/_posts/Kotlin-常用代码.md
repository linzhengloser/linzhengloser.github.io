---
title: Kotlin-常用代码
date: 2017-06-04 16:26:34
tags: Kotlin
---

# 开始
上一篇文章学习了一下 Kotlin 的基本语法，这一篇来学习一下 Kotlin 的一些常用代码。

下面例子都来自 Kotlin 的官方文档. http://kotlinlang.org/docs/reference/idioms.html

<!-- more -->

# 创建实体类( JavaBean )

``` Kotlin

data class Customer(val name: String, val email: String)

```

# 方法的默认参数

``` Kotlin

fun foo(a: Int, b: String = ""){ ... }

```

# 过滤集合

``` Kotlin

val positives = list.filter{ x -> x > 0 }

val positives = list.filter{ it -> 0 }

```

# 字符串插入

``` Kotlin

println("name $name")

```

# 实例检测

``` kotlin

when(x){
    is Foo -> ...
    is Bar -> ...
    else   -> ...
}

```

# 遍历 Map

``` Kotlin

for((k, v) in map){
    println("$k -> $v")
}

```

# 使用范围
``` Kotlin

for (i in 1..100){ ... } //循环 1 到 100
for (i in 1 until 100){ ... }

```

# 创建只读 Map

``` Kotlin

val list = listoOf("one","two","three")

val map = mapOf("a" to 1, "b" to 2, "c" to 3)

```

# 访问Map中的对象

``` Kotlin

println(map["key"])
map["key"] = value

```

# 懒属性

``` Kotlin

val p: String by lazy {
    //compute the string
}

```

# 扩展方法

``` Kotlin

fun String.spaceToCamelCase(){ ... }
"Convert this to camelcase".spaceToCameCase()

```

# 创建一个单例

``` Kotlin

object Resource {
    val name = "Name"
}

```

# 判断不是 null 

``` Kotlin

val files = File("Test").listFiles()
println(files?.size ?: "empty")

```

# 判断不是 null 的缩写

``` Kotlin

val files = File("Test").listFiles()
println(files?.size)

```

# 不为空才执行

``` Kotlin

val data = ...
data?.lat {
    //execute this block if not null
}

```

# 返回一个 when 语句

``` Kotlin

fun transform(color: String): Int{
    return when (color){
        "Red" -> 0
        "Green" -> 1
        "Blue" -> 2
        else - throw IllegalArgumentException("Invalid color param value")
    }
}

```

# try/catch 表达式

``` Kotlin

fun test(){
    val result = try{
        count()
    } catch (e: ArithmeticException){
        throw IllegalStateException(e)
    }

    //working with result
}

```

# if 表达式

``` Kotlin

fun foo(param: Int){
    val result = if(param == 1){
        "one"
    } else if(param == 2){
        "two"
    } else {
        "three"
    }
}

```

# 一行代码表示方法

``` Kotlin

fun theAnswer() = 42

//和下面的代码作用一样

fun theAnswer(): Int{
    return 42;
}

```

## `when` 表达式也能这样写，可以使代码更简短

``` Kotlin

fun transform(color: String): Int = when (color) {
    "Red" -> 0
    "Green" -> 1
    "Blue" -> 2
    else -> throw IllegalArgumentException("Invalid color param value")
}

```

# 在对象实例上调用多个方法 `with`

``` Kotlin

class Turtle{
    fun penDow()
    fun penUp()
    fun turn(degrees: Double)
    fun forward(pixels: Double)
}

val myTurtle = Turtle()
with(myTurtle) {
    penDown()
    //画一个 100 像素的正方形
    for(i in 1..4 ) {
        forward(100.0)
        turn(90.0)
    }
    penUp()
}

```

# 使用可空的 Boolean 值

``` Kotlin

val b: Boolean? = ...
if (b == true) {
    ...
} else {
    //Boolean 等于 false 或者 Boolean 等于 null
}

```