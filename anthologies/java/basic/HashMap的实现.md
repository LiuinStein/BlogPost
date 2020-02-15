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

**`HashMap`和`HashTable`的区别**

首先，`HashMap`是非线程安全的，而`HashTable`则是线程安全的，也因此，`HashMap`的效率较高。除此之外，在`HashMap`中`null`可以用来作为键，而`HashTable`则不可以，否则抛出`NullPointerException`。

**`HashMap`和`HashSet`的区别**

`HashSet`底层基于`HashMap`实现，其实现了`Set`接口，仅用来存储对象而不是键值对。而且`HashSet`较`HashMap`效率低。

**`HashMap`的长度为什么是2的幂次方**

因为经计算出来的hash值需要对数组长度进行取余运算，然后用余数定位其在hash表中的位置，为了优化取余运算的效率，人们发现取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(\&)操作，即：hash%length​等价于hash\&(length-1)，前提是length是2的n次方，采用二进制位操作\&，相对于%能够提高运算效率。

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

**JDK7 `put`方法为什么线程不安全**

**以下内容仅适用于JDK7之前的版本，即对链表的插入方法为头插法，JDK8以后，链表的插入方法改为了尾插法，从此之后就不存在这个问题了**。

众所周知，`HashMap`不是线程安全的，多线程的情况下操作`HashMap`的`put`操作有可能导致死循环，原因在于`HashMap`扩容使用的`resize`方法，由于扩容是新建一个数组，然后复制原数据到数组，由于数组下标挂有链表，所以需要复制链表，但多线程操作有可能导致环形链表。

