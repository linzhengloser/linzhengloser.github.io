---
title: Java集合框架之HashMap
date: 2017-09-13 10:26:35
tags: 数据结构
---
# 简介
这篇博客要分析的是 HashMap 这个数据结构。这个数据结构相比于 ArrayList 和 LinkedList 有很大的区别。首先在类继承结构上来看，HashMap 实现的是 Map 接口，而 LinkedList 和 ArrayList 实现的是 List 接口。(从名字上就能看出来)。

<!-- more -->

HashMap 采用键值对的方式存储数据，一个 Key 对应一个 Value，Key 和 Value 都允许为 null，且 Key 不能重复。

# 一般用法

``` java

Map<String,String> map = new HashMap<>();
map.put("key1","value1");
map.put("key2","value2");
map.put("key3","value3");
map.put("key4","value4");

//遍历 HashMap
for (Map.Entry<String, String> entry : map.entrySet()) {
    System.out.println("key = " + entry.getKey() + " value = " + entry.getValue());
}

//输出 newValue4，HashMap 的特性 newValue 把原本的 value4 顶掉
map.put("key4","newValue4");
System.out.println(map.get("key4"));

//移除 Key3 并把其对应的 value3 也一起移除
map.remove("key3");

```

HashMap 中用的比较多的方法差不多就上面这几个，要注意的是 HashMap 的遍历有点特变，可以通过 HashMap 的 keySet() 和 values() 方法来获取 HaspMap 的 key 和 value 的集合，从而实现遍历，还有一种遍历的方法就是上面的方法，通过 entrySet() 方法获取一个 Set<Map.Entry<K, V>>，然后我们只需要使用 for 循环遍历这个 set 即可，至于这个 Map.Entry 是什么，我们后面分析。

# 源码分析
首先看看下面几个问题:

1. HashMap 是怎么实现让一个 Key 只对应一个 Value 的。
2. HashMap 内部是采用数组还是对象实现的？
3. HashMap 是有序的吗？

接下来我们将会通过分析源码来知道上面问题的答案。

## 字段

HashMap 中有很多字段，挑几个比较重要的来分析下

``` java

transient Node<K,V>[] table;

transient Set<Map.Entry<K,V>> entrySet;

//数据的大小
transient int size;

//修改的次数
transient int modCount;

int threshold;

//加载因数
final float loadFactor;

```

size 和 modCount 这两个不用做多解释了，Java 的集合框架中基本上每个类都有这两个字段。threshold 这个字段翻译过来意识是阈值，暂时不知道是干嘛用的，后面再看。loadFactor 翻译过来意识是因素，也不知道是干嘛用的，后面再分析。

table 是一个 Node 类型的数组，entrySet 是一个 Map.Entry 类型的 set，这里需要注意下，Node 其实是 Map.Entry 的实现类，Map.Entry 只是一个接口。

## 构造函数

HashMap 的构造函数很简单，大概如下

``` java

static final float DEFAULT_LOAD_FACTOR = 0.75f;

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

```

可以看到在默认的无参构造函数里面，HashMap 只初始化了 loadFactor 这个字段。

我们主要看看两个参数的构造函数，initialCapacity 表示 HashMap 的初始容量。然后会根据这个初始容量算出 threshold 这个字段的值，具体的计算过程在 tableSizeFor() 方法中，代码如下：

``` java

static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

```

咋一看好像看不太懂是干嘛用的，但是我们可以把代码 copy 出来，然后随便传几个值看看结果。我们分别传入 10 ，20 ，30 ，40，50。输出结果为 16 ，32 ，32 ，64 ，64。显而易见，这个方法的作用就是最近接近 cap 的 2 的 N 次方。然后在将这个值设置给 threshold，举个例子，当我们设置 HashMap 的初始容量为 100 的时候，其阈值就是 124。

## put() 方法

上面分析了构造函数，下面来分析分析 put() 方法，代码如下：

``` java

public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //省略部分代码
    return null;
}

```

在 put() 方法中调用了 putVal() 方法，putVal() 首先会判断 table 如果是空的，就会调用 resize() 方法，然后 n = resize.length。我们先看看 resize() 方法。代码如下：

