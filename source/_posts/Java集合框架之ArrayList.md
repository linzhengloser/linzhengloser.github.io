---
title: Java集合框架之ArrayList
date: 2017-08-02 15:05:21
tags: 数据结构
---

# 开始
最近准备好好的学习一下数据结构，因为我最熟悉的语言是 Java，所以准备从 Java 语言中实现好的数据结构来入手。

Java 本身帮我们实现了很多现成的数据接口，如：ArrayList，LinkedList，Queue 等等....，这些现成的实现都是经过岁月的打磨，可以说是最佳的实现方式了，所以其内部实现原理还是有很多值得我们学习的地方的。

<!-- more -->

# Java 的集合框架
Java 中的集合框架分为两大类，下面列出一些常用集合类的继承结构。

* Collection
    * ArrayList
    * LinkedList
* Map
    * HashMap

本篇博客主要分析的 ArrayList。

# Array 的一般用法
在分析 ArrayList 一般用法之前，我们先来看看 Java 中数组的用法，因为 ArrayList 的实现离不开数组。

``` java

// 声明一个数组，有一下几种方法
int [] array = {1,2,3,4,5};
// or
array = new int[]{1,2,3,4,5};
// or
array = new int[5];
array[0] = 1;
....
array[4] = 5;

//获取数组中索引为0的元素
Systmen.out.println(array[0]);

```

因为 Java 中的数组是不可变的，所以删除和添加操作无法实现，但是我们可以使用一些别的方法来实现这两种操作，比如 ArrayList 的 remove() 方法和 add() 方法。


# ArrayList

ArrayList 翻译过来的意识为"数组集合"。

集合的数学含义: 集合是指具有某种特定性质的具体的或抽象的对象汇总成的集体，这些对象称为该集合的元素。

通过上面的描述，"数组集合" 即为多个数组组成的一个东西，那么这个东西就叫做 ArrayList。在这里不得不说一下，ArrayList 这个类名取的非常贴切，因为 ArrayList 本身内部的实现就是 将一个一个的数组拼在一起，最终组合成一个大数组。非常符合 ArrayList 这个名字。

## 一般用法

``` java

//创建 ArrayList
ArrayList<String> arrayList = new ArrayList<>();

//添加元素
arrayList.add("1");
arrayList.add("2");

//删除元素
arrayList.remove(0);
//or 
arrayList.remove("1");

//获取指定索引下的元素
arrayList.get(0);

//判断一个元素是否在集合中
arrayList.contains("1"); // return true is exist.

//获取元素的个数
arrayList.size();

//获取一个元素在集合中的索引
arrayList.indexOf("1");

```

ArrayList常用的方法就上面集合，这里有几个特别需要注意的地方。

1. 如果使用上面的 contains(对象)，indexOf(对象)，remove(对象)或涉及到对象的方法，都需要重写对应对象的 hashCode() 和 equals() 方法。具体原因后面我们通过源码来解释。
2. 上面的 size() 方法获取的是元素的个数而并非是 ArrayList 的大小(内部数组的长度)，具体原因我们后面通过源码来解释。


## 构造函数
直接看源码

``` java

private static final Object[] EMPTY_ELEMENTDATA = {};

// initialCapacity 集合初始大小
public ArrayList(int initialCapacity) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
}

// 此构造函数用的最多，elementData 默认是一个 空的数组
public ArrayList() {
    super();
    this.elementData = EMPTY_ELEMENTDATA;
}

//此方法用的并不多，主要是将别的集合对象转换成 ArrayList 前提条件是 实现了 Collection 接口
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    size = elementData.length;
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, size, Object[].class);
}

```

上面代码中，注释已经写的很清楚了。平时用的最多的就是无参的构造函数。

## 属性
看下 ArrayList 中有那些属性
``` java

// 核心属性，用来存储数据
transient Object[] elementData;

//存储元素的个数，size 不一定等于 elementData.length
private int size;

//默认容量
private static final int DEFAULT_CAPACITY = 10;

```

## add() 方法
添加元素方法

``` java

public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

```

add() 方法逻辑如下:

