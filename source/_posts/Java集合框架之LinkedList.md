---
title: Java集合框架之LinkedList
date: 2017-08-04 09:43:50
tags: 数据结构
---

# 开始

上一篇分析完了 ArrayList，这一篇来看看平时我们用的的不多，但是在框架里面很常见的一种数据结构:LinkedList。ArrayList 虽然使用起来非常方便，但是当数据过大时，删除和插入的效率过低，具体原因可以看上一篇博客。所以就有人想了另一种数据结构来优化插入和删除的效率，这种数据结构就叫做`链表`。

<!-- more -->

>链表是一种物理存储单元上非连续、非顺序的存储结构，数据元素的逻辑顺序是通过链表中的指针链接次序实现的。 ---> by baidu

链表分为单链表和双向链表，在单链表中每个元素只会记录下一个元素，而在双向链表中每个元素不仅会记录下一个元素还会记录上一个元素。

# LinkedList 一般使用方法

``` java

LinkedList<Integer> linkedList = new LinkedList<>();

//添加元素
linkedList.add(1);
linkedList.add(2);
linkedList.add(3);
linkedList.add(4);
linkedList.add(5);
linkedList.add(6);

//删除指定元素
linkedList.remove(2);

//修改指定元素的值
linkedList.set(4,7);


```

乍一看好像跟 ArrayList 一毛一样，是的，从操作上看是和 ArrayList 一样，原因是 LinkedList 和 ArrayList 都实现了 List 接口。但是 LinkedList 不仅实现了 List 接口 还实现 Deqe 接口，这个接口定一些队列的操作，具体我们后面分析。

在链表中每个元素被称之为节点，映射成 Java 中的类就是 LinedList 的一个叫做 Node 的内部类。我们来看看其内部实现:

``` java

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}

```

代码非常简单，其中 item 就是每个节点的 指向的数据，next 就是下一个 Node，prev 就是上一个节点。

# 构造函数

首先看看LinkedList 的构造函数。

``` java

public LinkedList() {

}

```

然而发现什么都没有，我们在看看 LinkedList 的字段。代码如下:

``` java

transient int size = 0;

transient Node<E> first;

transient Node<E> last;

```

size 表示当前 LinkedList 的节点个数，first 用来指向 链表的第一个节点，last 用来指向最后一个节点。

## add() 方法

下面在来看看 LinkedList 的 add() 方法。代码如下:

``` java

public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}

```

重点在 linkLast 方法中，其主要作用就是将元素添加在链表末尾，当 last 为 null 的时候，即当添加第一个元素的时候，这个时候 会将 first 指向添加的元素，且 last 也是指向添加的元素。注意上面的第三行代码，虽然将 last 指向了 newNode，但是 l 节点并没有受到它的影响。

所以当LinkedList 只有一个节点的时候，first 和 last 都是指向这个唯一的节点。

上面是当添加第一个节点的情况，我们在看看如果添加第二节点会怎么样，当添加第二个节点的时候，l 指向的是 第一个节点，第二个节点的上一个节点是 l，last 指向 第二个节点，这个时候 l 是不等于null 的。所以最后就将 l 即第一个节点的 next 指向 第二个节点。

上面描述的有点乱，简单一句话就是: 如果将一个节点添加到链表的末尾且这个节点已近有节点了，步骤为:添加节点的 prev 指向 last ，last 指向添加的节点，上一个节点的 next 指向新添加的节点。

## add(index,node) 方法

和 ArrayList 一样，LinkedList 也有将元素添加到指定索引的方法。代码如下:

``` java

public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

```

checkPositionIndex() 方法是对所以边界的判断。然后下面会对 index 的值判断，如果 index == size 即表示要将节点添加到链表末尾，所以直接调用上面的 linkLast() 方法，说我们主要看 linkBefore() 这个方法，代码如下:

``` java

void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}

Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

```

在分析 linkBefore() 方法之前，先看看 node() 方法，这个方法的作用是根据索引返回指定索引的节点对象，我们来看看其实现方法，首先会判断 index 是否小于 size >> 1，为什么要判断这个呢？在说原因之前我们先做个假设，假设我们链表有 20 个节点，我们要获取索引为 19 的节点，我们如果只从一端开始造，比如从 first 开始找，就要执行 19次 才能找到，但是如果我们从 last 开始找，那么只用执行 2 次就能找到，这就体现出双向链表的好处了，如果要找的节点的索引小于 size /2 则从头开始找，否者从后开始找。

回到 linkBefore() 方法，succ 就是通过 node() 方法找到的节点，首先创建变量 pred 指向 succ 的上个节点，然后创建新节点，新节点的 next 是 pred.prev，并且把 succ 的 prev 指向新节点。最后判断 pred 是否等于null，如果等于 null 说明插入的索引是 0 就将 first 指向 新节点，否者就将 pred 的 next 点指向 新节点。

## remove() 方法

