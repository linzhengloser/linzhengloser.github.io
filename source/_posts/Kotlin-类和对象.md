---
title: Kotlin-类和对象
date: 2018-01-02 20:35:41
tags: Kotlin
---

# 开始

本篇博客介绍 Kotlin 中关于类和对象的使用。

下面例子都来自 Kotlin 的官方文档。

<!-- more -->

# 类和继承

## 类的声明

在 Kotlin 中使用 `class` 关键字来声明类，和 Java 类似，代码如下：

``` kotlin

class Invoice{

}

```

如果类没有主体，即该类没有变量和方法，可以省略大括号，代码如下：

``` kotlin

class Invoice

```

在 Kotlin 中一个只能有一个主构造函数，但是可以有多个次构造函数，声明方式如下：

``` kotlin

class Person constructor(firstName: String) {
}

//如果主构造函数没有修饰符或注释，可以省略 constructor 关键字。
class Person(firstName: String){
}

```

如果我们需要在类构造的时候执行一些代码，可以使用 `init` 关键字，这个关键字可以让代码在类构造的时候被执行，且一个类可以有多个 `init` 关键字。

`init` 关键字和属性初始化的执行顺序是从上到下依次执行的。代码如下：

``` kotlin

class InitOrderDemo(name: String) {
    init{
        println("First initializer block that prints $name")
    }

    val firstProperty = "First property: $name".also(::prinln)

    val secondProperty = "Second property: ${name.length}".also(::println)

    init{
        println("Second initializer block that prints ${name.length}")
    }
}

//执行如下代码
InitOrerDemo("hello")

//输出：
//First initializer block that prints hello
//First property: hello
//Second property: 5
//Second initializer block that prints 5

```

关于 `init` 关键字，需要注意的是，如果一个类中既有 `init` 又有次级构造函数，那么 `init` 中的代码会被优先执行，其次才会执行次构造函数中的代码。

跟 Java 中一样，Kotlin 中每个类可以声明多个构造函数，且构造函数之间可以相互调用，在声明次构造函数的时候，使用 `constructor` 关键，如果需要掉用其他的构造函数，可以使用 `this` 关键字，代码如下：

``` kotlin

class Person(val name: String) {

    //声明次构造函数
    constructor(name: String, parent:Person) : this(name){
        parent.children.add(this)
    }

}

```

在 Java 中如果我们希望一类不能被实例化，可以使用 private 关键在声明构造函数，这样的话，就无法使用这个被 private 声明的构造函数的方式来构建该类的对象，Kotlin 也是如此，只需要把 private 关键字写在 constructor 关键字前面即可。


## 类的创建

Java 里面创建一个类型的对象只需要使用 new 关键字加类名即可，但是在 Kotlin 中，帮我们省略了这个 new 关键字，直接使用类名就可以创建这个类的实例，代码如下：

``` kotlin

//使用无参数的构造函数创建对象
val invoice = Invoice()

//使用有参数的构造函数创建对象
val customer = Customer("Joe Smith")

```

## 类的继承

在 Java 中所有类默认继承 Object 类，在 Kotlin 中，所有类默认继承来 Any 类，而 Kotlin 中的 Any 类是和 Java 中的 Object 类有区别的，区别在于 Java 中的 Object 类有很多方法，而 Kotlin 中的 Any 类只有 3 个方法，分别是 hashCode()，equals() 和 toString()。

需要注意的是 Kotlin 中默认所有类都是 final 类型的，即在 Kotlin 中默认一个类是不能被继承的，所以 Kotlin 有一个 open 关键字在标识一个类可以被继承。代码如下：

``` kotlin

//此处必须要使用 open 关键字
open class Base(p: Int)

class Derived(p:Int ) : Base(p)

```

通过上面代码可以看到，如果父类有一个有参数的构造函数(在 Kotlin 中叫做主构造函数)，那么子类也必须要调用这个构造函数，这一点是和 Java 中是一样的。

如果一个类有多个构造函数(在 Kotlin 中叫做多个次构造函数，且没有主构造函数)，那么这个类的子类必须调用父类所有构造函数，这个和 Java 中也是一样的。

代码如下：

``` kotlin

class MyView : View {

    constructor(ctx: Context) : super(ctx)

    constructor(ctx: Context, attrs: AttributeSet) : super(ctx, attrs)

}

```


### 重写方法

