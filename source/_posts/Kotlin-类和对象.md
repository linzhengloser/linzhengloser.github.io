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


