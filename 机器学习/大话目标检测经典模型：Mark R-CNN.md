# 大话目标检测经典模型：Mark R-CNN
![](https://static.oschina.net/uploads/space/2018/0428/135246_VdTi_876354.png)

在之前的文章中介绍了[目标检测经典模型（R-CNN、Fast R-CNN、Faster R-CNN）](https://my.oschina.net/u/876354/blog/1787921)，目标检测一般是为了实现以下效果：  
![](https://static.oschina.net/uploads/space/2018/0428/135256_Z0DL_876354.png)
   
在 R-CNN、Fast R-CNN、Faster R-CNN 中，实现了对目标的识别和定位，如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0428/135302_LmjM_876354.png)
   
为了更加精确地识别目标，实现在像素级场景中识别不同目标，利用 “图像分割” 技术定位每个目标的精确像素，如下图所示（精确分割出人、汽车、红绿灯等）：  
![](https://static.oschina.net/uploads/space/2018/0428/135309_i37X_876354.png)
   
Mask R-CNN 便是这种 “图像分割” 的重要模型。

Mask R-CNN 的思路很简洁，既然 Faster R-CNN 目标检测的效果非常好，每个候选区域能输出种类标签和定位信息，那么就在 Faster R-CNN 的基础上再添加一个分支从而增加一个输出，即物体掩膜（object mask），也即由原来的两个任务（分类 + 回归）变为了三个任务（分类 + 回归 + 分割）。如下图所示，Mask R-CNN 由两条分支组成：  
![](https://static.oschina.net/uploads/space/2018/0428/135324_fo5L_876354.png)
   
Mask R-CNN 的这两个分支是并行的，因此训练简单，仅比 Faster R-CNN 多了一点计算开销。  
分类和定位在 Faster R-CNN 中有介绍过了（详见文章：[大话目标检测经典模型 RCNN、Fast RCNN、Faster RCNN](https://my.oschina.net/u/876354/blog/1787921)），在此就不再重复介绍，下面重点介绍一下第二条分支，即如何实现像素级的图像分割。

如下图所示，Mask R-CNN 在 Faster R-CNN 中添加了一个全卷积网络的分支（图中白色部分），用于输出二进制 mask，以说明给定像素是否是目标的一部分。所谓二进制 mask，就是当像素属于目标的所有位置上时标识为 1，其它位置标识为 0  
![](https://static.oschina.net/uploads/space/2018/0428/135331_IOGZ_876354.png)
   
从上图可以看出，二进制 mask 是基于特征图输出的，而原始图像经过一系列的卷积、池化之后，尺寸大小已发生了多次变化，如果直接使用特征图输出的二进制 mask 来分割图像，那肯定是不准的。这时就需要进行了修正，也即使用 RoIAlign 替换 RoIPooling  
![](https://static.oschina.net/uploads/space/2018/0428/135338_eB1d_876354.png)
   
如上图所示，原始图像尺寸大小是 128x128，经过卷积网络之后的特征图变为尺寸大小变为 25x25。这时，如果想要圈出与原始图像中左上方 15x15 像素对应的区域，那么如何在特征图中选择相对应的像素呢？  
从上面两张图可以看出，原始图像中的每个像素对应于特征图的 25/128 像素，因此，要从原始图像中选择 15x15 像素，则只需在特征图中选择 2.93x2.93 像素（15x25/128=2.93），在 RoIAlign 中会使用双线性插值法准确得到 2.93 像素的内容，这样就能很大程度上，避免了错位问题。  
修改后的网络结构如下图所示（黑色部分为原来的 Faster R-CNN，红色部分为 Mask R-CNN 修改的部分）  
![](https://static.oschina.net/uploads/space/2018/0428/135344_OMA5_876354.png)
   
从上图可以看出损失函数变为  
![](https://static.oschina.net/uploads/space/2018/0428/135350_ONwm_876354.png)
   
损失函数为分类误差 + 检测误差 + 分割误差，分类误差和检测（回归）误差是 Faster R-CNN 中的，分割误差为 Mask R-CNN 中新加的。  
对于每个 MxM 大小的 ROI 区域，mask 分支有 KxMxM 维的输出（K 是指类别数量）。对于每一个像素，都是用 sigmod 函数求二值交叉熵，也即对每个像素都进行逻辑回归，得到平均的二值交叉熵误差 Lmask。通过引入预测 K 个输出的机制，允许每个类都生成独立的 mask，以避免类间竞争，这样就能解耦 mask 和种类预测。  
对于每一个 ROI 区域，如果检测得到属于哪一个分类，就只使用该类的交叉熵误差进行计算，也即对于一个 ROI 区域中 KxMxM 的输出，真正有用的只是某个类别的 MxM 的输出。如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0428/135358_oAD1_876354.png)
   
例如目前有 3 个分类：猫、狗、人，检测得到当前 ROI 属于 “人” 这一类，那么所使用的 Lmask 为 “人” 这一分支的 mask。

Mask R-CNN 将这些二进制 mask 与来自 Faster R-CNN 的分类和边界框组合，便产生了惊人的图像精确分割，如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0428/135415_Ez1c_876354.png)

Mask R-CNN 是一个小巧、灵活的通用对象实例分割框架，它不仅可以对图像中的目标进行检测，还可以对每一个目标输出一个高质量的分割结果。另外，Mask R-CNN 还易于泛化到其他任务，比如人物关键点检测，如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0428/135533_2rQ1_876354.png)
  
从 R-CNN、Fast R-CNN、Faster R-CNN 到 Mask R-CNN，每次进步不一定是跨越式的发展，这些进步实际上是直观的且渐进的改进之路，但是它们的总和却带来了非常显著的效果。  
最后，总结一下目标检测算法模型的发展历程，如下图所示：

![](https://static.oschina.net/uploads/space/2018/0428/135513_UZ6v_876354.png)

**墙裂建议**

2017 年，Kaiming He 等人发表了关于 Mask R-CNN 的经典论文《Mask R-CNN》，在论文中详细介绍了 Mask R-CNN 的思想、原理和测试效果，建议阅读该论文以进一步了解该模型。

关注本人公众号 “大数据与人工智能 Lab”（BigdataAILab），然后回复 “**论文**” 关键字可在线阅读经典论文的内容**。** 

![](https://static.oschina.net/uploads/space/2018/0213/155533_IdYn_876354.jpg)

**推荐相关阅读**

*   [【精华整理】CNN 进化史](https://my.oschina.net/u/876354/blog/1797489)
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
*   [浅说 “迁移学习”](https://my.oschina.net/u/876354/blog/1614883)
*   [什么是 “强化学习”](https://my.oschina.net/u/876354/blog/1614879)
*   [AlphaGo 算法原理浅析](https://my.oschina.net/u/876354/blog/1594849)
*   [大数据究竟有多少个 V](https://my.oschina.net/u/876354/blog/1604254)
*   [Apache Hadoop 2.8 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/993836)
*   [Apache Hive 2.1.1 安装配置超详细教程](https://my.oschina.net/u/876354/blog/1057639)
*   [Apache HBase 1.2.6 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/1163018)
*   [离线安装 Cloudera Manager 5 和 CDH5（最新版 5.13.0）超详细教程](https://my.oschina.net/u/876354/blog/1605320)