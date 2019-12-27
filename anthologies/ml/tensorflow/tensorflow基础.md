### 0x00 数据流图

首先明确一个问题，Tensorflow是基于数据流图的计算。就比如说如下代码：

```python
import tensorflow as tf

a = tf.constant([[1, 2]])
b = tf.constant([[3], [4]])
c = tf.matmul(a, b)

with tf.Session() as sess:
    result = sess.run(c)
    print(result)
```

a和b是两个矩阵，c是a和b这两个矩阵乘法运算后的结果。在代码执行到第8行`sess.run(c)`之前，c的值时没有经过计算。相反，之前的代码只是简单地创建了一个数据流图中的一个结点。只有当执行到`sess.run(c)`的时候，Tensorflow才开始计算c的值，继而顺着数据流图首先计算出a和b的值，然后再计算两个矩阵的乘积得出c的结果。

### 0x01 reduce_sum和reduce_mean

一开始的时候着实搞不懂这两个函数到底是干什么的。说得直白一点，reduce就是消除一个维度，sum就是使用求和的方法，`reduce_sum`即为使用求和的方法来消除一个维度。同理，`reduce_mean`即为使用取平均值的方法来消除一个维度。先来看其文档中的一个2维的例子：

```python
x = tf.constant([[1, 1, 1], [1, 1, 1]])
```

显然，这是一个2行3列——2x3——的矩阵。当执行`tf.reduce_sum(x, 0)`时，第二个参数的0即为消除维度0，也就是消除2x3里面的那个2，将两行中每一列的元素相加，得`[2 2 2]`即为最终结果。如果指定第二个参数为1，即为消除维度1，将每一列的元素相加，即可得`[3 3]`即为最终结果。如果不指定`reduce_sum`的第二个参数的话，默认即为消除所有维度，得到的结果即为矩阵中所有元素的相加，即为数字6。

然后，再来看一个复杂的例子：

```python
x = tf.constant([[[1, 1, 1, 1], [1, 1, 1, 1], [1, 1, 1, 1]], [[1, 1, 1, 1], [1, 1, 1, 1], [1, 1, 1, 1]]])
```

显然，这是一个2x3x4的矩阵，执行`tf.reduce_sum(x, 0)`我们将得到一个3x4的矩阵，如下：

```
[[2 2 2 2]
 [2 2 2 2]
 [2 2 2 2]]
```

执行`tf.reduce_sum(x, 1)`我们将得到一个2x4的矩阵，如下：

```
[[3 3 3 3]
 [3 3 3 3]]
```

执行`tf.reduce_sum(x, 2)`我们将得到一个2x3的矩阵，如下：

```
[[4 4 4]
 [4 4 4]]
```

执行`tf.reduce_sum(x)`我们将得到原矩阵中，所有元素的和，即为数字24

### 0x02 第一次训练

我们的第一次训练，使用机器学习的方法通过既定的点来拟合一条直线。代码如下：

```python
import tensorflow as tf
import numpy as np

# 生成100个随机数，形状为1x100
x = np.random.rand(100)
# 正确答案序列
y_correct = 3 * x + 4

# 设置斜率（权重）和偏移，以1和2作为初始值
k = tf.Variable(1.)
b = tf.Variable(2.)

# y的预测值
y_predict = k * x + b

# 以方差作为损失函数
loss = tf.reduce_mean(tf.square(y_predict - y_correct))

# 以0.1为学习率进行梯度下降
optimizer = tf.train.GradientDescentOptimizer(0.1)

# 梯度下降的目的是为了降低损失函数loss的值
train = optimizer.minimize(loss)

with tf.Session() as sess:
    # 初始化所有变量
    sess.run(tf.global_variables_initializer())
    # 训练200次
    for i in range(200):
        # 进行梯度下降
        sess.run(train)
        # 每隔20次循环输出一次k和b的值
        if i % 20 == 0:
            print(sess.run([k, b]))
```

