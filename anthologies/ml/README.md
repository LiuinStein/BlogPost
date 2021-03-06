### 0x00 索引目录

我们先来看一副图，形象描绘深度学习的图。这是Reddit上Programming  Humor下的一个话题，图下点赞最高的一条评论写道：如同大多数上课的学生一样，他们都没带硬盘（脑子）、CPU（脑子）和内存（还是脑子）。

![](https://bucket.shaoqunliu.cn/image/0333.png)

一开始学习深度学习的时候，比较急功近利，迫切想知道代码怎么写，TensorFlow怎么用。一开始学的时候就想，我又不当数学家，那些数学上的东西还是交给专精的人去考虑吧，我还是主要学学怎么用他们实现好的框架来搭建现成的应用吧。但是这样学总觉得差了些什么，只学会了框架，知道其可以做什么，而不知道其具体是怎么实现的，心理上总觉得少了些什么。所以在此建议各位，还是尽量搞懂其内部最基础的数学原理吧，最起码日后理解起来都会更方便一些。

#### 0x01 TensorFlow应用篇

[Tensorflow基础](/anthologies/ml/tensorflow/tensorflow基础) 本文共4104字，阅读大约需要17分钟

> 本文首先从数据流图的视角介绍了TensorFlow的计算方法，然后分别介绍了学习率、激活函数、One-hot编码、梯度下降法原理等机器学习基础概念。以线性拟合的方式搭建了第一个神经网络。此后分别以使用二次代价函数和交叉熵代价函数的方法训练了MNIST数据集，并阐述了相关的不同。

[卷积神经网络](/anthologies/ml/tensorflow/卷积神经网络) 本文共2310字，阅读大约需要9分钟

> 本文介绍了卷积神经网络的基本概念，以及相关算法流程，同时以使用卷积神经网络训练MNIST为例，阐述了如何使用TensorFlow实现一个基本的卷积神经网络。

[过拟合问题](/anthologies/ml/tensorflow/过拟合问题) 本文共2125字，阅读大约需要8分钟

> 本文详细概述了过拟合现象产生的原因，基于先前训练MNIST的代码给出了过拟合现象的实例。同时使用常用的过拟合处理方法Dropout对已经产生过拟合现象的实例进行优化。同时阐述了Dropout的基本原理和过程。

[重新认识优化器](/anthologies/ml/tensorflow/重新认识优化器) 本文共2344字，阅读大约需要9分钟

> 本文详细讲述了机器学习过程中常用的优化器及其算法。其中包含标准梯度下降法（GD）、批量梯度下降法（BGD）、随机梯度下降法（SGD）、基于动量的随机梯度下降法、Nesterov Accelerated Gradient (NAG)、AdaGrad、RMSProp、AdaDelta、Adam算法。

[循环神经网络](/anthologies/ml/tensorflow/循环神经网络) 本文共2767字，阅读大约需要11分钟

> 本文讲述了循环神经网络及其3种变体——双向循环神经网络、深度循环神经网络以及长短时记忆网络的基本概念及其相关结构。同时给出了使用TensorFlow基于LSTM来训练MNIST的过程和代码。

#### 0x02 手撕机器学习算法

[感知器](/anthologies/ml/algorithms/感知器) 本文共1357字，阅读大约需要6分钟

> 本文以最简单的机器学习单元——感知器为例，从数学层面阐述了其计算原理，并给出了相关的示例的演示——训练一个可以计算AND运算的感知器——以及相关代码。

[线性单元和梯度下降](/anthologies/ml/algorithms/线性单元和梯度下降) 本文共2813字，阅读大约需要11分钟

> 本文从感知器在处理线性不可分问题时可能导致的无法收敛的角度引出线性单元及其表示方法，同时给出了梯度下降法的基本原理及相关数学推导，并以线性拟合为例给出相关代码示例。

[神经网络](/anthologies/ml/algorithms/神经网络) 本文共2736字，阅读大约需要11分钟

> 本文给出了神经元的基本概念及其表示方法，同时给出了训练神经元的方法——反向传播算法及其相关数学推导。同时基于其基本原理实现一个简单的全连接神经网络框架并用其训练了一个MNIST手写数字识别模型。

[kNN](/anthologies/ml/algorithms/kNN) 本文共1076字，阅读大约需要5分钟

> 本文阐述了kNN算法的基本原理，并以电影分类为例，给出相关示例及代码。