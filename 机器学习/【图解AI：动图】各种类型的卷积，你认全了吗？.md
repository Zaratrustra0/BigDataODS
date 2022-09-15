# 【图解AI：动图】各种类型的卷积，你认全了吗？
![](https://oscimg.oschina.net/oscnet/a72876b488df2b747f250e9e6b9a8373e9d.jpg)

卷积（convolution）是深度学习中非常有用的计算操作，主要用于提取图像的特征。在近几年来深度学习快速发展的过程中，卷积从标准卷积演变出了反卷积、可分离卷积、分组卷积等各种类型，以适应于不同的场景，接下来一起来认识它们吧。

**一、卷积的基本属性**  
**卷积核（Kernel）：** 卷积操作的感受野，直观理解就是一个滤波矩阵，普遍使用的卷积核大小为 3×3、5×5 等；  
**步长（Stride）：** 卷积核遍历特征图时每步移动的像素，如步长为 1 则每次移动 1 个像素，步长为 2 则每次移动 2 个像素（即跳过 1 个像素），以此类推；  
**填充（Padding）：** 处理特征图边界的方式，一般有两种，一种是对边界外完全不填充，只对输入像素执行卷积操作，这样会使输出特征图的尺寸小于输入特征图尺寸；另一种是对边界外进行填充（一般填充为 0），再执行卷积操作，这样可使输出特征图的尺寸与输入特征图的尺寸一致；  
**通道（Channel）：** 卷积层的通道数（层数）。  
如下图是一个卷积核（kernel）为 3×3、步长（stride）为 1、填充（padding）为 1 的二维卷积：  
![](https://oscimg.oschina.net/oscnet/ed70c6c8660ea0d8c23a60b69a750aaf1ef.jpg)

**二、卷积的计算过程**  
卷积的计算过程非常简单，当卷积核在输入图像上扫描时，将卷积核与输入图像中对应位置的数值逐个相乘，最后汇总求和，就得到该位置的卷积结果。不断移动卷积核，就可算出各个位置的卷积结果。如下图：  
![](https://oscimg.oschina.net/oscnet/3e0200b4dbbc34d2f96e20aa5e072c1ff4a.jpg)
   
**三、卷积的各种类型**  
卷积现在已衍生出了各种类型，包括标准卷积、反卷积、可分离卷积、分组卷积等等，下面逐一进行介绍。  
**1、标准卷积**  
**（1）二维卷积（单通道卷积版本）（2D Convolution: the single channel version）**  
只有一个通道的卷积。  
如下图是一个卷积核（kernel）为 3×3、步长（stride）为 1、填充（padding）为 0 的卷积：  
![](https://oscimg.oschina.net/oscnet/52133a0ad3aed83cea2f39519af90b0c1db.jpg)
   
**（2）二维卷积（多通道版本）（2D Convolution: the multi-channel version）**  
拥有多个通道的卷积，例如处理彩色图像时，分别对 R, G, B 这 3 个层处理的 3 通道卷积，如下图：  
![](https://oscimg.oschina.net/oscnet/b0e0a7250330752a10864e028ac2dae99ec.jpg)
   
再将三个通道的卷积结果进行合并（一般采用元素相加），得到卷积后的结果，如下图：  
![](https://oscimg.oschina.net/oscnet/236ee5ebeaf87288f3f61d0157731d263fd.jpg)
   
**（3）三维卷积（3D Convolution）**  
卷积有三个维度（高度、宽度、通道），沿着输入图像的 3 个方向进行滑动，最后输出三维的结果，如下图：  
![](https://oscimg.oschina.net/oscnet/9d95948c9d59894995d50fcbfe107dc2299.jpg)
   
**（4）1x1 卷积（1 x 1 Convolution）**  
当卷积核尺寸为 1x1 时的卷积，也即卷积核变成只有一个数字。如下图：  
![](https://oscimg.oschina.net/oscnet/c86bf4291094f83aa93208008a0bc31676c.jpg)
   
从上图可以看出，1x1 卷积的作用在于能有效地减少维度，降低计算的复杂度。1x1 卷积在 GoogLeNet 网络结构中广泛使用。

**2、反卷积（转置卷积）（Deconvolution / Transposed Convolution）**  
卷积是对输入图像提取出特征（可能尺寸会变小），而所谓的 “反卷积” 便是进行相反的操作。但这里说是 “反卷积” 并不严谨，因为并不会完全还原到跟输入图像一样，一般是还原后的尺寸与输入图像一致，主要用于向上采样。从数学计算上看，“反卷积” 相当于是将卷积核转换为稀疏矩阵后进行转置计算，因此，也被称为 “转置卷积”  
如下图，在 2x2 的输入图像上应用步长为 1、边界全 0 填充的 3x3 卷积核，进行转置卷积（反卷积）计算，向上采样后输出的图像大小为 4x4  
![](https://oscimg.oschina.net/oscnet/732a36b4f27737fbe8cb46e166b418005e4.jpg)
   
**3、空洞卷积（膨胀卷积）（Dilated Convolution / Atrous Convolution）**  
为扩大感受野，在卷积核里面的元素之间插入空格来 “膨胀” 内核，形成 “空洞卷积”（或称膨胀卷积），并用膨胀率参数 L 表示要扩大内核的范围，即在内核元素之间插入 L-1 个空格。当 L=1 时，则内核元素之间没有插入空格，变为标准卷积。  
如下图为膨胀率 L=2 的空洞卷积：  
![](https://oscimg.oschina.net/oscnet/239b526729ef1ca62868d6269c62831ce24.jpg)
   
**4、可分离卷积（Separable Convolutions）**  
**（1）空间可分离卷积（Spatially Separable Convolutions）**  
空间可分离卷积是将卷积核分解为两项独立的核分别进行操作。一个 3x3 的卷积核分解如下图：  
![](https://oscimg.oschina.net/oscnet/1a243d8d808a9f40b9f167946aa7af28fcc.jpg)
   
分解后的卷积计算过程如下图，先用 3x1 的卷积核作横向扫描计算，再用 1x3 的卷积核作纵向扫描计算，最后得到结果。采用可分离卷积的计算量比标准卷积要少。  
![](https://oscimg.oschina.net/oscnet/7f02b4fcc0cf58a07a536dfe40ff6cbe816.jpg)
   
**（2）深度可分离卷积（Depthwise Separable Convolutions）**  
深度可分离卷积由两步组成：深度卷积和 1x1 卷积。  
首先，在输入层上应用深度卷积。如下图，使用 3 个卷积核分别对输入层的 3 个通道作卷积计算，再堆叠在一起。  
![](https://oscimg.oschina.net/oscnet/bb3b53ab9b1c517b88df6190785c314d19a.jpg)
   
再使用 1x1 的卷积（3 个通道）进行计算，得到只有 1 个通道的结果  
![](https://oscimg.oschina.net/oscnet/b57852e7bef6abe667acb8a804b8e7719ee.jpg)
   
重复多次 1x1 的卷积操作（如下图为 128 次），则最后便会得到一个深度的卷积结果。  
![](https://oscimg.oschina.net/oscnet/7de52ee96501d2ab58f213cd9bc42e31b69.jpg)
   
完整的过程如下：  
![](https://oscimg.oschina.net/oscnet/c36f99138e32724705aa5c210a1067013ef.jpg)
   
**5、扁平卷积（Flattened convolutions）**  
扁平卷积是将标准卷积核拆分为 3 个 1x1 的卷积核，然后再分别对输入层进行卷积计算。这种方式，跟前面的 “空间可分离卷积” 类似，如下图：  
![](https://oscimg.oschina.net/oscnet/2d07be592c6891e7e4557c10a86689290e7.jpg)
   
**6、分组卷积（Grouped Convolution）**  
2012 年，AlexNet 论文中最先提出来的概念，当时主要为了解决 GPU 显存不足问题，将卷积分组后放到两个 GPU 并行执行。  
在分组卷积中，卷积核被分成不同的组，每组负责对相应的输入层进行卷积计算，最后再进行合并。如下图，卷积核被分成前后两个组，前半部分的卷积组负责处理前半部分的输入层，后半部分的卷积组负责处理后半部分的输入层，最后将结果合并组合。  
![](https://oscimg.oschina.net/oscnet/785efa16bf500dd13c3d099d7746cfa4039.jpg)
   
**7、混洗分组卷积（Shuffled Grouped Convolution）**  
在分组卷积中，卷积核被分成多个组后，输入层卷积计算的结果仍按照原先的顺序进行合并组合，这就阻碍了模型在训练期间特征信息在通道组之间流动，同时还削弱了特征表示。而混洗分组卷积，便是将分组卷积后的计算结果混合交叉在一起输出。  
如下图，在第一层分组卷积（GConv1）计算后，得到的特征图先进行拆组，再混合交叉，形成新的结果输入到第二层分组卷积（GConv2）中：

![](https://oscimg.oschina.net/oscnet/d1e03fe76ed80f9a4396730d150aa3b3d93.jpg)

欢迎关注本人的微信公众号 “大数据与人工智能 Lab”（BigdataAILab），获取更多信息

![](https://static.oschina.net/uploads/space/2018/0213/155533_IdYn_876354.jpg)

**推荐相关阅读**

**1、AI 实战系列**

*   [【AI 实战】手把手教你文字识别（文字检测篇：MSER、CTPN、SegLink、EAST 等）](https://my.oschina.net/u/876354/blog/3054322)
*   [【AI 实战】手把手教你文字识别（入门篇：验证码识别）](https://my.oschina.net/u/876354/blog/3048523)
*   [【AI 实战】快速掌握 TensorFlow（一）：基本操作](https://my.oschina.net/u/876354/blog/1930175)
*   [【AI 实战】快速掌握 TensorFlow（二）：计算图、会话](https://my.oschina.net/u/876354/blog/1930490)
*   [【AI 实战】快速掌握 TensorFlow（三）：激励函数](https://my.oschina.net/u/876354/blog/1937296)
*   [【AI 实战】快速掌握 TensorFlow（四）：损失函数](https://my.oschina.net/u/876354/blog/1940819)
*   [【AI 实战】搭建基础环境](https://my.oschina.net/u/876354/blog/1924805)
*   [【AI 实战】训练第一个模型](https://my.oschina.net/u/876354/blog/1926060)
*   [【AI 实战】编写人脸识别程序](https://my.oschina.net/u/876354/blog/1926679)
*   [【AI 实战】动手训练目标检测模型（SSD 篇）](http://my.oschina.net/u/876354/blog/1927351)
*   [【AI 实战】动手训练目标检测模型（YOLO 篇）](https://my.oschina.net/u/876354/blog/1927881)

**2、大话深度学习系列**

*   [【精华整理】CNN 进化史](https://my.oschina.net/u/876354/blog/1797489)
*   [大话文本识别经典模型（CRNN](https://my.oschina.net/u/876354/blog/3047853)）
*   [大话文本检测经典模型（CTPN）](https://my.oschina.net/u/876354/blog/3047851)
*   [大话文本检测经典模型（SegLink）](https://my.oschina.net/u/876354/blog/3049704)
*   [大话文本检测经典模型（EAST）](https://my.oschina.net/u/876354/blog/3050127)
*   [大话文本检测经典模型（PixelLink）](https://my.oschina.net/u/876354/blog/3056318)
*   [大话文本检测经典模型（Pixel-Anchor）](https://my.oschina.net/u/876354/blog/3062706)
*   [大话卷积神经网络（CNN）](http://my.oschina.net/u/876354/blog/1620906)
*   [大话循环神经网络（RNN）](https://my.oschina.net/u/876354/blog/1621839)
*   [大话深度残差网络（DRN）](https://my.oschina.net/u/876354/blog/1622896)
*   [大话深度信念网络（DBN）](https://my.oschina.net/u/876354/blog/1626639)
*   [大话 CNN 经典模型：LeNet](https://my.oschina.net/u/876354/blog/1632862)
*   [大话 CNN 经典模型：AlexNet](https://my.oschina.net/u/876354/blog/1633143)
*   [大话 CNN 经典模型：VGGNet](https://my.oschina.net/u/876354/blog/1634322)
*   [大话 CNN 经典模型：GoogLeNet](https://my.oschina.net/u/876354/blog/1637819)
*   [大话目标检测经典模型：RCNN、Fast RCNN、Faster RCNN](https://my.oschina.net/u/876354/blog/1787921)
*   [大话目标检测经典模型：Mask R-CNN](https://my.oschina.net/u/876354/blog/1802743)
*   [大话注意力机制](https://my.oschina.net/u/876354/blog/3061863)

**3、图解 AI 系列**

*   [什么是语义分割、实例分割、全景分割](https://my.oschina.net/u/876354/blog/3055850)
*   [各种深度学习卷积（标准卷积、反卷积、可分离卷积、分组卷积…）](https://my.oschina.net/u/876354/blog/3064227)

**4、AI 杂谈**

*   [27 种深度学习经典模型](https://my.oschina.net/u/876354/blog/1924779)
*   [浅说 “迁移学习”](https://my.oschina.net/u/876354/blog/1614883)
*   [什么是 “强化学习”](https://my.oschina.net/u/876354/blog/1614879)
*   [AlphaGo 算法原理浅析](https://my.oschina.net/u/876354/blog/1594849)
*   [大数据究竟有多少个 V](https://my.oschina.net/u/876354/blog/1604254)

**5、大数据超详细系列**

*   [Apache Hadoop 2.8 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/993836)
*   [Apache Hive 2.1.1 安装配置超详细教程](https://my.oschina.net/u/876354/blog/1057639)
*   [Apache HBase 1.2.6 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/1163018)
*   [离线安装 Cloudera Manager 5 和 CDH5（最新版 5.13.0）超详细教程](https://my.oschina.net/u/876354/blog/1605320)