### 0x03 学习率

在上述代码中，我们设置的学习率为0.1，训练200次，所得的`[k, b]`的最终结果为：

```
[2.9544437, 4.025851]
```

当我们试图扩大学习率至1的时候，训练200次，所得的`[k, b]`的最终结果为：

```
[-2.7914815e+34, -5.4713246e+34]
```

当我们将学习率缩小10倍至0.01时，训练200次，所得的`[k, b]`的最终结果为：

```
[2.526483, 4.2690997]
```

我们抛开复杂的数学知识，来看学习率，学习啦实际上管控的是梯度下降时的下降步幅。如下图：

![](https://bucket.shaoqunliu.cn/image/0303.jpg)

如果学习率太小，梯度下降的速度就会变得缓慢，如上面的例子中，学习率为0.1和0.01时的对比。如果学习率太大，梯度下降的步伐就会摇摆不定，易产生震荡，始终难以找到一个最小值，如上面例子中，学习率为0.1和1时的对比。所以说，在训练过程中，一般根据训练轮数设置动态变化的学习率。

- 刚开始训练时：学习率以 0.01 ~ 0.001 为宜。
- 一定轮数过后：逐渐减缓。
- 接近训练结束：学习速率的衰减应该在100倍以上。

### 0x04 第一个神经网络

在这个例子中，我们拟合一条曲线，并且为其中的点加入随机的噪音值，然后使用神经网络去拟合这条曲线，代码如下：

```python
import tensorflow as tf
import numpy as np
import matplotlib.pyplot as plt

# np.linspace(起始点, 终止点(含), 数据量)，此函数返回一个数组
# 数组内数据自起始点开始到终止点结束，共有200个元素，且这200个元素之间是等距的
# np.newaxis用于在某位置增加一个维度，比如初始为[1,2,3]
# 后经[:, np.newaxis]在末尾增加一个维度即为[[1],[2],[3]]
# x_data即为一个200x1的矩阵
x_data = np.linspace(-0.5, 0.5, 200)[:, np.newaxis]
# 产生一个均值为0方差为0.05的形状为200x1的符合正态分布的随机数矩阵
noise = np.random.normal(0, 0.05, (200, 1))
# 平方函数加上一定的噪声，作为最终值
y_correct = np.square(x_data) + noise

# 因为需要使用sess.run进行数据投喂，所以定义两个占位符
# None的标识为有多少行都行
x = tf.placeholder(tf.float32, [None, 1])
y = tf.placeholder(tf.float32, [None, 1])

# 神经网络中间层
weight_1 = tf.Variable(tf.random.normal([1, 10]))
bias_1 = tf.Variable(tf.zeros([1, 10]))
# y_predict_1的矩阵形状为[None, 10]
y_predict_1 = tf.matmul(x, weight_1) + bias_1
prediction_1 = tf.nn.tanh(y_predict_1)

# 神经网络输出层
weight_2 = tf.Variable(tf.random.normal([10, 1]))
bias_2 = tf.Variable(tf.zeros([1, 1]))
y_predict_2 = tf.matmul(prediction_1, weight_2) + bias_2
prediction_2 = tf.nn.tanh(y_predict_2)

loss = tf.reduce_mean(tf.square(prediction_2 - y))
train = tf.train.GradientDescentOptimizer(0.1).minimize(loss)

with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    for i in range(2000):
        sess.run(train, feed_dict={x: x_data, y: y_correct})
    # 通过学习的结果，带入初始的数据进行预测，并获取预测值
    predict_value = sess.run(prediction_2, feed_dict={x: x_data})
    # 画图
    plt.figure()
    # 对正确值进行描点
    plt.scatter(x_data, y_correct)
    # 画出预测值图像
    plt.plot(x_data, predict_value, 'r-', lw=5)
    plt.show()
```

使用`tanh`作激活函数，拟合结果如图所示：

![](https://bucket.shaoqunliu.cn/image/0303.png)

### 0x05 激活函数

激活函数用于为神经元引入非线性元素，使其可以拟合任何非线性函数。如果没有激活函数，神经网络无论有多少层，输入输出都是线性组合。常见的激活函数有sigmoid, tanh, Relu, softmax。

sigmoid函数定义域可以到任何值，值域为[0, 1]，此输出范围和概率范围一致，因此可以用概率的方式解释输出。其表达式为：
$$
\text{sigmoid}(x)=\frac{1}{1+e^{-x}}
$$
其导数为：
$$
y'=y(1-y)
$$
tanh函数就是数学上的双曲正切函数，其与sigmoid函数之间的关系为：
$$
\tanh(x)=2\times\text{sigmoid}(2x)-1
$$
其导数为：
$$
y'=1-y^2
$$
对比sigmoid和tanh两者导数输出可知，tanh函数的导数比sigmoid函数导数值更大，即梯度变化更快，也就是在训练过程中收敛速度更快。

Relu函数如下：
$$
\text{Relu}(x)=\max(0,x)
$$
其导数为常数，当x小于0时导数为0，大于0时导数为1。计算速度和收敛速度非常快。

### 0x06 One-hot 编码

one-hot编码，一般译为独热编码，例如，MNIST手写数字识别中，目标答案只有0-9十个数字，在one-hot编码中这就是一个有10个元素的数组，则数字1使用one-hot编码表示即为`[0 1 0 0 0 0 0 0 0 0]`，同理数字2即为`[0 0 1 0 0 0 0 0 0 0]`。即为：除了答案的索引之外，其他值均为0。

### 0x07 MNIST 初上手

下面给出一段训练MNIST手写数字识别的代码的代码：

```python
import tensorflow as tf
from tensorflow.examples.tutorials.mnist import input_data

# 以one-hot编码读入data文件夹中的数据
mnist = input_data.read_data_sets("data", one_hot=True)

# 行表示有多少张图片，列表示每张图片的28*28=784个像素点
x = tf.placeholder(tf.float32, shape=[None, 784])
# 行表示有多少张图片，列表示返回有10种不同的可能（one-hot集有10列）
y = tf.placeholder(tf.float32, shape=[None, 10])

# x * weight所得矩阵即为[None, 10]
weight = tf.Variable(tf.zeros([784, 10]))
bias = tf.Variable(tf.zeros([10]))

# 使用softmax函数作为激活函数，softmax返回一个概率矩阵，即预测的one-hot集中每一个数据的概率
prediction = tf.nn.softmax(tf.matmul(x, weight) + bias)
# 用方差来衡量损失
loss = tf.reduce_mean(tf.square(y - prediction))

# 以0.2的学习率进行训练
train = tf.train.GradientDescentOptimizer(0.2).minimize(loss)

with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    # 整个数据集将分批投放，每一批的大小为100
    batch_size = 100
    # 一共有多少批
    batch_num = mnist.train.num_examples // batch_size
    # 将所有的数据循环训练100次
    for _ in range(100):
        # 分批进行训练
        for i in range(batch_num):
            # 获取数据集及答案集
            images, labels = mnist.train.next_batch(batch_size)
            sess.run(train, feed_dict={x: images, y: labels})
        # 测量准确率
        # argmax(input, axis)用于获取input沿着axis的方向的最大值
        # 其中axis默认为0，表示列方向，axis=1表示行方向
        # softmax函数返回一个概率矩阵，此处使用
        # tf.argmax(prediction, 1)和tf.argmax(y, 1)
        # 用于获取预测所得最大概率的那一方
        # 和答案集one-hot数据中标识为1的那一方所在的下标是否相等
        correct_prediction = tf.equal(tf.argmax(prediction, 1), tf.argmax(y, 1))
        # 将数据转换为float32类型
        # 然后对矩阵中所有的数据求均值得出准确率
        accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
        print(sess.run(accuracy, feed_dict={x: mnist.test.images, y: mnist.test.labels}) * 100, "%")
```

### 0x08 重新认识学习率

在上述训练MNIST数据集的例子中，我们使用的学习率为恒定值0.2，循环训练100遍后对验证集所产生的准确率大约为92.68%。正如我们之前所述，学习率太小，梯度下降的速率会比较低下，如果学习率太大，梯度下降的步伐就会摇摆不定，容易产生震荡。所以，此处我们可以使用动态学习率，初始设置一个较大的学习率，此后随着迭代次数的增加，**指数减缓**学习率的值，例如：
$$
\alpha = 0.95^{\text{epoch_num}} \cdot \alpha_0
$$
依据上述公式，我们假设初始学习率为2，每训练5次，指数减小一次学习率为例，改进上述代码，得：

```python
import tensorflow as tf
import math
from tensorflow.examples.tutorials.mnist import input_data

mnist = input_data.read_data_sets("data", one_hot=True)

x = tf.placeholder(tf.float32, shape=[None, 784])
y = tf.placeholder(tf.float32, shape=[None, 10])

weight = tf.Variable(tf.zeros([784, 10]))
bias = tf.Variable(tf.zeros([10]))

prediction = tf.nn.softmax(tf.matmul(x, weight) + bias)

loss = tf.reduce_mean(tf.square(y - prediction))

with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    batch_size = 100
    batch_num = mnist.train.num_examples // batch_size
    learn_rate_t = 0
    train = tf.train.GradientDescentOptimizer(2).minimize(loss)
    for _ in range(100):
        if _ % 5 == 0:
            # 动态学习率
            learn_rate = math.pow(0.95, learn_rate_t) * 2
            learn_rate_t = learn_rate_t + 1
            train = tf.train.GradientDescentOptimizer(learn_rate).minimize(loss)
        for i in range(batch_num):
            images, labels = mnist.train.next_batch(batch_size)
            sess.run(train, feed_dict={x: images, y: labels})
        correct_prediction = tf.equal(tf.argmax(prediction, 1), tf.argmax(y, 1))
        accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
        print(sess.run(accuracy, feed_dict={x: mnist.test.images, y: mnist.test.labels}) * 100, "%")

```

使用此代码再对MNIST进行训练，此后使用测试集来验证所得的准确率为93.10%。

### 0x09 梯度下降法原理

梯度下降法核心公式就一个：
$$
x_1=x_0-\alpha f'(x_0)
$$
其中：

* $x_0$代表x的当前位置
* $x_1$代表x的下一个位置
* $\alpha$代表学习率

假设对于函数$f(x)=x^2$，我们要通过梯度下降找出其最小值，假设学习率我们固定为0.25，当前位置$x_0=3$，我们来逐步进行梯度下降：
$$
f(x)=x^2\Rightarrow f'(x)=2x \\ 
\text{step 1. }~x=3-0.25\times2\times 3=1.5 \\ 
\text{step 2. }~x=1.5-0.25\times2\times 1.5=0.75 \\ 
\text{step 3. }~x=0.75-0.25\times2\times 0.75=0.375 \\ 
\cdots
$$
通过梯度下降，$x$的值逐渐向$f(x)$的最小值点$x=0$靠近了。

### 0x0A 二次代价函数和交叉熵代价函数

二次代价函数的数学表达实为：
$$
C=\frac{1}{2n}\sum_x(y-\text{prediction})^2 \\ 
\text{prediction}=\sigma(z) \\ 
z=wx+b
$$
其中：

* $n$为样本数量
* $x$为输入值，$y$为正确的输出值
* prediction为预测值，$\sigma(z)$为激活函数

在python中，我们一般以如下的形式表达二次代价函数：

```python
loss = tf.reduce_mean(tf.square(y - prediction))
```

我们假设只有一组样本，来对上述的公式进行简化得：
$$
C=\frac{(y-\text{prediction})^2}{2} \\ 
\text{prediction}=\sigma(z) \\ 
z=wx+b
$$
在深度学习的过程中，我们通过梯度下降的方法不断优化$w$和$b$的值，使最终产生的代价函数值尽可能地小。由此，对上述简化的例子，在梯度下降时所使用的2个偏导数分别为：
$$
\frac{\partial C}{\partial w}=(y-\text{prediction})\sigma'(z)x \\ 
\frac{\partial C}{\partial b}=(y-\text{prediction})\sigma'(z) \\ 
\text{prediction}=\sigma(z) \\ 
z=wx+b
$$
由上式，我们发现其梯度下降的速度与激活函数的导数成正比，激活函数的导数越大，w和b调整地就越快，训练收敛地也就越快。鉴于，我们一般使用sigmoid函数作为激活函数，我们来看一下这个函数的图像：

![](https://bucket.shaoqunliu.cn/image/0304.png)

由上图我们可以发现，在绿点的时候，sigmoid函数的斜率是低于其在红点时的斜率的。由此在梯度下降的过程中，在绿点的下降速率也是低于在红点时的下降速率的。但是，我们可以看到，绿点离最小值的距离是大于红点离最小值的距离的。这往往不是我们想要的，我们想要的是错误越大改正的幅度越大，从而学习得越快。

由此，我们抛弃了二次代价函数，引入了交叉熵代价函数，其基本的数学形式如下：
$$
C=-\frac{1}{n}\sum_x[y\ln(\text{prediction})+(1-y)\ln(1-\text{prediction})] \\ 
\text{prediction}=\sigma(z) \\ 
z=wx+b
$$
我们对交叉熵代价函数求导可得：
$$
\frac{\partial C}{\partial w}=\frac{1}{n}\sum_x x(\sigma(z)-y) \\ 
\frac{\partial C}{\partial b}=\frac{1}{n}\sum_x (\sigma(z)-y) \\ 
z=wx+b
$$
由此，我们可以看出，当误差$\sigma(z)-y$越大，参数w和b调整的速度就越快，梯度下降的速率也就越大。

### 0x0B 使用交叉熵代价函数优化MNIST的训练

下面我们将使用交叉熵代价函数，对上文中所给出的MNIST训练集进行优化，代码如下：

```python
import tensorflow as tf
from tensorflow.examples.tutorials.mnist import input_data

mnist = input_data.read_data_sets("data", one_hot=True)

x = tf.placeholder(tf.float32, shape=[None, 784])
y = tf.placeholder(tf.float32, shape=[None, 10])

weight = tf.Variable(tf.zeros([784, 10]))
bias = tf.Variable(tf.zeros([10]))

# tensorflow所自带的交叉熵代价函数已内置求解softmax函数
# 所以此处的prediction无需计算softmax
prediction = tf.matmul(x, weight) + bias

# 使用tf.nn.softmax_cross_entropy_with_logits来计算
# softmax的交叉熵结果，其中参数labels表示正确答案集
# logits表示预测的结果集
loss = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(labels=y, logits=prediction))

# 以0.2的学习率进行训练
train = tf.train.GradientDescentOptimizer(0.2).minimize(loss)

with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    batch_size = 100
    batch_num = mnist.train.num_examples // batch_size
    for _ in range(100):
        for i in range(batch_num):
            images, labels = mnist.train.next_batch(batch_size)
            sess.run(train, feed_dict={x: images, y: labels})
        correct_prediction = tf.equal(tf.argmax(prediction, 1), tf.argmax(y, 1))
        accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
        print(sess.run(accuracy, feed_dict={x: mnist.test.images, y: mnist.test.labels}) * 100, "%")
```

通过对比上述2代码，我们可以发现使用交叉熵代价函数，可以使训练快速地收敛到一定范围，上述代码训练所得的准确率在92.61%左右。

