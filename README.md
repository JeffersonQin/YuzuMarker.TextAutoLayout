# YuzuMarker.TextAutoLayout

本 repo 将描述 YuzuMarker 中用到的嵌字文本自动布局模型，并给出实现。

# 模型概述

## 2021/09/24 v1

* 本模型仅讨论文本块为水平或者竖直的情况，更复杂的情况将在以后进行讨论
* 下文中假设文本为竖排，横排情况类似，不做过多讨论
* 下标一般从0开始

### 主要思想

* 首先对于原图进行处理，得到便于计算的原文字区域表达
* 通过建立一系列参数和一个区域计算模型，可以通过参数计算出排版区域
* 建立指标函数，计算生成的排版区域和原来文字区域之间的重合程度
* 设定初始参数，最大化衡量指标，最后得到一个全局 / 局部最优解

### 图像处理

讨论一张图片中的一部分文字，假设我们能够通过 Erode / Dilate 以及二值化操作将文本处理为黑块，记这些点组成的点集为![](https://latex.codecogs.com/gif.latex?T)。对点集![](https://latex.codecogs.com/gif.latex?T)逐列扫描，记每一列的子点集为![](https://latex.codecogs.com/gif.latex?T_i)。引入超参数![](https://latex.codecogs.com/gif.latex?\lambda)，对于每一个![](https://latex.codecogs.com/gif.latex?T_i)，若

![](https://latex.codecogs.com/gif.latex?|T_i|<\lambda(\max{T_i}-\min{T_i}))

则将![](https://latex.codecogs.com/gif.latex?T_i)从![](https://latex.codecogs.com/gif.latex?T)中舍去，即

![](https://latex.codecogs.com/gif.latex?T:=T-T_i)

经过处理，我们能够得到一个大致准确的文本位置的点集![](https://latex.codecogs.com/gif.latex?T)，为了方便地计算损失函数，我们不妨假设其在纵向是一个凸图形，然后将其表示为形如

![](https://latex.codecogs.com/gif.latex?T^*=[(0,y_{1s},y_{1e}),(1,y_{2s},y_{2e}),...,(n,y_{ns},y_{ne})])

的形式。上述式子的含义是将其描述为列的集合的形式，而每列又可以表示为列的横坐标和开始和结束的纵坐标的形式（因为我们假设他是凸图形），如果文字是横排的，那只需要假设在横向是凸图形，然后可以得到一个类似的式子，这里就不作展开了。

注意：
* 如果一列没有任何像素，那这一列的表达记作![](https://latex.codecogs.com/gif.latex?(x,0,0))
* 这里我们用的横坐标记号是![](https://latex.codecogs.com/gif.latex?0...n)，也就是横跨整张图的横坐标，之所以这样做也是为了计算损失函数时可以直接以矩阵的形式计算，然后搭配自动求导框架可以帮助我们自动更新参数。当然，我们也可以进行优化，即框定一个更小的区域，然后计算那个区域中的相对坐标，可以节省计算资源。这是后话。

### 排版计算函数

有非常多的决定排版位置的因素，这里我们选取几个重要的、可推广的因素：
* 起始横坐标 (文本最右侧的横坐标) ![](https://latex.codecogs.com/gif.latex?x)
* 每一列的起始纵坐标 (文本最右侧的纵坐标) ![](https://latex.codecogs.com/gif.latex?\vec{y})
* 文本换行位置 (这个我们将重点讨论)
* 字体大小，在这里我们取每个字占的像素个数 ![](https://latex.codecogs.com/gif.latex?l_f)
* 文本间距，这里我们取两列字间的空隙像素个数 ![](https://latex.codecogs.com/gif.latex?d)

接下来我将先讨论文本换行位置，换行位置可以有多种表达形式：
* 一个多维向量，描述每列多少个字 / 多长
* 一个多维向量，描述每个剖分所在的位置
* 一个 One-Hot 编码的多维向量，记录每个字间是被分割

首先，这三种表达都是不可微不可导的，也就带来了一个问题，我们不能用梯度下降来更新这些参数。第二，看第一种表达，他还有一个约束限制，那就是向量的所有维度的值的和必须为文本长度，记作![](https://latex.codecogs.com/gif.latex?\inline\sum_i{w_i}=l_t)，增加了处理难度。

然而，我们可以发现，换行情况是可以在有限时间内被枚举出来的。

如果每个文字都可以被换行，假设字符总数为![](https://latex.codecogs.com/gif.latex?m)，那么就有![](https://latex.codecogs.com/gif.latex?2^{m-1})中字符情况，这无疑是灾难性的，毕竟这已经是指数复杂度了。

所以，我们需要增加一定的约束条件：
* 列的数量 ![](https://latex.codecogs.com/gif.latex?k)
* 列的最长字符数量 ![](https://latex.codecogs.com/gif.latex?m)

基于上面两个约束条件，我们可以给出所有可能的情况枚举，记为![](https://latex.codecogs.com/gif.latex?C)。接下来，我们正对每个字符串分割情况进行讨论，字符串分割记作![](https://latex.codecogs.com/gif.latex?\vec{c})，这里的编码使用上面的第一种编码，即这个多维向量的每一维记录这一列有多少个字。

总结一下，我们现在可以用![](https://latex.codecogs.com/gif.latex?x,\vec{y},l_f,d,\vec{c})来建立排版函数，除![](https://latex.codecogs.com/gif.latex?\vec{c})外都是需要优化的参数，所以记：

![](https://latex.codecogs.com/gif.latex?\vec\theta=(x,\vec{y},l_f,d)) 为参数

设排版面积函数为：

![](https://latex.codecogs.com/gif.latex?A(\vec\theta,\vec{c}))

这个函数的结果与![](https://latex.codecogs.com/gif.latex?\vec{c})的维度相同，记录每行的横坐标和纵坐标的起始结束位置：

![](https://latex.codecogs.com/gif.latex?A(\vec\theta,\vec{c})_i=(x-i\times{(l_f+d)},y_i,y_i+c_i\times{l_f}))

将![](https://latex.codecogs.com/gif.latex?A)扩展到整个图像大小长度的矩阵![](https://latex.codecogs.com/gif.latex?A^*)来得到与![](https://latex.codecogs.com/gif.latex?T^*)类似的结果：

![](https://latex.codecogs.com/gif.latex?\forall{i\in[0,|\vec{c}|\)},\forall{j\in{[0,l_f+d\)}},k=x-i\times{(l_f+d)})时

![](https://latex.codecogs.com/gif.latex?A^*(\vec\theta,\vec{c})_{j-k}=(j-k,y_i,y_i+c_i\times{l_f}))

其他情况下：

![](https://latex.codecogs.com/gif.latex?A^*(\vec\theta,\vec{c})_p=(p,0,0))

### 计算重合度指标

为了定义原区域![](https://latex.codecogs.com/gif.latex?T^*)和计算排版区域![](https://latex.codecogs.com/gif.latex?A^*)间的重合程度，定义重合度指标：

![](https://latex.codecogs.com/gif.latex?L(\vec\theta,\vec{c})=\frac{A^*\cap{T^*}}{A^*\cup{T^*}})