在 Java 中如果子类需要重写父类中的方法，只需要方法的签名相同则符合重写的规则，但是在 Kotlin 中，子类如果想要重写父类的某个方法，不仅需要方法签名相同，还需要使用 `override` 关键字，并且需要在 `fun` 前面加上 open，因为在 Kotlin 中方法和类一样默认是 `final` 的，所以如果想要重写 or 继承，都需要时手动声明 `open`。 代码如下：

``` kotlin

open class Base {

    open fun v() {}

    fun nv() {}

}

class Derived(): Base() {
    //使用 override 关键字，重写父类 v 方法。
    override fun v() {}

    //此行代码会报错，因为 nv 方法不是 open 的。
    override fun nv() {}
}

```


### 重写属性

在 Kotlin 中重写属性和重写方法的方式基本一致，都是使用 override 关键字来实现。可以使用 `val` 实现来重写 `var` 属性，但是不能用 `var` 属性来重写 `val` 属性。

`override` 关键还可以在主构造函数中使用

代码如下：

``` kotlin

//这里声明的是接口所以默认 count 不用初始化值。
interface Foo {
    val count: Int
}

//直接在主构造函数中重写
class Bar1(override val count: Int) : Foo

//在类的内部使用 override 关键字重写
class Bar2 : Foo {
    override var count: Int = 0
}


```

### 调用父类实现

既然可以重写父类的方法和属性，那么肯定也能调用父类的属性和方法，在 Java 中使用 super 关键字调用，Kotlin 也是一样。代码如下：

``` kotlin

open class Foo {
    open fun f() { println("Foo.f()") }
    open val x: Int get() = 1
}

class Bar : Foo(){
    override fun f() {
        super.f()
        println("Bar.f()")
    }
    override val x: Int get() = super.x + 1
}

```

还有一种特殊的情况，就是在一个内部类中如果想调用这个内部类的外部类的父类的方法，需要使用 `super@Outer` 来调用，代码如下：

``` kotlin

class Bar : Foo() {
    override fun f() {  }
    override val x: Int get() = 0

    //使用 inner 关键字声明内部类
    inner class Baz {
        fun g() {
            //调用的 Foo 类中的 f() 方法
            super@Bar.f()
        }
    }
}

```

除了上面这种情况，还有一种比较特殊的情况，我们都知道在 Java 中类是单继承的，即一个类有且只有一个父类。但是一个类可以实现多个接口，那么如果当一个类的父类中和这类所实现的接口中有方法的方法签名是一样的，那么这个时候子类需要重写这些签名一样的方法，不然连编译都不能通过。具体代码如下：

``` kotlin

open class A {
    open fun f() { print("A") }
    open a() { print("a") }
}

interface B {
    //在 kotlin 的接口中，方法默认是 open 的
    fun f() { print("B") }
    fun b() { print("b") }
}

class C() : A(), B {
    //这里必须重写 f() 方法
    override fun f() {
        //调用 A 类中
        super<A>.f()
        super<B>.f()
    }
}

```


# 属性和字段

在 Java 中其实没有属性这种概念，但是可以使用私有化变量加提供 `getter` 和 `setter` 来实现类似属性的特性。属性这种叫法是 C# 中的(可以参考这篇博客中的描述 https://www.cnblogs.com/mcgrady/p/3411405.html )，Kotlin 结合了属性这种概念并和 Java 中的语法相结合，规定了一套自己的属性声明规则。

## 声明属性

在 kotlin 中声明属性，使用 `var` 和 `val` 关键字，其中 `var` 声明的属性有 `getter` 和 `setter`，而 `val` 声明的属性没有 `setter` 只有 `getter`。可以把 `val` 声明的属性理解成是只读属性。

属性的声明方式如下：

``` kotlin

class Address {
    var name: String = ""
    var street: Strig = ""
    var city: String = ""
    var state: String? = null
    var zip: String = ""
}

```

属性的使用方式如下：

``` kotlin

fun copyAddress(address: Address)：Address {
    val result = Address()
    result.name = address.name
    result.street = address.street
    //....
    return result
}

```

### Getter and Setter

属性的完整语法是：

``` kotlin

var <propertyName>[: <PropertyType>] [= <property_initializer>]
[<getter>]
[<setter>]

```

一个属性的声明属性名称，其他的都可以省略，比如属性的类型是可以通过属性的值来自动推断的。

下面有几个声明属性的例子，代码如下：