``` java

static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    //记录容量和阈值
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //容量边界判断
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //阈值大于0
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {
        //阈值为0 设置默认容量和默认阈值
        //默认阈值等于 默认容量 * DEFAULT_LOAD_FACTOR
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //重新设新置阈值
    threshold = newThr;
    //创建长度为 新容量的 Node 数组
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    //重新设置 table 为 newTab
    table = newTab;
    //如果 HashMap 中有值 那么就要把原本数组中的数据 copy 到新的数组中去
    //默认第一次执行此方法，此处为false
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //判断是否是链表末尾
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```

resize() 主要做了如下几件事情：

1. 如果是第一次调用，那么会对 table 数组进行初始化，默认长度为 1 << 4 即 16。
2. table 数组默认扩容为原先数组的两倍，即 16 扩容为 32。
3. 如果默认指定过 threshold (阈值)，那么第一次调用 resize() 方法，table.length = threshold。
4. 默认阈值 = 默认容量 * DEFAULT_LOAD_FACTOR。


在回到 putVal() 方法，接着上面的分析，代码如下:

``` java

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //table 中对应索引处没有数据，那么就直接 newNode 并指向对应索引处。
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        //对应索引处已经有元素。
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //循环链表中的节点
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //如果 Hash 值相等且 equals 和 == 要有一个为 true
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

```



整体逻辑分为两种情况，这一种是对应索引处没有数据，这就很简单了，直接 new Node 对象并把数组对应索引处指向 newNode。另一种情况比较复杂，上面说到了，对应索引处，这个索引是通过 table.length-1 & hash 算出来的。

为什么要这么算，因为 HashMap 是 Key -> Value ,而我们上面发现，无论我们输入什么类型的 Key 值，HashMap 都会转换成一个 int 类型的 Hash 值，这个 hash 值是肯定不能用来直接当 table 的索引。那 HashMap 是怎么让 索引和 Hash 值之间建立关系的呢？就是使用 table.length-1 & hash，算出这个索引值，其实这个操作就是取模。

这样虽然解决了 索引和 Hash 值的对应关系，但是我们知道取模是会有重复的。这时代码就会走到上面 else 的代码中。在分析下面代码之前我们需要先了解 Node 这个类的结构。代码如下：

``` java

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    //省略部分代码...

}

```

Node 类是 HashMap 的内部类，实现了 Entry 接口，注意到其实这个类是一个单链表中的节点类。其内部 next 字段指向下一个节点。看到这里我们大概就知道了 HashMap 是怎么存储数据的了。图如下：

![HashMap内存结构图](Java集合框架之HashMap/HashMap内存结构图.png "HashMap内存结构图")

图片来自: https://tech.meituan.com/java-hashmap.html

其实我们可以把 HashMap 理解成一个类型为链表的数组，为什么要这么设计呢？我们上面分析说到 HashMap 是使用取模操作来建立 Hash 值和数组索引之间的关系的，那们就存在取模得出来的值是一样的情况，这个时候就需要用一个链表来存这些 Hash 值一样的 Key。

在回到 putVal() 方法，当指定索引处已经有值的情况下，HashMap 会遍历这个链表，判断没有 Node 的 Hash 值是否等于插入 Key 的 Hash 值。如果相等就直接把 节点的 Value 指向插入的 Value，如果没找到，就把 Key 和 Value 封装成一个 Node 对象并放在链表的末尾处。

在 putVal() 方法最后会判断修改后的 size 如果大于阈值就会 table 数组进行扩容，也就是调用上面的 resize 方法。

注意，在 jdk1.8 中，当链表的长度大于 8 时，HashMap 会把链表转换成红黑树。这里我们只分析 HashMap 的实现，至于这个红黑树我们后面有机会在分析。

# get() 方法

分析完 put() 方法，在来看看 get() 方法，相比于 put() 方法，get() 方法明显简单许多。代码如下：

``` java

public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            //红黑树处理
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                //循环遍历链表
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

```