1. 调用 add() 方法的时候，会先调用 ensureCapacityInternal(page+1) 方法。
2. 在 ensureCapacityInternal(page+1) 方法中会先判断 elementData 这个数字是否指向 EMPTY_ELEMENTDATA，当使用无参构造函数的创建 ArrayList 且第一次调用 add() 方法的时候，minCapacity = 10。
3. 调用 ensureExplicitCapacity(minCapacity)，在 ensureExplicitCapacity() 方法中 会对 modCount ++，然后判断当前数组即 elementData 是否还有空余的位置，如果没有就调用 grow(minCapacity) 方法对数进行扩容。
4. 最后将 elementData[size++] 指向要添加的元素。

grow() 方法逻辑如下:

首先我们要知道，只有当 elementData 数组中的位置被占满了，即 size + 1 - elementData.length() > 0。这个时候才会调用 grow() 方法对 elementData数组扩容，扩容后的数组的长度为原数组长度+原数组长度 >> 1，即每次扩容的个数是当前数组的一半。
我们假设现在数组中元素的个数为 10 ，当我们调用 add() 方法添加第十一个元素的时候，这个时候 grow() 方法会调用，将原本 长度为 10 的 elementData 扩容成长度为 10 + 10 >> 1 即长度为 15 的数组。然后在将 elementData[size+1] 指向新添加的元素，这样元素就被添加到了 ArrayList 中。

 grow() 方法中，使用的是 Arrays.copyOf(elementData, 扩容后的长度) 方法来对数组扩容，注意这里是 创建了一个新的数组，而并非在原本数组的基础上，理论上不可能对原数组扩容，因为 Java 中数组的长度是不可变的，改变长度的唯一方法就是 new 一个新的数组。

### add(index,data) 方法

上面的 add() 方法默认是将元素添加到数组的 size + 1 的位置，当我们需要将元素插入到指定索引的位置的时候我们可以使用 add() 方法的一个`重载` ，代码如下 :

``` java

public void add(int index, E element) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}

```

在上面代码中，首先对 index 进行边界判断，如果 index 不合法就抛出 Java 里面经常遇到的异常 `IndexOutOfBoundsException`。
然后调用 ensureCapacityInternal() 方法，这个方法我们上面分析过，作用就是对 elementData 进行扩容，当然前提是需要扩容。
在然后调用 System.arraycopy() 将数组的元素往后移一位。

比如: 现在 elementData 中的元素为 {1,2,3,4,5,0,0,0,0,0}，这个时候 ArrayList 的 size 是 5，这个时候我们调用 add(2,6) 方法，执行 System.arraycopy(elementData, 2, elementData, 3,3) 方法之后，elementData 为 {1,2,3,3,4,5,0,0,0,0}，这就是 System.arraycopy() 的作用。

将元素往后移完之后，我们发现移完后的数组中有两个3，我们本意是想把 6 插入到索引为 2 的位置，所以后面将 elementData[index] 指向要插入的数据，这个时候 elementData 为 {1,2,6,3,4,5,0,0,0,0},这样就完成了插入的操作。


## remove() 方法

remove() 方法的实现个 add(index,data) 方法类似，代码如下:

``` java

public E remove(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    modCount++;
    E oldValue = (E) elementData[index];

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}

```

首先还是对数组边界的判断，然后获取到需要移除的对象，并算出需要移动的元素个数。所谓的需要移动的元素个数指的是，假设现在 elementData = {1,2,3,4,5,0,0,0,0,0} size = 5，我们需要移除索引为 2 的元素，我们就需要把 4 和 5 像前移一位，即需要移动的元素个数为 2。

最后调用 System.arraycopy() 方法，又是这个方法，此方法在移动数组元素的时候异常的好用。还是上面的例子，elementData = {1,2,3,4,5,0,0,0,0,0} 通过调用 System.arraycopy(elementData,3,elementData,2,2) 方法之后，elementData = {1,2,4,5,5,0,0,0,0,0} 然后在将 elementData[--size] 即最后一个元素指向 null ，方遍 GC 回收内存。

## get() 和 set() 方法

get() 方法和 set() 方法相对来说比较简单。代码如下:

``` java

public E get(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    return (E) elementData[index];
}

public E set(int index, E element) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    E oldValue = (E) elementData[index];
    elementData[index] = element;
    return oldValue;
}

```

get() 方法就是返回指定索引处的数据，set() 方法就是将指定索引位置的对象进行替换。这个要注意，这两个方法的传入的索引都不能大于等于 size (因为索引等于 size-1) 否则会触发 `IndexOutOfBoundsException`。

## 重写 equals() 方法

在 ArrayList 中有很多诸如此类的方法，比如: remove(data)，indexOf(data)，contains(data) 等等.... 这些方法都是通过传入一个对象来执行操作的，那么就有一个问题，他是怎么判断对象是相等的，或者说当我调用 remove(data) 的时候，ArrayList 是怎么知道我么您是想删除哪个对象的？

源码如下:

``` java

public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}

```

代码很简单，循环 elementData 中的所有元素。有趣的地方是 remove 方法居然可以传 null，如果传的是null 可以把数组中的 null 移除调用。下面的 fastRemove() 方法和 remove(index) 方法是一样的逻辑。可以看到在 remove(data) 方法内部当传入的对象不为 null 的时候，回调用传入对象的 equals() 方法，这个方法是 Object 类中的，默认实现如下:

``` java

public boolean equals(Object obj) {
    return (this == obj);
}

```

在 Java 中用 == 比较的是两个对象的引用，即指向的对象是否是同一个。上面是 equals 的默认实现，我们可以重写这个方法来实现一些特殊的效果，如下:

``` java

class Item {

    private int id;

    private String name;

    private int age;

    //省略 get set 方法 和 构造函数

}

ArrayList<Item> arrayList = new ArrayList<>();
arrayList.add(new Item(1,"张三",18));
arrayList.add(new Item(2,"赵四",19));
arrayList.add(new Item(3,"王五",20));
arrayList.add(new Item(4,"赵六",21));

```

比如上述代码，我们想要删除 id 为 2 的 Item ，假设我们并不知道 id 为 2 的 Item 在 ArrayList 的索引是多少，那么我们只能调用 arrayList.remove(new Item(2,"赵四",19)); 如果我们不重写 equals() 方法，是没法正常删除的，原因在上面说明了，所以我们要重新 Item 的 equals() 方法。代码如下:

``` java

class Item {

    private int id;

    private String name;

    private int age;

    //省略 get set 方法 和 构造函数

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        Item item = (Item) o;

        return id == item.id;

    }

}

```

上面 equals() 方法是系统帮我们生成的，我们这里使用 id 做为对比两个对象的是属性。同理 indexOf(data)，和 contains(data) 套路一样，源码如下:

``` java

public boolean contains(Object o) {
    return indexOf(o) >= 0;
}

public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

```

## Collections 类

上面把 ArrayList 大致实现 原理分析完了，除了这些以外，ArrayList 还有很多别的功能，比如 Java 提供了一套排序的逻辑，内部已经帮我们优化了算法，我们只需要传入对象即可。具体代码如下:

``` java

ArrayList<Integer> arrayList = new ArrayList<>();
arrayList.add(1);
arrayList.add(99);
arrayList.add(22);
arrayList.add(88);
arrayList.add(33);
arrayList.add(77);
arrayList.add(44);
Collections.sort(arrayList, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1.compareTo(o2);
    }
});

```

Collections 类中有很多关于 ArrayList 类的工具方法，其中 sort() 方法帮我们实现排序的功能，我们只需要实现 Compartor 接口即可，具体实现网上有很多，套路都是一样的，这里就不做说明了。

# 总结
总的来说，ArrayList 就是一个可变长度的数字，使用 System.arraycopy() 来实现对数字的扩容操作，默认每次扩容原先数字 size 的一半，因为是用的是数组实现，所以查询效率快，但是插入效率低，所以如果需要频繁变动的数据源推荐使用别的实现。

Collections 类中的 sort() 方法的排序性能毋庸置疑，所以如果需要对 ArrayList 排序，最好还是使用 Collections 的 sort() 方法。