分析完 add() 方法，在来分析下 remove() 方法。代码如下:

``` java

public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}

```

第一个 remove() 方法是根据索引删除节点，第二个 remove() 方法是通过对象删除。

因为两个方法都调用了 unlink()，所以首先分析下 unlink() 方法。这个方法的作用就是将指定节点从链表中移除，移除操作主要是对 prev 和 next 这两个属性做操作。具体逻辑如下:

1. 如果 prev == null 则表示移除的节点是头结点，所以要把 first 指向 next。否则就将 prev.next 指向 next，并把被移除节点的 prev 置为 null。
2. 如果 next == null 这表示移除的节点是尾节点(自己 YY 的名字)，所以要把 last 指向 prev。否则就将 next.prev 指向 prev，并把被移除节点的 next 置为 null。
3. 最后将被移除节点的 item 置为 null。

unlink() 逻辑差不多就上面这些，在回到 remove() 方法，发现和 ArrayList() 中的 remove() 方法差不多，如果对对象进行删除操作，需要重写 equals() 方法。


## get() 和 set() 方法

在来看看 get() 和 set() 方法。

``` java

public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}

```

get() 方法没什么好说的，主要使用 node() 方法获取节点对象并返回 item。set() 方法也差不多，首先通过 node() 方法获取节点对象，然后获取 item 用于放回，然后将 item 设置为新值。

# Deque 接口
LinkedList 不仅实现了 List 接口，还实现了 Deque 接口，List 接口中定义的是 add() remove() get() 等一系列集合操作的方法，而 Deque() 接口中定义的是一些关于 队列操作的方法()，如 pop() poll() 等。下面来看看 LinkedList 是怎么实现这些接口的。

Deque 接口定义如下:

![Deque接口定义](Java集合框架之LinkedList/Deque接口定义.png "Deque接口定义")

首先介绍下常用的方法。

## poll() 方法
放回链表的头节点并把返回的节点对象从链表中移除，代码如下:

``` java

public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

```

可以看到，LinkedList 对象移除头节点做了单独的处理，主要来看看 unlinkFirst() 方法。传入的节点 f 其实就是 first 节点，首先将 first 的 item 和 next 都置为 null，然后 first 指向 first.next，最后判断 如果 next == null 即表示现在链表中没有节点了，所以将 last 置为 null，否则就将 next.prev 置为 null。

## peek() 方法
返回 first 节点对象。 代码如下:

``` java

public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

```

代码很简单，就是返回 first 的 item 。 

# xxxFirst() 方法 和 xxxLast() 方法

Deque 翻译过来的意思是 双端队列，关于队列的定义如下:

>队列是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。                    --> by baidu

而说所谓的 双端队列，就是既可以从队列的前端插入数据，也可以从队列的尾端插入数据，来看看 LinkedList 是怎么实现的。代码如下:

``` java

public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

```

addFirst() 方法，将节点对象添加到链表第一的位置，linkFirst() 方法和 上面的 unLinkFirst() 方法大同小异，这里不做详述。

既然有 addFirst() 方法，那么肯定有与之对应的 addLast() 方法。代码如下:

``` java

public void addLast(E e) {
    linkLast(e);
}

```

addLast() 和 add() 方法功能一致，都是将元素添加到链表的末尾。

最后在看看 removeFirst() 和 removeLast()。代码如下

``` java

public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}

public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

```

removeFirst() 和 removeLast() 方法，分别表示移除链表的第一个节点和最后一个节点。

# 总结

大致总结下链表的操作步骤:

插入操作:假设我们要将元素插入到 nextNode 的前面。

1. 首先创建节点 prevNode 并指向 nextNode 的 prev。
2. 然后创建节点 newNode 并将它的 prev 指向 prevNode，将它的 next 指向 nextNode。
3. 再然后将 nextNode 的 prev 指向 newNode。
4. 最后将 prevNode 的 next 指向 newMode。

移除操作:假设我们要移除索引为2的节点。
1. 首先获取索引为2的节点，removeNode 节点。
2. 然后分别获取 removeNode 的 next 和 prev 节点，nextNode 和 prevNode
3. 将 prevNode.next 指向 nextNode
4. 将 nextNode.prev 指向 prevNode
5. 如果 prevNode == null 或者 nextNode == null 就将 first 指向 nextNode 或者将 last 指向 prevNode


上一篇中我们分析了 ArrayList 其内部的实现，发现他其实就是一个可变的数组，而 LinkedList 跟数组没有什么关系，其内部是通过对象与对象之间的引用来设置关系的，在执行删除操作和插入操作的时候，ArrayList 和效率远远不如 LinkedList 。所以如果需要频繁插入和删除的数据的情景中，推荐使用链表做为容器。像在 Android 系统中 MessageQueue 就是使用一个单链表来存储消息列表的。

至于为什么 LinkedList 要使用双向链表的原因 可能是因为为了提高查询的效率吧。