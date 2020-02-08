### 0x00 分布式唯一`id`的生成策略简介

作为一个分布式唯一`id`，一般要求其具有以下特征：

* 在同一业务场景里面必须保持全局的唯一
* 趋势递增

### 0x01 数据库自增长序列或字段

最常见的方法，利用数据库，可保证全数据库唯一。

优点：

* 简单，性能可以接受；
* 代码方便，在`INSERT`语句中直接把主键字段空出来，交由数据库自动填充即可；
* 生成的数字`id`天然排序，对分页或者需要排序的结果很有帮助。

缺点：

* 不同数据库的语法和内部实现方法不同，在数据库迁移的时候可能需要额外的处理；
* 在单个数据库或者一主多从的情况下，只有一个主库可以生成，有单点故障的风险；
* 在性能达不到要求的情况下，难于扩展；
* 分表分库时可能会带来麻烦。

对于有多个主库的情况下，可以设置每个主库的起始数字不一样但步长一样。比如一个主库生成的`id`全是奇数，另外一个生成的`id`全是偶数，也可有效生成分布式唯一`id`。

### 0x02 UUID

UUID即通用唯一识别码，其设计的目的就是为了让分布式系统中的所有元素都有唯一的辨识信息，一般来说这个值生成出来就是全球唯一的。GUID是微软公司出的一套UUID生成组件。

UUID的生成即可交给数据库去做，比如可以在MySQL数据库中使用`UUID()`函数来生成UUID，MySQL中的`UUID()`函数返回的是带`-`分隔符的UUID。同时，也可以交给ORM框架去做，例如Hibernate框架就有一套自己的UUID生成机制，其返回的UUID符合如下规则：IP-JVM启动时间-当前时间右移32位-当前时间-内部计数（8-8-4-8-4）。

优点：

* 简单，很多数据库和编程语言都内置UUID的生成函数；
* 生成ID的性能好，基本不会有性能问题；
* 全球唯一，在数据迁移、数据合并或数据库变更等情况下可以从容应对。

缺点：

* 无法保证趋势递增
* 需使用字符串来存储，查询效率低
* 存储空间比较大，如果是海量数据就需要考虑存储量的问题
* 传输数据量大

### 0x03 Snowflake算法

这是Twitter公司出的分布式唯一ID的生成算法，其结果是一个64bit的整数（Java中的`long`类型），其结构如下：

