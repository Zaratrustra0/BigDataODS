# 大话目标检测经典模型（RCNN、Fast RCNN、Faster RCNN）
![](https://static.oschina.net/uploads/space/2018/0331/144234_Dip5_876354.png)

目标检测是深度学习的一个重要应用，就是在图片中要将里面的物体识别出来，并标出物体的位置，一般需要经过两个步骤：  
1、分类，识别物体是什么  
![](https://static.oschina.net/uploads/space/2018/0331/144252_c4bd_876354.png)
   
2、定位，找出物体在哪里  
![](https://static.oschina.net/uploads/space/2018/0331/144256_1bbr_876354.png)
   
除了对单个物体进行检测，还要能支持对多个物体进行检测，如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0331/144302_eZmG_876354.png)
   
这个问题并不是那么容易解决，由于物体的尺寸变化范围很大、摆放角度多变、姿态不定，而且物体有很多种类别，可以在图片中出现多种物体、出现在任意位置。因此，目标检测是一个比较复杂的问题。  
最直接的方法便是构建一个深度神经网络，将图像和标注位置作为样本输入，然后经过 CNN 网络，再通过一个分类头（Classification head）的全连接层识别是什么物体，通过一个回归头（Regression head）的全连接层回归计算位置，如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0331/144308_Raws_876354.png)
   
但 “回归” 不好做，计算量太大、收敛时间太长，应该想办法转为 “分类”，这时容易想到套框的思路，即取不同大小的 “框”，让框出现在不同的位置，计算出这个框的得分，然后取得分最高的那个框作为预测结果，如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0331/144314_Sgr2_876354.png)
   
根据上面比较出来的得分高低，选择了右下角的黑框作为目标位置的预测。

但问题是：框要取多大才合适？太小，物体识别不完整；太大，识别结果多了很多其它信息。那怎么办？那就各种大小的框都取来计算吧。

