### 0x00 运行时数据区组成

1. 程序计数器（Program Counter Register）：如果线程执行的是非native方法，则程序计数器中保存的是当前需要执行的指令的地址；如果线程执行的是native方法，则程序计数器中的值是`undefined`。由于程序计数器中存储的数据所占空间的大小不会随程序的执行而发生改变，因此，对于程序计数器是不会发生内存溢出现象(`OutOfMemory`)的。
2. 虚拟机栈（VM Stack）：虚拟机栈中存放的是一个个的栈帧，每个栈帧对应一个被调用的方法。当线程执行一个方法时，就会随之创建一个对应的栈帧，并将建立的栈帧压栈。当方法执行完毕之后，便会将栈帧出栈。同理，这也是递归容易导致内存溢出现象的原因。
3. 本地方法栈（Native Method Stack）：Java栈是为执行Java方法服务，而本地方法栈则是为执行本地方法（Native Method）服务。在JVM规范中，并没有对其具体实现方法以及数据结构作强制规定，虚拟机可以自由实现它。在Hotspot虚拟机中直接就把本地方法栈和Java栈合二为一。
4. 方法区（Method Area）：存储每个类的信息（包括类的名称、方法信息、字段信息）、静态变量、常量以及编译器编译后的代码等。
5. 堆（Heap）：用来存储对象本身及数组，**堆是被所有线程共享的**，在JVM中只有一个堆，因此在堆上分配内存是需要加锁的。

### 0x01 判断对象是否存活

#### 0x00 引用计数法

给对象头处添加一个引用计数器counter，每当有一个地方引用了对象，计数器加1；引用失效，计数器减1；当计数器为0表示该对象已死、可回收。**此方法可能导致循环引用的两个或多个对象都无法进行垃圾回收**，例如：

```java
public class Test {
	public Test ref;

	public static void main(String[] args) {
        Test a = new Test();
        Test b = new Test();
        // a和b交叉引用
        a.ref = b;
        b.ref = a;
        // 将a和b均置为空
        a = null;
        b = null;
        // 手动进行垃圾回收
        // 若采用引用计数法，a, b实则均无法进行垃圾回收
        System.gc();
    }
}
```

#### 0x01 可达性分析法

首先定义一些对象作为GC Roots，以GC Roots为起点可达对象即为存活对象，不可达对象即为需要回收的垃圾内存。可以作为GC Roots对象的如下：

1. 虚拟机栈中的栈帧的局部变量表所引用的对象。
2. 本地方法栈中JNI(Java Native Interface)所引用的对象。
3. 方法区的静态变量和常量所引用的对象。

如下图，绿色框起来的部分即为GC Roots，从GC Roots结点出发可以抵达的对象实例有1, 2, 4, 6，对象实例3和5虽然连通，但并没有任何一个GC Roots与之相连，故对象实例3, 5即为需要进行垃圾回收的对象。

![](https://bucket.shaoqunliu.cn/image/0353.png)

### 0x02 垃圾回收算法

#### 0x00 标记-清除算法（Mark-Sweep）

顾名思义，其分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，标记完成后统一回收所有被标记的对象。这种算法的不足主要体现在效率和空间，从效率的角度讲，标记和清除两个过程的效率都不高；从空间的角度讲，**标记清除后会产生大量不连续的内存碎片**，内存碎片太多可能会导致以后程序运行过程中在需要分配较大对象时，无法找到足够的连续内存，而不得不提前触发一次垃圾收集动作，如图：

![](https://bucket.shaoqunliu.cn/image/0360.png)

#### 0x01 复制算法（Copying）

复制算法是为了解决效率问题而出现的，它将可用的内存分为两块，每次只用其中一块，当这一块内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已经使用过的内存空间一次性清理掉。这样每次只需要对整个半区进行内存回收，内存分配时也不需要考虑内存碎片等复杂情况，只需要移动指针，按照顺序分配即可。复制算法的执行过程如图：

![](https://bucket.shaoqunliu.cn/image/0361.png)

#### 0x02 标记-整理算法（Mark-Compact）

让所有存活对象都向一端移动，然后直接清理掉边界以外的内存。如图：

![](https://bucket.shaoqunliu.cn/image/0362.png)

#### 0x03 分代收集算法（Generational GC）

分代收集算法将堆区分为如下几类：

![](https://bucket.shaoqunliu.cn/image/0354.png)

1. Young Generation（新生代区）
   1. Eden Space（伊甸园）：新对象分配内存的地方。
   2. Survivor Space（幸存者区）：用于存放minor gc之后存活的对象，其分为两块survivor 0和survivor 1。
2. Tenured Generation（老年代区）：存放存活时间较久的，体积较大的对象。新生代与老年代的比例在1:2左右。
3. Permanent Generation（永久代区）：用于存放一些类的永久数据，JDK8之后不再有永久代。

JVM的内存分配和回收过程如下：

所有新对象都是在Eden区进行分配的，两个survivor区在一开始都是空的：

![](https://bucket.shaoqunliu.cn/image/0355.png)

当Eden区满了之后，将会进行一次minor gc。minor gc时，经引用的对象都会被移动到survivor 0区，未经引用的对象将会直接删除：

![](https://bucket.shaoqunliu.cn/image/0356.png)

再到下一次minor gc时，Eden中经引用的对象将会被移动到survivor 1区，目前在survivor 0区的对象，经引用的，将引用计数+1，然后从survivor 0区移动到survivor 1区，未经引用的，将连同Eden区一块进行内存释放。经由此过程后，survivor 1区内将会有2中不同年龄（引用计数）的对象：

![](https://bucket.shaoqunliu.cn/image/0357.png)

再到下一次minor gc时，Eden区和survivor 1区中被引用的对象将会移动到survivor 0区（survivor区互换），然后移动的对象引用计数+1，Eden区和survivor1区将会被回收：

![](https://bucket.shaoqunliu.cn/image/0358.png)

经过一系列的minor gc之后，一些对象的引用计数将达到设定的阈值（例如8），这些足够老（达到阈值）的将会从新生代移动到老年代：

![](https://bucket.shaoqunliu.cn/image/0359.png)

当老年代满了之后，会触发major gc (full gc)，major gc的发生频率较低，且老年代对象存活时间较长，存活标记率较高。