``` kotlin

//这里是错误的，在 Java 中我们声明一个字段是可以不给初始值，
//默认是类型对应的初始值，但是在 Kotlin 中除非使用 lateinit 关键字
//否则所有属性必须在声明的时候指定其初始值。当然这里不包括构造函数中声明的。
var allByDefalut: Int? 

//声明一个名称为 initialized  的属性，使用类型推断编译器推断出 initialized  的类型为 Int，
//其初始值为 1
var initialized = 1

//这里是错误的，必须指定其初始值，且使用 val 关键声明的属性不能再其他地方修改其值，即表示使用
//val 声明的属性没有 setter
val simple: Int?

//和上面的 initialized 一样，除了没有 setter 外，
val inferredType = 1

```

默认情况下，Kotlin 自动帮我们生成属性的 `getter` 和 `setter`，如果我们需要修改一个属性的 `getter` 和 `setter`，可以使用下面方法：

``` kotlin

//修改 isEmpty 属性的 getter，其类型是 Boolean 类型
//在 Kotlin 1.1 之前，这里可能需要手动指定 isEmpty 的类型，但是在 1.1 之后 getter 可以自动推断出类型。
val isEmpty get() = this.size == 0

//修改  stringRepresentation 属性的 setter，默认 setter 的参数名称为 value，当然也可以自定义。
var stringRepresentation: String 
    get() = this.toString()
    set(value) {
        setDataFromString(value)
    }

//设置属性的 setter 为私有的
var setterVisibility: String = "abc"
    private set

//使用依赖注入框架对属性进行初始化
var setterWithAnnotation:Any? = null
    @Inject set

//可以使用 field 关键字在 setter 中指定属性的值，且 field 关键字只能在 getter 或者 setter 中使用
var counter = 0
    set(value) {
        if(value >= 0) field = value
    }

```

## 编译期常量

在 Kotlin 中可以声明编译期常量，即一些属性的值是"死代码"，有点类似常量，但是又有区别，因为常量可以在类的 `init` 中初始化，总而言之，编译期常量就是"死代码"。

在 Kotlin 中可以使用 `const` 关键字声明编译期常量，但是又下面几点注意事项：

1. 编译期常量必须在一个文件的最顶级或者在 `object` 中。
2. 编译期常量只能是 String 或者基本数据类型。
3. 编译器常量无法自定义 getter。

下面是编译期常量的声明方法：

``` kotlin

//编译期常量的声明
const val SUBSYSTEM_DEPRECATED: String = "This subsystem is deprecated"

//可以在注解中使用编译期常量
@Deprecated(SUBSYSTEM_DEPRECATED) fun foo() { ... }

```

## 延迟初始化和变量

在 Kotlin 中，一般情况下非空类型的属性必须在构造函数中初始化，但是也有特殊的情况，比如有些值必须在某个特定的地方才能获取到其的具体值比如在 Android 中获取 Intent 传递过来的值，虽然我们可以手动给属性知道一个初始值，然后在在后面修改这个初始值，但是这样做是不好的，推荐的做法是使用 `lateinit` 关键字，代码如下：

``` kotlin

public class MyTest {
    //如果不是用 lateinit 关键字，这里会编译不通过。
    lateinit var subject: TestSubject

    @Setup fun setup() {
        subject = TestSubject()
    }

    @Test fun test() {
        subject.method()
    }

}

```

值得注意的是 lateinit 关键字只能作用于非空类型，且不能是原生类型(String 不是原生类型)。如果我们试图访问一个没有被初始化但是是使用 lateinit 声明的属性，会抛出一个异常。在 Kotlin 1.2 中提供了 `isInitialized` 来判断用 `lateinit` 声明的属性是否有初始化。代码如下：

``` kotlin

if (foo::bar.isInitialized) {
    println(foo.bar)
}

```


# 接口

Kotlin 中的接口和 Java8 中的类似，除了可以声明抽象方法以外，还可以声明默认方法，但是在 Java8 中需要使用 `defalut` 关键字来声明默认方法，而 Kotlin 的接口中的默认方法的声明和普通方法一致。代码如下：

``` kotlin

interface MyInterface {

    //抽象方法
    fun bar()

    //有方法体的方法
    fun foo() {

    }

}

class Child : MyInterface {

    //只需要实现 bar() 方法
    override fun bar(){

    }
}

```

## 接口中的属性

在 Kotlin 中可以在接口里面声明属性，但是属性必须是没有初始值或者提供了 getter，代码如下：

