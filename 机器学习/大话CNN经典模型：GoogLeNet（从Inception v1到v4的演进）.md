# 大话CNN经典模型：GoogLeNet（从Inception v1到v4的演进）
![](https://static.oschina.net/uploads/space/2018/0317/141419_uDBe_876354.png)

2014 年，GoogLeNet 和 VGG 是当年 ImageNet 挑战赛 (ILSVRC14) 的双雄，GoogLeNet 获得了第一名、VGG 获得了第二名，这两类模型结构的共同特点是层次更深了。VGG 继承了 LeNet 以及 AlexNet 的一些框架结构（详见  [大话 CNN 经典模型：VGGNet](https://my.oschina.net/u/876354/blog/1634322)），而 GoogLeNet 则做了更加大胆的网络结构尝试，虽然深度只有 22 层，但大小却比 AlexNet 和 VGG 小很多，GoogleNet 参数为 500 万个，AlexNet 参数个数是 GoogleNet 的 12 倍，VGGNet 参数又是 AlexNet 的 3 倍，因此在内存或计算资源有限时，GoogleNet 是比较好的选择；从模型结果来看，GoogLeNet 的性能却更加优越。

小知识：GoogLeNet 是谷歌（Google）研究出来的深度网络结构，为什么不叫 “GoogleNet”，而叫 “GoogLeNet”，据说是为了向 “LeNet” 致敬，因此取名为 “GoogLeNet”

那么，GoogLeNet 是如何进一步提升性能的呢？  
一般来说，提升网络性能最直接的办法就是增加网络深度和宽度，深度指网络层次数量、宽度指神经元数量。但这种方式存在以下问题：  
（1）参数太多，如果训练数据集有限，很容易产生过拟合；  
（2）网络越大、参数越多，计算复杂度越大，难以应用；  
（3）网络越深，容易出现梯度弥散问题（梯度越往后穿越容易消失），难以优化模型。  
所以，有人调侃 “深度学习” 其实是 “深度调参”。  
解决这些问题的方法当然就是在增加网络深度和宽度的同时减少参数，为了减少参数，自然就想到将全连接变成稀疏连接。但是在实现上，全连接变成稀疏连接后实际计算量并不会有质的提升，因为大部分硬件是针对密集矩阵计算优化的，稀疏矩阵虽然数据量少，但是计算所消耗的时间却很难减少。

那么，有没有一种方法既能保持网络结构的稀疏性，又能利用密集矩阵的高计算性能。大量的文献表明可以将稀疏矩阵聚类为较为密集的子矩阵来提高计算性能，就如人类的大脑是可以看做是神经元的重复堆积，因此，GoogLeNet 团队提出了 Inception 网络结构，就是构造一种 “基础神经元” 结构，来搭建一个稀疏性、高计算性能的网络结构。

**【问题来了】什么是 Inception 呢？**  
Inception 历经了 V1、V2、V3、V4 等多个版本的发展，不断趋于完善，下面一一进行介绍

**一、Inception V1**  
通过设计一个稀疏网络结构，但是能够产生稠密的数据，既能增加神经网络表现，又能保证计算资源的使用效率。谷歌提出了最原始 Inception 的基本结构：  
![](https://static.oschina.net/uploads/space/2018/0317/141510_fIWh_876354.png)
   
该结构将 CNN 中常用的卷积（1x1，3x3，5x5）、池化操作（3x3）堆叠在一起（卷积、池化后的尺寸相同，将通道相加），一方面增加了网络的宽度，另一方面也增加了网络对尺度的适应性。  
网络卷积层中的网络能够提取输入的每一个细节信息，同时 5x5 的滤波器也能够覆盖大部分接受层的的输入。还可以进行一个池化操作，以减少空间大小，降低过度拟合。在这些层之上，在每一个卷积层后都要做一个 ReLU 操作，以增加网络的非线性特征。  
然而这个 Inception 原始版本，所有的卷积核都在上一层的所有输出上来做，而那个 5x5 的卷积核所需的计算量就太大了，造成了特征图的厚度很大，为了避免这种情况，在 3x3 前、5x5 前、max pooling 后分别加上了 1x1 的卷积核，以起到了降低特征图厚度的作用，这也就形成了 Inception v1 的网络结构，如下图所示：

![](https://static.oschina.net/uploads/space/2018/0317/141520_31TH_876354.png)

**1x1 的卷积核有什么用呢？**  
1x1 卷积的主要目的是为了减少维度，还用于修正线性激活（ReLU）。比如，上一层的输出为 100x100x128，经过具有 256 个通道的 5x5 卷积层之后 (stride=1，pad=2)，输出数据为 100x100x256，其中，卷积层的参数为 128x5x5x256= 819200。而假如上一层输出先经过具有 32 个通道的 1x1 卷积层，再经过具有 256 个输出的 5x5 卷积层，那么输出数据仍为为 100x100x256，但卷积参数量已经减少为 128x1x1x32 + 32x5x5x256= 204800，大约减少了 4 倍。

基于 Inception 构建了 GoogLeNet 的网络结构如下（共 22 层）：

![](https://static.oschina.net/uploads/space/2018/0317/141544_FfKB_876354.jpg)

对上图说明如下：  
（1）GoogLeNet 采用了模块化的结构（Inception 结构），方便增添和修改；  
（2）网络最后采用了 average pooling（平均池化）来代替全连接层，该想法来自 NIN（Network in Network），事实证明这样可以将准确率提高 0.6%。但是，实际在最后还是加了一个全连接层，主要是为了方便对输出进行灵活调整；  
（3）虽然移除了全连接，但是网络中依然使用了 Dropout ;   
（4）为了避免梯度消失，网络额外增加了 2 个辅助的 softmax 用于向前传导梯度（辅助分类器）。辅助分类器是将中间某一层的输出用作分类，并按一个较小的权重（0.3）加到最终分类结果中，这样相当于做了模型融合，同时给网络增加了反向传播的梯度信号，也提供了额外的正则化，对于整个网络的训练很有裨益。而在实际测试的时候，这两个额外的 softmax 会被去掉。

GoogLeNet 的网络结构图细节如下：  
![](https://static.oschina.net/uploads/space/2018/0317/141605_c1XW_876354.png)
   
注：上表中的 “#3x3 reduce”，“#5x5 reduce” 表示在 3x3，5x5 卷积操作之前使用了 1x1 卷积的数量。

GoogLeNet 网络结构明细表解析如下：  
**0、输入**  
原始输入图像为 224x224x3，且都进行了零均值化的预处理操作（图像每个像素减去均值）。  
**1、第一层（卷积层）**  
使用 7x7 的卷积核（滑动步长 2，padding 为 3），64 通道，输出为 112x112x64，卷积后进行 ReLU 操作  
经过 3x3 的 max pooling（步长为 2），输出为 ((112 - 3+1)/2)+1=56，即 56x56x64，再进行 ReLU 操作  
**2、第二层（卷积层）**  
使用 3x3 的卷积核（滑动步长为 1，padding 为 1），192 通道，输出为 56x56x192，卷积后进行 ReLU 操作  
经过 3x3 的 max pooling（步长为 2），输出为 ((56 - 3+1)/2)+1=28，即 28x28x192，再进行 ReLU 操作  
**3a、第三层（Inception 3a 层）**  
分为四个分支，采用不同尺度的卷积核来进行处理  
（1）64 个 1x1 的卷积核，然后 RuLU，输出 28x28x64  
（2）96 个 1x1 的卷积核，作为 3x3 卷积核之前的降维，变成 28x28x96，然后进行 ReLU 计算，再进行 128 个 3x3 的卷积（padding 为 1），输出 28x28x128  
（3）16 个 1x1 的卷积核，作为 5x5 卷积核之前的降维，变成 28x28x16，进行 ReLU 计算后，再进行 32 个 5x5 的卷积（padding 为 2），输出 28x28x32  
（4）pool 层，使用 3x3 的核（padding 为 1），输出 28x28x192，然后进行 32 个 1x1 的卷积，输出 28x28x32。  
将四个结果进行连接，对这四部分输出结果的第三维并联，即 64+128+32+32=256，最终输出 28x28x256  
**3b、第三层（Inception 3b 层）**  
（1）128 个 1x1 的卷积核，然后 RuLU，输出 28x28x128  
（2）128 个 1x1 的卷积核，作为 3x3 卷积核之前的降维，变成 28x28x128，进行 ReLU，再进行 192 个 3x3 的卷积（padding 为 1），输出 28x28x192  
（3）32 个 1x1 的卷积核，作为 5x5 卷积核之前的降维，变成 28x28x32，进行 ReLU 计算后，再进行 96 个 5x5 的卷积（padding 为 2），输出 28x28x96  
（4）pool 层，使用 3x3 的核（padding 为 1），输出 28x28x256，然后进行 64 个 1x1 的卷积，输出 28x28x64。  
将四个结果进行连接，对这四部分输出结果的第三维并联，即 128+192+96+64=480，最终输出输出为 28x28x480

第四层（4a,4b,4c,4d,4e）、第五层（5a,5b）……，与 3a、3b 类似，在此就不再重复。

从 GoogLeNet 的实验结果来看，效果很明显，差错率比 MSRA、VGG 等模型都要低，对比结果如下表所示：  
![](https://static.oschina.net/uploads/space/2018/0317/141641_H28K_876354.png)

**二、Inception V2**  
GoogLeNet 凭借其优秀的表现，得到了很多研究人员的学习和使用，因此 GoogLeNet 团队又对其进行了进一步地发掘改进，产生了升级版本的 GoogLeNet。  
GoogLeNet 设计的初衷就是要又准又快，而如果只是单纯的堆叠网络虽然可以提高准确率，但是会导致计算效率有明显的下降，所以如何在不增加过多计算量的同时提高网络的表达能力就成为了一个问题。  
Inception V2 版本的解决方案就是修改 Inception 的内部计算逻辑，提出了比较特殊的 “卷积” 计算结构。

**1、卷积分解（Factorizing Convolutions）**  
大尺寸的卷积核可以带来更大的感受野，但也意味着会产生更多的参数，比如 5x5 卷积核的参数有 25 个，3x3 卷积核的参数有 9 个，前者是后者的 25/9=2.78 倍。因此，GoogLeNet 团队提出可以用 2 个连续的 3x3 卷积层组成的小网络来代替单个的 5x5 卷积层，即在保持感受野范围的同时又减少了参数量，如下图：  
![](https://static.oschina.net/uploads/space/2018/0317/141655_QXsH_876354.png)
   
那么这种替代方案会造成表达能力的下降吗？通过大量实验表明，并不会造成表达缺失。  
可以看出，大卷积核完全可以由一系列的 3x3 卷积核来替代，那能不能再分解得更小一点呢？GoogLeNet 团队考虑了 nx1 的卷积核，如下图所示，用 3 个 3x1 取代 3x3 卷积：  
![](https://static.oschina.net/uploads/space/2018/0317/141700_GV5l_876354.png)
   
因此，任意 nxn 的卷积都可以通过 1xn 卷积后接 nx1 卷积来替代。GoogLeNet 团队发现在网络的前期使用这种分解效果并不好，在中度大小的特征图（feature map）上使用效果才会更好（特征图大小建议在 12 到 20 之间）。

![](https://static.oschina.net/uploads/space/2018/0317/141713_bGpL_876354.png)

**2、降低特征图大小**  
一般情况下，如果想让图像缩小，可以有如下两种方式：  
![](https://static.oschina.net/uploads/space/2018/0317/141726_LQNh_876354.png)
   
先池化再作 Inception 卷积，或者先作 Inception 卷积再作池化。但是方法一（左图）先作 pooling（池化）会导致特征表示遇到瓶颈（特征缺失），方法二（右图）是正常的缩小，但计算量很大。为了同时保持特征表示且降低计算量，将网络结构改为下图，使用两个并行化的模块来降低计算量（卷积、池化并行执行，再进行合并）

![](https://static.oschina.net/uploads/space/2018/0317/141734_OEPA_876354.png)

使用 Inception V2 作改进版的 GoogLeNet，网络结构图如下：  
![](https://static.oschina.net/uploads/space/2018/0317/141739_flcW_876354.png)
   
注：上表中的 Figure 5 指没有进化的 Inception，Figure 6 是指小卷积版的 Inception（用 3x3 卷积核代替 5x5 卷积核），Figure 7 是指不对称版的 Inception（用 1xn、nx1 卷积核代替 nxn 卷积核）。

经实验，模型结果与旧的 GoogleNet 相比有较大提升，如下表所示：  
![](https://static.oschina.net/uploads/space/2018/0317/141746_Fw1D_876354.png)

**三、Inception V3**  
Inception V3 一个最重要的改进是分解（Factorization），将 7x7 分解成两个一维的卷积（1x7,7x1），3x3 也是一样（1x3,3x1），这样的好处，既可以加速计算，又可以将 1 个卷积拆成 2 个卷积，使得网络深度进一步增加，增加了网络的非线性（每增加一层都要进行 ReLU）。  
另外，网络输入从 224x224 变为了 299x299。

**四、Inception V4**  
Inception V4 研究了 Inception 模块与残差连接的结合。ResNet 结构大大地加深了网络深度，还极大地提升了训练速度，同时性能也有提升（ResNet 的技术原理介绍见本博客之前的文章：[大话深度残差网络 ResNet](https://my.oschina.net/u/876354/blog/1622896)）。  
Inception V4 主要利用残差连接（Residual Connection）来改进 V3 结构，得到 Inception-ResNet-v1，Inception-ResNet-v2，Inception-v4 网络。  
ResNet 的残差结构如下：  
![](https://static.oschina.net/uploads/space/2018/0317/141800_uS67_876354.png)
   
将该结构与 Inception 相结合，变成下图：  
![](https://static.oschina.net/uploads/space/2018/0317/141810_oD01_876354.png)
   
通过 20 个类似的模块组合，Inception-ResNet 构建如下：

![](https://static.oschina.net/uploads/space/2018/0317/141818_MrSL_876354.png)

**墙裂建议**

2014 至 2016 年，GoogLeNet 团队发表了多篇关于 GoogLeNet 的经典论文《Going deeper with convolutions》、《Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift》、《Rethinking the Inception Architecture for Computer Vision》、《Inception-v4, Inception-ResNet and the Impact of Residual Connections on Learning》，在这些论文中对 Inception v1、Inception v2、Inception v3、Inception v4 等思想和技术原理进行了详细的介绍，建议阅读这些论文以全面了解 GoogLeNet。

关注本人公众号 “大数据与人工智能 Lab”（BigdataAILab），然后回复 “**论文**” 关键字可在线阅读这篇经典论文的内容**。** 

![](https://static.oschina.net/uploads/space/2018/0213/155533_IdYn_876354.jpg)

**推荐相关阅读**

*   [大话卷积神经网络（CNN）](https://my.oschina.net/u/876354/blog/1620906)
*   [大话循环神经网络（RNN）](https://my.oschina.net/u/876354/blog/1621839)
*   [大话深度残差网络（DRN）](https://my.oschina.net/u/876354/blog/1622896)
*   [大话深度信念网络（DBN）](https://my.oschina.net/u/876354/blog/1626639)
*   [大话 CNN 经典模型：LeNet](https://my.oschina.net/u/876354/blog/1632862)
*   [大话 CNN 经典模型：AlexNet](https://my.oschina.net/u/876354/blog/1633143)
*   [大话 CNN 经典模型：VGGNet](https://my.oschina.net/u/876354/blog/1634322)
*   [浅说 “迁移学习”](https://my.oschina.net/u/876354/blog/1614883)
*   [什么是 “强化学习”](https://my.oschina.net/u/876354/blog/1614879)
*   [AlphaGo 算法原理浅析](https://my.oschina.net/u/876354/blog/1594849)
*   [大数据究竟有多少个 V](https://my.oschina.net/u/876354/blog/1604254)
*   [Apache Hadoop 2.8 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/993836)
*   [Apache Hive 2.1.1 安装配置超详细教程](https://my.oschina.net/u/876354/blog/1057639)
*   [Apache HBase 1.2.6 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/1163018)
*   [离线安装 Cloudera Manager 5 和 CDH5（最新版 5.13.0）超详细教程](https://my.oschina.net/u/876354/blog/1605320)