我们先来看JDK7 `HashMap`实现中的`transfer`方法：

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i]; // 才用头插入，在链表头部插入元素
            newTable[i] = e; // 线程执行完上一行后，在此处挂起
            e = next; // 循环遍历链表内元素
        }
    }
}
```

我们假设没有执行`resize`之前`Entry`数组的结构如下：

![](https://bucket.shaoqunliu.cn/image/0367.png)

则其在单线程的情况下，正确执行扩容操作后的样子如下：

![](https://bucket.shaoqunliu.cn/image/0368.png)

好了，现在我们来看多线程的情况，假设线程A执行扩容操作，在上述代码片中标记部分线程A被挂起，则目前线程A内的状态如下：

![](https://bucket.shaoqunliu.cn/image/0369.png)

这个时候线程B介入了，线程B也想进行扩容操作，而且恰好线程B的扩容操作还操作完了，则线程B内的状态如下：

![](https://bucket.shaoqunliu.cn/image/0370.png)

这是对于值为3, 7, 5的这三个对象来说，其状态为：

```java
tab[1]=5, 5.next=null
tab[3]=7, 7.next=3, 3.next=null
```

然后CPU又切换时间片到线程A，线程A继续运行，在Java中代码中的临时变量`e`和`next`实际上都为引用类型变量。线程A继续操作，线程A的`newTable`变为了如下状态：

![](https://bucket.shaoqunliu.cn/image/0371.png)

继续循环：

```JAVA
e=7
next=e.next ----> next=3【从主存中取值】
e.next=newTable[3] ----> e.next=3（7.next=3）【从主存中取值】
newTable[3]=e ----> newTable[3]=7
e=next ----> e=3
```

此时状态如下：

![](https://bucket.shaoqunliu.cn/image/0373.png)

`e=3`于是又再一次进入循环：

```java
e=3
next=e.next ----> next=null
e.next=newTable[3] ----> e.next=7（3.next=7）
newTable[3]=e ----> newTable[3]=3
e=next ----> e=null
```

此时线程状态如下，并结束循环：

![](https://bucket.shaoqunliu.cn/image/0372.png)

此时`newTable[3]`中所保存的链表即成了循环链表，当查找一个元素，假设说8，经计算`indexOf(8) = 3`，那么将会在`table[3]`所存储链表中查找，此时`get(8)`操作即为一个死循环。

**JDK8之后的`put`方法为什么线程不安全？**

JDK8中链表的插入方法由JDK7中的头插入改为了尾插入，所以上面会导致死循环的那个问题在JDK8中已经不复存在了。但`HashMap`依旧是线程不安全的，因为写入丢失。

现在考虑一种情况，2个线程同时要向一个`HashMap`中插入元素，而这两个元素恰好又有相同的hash值，假设这个hash值所对应的数组下标所在位置原先是没有元素的，那线程1直接进行赋值，然后CPU切换到了线程2，线程2又进行了一次直接复制，线程2赋的值覆盖了线程1的值，导致了写入丢失的问题。

其次，如果线程1和2向`HashMap`插入元素时，需要进行扩容操作的话，线程1和线程2都会各自生成各自的`newTab`，然后最后`tab = resize()`，到最后`table`里面只保存了最后一个线程的`newTab`，其余线程的均会丢失。

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
    // oldCap翻番后的newCap大于MAXIMUM_CAPACITY也会使newThr=0
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        // 如果说新的容量已经超过最大容量的话
        // 那新的门限值就设置为INT_MAX以后不再扩容
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 复制现有表到新表
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 重新计算在新表中的位置
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // loHead和loTail标记的这个链表
                    // 表示插入到当前数组位置的元素们
                    Node<K,V> loHead = null, loTail = null;
                    // hiHead和hiTail标记的这个链表
                    // 表示插入到新数组位置的元素们
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 对于这个判断条件的理解请看下文
                        if ((e.hash & oldCap) == 0) {
                            // 扩容后的位置没有发生变化
                            // 在链表中顺序插入元素
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            // 扩容后当前结点的位置需要变化
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        // 插入到当前数组位置
                        loTail.next = null;
                        // 有人看了下面的代码
                        // newTab[j + oldCap] = hiHead;
                        // 可能会疑惑这是否会产生覆盖问题
                        // 答案是不会，直接赋值不会产生覆盖问题，因为在下方的j + oldCap是始终大于等于oldCap的，而j是始终小于oldCap的
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        // 插入到新数组位置
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

**对判断条件`(e.hash & oldCap) == 0`的理解**

扩容后重新计算元素在新表中的位置的时候，用到了一个非常巧妙的算法，首先我们每次重新计算，都是将原数组大小n乘以2。我们现在假设原数组大小为16，则新数组大小即为32，我们计算某hash值在数组中的位置时，使用的方法为`e.hash & (n - 1)`，当n翻一番的时候，这个产生的结果只可能出现2种情况，一种是不变，如下：

```
扩容前n为16时：
n-1          0000 0000 0000 0000 0000 0000 0000 1111
hash         1111 1111 1111 1111 0000 1111 0000 0101
hash & (n-1) 0000 0000 0000 0000 0000 0000 0000 0101
扩容后n为32时：
n-1          0000 0000 0000 0000 0000 0000 0001 1111
hash         1111 1111 1111 1111 0000 1111 0000 0101
hash & (n-1) 0000 0000 0000 0000 0000 0000 0000 0101
```

首先一个对象的hash值只与这个对象有关，与其他因素无关，所以扩容前后hash值是不变的，变的只有`n-1`的值，在上例中，扩容前后的这个元素在数组中的位置依旧保持不变

另一种是结果也跟着移动2次幂的位置，如下：

```
扩容前n为16时：
n-1          0000 0000 0000 0000 0000 0000 0000 1111
hash         1111 1111 1111 1111 0000 1111 0001 0101
hash & (n-1) 0000 0000 0000 0000 0000 0000 0000 0101
扩容后n为32时：
n-1          0000 0000 0000 0000 0000 0000 0001 1111
hash         1111 1111 1111 1111 0000 1111 0001 0101
hash & (n-1) 0000 0000 0000 0000 0000 0000 0001 0101
```

在这种情况下，我们只需要看新增的那个bit位是0还是1就好了，是0的话索引没变，是1的话索引变成原索引加`oldCap`。

我们发现新增的那个bit位的位置实际上就是`oldCap`（n）的位置，如下：

```
n            0000 0000 0000 0000 0000 0000 0001 0000
扩容后数组位置不变的情况：
hash         1111 1111 1111 1111 0000 1111 0000 0101
扩容后数组位置发生变化的情况：
hash         1111 1111 1111 1111 0000 1111 0001 0101
```

我们发现只要判断`hash & n`是否等于1就可以判断出扩容之后的位置是否发生变化来，由此判断条件`(e.hash & oldCap) == 0`的意思即为判断扩容后的位置是否发生变化。

### 0x05 hash方法

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

这样的hash计算方法产生的hash碰撞的概率在其源码里有详细的介绍，摘抄如下：

> Ideally, under random `hashCodes`, the frequency of
> nodes in bins follows a Poisson distribution
> (http://en.wikipedia.org/wiki/Poisson_distribution) with a
> parameter of about 0.5 on average for the default resizing
> threshold of 0.75, although with a large variance because of
> resizing granularity. Ignoring variance, the expected
> occurrences of list size k are (exp(-0.5) pow(0.5, k) /
> factorial(k)). The first values are:
> 0:    0.60653066
> 1:    0.30326533
> 2:    0.07581633
> 3:    0.01263606
> 4:    0.00157952
> 5:    0.00015795
> 6:    0.00001316
> 7:    0.00000094
> 8:    0.00000006
> more: less than 1 in ten million

大意是在理想状况下，链表内元素的个数是服从泊松分布（Poisson distribution）的，其还给出了链表内元素个数所出现的概率，例如链表内有1个元素的概率是0.3032，超过8个元素的概率就在千万分之一了。也就说真正需要转化成红黑树的概率其实是很小的。