``` kotlin

interface MyInterface {

    val prop: Int

    val propertyWithImplementation: String
        get() = "foo"

    fun foo() {
        print(prop)
    }

}

class Child: MyInterface {

    //此处必须重写 prop 属性
    override val prop = 123

}

```

# 可见性修饰符

Kotlin 中的类，对象，接口，构造函数，方法和属性 `setter` 都是可以设置可见性修饰符的(`getter` 使用属性的可见性修饰符)。

Kotlin 中的可见性修饰符有 `private`，`protected`，`internal` 和 `public`，其中除了 `internal` 是 Kotlin 中特有的，其他几个修饰符和 Java 里面的修饰符一致。

## 包

Kotlin 中可以直接在文件中声明函数，属性，类，对象和接口，但是 Java 中不行，Java 中的函数，对象，属性必须在类中声明。代码如下：

``` Kotlin

//example.kt
package foo

fun baz() {}

var number: Int = 1


```

关于可见性修饰符，需要注意以下几点：

1. 如果不指定修饰符，那么默认是 `public` 修饰符，这里和 Java 是不一样的，Java 的默认修饰符是同包下可以访问。
2. 如果使用 `private` 修饰符，它只能在它声明的文件中可见。
3. 如果使用 `internal` 修饰符，它只能在他声明的模块中可见。
4. `protected` 修饰符不能用于顶层。

代码如下：

``` kotlin

//example.kt
package foo

private fun foo() {}//只能在 example.kt 中访问

public var bar: Int = 5 //bar 在任何地方都可以访问
    private set //setter 只能在 example.kt 中访问

```


## 类和接口

在类和接口中使用修饰符，需要注意一下几点：

1. `private` 表示只能在该类的内部可见。
2. `protected` 表示只能在该类和该类的子类中可见。
3. `internal` 表示在该模块中的任何地方可见。
4. `public` 表示任何地方都可见。

在使用基础的时候，如果覆盖一个 `protected` 修饰的成员，默认被覆盖的成员也是 `protected`。

需要注意的是 Kotlin 中外部类不能访问内部类的 `private` 修饰的成员。

代码如下：

``` kotlin

open class Outer {
    private val a = 1
    protected open val b = 2
    internal val c = 3
    val d = 4

    protected class Nested {
        pubic val e:Int = 5
    }
}

class Subclass : Outer {
    //Outer 类中的
    //a 不可见
    //b,c 和 d 可见
    //内部类 Nested 中的 e 可见
    override val b = 5 //此次覆盖父类中的 b 修饰符还是 protected
}

class Unrelated(o: Outer) {
    //o 中的 a 和 b 不可见
    //o 中的 c 和 d 可见
    //o 中的 Nested 内部类和内部类中的 e 都是不可见的
}

```

## 构造函数

类的构造函数也是可以指定修饰符的，在指定修饰符的时候，必须要使用 `constructor` 关键字。默认情况下，构造函数的修饰符是 `public`。

代码如下：

``` kotlin

class C private constructor(a: Int) { ... }

```


## 局部变量

局部变量和局部方法都是不能添加修饰符的。

## 模块

上面提到的 `internal` 修饰符所指的模块的含义如下：

1. 一个 IntelliJ IDEA 模块。
2. 一个 Maven 项目。
3. 一个 Gradle 源集。
4. 一次 kotlinc Ant 任务执行所编译的一套文件。

# 扩展

Kotlin 中提供了扩展这一特性，这个特性主要是帮助开发者，在不继承一个类，或者不使用任何设计模式的情况下，对一个类进行扩展。所谓扩展就是给这个类添加额外的函数或者属性。

## 扩展函数

使用扩展函数这一特性，我们为 MutableList<Int> 添加一个 swap(index1: Int,index2： Int) 函数，具体代码如下：

``` kotlin

fun MutableList<Int>.swap(index1:Int, index2:Int) {
    val temp = this[index1]
    this[inde1] = this[index2]
    this[index2] = temp
}

```

在上面的 swap 方法中，`this` 关键字指的就是调用该方法的 list 对象实例。调用方法的代码如下：

``` Kotln

val l = mutableListOf(1,2,3)
l.swap(0,2)

```

上面我们只是扩展了 MutableList<Int>，只有当 list 泛型为 Int 的时候才能调用，其实是可以直接扩展所有类型，代码如下：

