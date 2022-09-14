# 大话CNN经典模型：VGGNet
_—— 原文发布于本人的微信公众号 “大数据与人工智能 Lab”（BigdataAILab），欢迎关注。_

![](https://static.oschina.net/uploads/space/2018/0314/022831_wOGN_876354.png)
   
2014 年，牛津大学计算机视觉组（Visual Geometry Group）和 Google DeepMind 公司的研究员一起研发出了新的深度卷积神经网络：VGGNet，并取得了 ILSVRC2014 比赛分类项目的第二名（第一名是 GoogLeNet，也是同年提出的）和定位项目的第一名。  
VGGNet 探索了卷积神经网络的深度与其性能之间的关系，成功地构筑了 16~19 层深的卷积神经网络，证明了增加网络的深度能够在一定程度上影响网络最终的性能，使错误率大幅下降，同时拓展性又很强，迁移到其它图片数据上的泛化性也非常好。到目前为止，VGG 仍然被用来提取图像特征。  
VGGNet 可以看成是加深版本的 AlexNet，都是由卷积层、全连接层两大部分构成。卷积神经网络技术原理、AlexNet 在本博客前面的文章已经有作了详细的介绍，有兴趣的同学可打开链接看看（[大话卷积神经网络](https://my.oschina.net/u/876354/blog/1620906)，[大话 CNN 经典模型：AlexNet](https://my.oschina.net/u/876354/blog/1633143)）。

**一、VGG 的特点**  
先看一下 VGG 的结构图  
![](https://static.oschina.net/uploads/space/2018/0314/022939_Pl12_876354.png)
   
**1、结构简洁**  
VGG 由 5 层卷积层、3 层全连接层、softmax 输出层构成，层与层之间使用 max-pooling（最大化池）分开，所有隐层的激活单元都采用 ReLU 函数。  
**2、小卷积核和多卷积子层**  
VGG 使用多个较小卷积核（3x3）的卷积层代替一个卷积核较大的卷积层，一方面可以减少参数，另一方面相当于进行了更多的非线性映射，可以增加网络的拟合 / 表达能力。  
小卷积核是 VGG 的一个重要特点，虽然 VGG 是在模仿 AlexNet 的网络结构，但没有采用 AlexNet 中比较大的卷积核尺寸（如 7x7），而是通过降低卷积核的大小（3x3），增加卷积子层数来达到同样的性能（VGG：从 1 到 4 卷积子层，AlexNet：1 子层）。  
VGG 的作者认为两个 3x3 的卷积堆叠获得的感受野大小，相当一个 5x5 的卷积；而 3 个 3x3 卷积的堆叠获取到的感受野相当于一个 7x7 的卷积。这样可以增加非线性映射，也能很好地减少参数（例如 7x7 的参数为 49 个，而 3 个 3x3 的参数为 27），如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0314/022956_LO3Y_876354.png)
   
**3、小池化核**  
相比 AlexNet 的 3x3 的池化核，VGG 全部采用 2x2 的池化核。  
**4、通道数多**  
VGG 网络第一层的通道数为 64，后面每层都进行了翻倍，最多到 512 个通道，通道数的增加，使得更多的信息可以被提取出来。  
**5、层数更深、特征图更宽**  
由于卷积核专注于扩大通道数、池化专注于缩小宽和高，使得模型架构上更深更宽的同时，控制了计算量的增加规模。  
**6、全连接转卷积（测试阶段）**  
这也是 VGG 的一个特点，在网络测试阶段将训练阶段的三个全连接替换为三个卷积，使得测试得到的全卷积网络因为没有全连接的限制，因而可以接收任意宽或高为的输入，这在测试阶段很重要。  
如本节第一个图所示，输入图像是 224x224x3，如果后面三个层都是全连接，那么在测试阶段就只能将测试的图像全部都要缩放大小到 224x224x3，才能符合后面全连接层的输入数量要求，这样就不便于测试工作的开展。  
而 “全连接转卷积”，替换过程如下：  
![](https://static.oschina.net/uploads/space/2018/0314/023015_rDZR_876354.png)
   
例如 7x7x512 的层要跟 4096 个神经元的层做全连接，则替换为对 7x7x512 的层作通道数为 4096、卷积核为 1x1 的卷积。  
这个 “全连接转卷积” 的思路是 VGG 作者参考了 OverFeat 的工作思路，例如下图是 OverFeat 将全连接换成卷积后，则可以来处理任意分辨率（在整张图）上计算卷积，这就是无需对原图做重新缩放处理的优势。

![](https://static.oschina.net/uploads/space/2018/0314/023026_99ma_876354.png)

**二、VGG 的网络结构**  
下图是来自论文《Very Deep Convolutional Networks for Large-Scale Image Recognition》（基于甚深层卷积网络的大规模图像识别）的 VGG 网络结构，正是在这篇论文中提出了 VGG，如下图：  
![](https://static.oschina.net/uploads/space/2018/0314/023044_X49R_876354.png)
   
在这篇论文中分别使用了 A、A-LRN、B、C、D、E 这 6 种网络结构进行测试，这 6 种网络结构相似，都是由 5 层卷积层、3 层全连接层组成，其中区别在于每个卷积层的子层数量不同，从 A 至 E 依次增加（子层数量从 1 到 4），总的网络深度从 11 层到 19 层（添加的层以粗体显示），表格中的卷积层参数表示为 “conv⟨感受野大小⟩- 通道数⟩”，例如 con3-128，表示使用 3x3 的卷积核，通道数为 128。为了简洁起见，在表格中不显示 ReLU 激活功能。  
其中，网络结构 D 就是著名的 VGG16，网络结构 E 就是著名的 VGG19。

以网络结构 D（VGG16）为例，介绍其处理过程如下，请对比上面的表格和下方这张图，留意图中的数字变化，有助于理解 VGG16 的处理过程：  
![](https://static.oschina.net/uploads/space/2018/0314/023055_OLcQ_876354.png)
   
1、输入 224x224x3 的图片，经 64 个 3x3 的卷积核作两次卷积 + ReLU，卷积后的尺寸变为 224x224x64  
2、作 max pooling（最大化池化），池化单元尺寸为 2x2（效果为图像尺寸减半），池化后的尺寸变为 112x112x64  
3、经 128 个 3x3 的卷积核作两次卷积 + ReLU，尺寸变为 112x112x128  
4、作 2x2 的 max pooling 池化，尺寸变为 56x56x128  
5、经 256 个 3x3 的卷积核作三次卷积 + ReLU，尺寸变为 56x56x256  
6、作 2x2 的 max pooling 池化，尺寸变为 28x28x256  
7、经 512 个 3x3 的卷积核作三次卷积 + ReLU，尺寸变为 28x28x512  
8、作 2x2 的 max pooling 池化，尺寸变为 14x14x512  
9、经 512 个 3x3 的卷积核作三次卷积 + ReLU，尺寸变为 14x14x512  
10、作 2x2 的 max pooling 池化，尺寸变为 7x7x512  
11、与两层 1x1x4096，一层 1x1x1000 进行全连接 + ReLU（共三层）  
12、通过 softmax 输出 1000 个预测结果

以上就是 VGG16（网络结构 D）各层的处理过程，A、A-LRN、B、C、E 其它网络结构的处理过程也是类似，执行过程如下（以 VGG16 为例）：  
![](https://static.oschina.net/uploads/space/2018/0314/023103_ME9y_876354.png)
   
从上面的过程可以看出 VGG 网络结构还是挺简洁的，都是由小卷积核、小池化核、ReLU 组合而成。其简化图如下（以 VGG16 为例）：

![](https://static.oschina.net/uploads/space/2018/0314/023111_GG9k_876354.png)

A、A-LRN、B、C、D、E 这 6 种网络结构的深度虽然从 11 层增加至 19 层，但参数量变化不大，这是由于基本上都是采用了小卷积核（3x3，只有 9 个参数），这 6 种结构的参数数量（百万级）并未发生太大变化，这是因为在网络中，参数主要集中在全连接层。  
![](https://static.oschina.net/uploads/space/2018/0314/023129_ppE7_876354.png)
   
经作者对 A、A-LRN、B、C、D、E 这 6 种网络结构进行单尺度的评估，错误率结果如下：  
![](https://static.oschina.net/uploads/space/2018/0314/023135_0OwO_876354.png)
   
从上表可以看出：  
**1、LRN 层无性能增益（A-LRN）**  
VGG 作者通过网络 A-LRN 发现，AlexNet 曾经用到的 LRN 层（local response normalization，局部响应归一化）并没有带来性能的提升，因此在其它组的网络中均没再出现 LRN 层。  
**2、随着深度增加，分类性能逐渐提高（A、B、C、D、E）**  
从 11 层的 A 到 19 层的 E，网络深度增加对 top1 和 top5 的错误率下降很明显。  
**3、多个小卷积核比单个大卷积核性能好（B）**  
VGG 作者做了实验用 B 和自己一个不在实验组里的较浅网络比较，较浅网络用 conv5x5 来代替 B 的两个 conv3x3，结果显示多个小卷积核比单个大卷积核效果要好。

最后进行个小结：  
**1、通过增加深度能有效地提升性能；  
2、最佳模型：VGG16，从头到尾只有 3x3 卷积与 2x2 池化，简洁优美；  
3、卷积可代替全连接，可适应各种尺寸的图片**

**墙裂建议**

2014 年，K. Simonyan 等人发表了关于 VGGNet 的经典论文《Very Deep Convolutional Networks for Large-Scale Image Recognition》（基于甚深层卷积网络的大规模图像识别），在该论文中对 VGG 的思想、测试情况进行了详细介绍，建议阅读这篇论文加深了解。

扫描以下二维码关注本人公众号 “大数据与人工智能 Lab”（BigdataAILab），然后回复 “**论文**” 关键字可在线阅读这篇经典论文的内容。

![](https://static.oschina.net/uploads/space/2018/0213/155533_IdYn_876354.jpg)

**推荐相关阅读**

*   [大话卷积神经网络（CNN）](https://my.oschina.net/u/876354/blog/1620906)
*   [大话循环神经网络（RNN）](https://my.oschina.net/u/876354/blog/1621839)
*   [大话深度残差网络（DRN）](https://my.oschina.net/u/876354/blog/1622896)
*   [大话深度信念网络（DBN）](https://my.oschina.net/u/876354/blog/1626639)
*   [大话 CNN 经典模型：LeNet](https://my.oschina.net/u/876354/blog/1632862)
*   [大话 CNN 经典模型：AlexNet](https://my.oschina.net/u/876354/blog/1633143)
*   [浅说 “迁移学习”](https://my.oschina.net/u/876354/blog/1614883)
*   [什么是 “强化学习”](https://my.oschina.net/u/876354/blog/1614879)
*   [AlphaGo 算法原理浅析](https://my.oschina.net/u/876354/blog/1594849)
*   [大数据究竟有多少个 V](https://my.oschina.net/u/876354/blog/1604254)
*   [Apache Hadoop 2.8 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/993836)
*   [Apache Hive 2.1.1 安装配置超详细教程](https://my.oschina.net/u/876354/blog/1057639)
*   [Apache HBase 1.2.6 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/1163018)
*   [离线安装 Cloudera Manager 5 和 CDH5（最新版 5.13.0）超详细教程](https://my.oschina.net/u/876354/blog/1605320)