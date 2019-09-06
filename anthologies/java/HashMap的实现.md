### 0x00 背景知识

首先`HashMap`是基于hash表的，hash表的相关知识及其存储原理，可以参考我之前写的一篇文章：[查找算法总结](/anthologies/fundamental/algorithm/查找?id=_0x02-哈希表)，要着重看此文章关于冲突处理所讲述的拉链法。

![](https://bucket.shaoqunliu.cn/image/0209.png)

我们仅在这稍微明确一点概念：首先对于拉链式的hash表，其是由数组和链表共同组成的，当然Java中还引入了红黑树，当链表长度大于8时，将链表转为红黑树以提高查找效率。我们先来看上面的这个图，保存这个hash表的数组称之为桶（bucket），而每个桶中保存了一系列的数据入口（entries）。

同理，我们在这明确一个概念“**等同**”：我们假定如果两个对象其`hashCode`相同，且任一一个调用`equals`方法与另一个返回`true`，即认为两个对象等同。

同时，在`HashMap`中有如下几个量需要注意，代码如下：

```java
/**
 * The number of key-value mappings contained in this map.
 */
// MAP中所包含的键值对的数量
transient int size;

/**
 * The next size value at which to resize (capacity * load factor).
 *
 * @serial
 */
// (The javadoc description is true upon serialization.
// Additionally, if the table array has not been allocated, this
// field holds the initial array capacity, or zero signifying
// DEFAULT_INITIAL_CAPACITY.)
// 阈值，超过此阈值就会诱发桶的扩容操作，值为：桶的容量乘以装填因子
int threshold;

/**
 * The load factor for the hash table.
 *
 * @serial
 */
// 装填因子
final float loadFactor;
```

### 0x01 put方法

先来看一张流程图：

![](https://bucket.shaoqunliu.cn/image/0294.png)

> 图片非原创，来自文章：[HashMap原理深入理解](https://blog.csdn.net/visant/article/details/80045154)

正好我们就着源代码来说说这张图，首先我们来看`put`方法的源代码：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

其直接将工作转交至了`putVal`方法中，我们直接来看`putVal`的代码：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    // 如果桶为空或者长度为0的话，那么调用resize方法对其进行扩容
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 通过(n - 1) & hash计算当前元素对应在table数组（桶）中的下标
    // 其中n为桶的长度，hash为依据key所计算的hash码值
    // 如果table数组中此项为空，意味着没有冲突发生
    // 那就直接在此项处新建一个Node直接将其插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果桶中已经有了至少一个元素
    else {
        Node<K,V> e; K k;
        // 首先判断链表的第一个结点与当前所插入的结点的`key`是否等同
        // 其中key的hash相同并且equals方法返回true被认为等同
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 如果等同的话，则将标记e指向p
            e = p;
        // 如果桶的entry类型为TreeNode的话
        else if (p instanceof TreeNode)
            // 那就调用红黑树的插入方法，将此键值对插入
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); 
        // 否则调用链表的插入方法，目标在链表尾部插入此结点
        else {
            for (int binCount = 0; ; ++binCount) {
                // 如果抵达链表尾部
                if ((e = p.next) == null) {
                    // 插入新结点
                    p.next = newNode(hash, key, value, null);
                    // 如果插入后链表的长度大于TREEIFY_THRESHOLD
                    // 即调用treeifyBin方法将链表变为红黑树
                    // 注：TREEIFY_THRESHOLD默认为8
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果在链表中找到了和插入键值对key等同的结点则退出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果标记e不为null的话
        // 即在当前hash表中找到了与所要插入的键值对的key相同的结点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent即为是否仅在oldValue为null的情况下更新键值对
            // 这个参数在put方法中被默认给定为false
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount; 
    // 键值对数量超过阈值时，调用resize方法进行扩容对桶进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

在上述两个代码中，我们可以看到两个方法`afterNodeAccess`和`afterNodeInsertion`这两个方法在`HashMap`里面的实现是空的，在此处什么都不做，其注释中写道：Callbacks to allow `LinkedHashMap ` post-actions，即适用于`LinkedHashMap `实现一些后置功能的回调函数。

### 0x02 get方法

先来看一下`get`方法：

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

其主要工作在`getNode`方法中，用于通过键值`hash`和键值对象获取`Node`，如果其得到的值为`null`那就返回`null`，否则返回`Node`对象的`value`，我们着重来看`getNode`方法：

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 首先检查桶是否为空，如果不为空的话，检查键值所在桶的位置是否为空
    // 如果二者均为空的话，返回null查找失败
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 检查第一个是否等同
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 如果桶的entry类型为TreeNode的话
            if (first instanceof TreeNode)
                // 在entry所指定的红黑树中查找相关键值
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 在链表中查找相关键值
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    } 
    // 查找失败
    return null;
}
```

### 0x03 remove方法

`remove`方法用于从`HashMap`中移除一个元素，代码如下：

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
}
```

其主要工作在`removeNode`中，我们来看其源代码：

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 要删除就要判断被删除的键值对在hash表中是否存在
    if ((tab = table) != null && (n = tab.length) > 0 &&
        // 判断被删除元素hash码值是否在表中存在
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;.
        // p.hash在Node<K,V>构造时就已经赋值，详见其构造函数
        // 判断被删除元素是否在拉链中存在
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // 在树中寻找被删除元素
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 在拉链中顺序寻找被删除元素
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
            // 被删除元素存在的话
            if (node instanceof TreeNode)
                // 在树节点中删除
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // p代表了拉链的第一个位置
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    // 被删除的键值对不存在，返回null
    return null;
}
```

### 0x04 resize方法

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 如果扩容前容量已经超出了所容许的最大容量
            // 那么提高门限值至int类型的最大值
            // 以防增加元素时再次触发resize操作
            threshold = Integer.MAX_VALUE;
            return oldTab; // 不再扩容
        }
        // 新容量翻番
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 如果翻番后的容量仍小于所容许的最大容量的话
            // 门限值也翻番
            newThr = oldThr << 1; // double threshold
    }
    // 原容量==0 && 原门限值>0
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 原容量和原门限值均为0，则全部使用默认值初始化
    else {  // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
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













下面再来看一下其采用的Hash函数：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

显然，通过其hash函数所得的hash码值与这个对象的`hashCode`有着直接的关联，这就产生了一个问题，首先`hashCode`的返回值是依据当前对象在JVM中的内存地址算出来的，当我们用一个自定义类的对象来做`key`的话，同样内容的两个对象，`get`操作就会出现问题，看代码：

```java
/* == Main.java == */
public class Main {
    public static void main(String[] args) {
        Person boy = new Person("Qun", 22);
        Person girl = new Person("Hui", 23);
        HashMap<Person, Person> couples = new HashMap<Person, Person>();
        couples.put(boy, girl);
        Person someone = new Person("Qun", 22);
        System.out.println(couples.get(someone));
    }
}

/* == Person.java == */
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // 省略getter, setter
}
```

`get`函数会返回`null`，`someone`和`boy`虽说看上去内容是一样的，但是实际上是两个对象，二者的`hashCode`会返回不一样的值，所得到的用于查找的hash码值也会不一样，所以`get`失败。好，我们来重写一下`Person`类的`hashCode`让他由属性`name`和`age`来决定，代码如下：

```java
@Override
public int hashCode() {
    return name.hashCode() + age;
}
```

> TL;DR
>
> `String`类型的对象所产生的`hashCode`值是由其所保存的字符串决定的，字符串相同，`hashCode`值也相同，字符串不同，`hashCode`值也不同。我们可以在其文档中，找到相关`hashCode`值产生算法的描述：
>
> ```
> s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
> ```

这样就可以保证两个内容相同的`Person`对象产生的`hashCode`也相同了，让我们重新运行代码，依旧输出`null`，经检查发现`get`函数不仅会判断`key`的hash值相同，同时也会使用`equals`函数来判断两个对象是否等同，重写`equals`函数如下：

```java
@Override
public boolean equals(Object obj) {
    return obj instanceof Person && 
             this.name.equals(((Person) obj).name) &&
               this.age == ((Person) obj).age;
}
```

这样程序就返回了正确的结果。由此可以得到一个结论，查找`HashMap`时，`get`操作会先通过`hashCode`计算其相关的`hash`码值，当这个`hash`码值相同时，会调用`equals`方法来检验当前对象与数组中`hash`码值相同的对象是否等同，如果等同即会返回其关联的`value`。