![](https://bucket.shaoqunliu.cn/image/0350.png)

我们从左往右看，**第一位一直是0**，这一位是不用的，而且它也不能是1，原因很简单，这一位如果是1，那整个值都会变成负数，`id`是负数这种情况显然是不合理的。所以我们实际可用的只有后面63位。

紧接着的41个bit是时间戳，这个时间戳转换成10进制的毫秒数最大值为2199023255551。Linux的时间戳是从1970年的某个时间开始计时的毫秒数，但我们一般将其即设为从某个时间点开始的毫秒数，比如统一设置的集群启动时间，然后这41位所记录的值即为现在的时间戳减去这个统一的集群启动时间的时间戳。

然后是10bit的集群中的机器id，2的10次方是1024，也就是这个算法最多支持1024个结点，全用上这个集群的规模也可见一斑了。

最后12位是交由集群中的每个结点自行生成的自增长序列值。2的12次方即为4096，也就是说每一毫秒，每个分布式节点都可以生成4096个分布式id而不重复。

需要注意的一点是，如果前端和后端需要用到这个id来打交道的话，JavaScript最大支持的整数的精度是53位，64位已经超出了JavaScript支持的最大整数精度，**所以需要用字符串来保存这个生成的id**。这一点在[Twitter官方有关Snowflake算法的介绍文档](https://developer.twitter.com/en/docs/basics/twitter-ids)中就有提到。

这种丢失精度的现象，你可以在你浏览器的控制台使用如下代码查看：

```javascript
(90071992547409921).toString()
```

除此之外，还可以自定义精简版的Snowflake算法，比如缩小64位至53位。

好了，我们来自个实现一下Snowflake算法：

```java
import java.util.concurrent.atomic.AtomicLong;

public class SnowflakeIdentifierHelper {

    private final long twepoch;  // 起始时间戳
    private final int timestampBits;  // 时间戳的位数
    private final int workerIdentifier;  // 工作机器id
    private final int workerIdentifierBits;  // 工作机器id的位数
    private final int sequenceBits;  // 最后序列号的位数
    // 毫秒内序列号
    private final AtomicLong sequence = new AtomicLong(0);
    // 最后一次生成id的时间戳
    private final AtomicLong lastTimeStamp = new AtomicLong(0);

    public SnowflakeIdentifierHelper(int workerIdentifier, long twepoch) {
        this(41, 10, 12, workerIdentifier, twepoch);
    }

    public SnowflakeIdentifierHelper(int timestampBits, int workerIdentifierBits, int sequenceBits, int workerIdentifier, long twepoch) {
        this.timestampBits = timestampBits;
        this.workerIdentifier = workerIdentifier;
        this.workerIdentifierBits = workerIdentifierBits;
        // 确保输入的工作机器id在其bit位数所限制的范围之内
        if (workerIdentifier >= (1 << workerIdentifierBits)) {
            throw new IllegalArgumentException("The worker's id not matches the specific length of bits of it.");
        }
        this.sequenceBits = sequenceBits;
        // 确保所有部分的位数加和小于等于63，以防long溢出
        if (this.timestampBits + this.workerIdentifierBits + this.sequenceBits > 63) {
            throw new IllegalArgumentException("The length of bits of Snowflake identifiers is larger than the length of a long integer.");
        }
        this.twepoch = twepoch;
    }

    public long generate() throws InterruptedException {
        long timestamp;
        synchronized (lastTimeStamp) {
            timestamp = System.currentTimeMillis() - twepoch;
            // 必须是1L而不能是1，因为1是int型，1L是long型，1<<41的结果会导致整型溢出
            if (timestamp >= (1L << timestampBits)) {
                throw new RuntimeException("Timestamp overflows.");
            }
            // 生成的时间戳小于最后一次生成的时间戳即出现了时钟回拨现象
            if (timestamp < lastTimeStamp.get()) {
                throw new RuntimeException("Clock moved backwards.");
            }
            // 归零毫秒内计数器，如果毫秒变了的话
            if (timestamp != lastTimeStamp.get()) {
                sequence.set(0);
            }
            lastTimeStamp.set(timestamp);
        }
        long sequenceNumber = sequence.incrementAndGet();
        if (sequenceNumber >= (1L << sequenceBits)) {
            // 如果当前毫秒内计数器值超过其最大值，那就小睡1ms再生成一遍
            Thread.sleep(1);
            return generate();
        }
        return (timestamp << (workerIdentifierBits + sequenceBits)) +
                (workerIdentifier << sequenceBits) + sequenceNumber;
    }
}

```

> TL;DR
>
> 在`generate()`函数中为什么要用`synchronized`？
>
> 我们必须用`synchronized`框起所有与时间戳计算有关的部分，原因在于`lastTimeStamp`是所有线程的共享变量，`AtomicLong`仅保证其在加减运算时是线程安全的，并不能保证下面的代码也是线程安全的：
>
> ```java
> timestamp = System.currentTimeMillis() - twepoch;
> if (timestamp >= (1L << timestampBits)) {
>     throw new RuntimeException("Timestamp overflows.");
> }
> if (timestamp < lastTimeStamp.get()) {
>     throw new RuntimeException("Clock moved backwards.");
> }
> if (timestamp != lastTimeStamp.get()) {
>     sequence.set(0);
> }
> lastTimeStamp.set(timestamp);
> ```
>
> 在上述代码片中，假设线程1执行了第一行代码，得到的时间戳我们假设是10000，此时处理器切换到了线程2，线程2也得到了一个时间戳假设是20000，线程2执行完了所有的代码，线程2执行完成最后一行代码代码之后将`lastTimeStamp`设置成了自己生成的时间戳20000，这个时候线程2再切换到线程1继续执行的话，就会抛出`RuntimeException("Clock moved backwards.")`。线程1就会错误地认为时钟回拨了。所以我们必须让所有有关时间戳的代码并发执行一次完成。
>
> 为什么要用`AtomicLong`而不直接用`long`？
>
> 在32位操作系统中，64位的`long`和`double`变量由于会被JVM当作两个分离的32位来进行操作，所以不具有原子性。而使用`AtomicLong`能让`long`的操作保持原子型。

上面这份代码的`id`生成效率是非常高的，我们用如下测试代码生成了1亿条`id`，耗时46.653s。

```java
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

public class Main {

    public static void main(String[] args) throws IOException {
        // 假设worker id是101，起始时间戳是1577808000L
        SnowflakeIdentifierHelper helper = new SnowflakeIdentifierHelper(101, 1577808000L);
        // 创建包含16个线程的线程池
        ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(16);
        FileOutputStream fileOutputStream = new FileOutputStream(new File("F:/Desktop/ids.txt"));
        // 循环调用1000次
        for (int j = 0; j < 1000; j++) {
            executor.execute(() -> {
                // 每个线程生成100000条id并写入文件
                for (int i = 0; i < 100000; i++) {
                    try {
                        long id = helper.generate();
                        // 将整数型的id转换成二进制字符串
                        // 方便我们对比查看程序输出结果是否有误
                        String s = Long.toBinaryString(id);
                        // toBinaryString函数会将二进制结果前面的0舍弃，我们给它手动加上
                        s = s.length() == 63 ? "0" + s : s;
                        String result = s.charAt(0) + " " + s.substring(1, 42) + " " + s.substring(42, 52) + " " + s.substring(52, 64) + "\r\n";
                        fileOutputStream.write(result.getBytes());
                    } catch (InterruptedException | IOException ignored) {

                    }
                }
            });
        }
        fileOutputStream.flush();
        executor.shutdown();
    }
}
```

此后我们将结果转换成二进制字符串写入文件，生成的`id`文件占地6.58GB，我们将程序的输出文件中节选了10条，如下：

```
0 10110111111000111001111000000000101101101 0001100101 000000000011
0 10110111111000111001111000000000101101011 0001100101 000000000100
0 10110111111000111001111000000000101101110 0001100101 000000000011
0 10110111111000111001111000000000110110011 0001100101 000000000001
0 10110111111000111001111000000000110110100 0001100101 000000000001
0 10110111111000111001111000000000110110100 0001100101 000000000010
0 10110111111000111001111000000000101101110 0001100101 000000000010
0 10110111111000111001111000000000110110100 0001100101 000000000100
0 10110111111000111001111000000000110110100 0001100101 000000000111
0 10110111111000111001111000000000110110100 0001100101 000000000110
```