``` Kotlin

fun <T> MutableList<T>.swap(index1: Int, index: Int) {
    val temp = this[index1]
    this[index1] = this[index2]
    this[index2] = temp
}

```

### 扩展函数的本质

Kotlin 中的扩展函数本身其实是使用了 Java 中的静态方法实现的。这也说明了扩展函数的调用只是简单的一个静态方法调用，起调用结果不会受到多态的影响。代码如下：

``` kotlin

open class C 

class D: C()

fun c.foo() = "C"

fun D.foo() = "D"

fun printFoo(c: C) {
    println(c.foo())
}

//最终输出的是 C
printFoo(D())

```

### 扩展函数和成员函数

当扩展函数的签名和成员函数的签名一样的时候，我们调用其中一个方法，会只调用成员函数，而不是扩展函数。扩展函数可以对成员函数进行重载。总之扩展函数的优先级比成员函数低，如果函数调用同时满足成员函数和扩展函数，这个时候只会调用成员函数。

代码如下:

``` kotlin

class C {
    fun foo() { println("Member") }

}

fun C.foo() { println("Extension") }

fun C.foo(number: Int) { println("Extension") }

//调用 C 的 foo() 方法，输出 Member
//调用 C 的 foo(number: Int) 方法，输出 Extension

```

### 扩展可空类型

可以通过定义可空类型的扩展函数，来让可空类型也可以使用扩展函数，即使这个可空类型的值是 null。代码如下：

``` kotlin

fun Any?.toString(): String {
    if(this == null) return "null"
    //在空检查后面的代码自动转换成非空类型
    return toString()
}

```

## 扩展属性

除了可以扩展函数之外，还可以扩展属性。具体代码如下：

``` kotlin

val <T> List<T>.lastIndex: Int
    get() = size - 1

val Foo.bar = 1 //异常，扩展属性不能有初始值

```

注意扩展属性不会修改原本的值，只会影响被扩展属性的 getter 和 setter，而且扩展函数是不能有初始值的。


## 伴生对象扩展

Kotlin 中提供了伴生对象这一特性来实现类似 Java 中的静态特性。扩展函数可以对伴生对象进行扩展，这样调用扩展函数的时候，就不需要创建对象的实例。具体代码如下：

``` kotlin

class MyClass {
    //声明伴生对象
    companion object { }
}

fun MyClass.Companion.foo() {
    ...
}

//调用
MyClass.foo()

```


## 声明扩展成员

在一个类的内部，可以声明其他类的扩展函数，并且在扩展函数中可以调用本来的方法，当出现方法签名一样的情况的时候，可以使用 `this@` 关键字手动指定调用本类中的方法。具体代码如下：

``` kotlin

class D {
    fun bar() { ... }
}

class C {
    fun baz() { ... }
    fun D.foo() {
        bar() //调用 D 类中的 bar() 方法
        baz() //调用 C 类中的 bar() 方法
    }

    fun caller(d: D) {
        //调用扩展函数
        d.foo()
    }

    fun D.foo() {
        toString() // D 类的 toString() 方法
        this@C.toString() //C 类的 toString() 方法
    }

}


```


## 扩展函数的用处

在日常开发者，为了节约时间，会编写大量的 xxxUtils 类来讲模板代码编写成方法，比如 Java 中的 `java.util.Collections`。这样做法方便了我们日常开发，但是在调用的时候，会出现大量的重复代码。但是如果使用扩展函数，就可减少很多重复的代码。

没有扩展函数：

``` java


Collections.swap(list,Collections.binarySearch(list,Collections.max(otherList)));

//使用静态导入
swap(list,binarySearch(list,max(otherList)));

```

虽然可以使用静态导入来减少重复的代码，但是 IDE 并不能制动识别静态导入，换言之，静态导入的代码自己手动编写。

有了扩展函数：

``` kotlin

list.swap(list.binarySearch(otherList.max()),list.max())

```

扩展函数这一特性可以省略很多以前的重复代码。

# 数据类

在日常开发中，经常为了存储数据而创建一些类，这些类代码的基本一致，Kotlin 为了节省代码，提供了 `data class` 来专门创建这些用来存储数据的类，声明方式如下：

``` kotlin

data class User(val name: String, val age: Int)

```


编译器会帮我们生产下列方法：

1. equals() 和 hashCode()。
2. toString() 格式为 (User=Linzheng, age=20)。
3. componentN() 用于排序。
4. copy() 用于复制对象。

为了确保生成的代码的一致性，创建的数据类需要瞒住下面这些条件：

