# 卷积神经网络基础 | AI技术聚合
本来想自己写点东西的，但是发现了一个很不错的博客，就什么也没做。

什么是卷积？

一句话：在某一时刻，一个点的能量（或值）等于其他多个点的叠加。

感受野
---

一句话：特征图上的一个点对应输入图上的一个区域。

如下所示：

[![](https://aitechtogether.com/wp-content/uploads/2022/03/462977d9b71c495ea92c860dc02fe1ab.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/462977d9b71c495ea92c860dc02fe1ab.webp)

卷积神经网络结构：
---------

[![](https://aitechtogether.com/wp-content/uploads/2022/03/f66d9495813c430b9d828487b7d8ea85.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/f66d9495813c430b9d828487b7d8ea85.webp)

> *   卷积层+激活函数（Conv+ReLU）  
>     
> *   池化层（Pololing）  
>     
> *   全连接层（FC）  
>     

2.1 卷积层
-------

### 2.1.1 卷积运算

> 总共有四个步骤：反转、移动、乘法和求和。
> 
> 神经网络中的操作不需要逆向操作，只需要三个步骤。

如下图所示输入矩阵I与卷积核K对应相乘，乘完后相加得到一块的值，移动步长1继续相乘相加最终得到输出矩阵。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/136d49e0b36144f89adc63d77c1f37f1.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/136d49e0b36144f89adc63d77c1f37f1.webp)

此处可以看到的是一个3X4的矩阵做完卷积后变成了2X3的矩阵，在图像处理中即为输入图像与卷积核进行卷积后的结果中损失了部分值，输入图像的边缘被“修剪”掉了（边缘处只检测了部分像素点，丢失了图片边界处的众多信息）。这是因为边缘上的像素永远不会位于卷积核中心，而卷积核也没法扩展到边缘区域以外。

这个结果我们是不能接受的，有时我们还希望输入和输出的大小应该保持一致。为解决这个问题，可以在进行卷积操作前，对原矩阵进行边界填充（Padding），也就是在矩阵的边界上填充一些值，以增加矩阵的大小，通常都用0来进行填充的。

如下图所示，输入5X5，输出仍然为5X5。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/266f3a145b57479eaf98c96383d2f728.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/266f3a145b57479eaf98c96383d2f728.webp)

Stride：对于卷积的stride size设计，padding的长度要尽可能的分割。

卷积的模式：Full、Same、Valid，如下图所示：

[![](https://aitechtogether.com/wp-content/uploads/2022/03/ae88436c53df4eaabb38dbee912d0cf7.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/ae88436c53df4eaabb38dbee912d0cf7.webp)

### 2.1.2 通道

通道（channel）：一般指图像的颜色通道。

> 单通道图像：一般指灰度图像
> 
> 多通道图像：一般指基于RGB的图像
> 
> Feature Map：经过卷积和激活函数处理的图像

单通道卷积：单通道图像的卷积（单通道图像单卷积核，单通道图像多卷积核）

多通道卷积：多通道图像的卷积（multi-channel image multi-convolution kernel）

在计算多通道卷积时，先将输入图像与卷积核对应的位置相乘相加，再将多个卷积核的结果相加，最后将偏置加到输出特征图上。如下所示：

[![](https://aitechtogether.com/wp-content/uploads/2022/03/85bbbd38ab1c4fb1b7e56d7d18ea6bde.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/85bbbd38ab1c4fb1b7e56d7d18ea6bde.webp)

x\[:,:,0\]与w0\[:,:,0\]结果为4，x\[:,:,1\]与w0\[:,:,1\]结果为0，x\[:,:,2\]与w0\[:,:,2\]结果为1，再加上偏置1最后结果为9，正如o\[:,:,0\]所示。

### 2.1.3 扩张卷积

为图像语义分割中的下采样而提出的一种卷积形式会降低图像分辨率并丢失信息。

Dilation rate：卷积核处理数据时原始值之间的距离。

利用添加空洞（空洞补0）扩大感受野，让原本3×3的卷积核，在相同参数量和计算量下拥有更大的感受野，从而无需下采样。

### 2.1.4 标准卷积与深度可分离卷积

标准卷积：如下图所示，卷积层有4个Filter，每个Filter有3个卷积核，每个卷积核为3X3的，故参数总量为4X3X3X3，共108个。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/d356548c539942a2b1414c1461db4a14.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/d356548c539942a2b1414c1461db4a14.webp)

深度可分离卷积：Depthwise Separable Convolution，先做Depthwise卷积，再做Pointwise 卷 积，实现空间维（卷积核大小）和通道维（特征图）的分离。

如下图所示，DSC由Depthwise Convolution和Pointwise Convolution两部分构成。Depthwise Convolution的计算非常简单，它对channel input的每个通道分别使用一个卷积核，然后将所有卷积核的输出再进行拼接得到它的最终输出。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/a7469896203b454fbb337b7c6a6b0e1a.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/a7469896203b454fbb337b7c6a6b0e1a.webp)

Pointwise Convolution实际为1×1卷积，相乘再叠加即可。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/29c01e6710df422e89996fd44ed18f9a.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/29c01e6710df422e89996fd44ed18f9a.webp)

相比标准卷积参数数量大幅减少，Depthwise卷积（3X3X3=27)，Pointwise（4X3X1X1=12），总共39个。

详细结束见：卷积神经网络之深度可分离卷积（Depthwise Separable Convolution） – 知乎

### 2.1.5 卷积层作用

浅卷积层：提取图像的基本特征，如边缘、方向和纹理特征。

深度卷积层：提取高级图像特征，出现高级语义模式，如“轮子”、“人脸”等特征。

2.2 激活函数
--------

常用激活函数主要有：identity、ReLU、PReLU、ERU、Maxout。

### 2.2.1 identity

[![](https://latex.codecogs.com/gif.latex?f%5Cleft%20%28%20x%20%5Cright%20%29%3D%20x%2C%20x%20%5Cepsilon%20%5Cleft%20%28%20-%5Cinfty%20%2C&plus;%5Cinfty%20%5Cright%20%29)
](https://latex.codecogs.com/gif.latex?f%5Cleft%20%28%20x%20%5Cright%20%29%3D%20x%2C%20x%20%5Cepsilon%20%5Cleft%20%28%20-%5Cinfty%20%2C+%5Cinfty%20%5Cright%20%29)

适合线性任务，比较稳定，但对卷积输出没有影响。

### 2.2.2 ReLU

[![](https://latex.codecogs.com/gif.latex?f%28x%29%20%3D%20%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%200%2C%20x%3C%200%20%26%20%5C%5Cx%2C%20x%5Cgeqslant%200%20%26%20%5Cend%7Bmatrix%7D%5Cright.%20x%5Cepsilon%20%28-%5Cinfty%20%2C&plus;%5Cinfty%20%29)
](https://latex.codecogs.com/gif.latex?f%28x%29%20%3D%20%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%200%2C%20x%3C%200%20%26%20%5C%5Cx%2C%20x%5Cgeqslant%200%20%26%20%5Cend%7Bmatrix%7D%5Cright.%20x%5Cepsilon%20%28-%5Cinfty%20%2C+%5Cinfty%20%29)

ReLU函数的优点：

•计算速度快。ReLU函数只有线性关系， 比Sigmoid和Tanh要快很多。

• 当输入为正时，不存在梯度消失问题。

ReLU函数的缺点：

• 强制性把负值置为0，可能丢掉一些特征。

• 当输入为负时，权重无法更新，导致“神经元死亡”（学习率不宜过大）。

### 2.2.3 PReLU

[![](https://latex.codecogs.com/gif.latex?f%28x%29%20%3D%20%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%20%5Calpha%20x%2C%20x%3C%200%20%26%20%5C%5Cx%2C%20x%5Cgeqslant%200%20%26%20%5Cend%7Bmatrix%7D%5Cright.%20x%5Cepsilon%20%28-%5Cinfty%20%2C&plus;%5Cinfty%20%29)
](https://latex.codecogs.com/gif.latex?f%28x%29%20%3D%20%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%20%5Calpha%20x%2C%20x%3C%200%20%26%20%5C%5Cx%2C%20x%5Cgeqslant%200%20%26%20%5Cend%7Bmatrix%7D%5Cright.%20x%5Cepsilon%20%28-%5Cinfty%20%2C+%5Cinfty%20%29)

当α=0.01时，称作Leaky ReLU，当从高斯分布中随机产生时，称为 Randomized ReLU（RReLU）。

PReLU函数的优点：

• 比sigmoid/tanh收敛快。

•解决了ReLU的“神经元死亡”问题。

PReLU函数的缺点：

• 需要再学习一个参数，工作量变大。

### 2.2.4 ELU

[![](https://latex.codecogs.com/gif.latex?f%28x%29%20%3D%20%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%20%5Calpha%20%28e%5E%7Bx%7D-1%29%2C%20x%3C%200%20%26%20%5C%5Cx%2C%20x%5Cgeqslant%200%20%26%20%5Cend%7Bmatrix%7D%5Cright.%20x%5Cepsilon%20%28-%5Cinfty%20%2C&plus;%5Cinfty%20%29)
](https://latex.codecogs.com/gif.latex?f%28x%29%20%3D%20%5Cleft%5C%7B%5Cbegin%7Bmatrix%7D%20%5Calpha%20%28e%5E%7Bx%7D-1%29%2C%20x%3C%200%20%26%20%5C%5Cx%2C%20x%5Cgeqslant%200%20%26%20%5Cend%7Bmatrix%7D%5Cright.%20x%5Cepsilon%20%28-%5Cinfty%20%2C+%5Cinfty%20%29)

ELU函数的优点：

• 处理噪声数据有一些优势。

•更容易收敛(带e的求导更易收敛）。

ELU函数的缺点：

• 计算量大，收敛速度慢。

### 2.2.5 Maxout

[![](https://aitechtogether.com/wp-content/uploads/2022/03/9408f8301a23454b9146e26be4ed6a60.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/9408f8301a23454b9146e26be4ed6a60.webp)

增加一层神经网络，神经元个数k自定。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/5a7a0573d9ea4e5dbe82c9ddf5db250d.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/5a7a0573d9ea4e5dbe82c9ddf5db250d.webp)

Maxout函数的优点：

• Maxout能够缓解梯度消失。

• 规避了ReLU神经元死亡的缺点。

Maxout函数的缺点：

• 添加了参数和计算。

### 2.2.6激活函数选择

> CNN在卷积层尽量不要使用Sigmoid和Tanh，将导致梯度消失。
> 
> 首先选用ReLU，使用较小的学习率，以免造成神经元死亡的情况。
> 
> 如果ReLU失效，考虑使用Leaky ReLU、PReLU、ELU或者 Maxout，此时一般情况都可以解决。

2.3 池化层
-------

在 CNN中，通常会在连续的 Conv 层之间定期插入一个池化层。其功能是逐步减小表示的空间大小，以减少网络中的参数和计算量，从而控制过拟合。池化层无包含需要训练的参数，指定池化的大小、步幅、类型即可。同时由于池化层取了多个位置的总体统计特征，增加了网络的鲁棒性。

> 池化操作使用一个位置的相邻输出的整体统计数据作为该位置的输出。
> 
> 常用最大池化（max-pooling）和 均值池化（average-pooling），如下图所示，其中最大池化使用较多。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/716ae5e6a67e4c53954fdba9071ef8ab.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/716ae5e6a67e4c53954fdba9071ef8ab.webp)

2.4 全连接层
--------

[![](https://aitechtogether.com/wp-content/uploads/2022/03/1f3af6938eae5f99c65c2854f8f6d288.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/1f3af6938eae5f99c65c2854f8f6d288.webp)

以上图为例，我们仔细看上图全连接层的结构，全连接层中的每一层是由许多神经元组成的（1x 4096）的平铺结构。

问题来了，3X3X5的输出又是怎么变到1X4096的呢？

[![](https://aitechtogether.com/wp-content/uploads/2022/03/956cfcd92cedbb6d8aabd0ce52d8b987.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/956cfcd92cedbb6d8aabd0ce52d8b987.webp)

可以理解为做了一个卷积，从上图我们可以看出，我们用一个3x3x5的filter 去卷积激活函数的输出，得到的结果就是一个fully connected layer 的一个神经元的输出，这个输出就是一个值。

因为我们有4096个神经元

实际上我们就是用一个3x3x5x4096的卷积层去卷积激活函数的输出

这样做有什么用？

在2.2中提到过，卷积层的作用是提取特征，到全连接层时特征已经提取完了，现在要做的是分类。

从后往前推，如果要识别修复过的猫，红色表示神经元被激活，将这些特征结合起来就可以识别出一只猫。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/3ce70927cfb9da8bdc723909c878dd25.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/3ce70927cfb9da8bdc723909c878dd25.webp)

继续往前推，一个猫头有多种特征，比如眼睛、耳朵、嘴巴等，组合起来就可以识别出猫头。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/92cf1dcb15f0f43f6ae7de9c5912d478.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/92cf1dcb15f0f43f6ae7de9c5912d478.webp)

这个细节特征从何而来？

它来自前面的卷积层和下采样层，使它们联系在一起。

参考：CNN 入门讲解：什么是全连接层（Fully Connected Layer）? – 知乎 (zhihu.com)

2.5 输出层
-------

### 2.5.1 分类问题

对于分类问题使用Softmax函数。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/c93264fc6dcd49af900776451a7b296f.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/c93264fc6dcd49af900776451a7b296f.webp)

其中Zi为第i个节点的输出值，n为输出节点的个数，即分类的类别个数。通过Softmax函数就可以将多分类的输出值转换为范围在\[0, 1\]和为1的概率分布。

Softmax的意义在于不再唯一的确定某一个最大值，而是为每个输出分类的结果都赋予一个概率值，表示属于每个类别的可能性。

### 2.5.2 递归问题

使用线性函数解决递归问题。

[![](https://aitechtogether.com/wp-content/uploads/2022/03/a3be24a63ffc4876989c31b84a2eb17c.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/a3be24a63ffc4876989c31b84a2eb17c.webp)

2.6 卷积神经网络的训练
-------------

> 步：
> 
> Step 1: 用随机数初始化所有的卷积核和参数/权重。
> 
> Step 2: 将训练图片作为输入，执行前向步骤（卷积，ReLU，池化 以及全连接层的前向传播）并计算每个类别的对应输出概率 。
> 
> Step 3: 计算输出层的总误差：总误差=1/2 ∑ (目标概率−输出概 率)^2。
> 
> Step 4: 使用BP算法计算误差相对于所有权重的梯度，并用梯度下降法更新所有的卷积核/权重和参数的值，以使输出误差最小。

卷积层和池化层在训练的时候要改成神经网络的形式，如下图

[![](https://aitechtogether.com/wp-content/uploads/2022/03/c085a8f7b0f648daaaa6c12c8f18692b.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/c085a8f7b0f648daaaa6c12c8f18692b.webp)

[![](https://aitechtogether.com/wp-content/uploads/2022/03/f2bac697c7e14644b037a5c12a4e607c.webp)
](https://aitechtogether.com/wp-content/uploads/2022/03/f2bac697c7e14644b037a5c12a4e607c.webp)