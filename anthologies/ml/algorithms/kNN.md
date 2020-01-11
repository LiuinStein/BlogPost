### 0x00 k-Nearest Neighbors Algorithm（kNN）

这里阐述一个最简单的**用于分类**问题的**适用于监督学习**环境下的机器学习算法——kNN算法。其主要具有**高精确度**、**对异常值（outliers）不敏感**的优点，以及**需要大量的计算时间和占用大量的内存**的缺点。而且其**只适用于数值数据**（Numeric values）。

这个算法的原理非常简单，首先给定一个训练集及其答案标签，此后对于每一个给出的测试数据，计算其与训练集中每一个数据的“距离”，然后取距离最近的k个数据作为当前训练样本的预测值。计算距离的过程实则就是我们高中数学中坐标系下计算两点之间距离的公式，对于一个二维坐标系下的两点$(x_1, y_1)$和$(x_2, y_2)$而言，其距离为：
$$
d=\sqrt{(x_1-x_2)^2+(y_1-y_2)}
$$
这么说可能有点抽象，我们现在举例说明，假设我们要做一个程序对电影进行分类，区分出这部电影是爱情电影还是动作电影。我们现在统计出了6部电影中亲吻的次数和打拳的次数作为我们的训练集，如下：

![](https://bucket.shaoqunliu.cn/image/0341.png)

我们对已知训练集作了人工的分类，为其标注出电影类型。我们可以在同一坐标系中以接吻的次数为x轴，以拳击的次数为y轴对上述数据进行描点，我们可以清晰地看到爱情片都集中在了坐标轴的左上角，武打片都集中在了右下角。

![](https://bucket.shaoqunliu.cn/image/0340.png)

现在我们引入一部新的电影，首先我们不知道其是武打片还是爱情片，我们仅仅统计出了其中拳击的次数为18次，接吻的次数为90次，对于训练集中已知的6部影片，我们计算这部新影片到其余6部影片的距离，依据上述距离计算公式，我们给出其到*California Man*这部电影距离的计算过程以作实例：
$$
d=\sqrt{(18-3)^2+(90-104)^2}\approx 20.518
$$

> 两部电影中接吻次数差值的平方加拳击次数差值的平方开根号，即为在上图横纵坐标轴中新电影到*California Man*这部电影的几何距离。

此后我们给出这部新电影到已知训练集中全部的6部电影的距离计算结果：

![](https://bucket.shaoqunliu.cn/image/0342.png)

使用kNN算法我们取k=3，由此可见距离最短的3部电影均为爱情片，那么我们即预测这部新电影为爱情电影。这个过程实则为我们在距离最短的k个类型中进行投票，票数最多的获胜，如果我们在此例中取值k=4，那么这部电影是爱情电影所获票数为3票，是动作电影所获票数为1票，取票数多的获胜为最终预测值。代码如下：

```python
import numpy as np


class Movie:
    def __init__(self, kicks, kisses, type, name=""):
        # type为1表示爱情电影，为2表示动作电影
        self.name = name
        self.kicks = kicks
        self.kisses = kisses
        self.type = type


class kNN:
    def __init__(self, k):
        self.k = k
        self.train_set = []

    def feed(self, kicks, kisses, type):
        self.train_set.append([0, kicks, kisses, type])

    def predict(self, kicks, kisses):
        for i in range(len(self.train_set)):
            self.train_set[i][0] = np.sqrt((kicks - self.train_set[i][1]) ** 2 +
                                           (kisses - self.train_set[i][2]) ** 2)
        self.train_set.sort()
        votes = {}
        for i in range(self.k):
            if self.train_set[i][3] in votes:
                votes[self.train_set[i][3]] += 1
            else:
                votes[self.train_set[i][3]] = 1
        max = 0
        max_type = -1
        for k in votes.keys():
            if votes[k] > max:
                max = votes[k]
                max_type = k
        return max_type


if __name__ == "__main__":
    # k为3的kNN算法
    predictor = kNN(3)
    # 从文件中读取训练集
    with open('F:\\Desktop\\test.txt', 'r') as file:
        content = file.read().splitlines()
        for i in range(len(content)):
            info = content[i].split('\t')
            predictor.feed(int(info[1]), int(info[2]), int(info[3]))
    # 从控制台读取亟待预测的数据
    kick = int(input("Please type count of kicks within this movie: "))
    kiss = int(input("Please type count of kisses within this movie: "))
    if predictor.predict(kick, kiss) == 1:
        print("The result of the predicted type of this movie is ", "Romance")
    else:
        print("The result of the predicted type of this movie is ", "Action")

```





