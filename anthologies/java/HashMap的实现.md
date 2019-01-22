### 0x00 理论知识

首先`HashMap`是基于hash表的，hash表的相关知识及其存储原理，可以参考我之前写的一篇文章：[查找算法总结](/anthologies/algorithm/查找?id=_0x02-哈希表)，要着重看此文章关于冲突处理所讲述的拉链法。

![](https://bucket.shaoqunliu.cn/image/0209.png)

我们仅在这稍微明确一点概念：首先对于拉链式的hash表，其是由数组和链表共同组成的，当然Java中还引入了红黑树，当链表长度大于8时，将链表转为红黑树以提高查找效率。我们先来看上面的这个图，保存这个hash表的数组称之为桶（bucket），而每个桶中保存了一系列的数据入口（entries）。

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

方法签名：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
```

如果桶为空或者长度为0的话，那么调用`resize`方法对其进行扩容

```java
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
```

通过`(n - 1) & hash`（其中n为桶的长度，hash为依据key所计算的hash码值）计算当前元素对应在`table`数组（桶）中的下标，如果`table`数组中此项为空（意味着没有冲突发生）那就直接在此项处新建一个Node直接将其插入。

```java
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
```

如果桶中已经有了至少一个元素，首先判断链表的第一个结点与当前所插入的结点的`key`是否等同（`key`的hash相同并且`equals`方法返回`true`被认为等同），如果相同的话，则将标记e指向p。

```java
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
```

如果桶的entry类型为`TreeNode`的话，那就调用红黑树的插入方法，将此键值对插入。

```java
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```

否则调用链表的插入方法，目标在链表尾部插入此结点：

```JAVA
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
```

如果在链表中找到了和插入键值对`key`等同的结点，则退出循环：

```java
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
```

如果标记`e`不为`null`的话，即在当前hash表中找到了与所要插入的键值对的`key`相同的结点，`onlyIfAbsent`即为是否仅在`oldValue`为`null`的情况下更新键值对，这个参数在`put`方法中被默认给定为`false`：

```java
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount; 
```

键值对数量超过阈值时，调用`resize`方法进行扩容对桶进行扩容：

```java
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

在上述两个代码片中，我们可以看到两个方法`afterNodeAccess`和`afterNodeInsertion`这两个方法在`HashMap`里面的实现是空的，在此处什么都不做，其注释中写道：Callbacks to allow `LinkedHashMap ` post-actions，即适用于`LinkedHashMap `实现一些后置功能的回调函数。

### 0x02 get方法



### 0x03 resize方法









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