如下图所示（要识别一只熊），用各种大小的框在图片中进行反复截取，输入到 CNN 中识别计算得分，最终确定出目标类别和位置。  
![](https://static.oschina.net/uploads/space/2018/0331/144327_aLBj_876354.png)
   
这种方法效率很低，实在太耗时了。那有没有高效的目标检测方法呢？

**一、R-CNN 横空出世**  
R-CNN（Region CNN，区域卷积神经网络）可以说是利用深度学习进行目标检测的开山之作，作者 Ross Girshick 多次在 PASCAL VOC 的目标检测竞赛中折桂，2010 年更是带领团队获得了终身成就奖，如今就职于 Facebook 的人工智能实验室（FAIR）。

R-CNN 算法的流程如下  
![](https://static.oschina.net/uploads/space/2018/0331/144342_iY9y_876354.png)
   
1、输入图像  
2、每张图像生成 1K~2K 个候选区域  
3、对每个候选区域，使用深度网络提取特征（AlextNet、VGG 等 CNN 都可以）  
4、将特征送入每一类的 SVM 分类器，判别是否属于该类  
5、使用回归器精细修正候选框位置

下面展开进行介绍  
**1、生成候选区域**  
使用 Selective Search（选择性搜索）方法对一张图像生成约 2000-3000 个候选区域，基本思路如下：  
（1）使用一种过分割手段，将图像分割成小区域  
（2）查看现有小区域，合并可能性最高的两个区域，重复直到整张图像合并成一个区域位置。优先合并以下区域：  
\- 颜色（颜色直方图）相近的  
\- 纹理（梯度直方图）相近的  
\- 合并后总面积小的  
\- 合并后，总面积在其 BBOX 中所占比例大的  
在合并时须保证合并操作的尺度较为均匀，避免一个大区域陆续 “吃掉” 其它小区域，保证合并后形状规则。  
（3）输出所有曾经存在过的区域，即所谓候选区域  
**2、特征提取**  
使用深度网络提取特征之前，首先把候选区域归一化成同一尺寸 227×227。  
使用 CNN 模型进行训练，例如 AlexNet，一般会略作简化，如下图：  
![](https://static.oschina.net/uploads/space/2018/0331/144354_qJvD_876354.png)
   
**3、类别判断**  
对每一类目标，使用一个线性 SVM 二类分类器进行判别。输入为深度网络（如上图的 AlexNet）输出的 4096 维特征，输出是否属于此类。  
**4、位置精修**  
目标检测的衡量标准是重叠面积：许多看似准确的检测结果，往往因为候选框不够准确，重叠面积很小，故需要一个位置精修步骤，对于每一个类，训练一个线性回归模型去判定这个框是否框得完美，如下图：  
![](https://static.oschina.net/uploads/space/2018/0331/144403_IPBp_876354.png)
   
R-CNN 将深度学习引入检测领域后，一举将 PASCAL VOC 上的检测率从 35.1% 提升到 53.7%。

**二、Fast R-CNN 大幅提速**  
继 2014 年的 R-CNN 推出之后，Ross Girshick 在 2015 年推出 Fast R-CNN，构思精巧，流程更为紧凑，大幅提升了目标检测的速度。  
Fast R-CNN 和 R-CNN 相比，训练时间从 84 小时减少到 9.5 小时，测试时间从 47 秒减少到 0.32 秒，并且在 PASCAL VOC 2007 上测试的准确率相差无几，约在 66%-67% 之间。  
![](https://static.oschina.net/uploads/space/2018/0331/144411_NCm0_876354.png)
   
Fast R-CNN 主要解决 R-CNN 的以下问题：  
1、训练、测试时速度慢  
R-CNN 的一张图像内候选框之间存在大量重叠，提取特征操作冗余。而 Fast R-CNN 将整张图像归一化后直接送入深度网络，紧接着送入从这幅图像上提取出的候选区域。这些候选区域的前几层特征不需要再重复计算。  
2、训练所需空间大  
R-CNN 中独立的分类器和回归器需要大量特征作为训练样本。Fast R-CNN 把类别判断和位置精调统一用深度网络实现，不再需要额外存储。

下面进行详细介绍  
**1、在特征提取阶段，**通过 CNN（如 AlexNet）中的 conv、pooling、relu 等操作都不需要固定大小尺寸的输入，因此，在原始图片上执行这些操作后，输入图片尺寸不同将会导致得到的 feature map（特征图）尺寸也不同，这样就不能直接接到一个全连接层进行分类。  
在 Fast R-CNN 中，作者提出了一个叫做 ROI Pooling 的网络层，这个网络层可以把不同大小的输入映射到一个固定尺度的特征向量。ROI Pooling 层将每个候选区域均匀分成 M×N 块，对每块进行 max pooling。将特征图上大小不一的候选区域转变为大小统一的数据，送入下一层。这样虽然输入的图片尺寸不同，得到的 feature map（特征图）尺寸也不同，但是可以加入这个神奇的 ROI Pooling 层，对每个 region 都提取一个固定维度的特征表示，就可再通过正常的 softmax 进行类型识别。  
![](https://static.oschina.net/uploads/space/2018/0331/144419_hVfy_876354.png)

**2、在分类回归阶段，**在 R-CNN 中，先生成候选框，然后再通过 CNN 提取特征，之后再用 SVM 分类，最后再做回归得到具体位置（bbox regression）。而在 Fast R-CNN 中，作者巧妙的把最后的 bbox regression 也放进了神经网络内部，与区域分类合并成为了一个 multi-task 模型，如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0331/144431_NbDC_876354.png)
   
实验表明，这两个任务能够共享卷积特征，并且相互促进。

Fast R-CNN 很重要的一个贡献是成功地让人们看到了 Region Proposal+CNN（候选区域 + 卷积神经网络）这一框架实时检测的希望，原来多类检测真的可以在保证准确率的同时提升处理速度。

**三、Faster R-CNN 更快更强**  
继 2014 年推出 R-CNN，2015 年推出 Fast R-CNN 之后，目标检测界的领军人物 Ross Girshick 团队在 2015 年又推出一力作：Faster R-CNN，使简单网络目标检测速度达到 17fps，在 PASCAL VOC 上准确率为 59.9%，复杂网络达到 5fps，准确率 78.8%。  
在 Fast R-CNN 还存在着瓶颈问题：Selective Search（选择性搜索）。要找出所有的候选框，这个也非常耗时。那我们有没有一个更加高效的方法来求出这些候选框呢？  
在 Faster R-CNN 中加入一个提取边缘的神经网络，也就说找候选框的工作也交给神经网络来做了。这样，目标检测的四个基本步骤（候选区域生成，特征提取，分类，位置精修）终于被统一到一个深度网络框架之内。如下图所示：  
![](https://static.oschina.net/uploads/space/2018/0331/144441_YVAg_876354.png)
   
Faster R-CNN 可以简单地看成是 “区域生成网络 + Fast R-CNN” 的模型，用区域生成网络（Region Proposal Network，简称 RPN）来代替 Fast R-CNN 中的 Selective Search（选择性搜索）方法。  
如下图  
![](https://static.oschina.net/uploads/space/2018/0331/144447_Pe7l_876354.png)
   
RPN 如下图：  
![](https://static.oschina.net/uploads/space/2018/0331/144456_Vava_876354.png)
   
RPN 的工作步骤如下：  
\- 在 feature map（特征图）上滑动窗口  
\- 建一个神经网络用于物体分类 + 框位置的回归  
\- 滑动窗口的位置提供了物体的大体位置信息  
\- 框的回归提供了框更精确的位置

Faster R-CNN 设计了提取候选区域的网络 RPN，代替了费时的 Selective Search（选择性搜索），使得检测速度大幅提升，下表对比了 R-CNN、Fast R-CNN、Faster R-CNN 的检测速度：  
![](https://static.oschina.net/uploads/space/2018/0331/144503_nUf1_876354.png)

**总结**  
R-CNN、Fast R-CNN、Faster R-CNN 一路走来，基于深度学习目标检测的流程变得越来越精简、精度越来越高、速度也越来越快。基于 region proposal（候选区域）的 R-CNN 系列目标检测方法是目标检测技术领域中的最主要分支之一。

**墙裂建议**

2014 至 2016 年，Ross Girshick 等人发表了关于 R-CNN、Fast R-CNN、Faster R-CNN 的经典论文《Rich feature hierarchies for accurate object detection and semantic segmentation》、《Fast R-CNN》、《Faster R-CNN: Towards Real-Time ObjectDetection with Region Proposal Networks》，在这些论文中对目标检测的思想、原理、测试情况进行了详细介绍，建议阅读些篇论文以全面了解目标检测模型。

关注本人公众号 “大数据与人工智能 Lab”（BigdataAILab），然后回复 “**论文**” 关键字可在线阅读经典论文的内容**。** 

![](https://static.oschina.net/uploads/space/2018/0213/155533_IdYn_876354.jpg)

**推荐相关阅读**

*   [大话卷积神经网络（CNN）](https://my.oschina.net/u/876354/blog/1620906)
*   [大话循环神经网络（RNN）](https://my.oschina.net/u/876354/blog/1621839)
*   [大话深度残差网络（DRN）](https://my.oschina.net/u/876354/blog/1622896)
*   [大话深度信念网络（DBN）](https://my.oschina.net/u/876354/blog/1626639)
*   [大话 CNN 经典模型：LeNet](https://my.oschina.net/u/876354/blog/1632862)
*   [大话 CNN 经典模型：AlexNet](https://my.oschina.net/u/876354/blog/1633143)
*   [大话 CNN 经典模型：VGGNet](https://my.oschina.net/u/876354/blog/1634322)
*   [大话 CNN 经典模型：GoogLeNet](https://my.oschina.net/u/876354/blog/1637819)
*   [浅说 “迁移学习”](https://my.oschina.net/u/876354/blog/1614883)
*   [什么是 “强化学习”](https://my.oschina.net/u/876354/blog/1614879)
*   [AlphaGo 算法原理浅析](https://my.oschina.net/u/876354/blog/1594849)
*   [大数据究竟有多少个 V](https://my.oschina.net/u/876354/blog/1604254)
*   [Apache Hadoop 2.8 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/993836)
*   [Apache Hive 2.1.1 安装配置超详细教程](https://my.oschina.net/u/876354/blog/1057639)
*   [Apache HBase 1.2.6 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/1163018)
*   [离线安装 Cloudera Manager 5 和 CDH5（最新版 5.13.0）超详细教程](https://my.oschina.net/u/876354/blog/1605320)