首先通过 table.length & hash 的出索引值，如果对于索引处有节点，那么就遍历该节点，比较每个节点的 hash 如果是一样就返回对应节点对象，如果没有找到或是说对应索引处是 null ，就返回 null。


# remove() 方法

最后再来看看 HashMap 的 remove() 方法，代码如下：

``` java

public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}


final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                //要移除的节点在链表的第一个
                tab[index] = node.next;
            else
                //将要移除的节点的上一个节点指向被移除节点的下一个节点
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}

```

和 get() put() 方法套路一样，首先算出 hash 值对应的索引，然后判断对应索引处的节点对象的 hash 值是否和要移除的 hash 值一样，如果一样就就将其移除，移除操作就是一个链表的移除操作。


# 其他方法

上面我们用到 entrySet() 这个方法，可以用这个方法的返回值来遍历 HashMap，我们来看看这个方法内部是怎么实现的。代码如下：

``` java

public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}


final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
    public final Spliterator<Map.Entry<K,V>> spliterator() {
        return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    //此方法也可以用来遍历。不过好像需要 jdk1.8 才能用。
    public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}

```

可以看到，entrySet() 方法中，new 了一个 EntrySet 类的对象并返回。在 EntrySet 类中，interator() 方法就是用来遍历用的。因为 EntrySet 类中并没有 get() 方法，所以如果我们想要遍历这个 set 只能通过 forEach 循环或者是用 interator() 方法返回的 EntryIterator 对象手动调用方法来实现迭代操作。

HashMap 还提供了 keySet() 和 values() 这两个方法，用来返回 HashMap 的 Key 集合 和 Value 集合，实现方式和 EntrySet 一模一样，这一就不做过多讨论。

# 一些关于 HashMap 的小技巧

HashMap 其实不仅可以用来存储数据，我们可以利用他的键值对特性来实现一些很有 "意思" 的代码。比如下面这段代码：

``` java

int number = 0;

switch(what){
    case 1:
        number = xxx;
        break;
    case 2:
        number = xxxx;
        break:
    //省略很多 csae
}

or

if(what==1){
    number = xxx;
}else ifwhat == 2{
    number = xxxxx;
}
//省略很多 else if

```

在很多年前，有一个大佬告诉过我，上面这种代码属于非常 low 的代码，只有新手才会这么写。然后还说，可以使用 HashMap 来处理类似的代码。代码如下：

``` java

HashMap map = new HashMap();
map.put(1,xxx);
map.put(2,xxxx);

number = map.get(what);

```

代码这么一改，是不是觉得瞬间高大上了？这么写代码不仅可读性高，扩展起来也比较方便。纯属个人看法。

Android 其实为我们提供了很多优化后的数据结构，其中就包括 HashMap 的优化，比如如果我创建的 HashMap 的 key 类型是 int 类型，那么我们应该考虑使用 SparseArray 这个类。此类是能用 int 类型的 key 来获取 value。其次 Android 还为我们提供了 ArrayMap 类，该类的注释上是这么写的：ArrayMap 是一个通用的键 - 值映射数据结构，比传统的 HashMap 更高的内存效率。我们都知道在手机上，内存是很吃紧的，所以合理的使用内存是我们有限考虑的。

# 总结

分析了这么多，基本上 HashMap 大致的工作流程基本上已经搞清楚了，回到文章开头的那几个问题，答案就显而易见了。

1. HashMap 通过数组加链表的方式实现键值对的数据结构，其键的 Hash 值可以通过计算，求出在数组中对应的索引值。
2. HashMap 内部采用数组加对象的方式实现。用数组实现整体结构，用对象实现链表结构。
3. HashMap 中数据存储在数组中的索引是无序的，其索引和其 Hash 值有关联。

在 HashMap 中的构造函数中，有一个参数叫做加载因子，通过上面我们分析  HashMap 的源码得出，这个参数其实是可以通过传入不同的值来应对不如的应用场景
，如果在内存比较吃紧的情况下，将次参数设置为比较大的值，可以压缩空间，反之，如果内存不是很吃紧且需要加快查询速度，可将次参数设置为较小的值。默认情况下我们不需要修改这个值。