1. 主构造函数至少需要一个参数。
2. 主构造函数的所有参数必须要是 `val` 或 `var`。
3. 数据类不能是抽象的，开放的，密封的，内部的。
4. 在 Kotlin 1.1 之前，数据类只能实现接口不能继承。

数据类生产的方法遵循下列规则：

1. 如果数据类本身有实现 toString()，hashCode() 或 equals() 方法，或者其父类中使用 `final` 实现了这些方法，那么编译器将不会帮我们生成这些方法，而是使用我们自己声明的。
2. 如果父类具有 `open` 的 componentN() 方法，并且返回兼容的类型，那么数据类会生成相应的方法，并且覆盖父类的实现。如果父类的这些方法的签名不兼容或者是 `final` 的，那么会报错。
3. 不能为 componentN() 已经 copy() 方法提供显示实现。

如果希望创建的类有一个无参的构造函数，那么可以为这个数据类的主构造函数的所有参数提供默认参数，代码如下：

``` kotlin

data class User(val name: String = "", val age: Int = 0)

```


## 复制

在很多情况下，我们需要修改一个对象的个别属性，并派生出一个新的对象，这个时候使用 copy() 就很方便，上面的 User 类就会生成如下的 copy() 方法：

``` kotlin

fun copy(name: String = this.name, age: Int = this.age) = User(name, age)

//copy

val linzheng = User(name = "Linzheng", age = 18)
val oldLinzheng = jack.copy(age = 20)

```

## 数据类和结构声明

关于结构声明，在后面的博客中会进行介绍，具体代码如下：

``` kotlin

val jane = User("Jane", 35)
val (name, age) = jane //解构声明
println("$name ,$age years of age") //输出 Jane，35 years of age

```

# 密封类

Kotlin 中的密封类，和枚举有点类似，可以限制一个类的类型，和枚举不一样的是，枚举只会有一个实例，而密封类的子类可以有多个实例。

密封类使用 `sealed` 关键字声明，且密封类的子类必须要和密封类声明在同一个文件中，在 Kotlin 1.1 之前，密封类的子类还必须要声明在密封类的内部。

密封类自身是抽象的，它不能直接实例化，且可以有抽象的成员。密封类值允许声明 `private` 的构造函数，在密封类中声明的构造函数默认就是 `private` 的，继承密封类的子类的类可以声明在其他的文件中。

在 `when` 表达式中使用密封类，可以不用写 `else`。代码如下：

``` kotlin

fun eval(expr: Expr): Double = when(expr) {
    is Const -> expr.number
    is Sum -> eval(expr.e1) + eval(expr.e2)
    NotANumber -> Double.NaN
    //不需要 else，因为我们已经覆盖了所有情况
}

```

# 泛型

Kotlin 中的泛型和 Java 中的泛型基本使用方法是一样的。代码如下：

``` kotlin

class Box<T>(t: T) {
    var value = t
}

val box = Box<Int>(1)

//类型推断
val box2 = Box(1)

```

## 型变

# 嵌套类与内部类

在 Kotlin 中，一个类声明在另一类内部，叫做嵌套类。代码如下：

``` kotlin

class Outer {
    private val bar: Int = 1
    class Nested {
        //在 Nested 类中，无法访问 Outer 类中的成员
        fun foo () = 2
    }
}

```

当我们使用 `inner` 关键字声明嵌套类的时候，这个内部类就可以访问外部类的长远。代码如下：

``` kotlin

class Outer {
    private val bar: Int = 1
    inner class Inner {
        fun foo() = bar
    }
}

val demo = Outer().Inner().foo() // 值为 1

```


## 匿名内部类

Kotlin 中使用 `object` 关键字创建匿名内部类。代码如下：

``` kotlin

window.addMouseListener(object : MouseAdapter(){

    override fun mouseClicked(e: MouseEvent) { ... }

    override fun mouseEntered(e: MouseEvent) { ... }

    })

```

如果对象是函数式 Java 接口，可以使用 `lambda` 表达式来简写代码。代码如下：

``` kotlin

val listener = ActionListener { println("clicked") } 

```

# 枚举

枚举的作用就是限定一个类的类型。基本用法如下：

``` kotlin

enum class Direction {
    NORTH, SOUTH, WEST, EAST
}

```

枚举中每个常量都是一个枚举类的对象，所以他们也可以通过构造函数来初始化，代码如下：

