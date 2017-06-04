---
title: Kotlin-基本语法
date: 2017-05-28 20:39:53
tags: Kotlin
---
# 开始
因为在今年的 Google的IO 大会上，谷爹宣布将Kotlin作为安卓的一级开发语言，以后肯定会接 Java 的班，所以趁现在赶紧来学习学习 Kotlin 这门语言。

<!-- more -->

Kotlin 的官方文档对使用 Kotlin 开发 Android 的描述是:

>Kotlin is a great fit for developing Android applications, bringing all of the advantages of a modern language to the Android platform without introducing any new restrictions:

翻译过来意识就是，Kotlin非常适合开发 Android 应用,将现代语言的所有优势带入Android平台。

下面先学习一下 Kotlin 的基本语法。

以下代码来自 Kotlin 的官方文档:https://kotlinlang.org/docs/reference/basic-syntax.html

# 方法(`Fun`)

``` Kotlin

fun sum(a: Int, b: Int): Int {
    return a + b
}

fun sum(a: Int, b: Int) a + b

fun printSum(a: Int, b: Int): Unit{
    println("sum of $a and $b is ${a + b}")
}

fun printSum(a: Int, b: Int){
    println("sum of $a and $b is ${a + b}")
}


```

定义方法，使用关键字`Fun`，和 Java 不一样的是，在不是在方法名前面定义方法的返回类型,而是在参数后面定义方法的返回类型(或者可以不写放回类型,参考第二种方法的定义)。

当方法体只有一行的时候可以直接使用第二种定义方式，而且返回的类型也不需要指定。

Kotlin 中如果想让方法不需要返回值，可以使用 `Unit` 类型，通常情况下 `Unit` 可以省略。

# 局部变量(`val` and `var`)

``` Kotlin

val a: Int = 1 //立即赋值
val b = 2; //`Int` 类型是推断出来的
val c: Int //声明的时候没有赋值，但是必须指定类型
c = 3 //必须给c赋值 否者会编译异常

var x = 5 //同上，Int类型是推断出来的
x += 1 

```

定义局部变量使用到两个关键字 `val` 和 `var`，其中 `val` 声明一个只读的变量，类似 Java 里面的常量

另外一个关键字 `var` 是定义一个可变的变量，类型 Java 里面的变量

和Java 不同的是 Kotlin 可以使用推断类型，就是让编译器去推断出这个变量是什么类型，有点类型 JavaScript里面的 `var` 关键字，当然也可以手动指定类型。

# 字符串模板

``` Kotlin

var a = 1
val s1 = "a is $a"

a = 2
val s2 = "${a.replace("is", "was")}, but now is $a"

```

字符串模板，顾名思义就是将一些值添加到已经写的字符串里面。Kotlin 中的字符串模板使用 `$` 关键字。

在字符串中可以使用关键字 `${}` 在字符串模板中调用方法，比如上面使用 String 类中的 `replace` 方法，将 s1 的 is 替换成 was。

# 条件表达式(`if` and `else`)

``` Kotlin

fun maxOf(a: Int, b: Int): Int{
    if(a > b){
        return a
    }else{
        return b
    }
}

fun maxOf(a: Int, b: Int) = if (a > b) a elase b

```

条件表达式使用 `if` 和 `else` 关键字，这个和 Java 里面用法一样,但是在 Kotlin 中如果只有一行代码 可以使用第二种写法，这个比 Java 中省略大括号要更简洁。

# 空值判断(`?`)

``` Kotlin

fun parseInt(str: String): Int?{
    return str.toIntOrNull()
}

fun printProduct(arg1: String, arg2 :String){
    val x = parseInt(arg1)
    val y = parseInt(arg2)

    if(x != null && y != null){
        println(x*y)
    }else{
        println("either $arg1 or $arg2 is not a number")
    }
}

```

在Kotlin中，在类型后面加上关键字`?`，表示这个值可能是null。

上面代码中 parseInt 方法如果Int后面去掉 `?` 会导致编译报错，因为 String 类的 toIntOrNull 会返回Null，并且在 printProduct 方法中必须要对 x 和 y 进行非空的判断，不然编译也会报错，因为上面的 parseInt 方法是会返回 null 的。

# 类型检查和类型转换(`is`)

``` Kotlin

fun getStringLength(obj: Any): Int? {
    if(obj is String){
        retrun obj.length
    }
    return null
}

fun printStringLength(obj: Any){
    println("'$obj' string length is ${getStringLength(obj) ?: "... err, not a string"} ")
}

```

在 Java 里面如果我们要对一个类型做判断，是用关键字 `instanceof` ，那在 Kotlin 中就是用 `is` 关键字，效果差不多。

上面用到了 `?:` 这个表达式(暂且叫它表达式)，这个表达式和 Java 中的 `三元运算符` 差不多，但是 `?:` 是对前面的值进行判空 ，如果为 null 就执行后面。这个 Java 里面的的`三元运算符`有点区别，个人觉得 `?:` 更好用些。

# 循环(`while` and `for`)

``` Kotlin

val items = listof("apple","banana","kiwi")

for(string in items){
    println(string)
}

for(index in items.indices){
    println(items[index])
}

var index = 0
whiel(index < items.size){
    println(items[index])
    index++
}

```

Kotlin中的循环和Java中的循环大同小异，唯一不一样的就是省略了类型的声明。

# `when`表达式

``` Kotlin

fun describe(object: Any): String = 
when(object){
    1 -> "One"
    "Hello" -> "Greeting"
    is Long -> "Long"
    !is String -> "Not a String"
    elase -> "Unknown"
}

```

Kotlin 中的 `when` 表达式和Java里面的 `switch` 很像，但是也有很明显的区别，就是 `when`中可以使用任何类型这个比 `switch 更有灵性。

# 范围

``` Kotlin

val x = 10
val y = 9

if(x in 1..y+1){
    println(fits in range)
}

val list = listof("a","b","c")
if(-1 !in 0..list.lastIndex){
    println("-1 is out of range")
}
if(list.size !in list.indices){
    println("list size is out of valid list indices range too")
}

for(x in 1..5){
    println(5)
}

for(x in 1..10 step 2){
    println(x)
}

for(x in 9 downTo 0 step 3){
    println(x)
}

```

用使用 `..` 来指定范围，并使用 in 可以判断某个值是否在一个范围内。在for循环中可以使用 step downTo 等方法来完成一些特殊的操作。

# 集合

``` Kotlin

val list = listOf("apple","banana","kiwi")
for(string in list){
    println(string)
}

when{
    "orange" in list -> println("juicy")
    "apple" in list -> println("apple is fine too")
}

val fruits = listOf("banana","avocado","apple","kiwi")
fruits
.filter{it.startsWith("a")}
.sortedBy{it}
.map{it.toUpperCase()}
.forEach{println(it)}

```

在 Kotlin 中，可以使用 `when` 关键字来筛选出集合中想要的数据，和 Java8 一样可以使用一些操作符加 Lambda 表达式来操作集合。