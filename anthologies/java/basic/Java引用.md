### 0x00 前言

说一说Java中的引用，Java没有C++中标记地那么明显，在C++中，引用类型是必需用`&`标记出来的，而Java不用，这就可能对初学者造成一定的困惑。

### 0x01 引用传递和值传递

在Java中，对于基本数据类型（如`int`, `char`, `double`等），值就直接保存在变量中。而对于对象类型（如`String`），我们使用如下的代码创建一个对象：

```java
String str = "Hello World!";
```

`str`实质上为一个`String`引用类型变量，变量中保存的实际上为这个对象在堆内存中的地址信息，而不是实际的对象数据。

对于基本数据类型，赋值操作会直接覆盖变量的值，原来的数据将被覆盖掉，被替换为新值。而对于引用类型变量`str`，赋值运算符会修改其所保存对象的地址信息，原来对象的地址被覆盖掉，重新写入新的对象的地址，但是原来对象本身并没有发生改变，当垃圾回收机制检测到这个对象在内存中没有引用的话，即被垃圾回收。

如下代码即会输出"Hello World!"，因为在`test`函数中，引用类型变量`a`传入进来的时候一开始保存的是引用类型变量`str`所指向的地址，但是随即`a`被重新赋予了一个新的对象地址，但是其原来指向的地址仍然有引用类型变量`str`所指向，这是一个强引用，所以不会被垃圾回收机制给回收，此时引用类型变量`a`和`str`分别指向两个不同的堆内存地址。

```java
private static void test(StringBuilder a) {
    a = new StringBuilder("Test");
}

public static void main(String[] args) {
    StringBuilder str = new StringBuilder("Hello World!");
    test(str);
    System.out.println(str);
}
```

但是如果是下面的方法的话，输出结果就变成了2行，一行是`Hello World!`另一行是`Hello China!`，因为`append`方法是直接作用于`a`所指向的对象的，所以原对象的值发生了改变。

```java
private static void test(StringBuilder a) {
    a.append("\nHello China!");
}

public static void main(String[] args) {
    StringBuilder str = new StringBuilder("Hello World!");
    test(str);
    System.out.println(str);
}
```

### 0x02 数组的传递

数组的传递为引用传递，因为如果我们定义一个数组：

```java
int[][]  arr = new int[2][4];
```

`arr`仍然标记着一个引用类型变量，指向在堆内存中的数组实体。而在这个数组实体中，有2个引用类型变量分别指向2个长度为4的数组实体。如图：

![](https://bucket.shaoqunliu.cn/image/0347.png)

所以，如下代码输出的结果即为`1 2 3`，因为我们传到`test`方法内部的是一个对数组的引用。

```java
private static void test(int[] a) {
    a[0] = 1;
    a[1] = 2;
    a[2] = 3;
}

public static void main(String[] args) {
    int[] arr = new int[3];
    test(arr);
    System.out.println(arr[0] + " " + arr[1] + " " + arr[2]);
}
```