``` kotlin

enum class Color(val rgb: Int) {
    RED(0xFF0000),
    GREEN(0x00FF00),
    BLUE(0x0000FF)
}

```

## 枚举中使用匿名类

枚举常量可以声明自己的匿名类，代码如下：

``` kotlin

enum class ProtocolState {

    WAITING {
        override fun signal() = TALKING
    },

    TALKING {
        override fun signal() = WAITING
    };

    abstract fun signal(): ProtocolState

}

```

需要注意的是，枚举中的常量和方法之间需要用分号分隔。

## 使用枚举

和 Java 中一样，Kotlin 中的枚举提供了两个方法，一个是用来获取枚举类中的所有常量，另外一个是用来通过常量的名称获取常量的实例。代码如下：

``` kotlin

//假设枚举类名称为 EnumClass
EnumClass.valueOf(value: String): EnumClass //通过枚举常量名称获取常量实例
EnumClass.values(): Array<EnumClass> //获取枚举中所有常量

```

# 对象表达式和对象声明

在 Java 中，如果我们只想对一个类进行细微的修改，但是又不想创建一个新类，这个时候，可以使用匿名内部类来实现，而在 Kotlin 中，使用 `object` 关键字声明匿名匿名内部类。具体代码如下：

``` kotlin

window.addMouseListener(object : MouseAdapter() {
    override fun mouseClicked(e: MouseEvent) {
        // ……
    }

    override fun mouseEntered(e: MouseEvent) {
        // ……
    }
})

//如果现在对象表达式中实现父类的构造函数或者是实现别的接口
//写法如下

open class A(x: Int) {
    public open val y: Int = x
}

interface B { ... }

val ab: A = object : A(1), B {
    override val y = 15
}

//在某些特殊情况下，对象表达式还可以这样使用

fun foo() {
    val adHoc = object {
        var x: Int = 0
        var y: Int = 0
    }
    print(adHoc.x + adHoc.y)
}

```

对象表达式是可以作为一个函数返回值，但是如果这个函数不是私有的，默认系统是返回 `Any` 类型的，具体代码如下：

``` kotlin

class C {

    private fun foo() = object {
        val x: String = "x"
    }

    fun publicFoo() = object {
        val x: String = "x"
    }

    fun bar() {
        val x1 = foo().x //可以正常运行
        val x2 = publicFoo().x //编译无法通过，因为 Any 类型没有 x 这个属性  

    }

}

```

在 Java 中，匿名内部类如果想访问外部的变量，就必须要加上 `final`，但是 Kotlin 中却不需要。代码如下：

``` kotlin

fun countClicks(window: JComponent) {
    var clickCount = 0
    var enterCount = 0

    window.addMouseListener(object : MouseAdapter() {
        override fun mouseClicked(e: MouseEvent) {
            clickCount++
        }

        overrid fun mouseEntered(e: MouseEvent) {
            enterCount++
        }
    })
}

```


## 对象声明

使用对象声明这一特性，可以很方便的实现单例设计模式，代码如下：

``` kotlin

object DataProviderManager {
    fun registerDataProvider(provider: DataProvider) {
        // ...
    }
    val allDataProviders: Collecation<DataProvider> 
        get() = //...
}

//对象声明也可以继承其他类
object DefaultListener : MouseAdapter() {
    override fun mouseClicked(e: MouseEvent) {
        //...
    }
    override fun mouseEntered(e: MouseEvent) {
        //...
    }
}

//使用对象声明
DataProviderManager.registerDataProvider(...)

```


## 伴生对象

在类中，使用关键字 `companion`，可以声明一个类的伴生对象，主要是用来实现类型其他语言中的静态成员的调用方式。注意 Kotlin 中的伴生对象使用起来和 Java 中的静态成员一样，但是 Kotlin 中的伴生对象并不是静态的，如果想要实现伴生对象是静态的需要使用 `@JvmStatic` 注解。具体代码如下：

``` kotlin

class MyClass {
    companion object Factory {
        fun create(): MyClass = MyClass()
    }
}

//使用伴生对象
val instance = MyClass.create()

//可以省略伴生对象的名称
class MyClass {
    companion object {
        // ...
    }
}

//当伴生对象没有名称的时候，直接使用 Companion 关键字访问伴生对象实例
val x = MyClass.Companion

//伴生可以实现接口
interface Factory<T> {
    fun create(): T
}

class MyClass {
    companion object : Factory<MyClass> {
        override fun create(): MyClass = MyClass()
    }
}

```

## 对象表达式和对象声明直接的区别

* 对象表达式是在使用它们的地方立即执行。
* 对象声明是在第一次被访问的时候被初始化的。
* 伴生对象的初始化是在类被加载时执行的，和 Java 中的 `static` 一样。


# 委托类

委托设计模式是继承这一特性很好的替代方案，Kotlin 默认支持使用 `by` 关键字来实现委托模式，省略了很多模板代码，具体代码如下：

``` kotlin

interfca Base {
    fun print()
}

//被委托类
class BaseImpl(val x: Int) : Base {

    override fun print() {
        print(x)
    }

}

//委托类
class Derived(d: Base) : Base by b

val b = BaseImpl(10)
Derived(b).print() //输出 10

```


我可以在 Derived 类中手动重写 print() 方法，这样最终调用的就是我们自己实现的 print() 方法，而不是被委托的 BaseImpl 中的 print() 方法。

# 委托属性

在日常开发中，为了实现一些常用的功能，我们需要编写一些重复的模板代码，比如下面这些情形：

1. 懒加载，只有在第一次使用的时候才去加载。
2. 观察属性，可以监听一个数据的变化，在数据变化时候做出一系列操作。
3. 将数据存储在 Map 中。

在 Kotlin 中使用委托属性这一特性，帮我们实现了上述的这些情况。委托属性的基本用法如下：

``` kotlin

class Example {
    var p: String by Delegate()
}

```

委托属性的语法是 `val/var <property name>: <Type> by <Expression>`，`by` 关键字后面的对象实例是被委托者，委托者的 getter 和 setter 会被委托给被委托者的 setValue 和 getValue，所以如果想要实现委托属性，就必须要实现 setValue 和 getValue 这两个方法。具体如下：

``` kotlin

class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        rturn "$thisRef,thank you for delegating '${property.name}' to me!"
    }
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println($value has been assigned to '${property.name}' in $thisRef.)
    }
}

```

当我们委托 Delegate 实例的 p 读取，将调用 Delegate 类的 getValue() 方法。如下：

``` kotlin

//Example 类的声明在上面
val e = Example()
println(e.p) //输出 Example@33a17727, thank you for delegating ‘p’ to me!

```

反之，当我们队 Example 中的 p 赋值的时候，会调用 Delegate 的 setValue() 方法，如下：

``` kotlin

e.p = "NEW" //输出 NEW has been assigned to ‘p’ in Example@33a17727.

```


## 懒加载

使用委托属性加 lazy 可以实现属性的懒加载，代码如下：

``` kotlin

val lazyValue: Strin by lazy {
    println("computed!")
    "Hello"
}

fun main(args: Array<String>) {
    println(lazyValue)
    println(lazyValue)
    //输出
    //computed
    //hello
    //hello
}

```

我们可以使用 lazy 这一特性来替换 Java 中的单例模式的实现，因为 Kotlin 中的 lazy 默认是加了锁，不用担心多线程的问题，如果可以确定不会出现多线程的问题，可以将 `LazyThreadSafetyMode.NONE` 作为参数传入 lazy，这样可以节省锁的性能开支。

## 观察属性

Kotlin 提供了 Delegates.observable() 来实现可以观察的属性，具体代码如下：

``` kotlin

import kotlin.propertys.Delegates

class User {
    var name: String by Delegates.observable("initital value") {
        prop, old, new ->
        println("$old -> $new")
    }
}

fun main(args: Array<String>) {
    val user = User()
    user.name = "first"
    user.name = "second"

    //输出
    //initial value -> first
    //first -> second
}

```

## 属性存储在 Map 中

可以使用 `by` 关键字把属性映射到 Map 中，具体代码如下：

``` kotlin

class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int     by map
}

//赋值
val user = User(mapOf(
    "name" to "John Doe"，
    "age" to 25
))


//var 属性
class User(val map: MutableMap<String,Any?>) {
    var name: String by map
    var age: Int by map
}

```

## 局部委托属性

在 Kotlin 1.1 之后，支持局部委托属性，代码如下：

``` kotlin

fun example(computeFoo: () -> Foo) {
    val memoizedFoo by lazy(computeFoo)

    if(someCondition && memoizedFoo.isValid()) {
        //只有当 someCondition 为 true 的时候，memoizedFoo 才会初始化
        memoizedFoo.doSomething()
    }
}

```