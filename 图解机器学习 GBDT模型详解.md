# 图解机器学习 | GBDT模型详解
1.GBDT算法
--------

GBDT（Gradient Boosting Decision Tree），全名叫梯度提升决策树，是一种迭代的决策树算法，又叫 MART（Multiple Additive Regression Tree），它**通过构造一组弱的学习器（树），并把多颗决策树的结果累加起来作为最终的预测输出**。该算法将决策树与集成思想进行了有效的结合。

![](https://img-blog.csdnimg.cn/img_convert/cac8253efbd543d8281ca5207ce05bc7.png)

（本篇GBDT集成模型部分内容涉及到机器学习基础知识、决策树、回归树算法，没有先序知识储备的宝宝可以查看ShowMeAI的文章 [图解机器学习 | **机器学习基础知识**](https://www.showmeai.tech/article-detail/185) 、[**决策树模型详解**](https://www.showmeai.tech/article-detail/190) 及 [**回归树模型详解**](https://www.showmeai.tech/article-detail/192)）。

### 1）Boosting核心思想

Boosting方法训练基分类器时采用串行的方式，各个基分类器之间有依赖。它的基本思路是将基分类器层层叠加，每一层在训练的时候，对前一层基分类器分错的样本，给予更高的权重。测试时，根据各层分类器的结果的加权得到最终结果。

![](https://img-blog.csdnimg.cn/img_convert/1b9e5f7f3f733bb0f6335829235e7a74.png)

Bagging 与 Boosting 的串行训练方式不同，Bagging 方法在训练过程中，各基分类器之间无强依赖，可以进行并行训练。

### 2）GBDT详解

**GBDT的原理很简单**：

*   所有弱分类器的结果相加等于预测值。
*   每次都以当前预测为基准，下一个弱分类器去拟合误差函数对预测值的残差（预测值与真实值之间的误差）。
*   GBDT的弱分类器使用的是树模型。

![](https://img-blog.csdnimg.cn/img_convert/8b35092a9eeca61fba14a1decd236d8b.png)

如图是一个非常简单的帮助理解的示例，我们用 GBDT 去预测年龄：

最终，四棵树的结论加起来，得到 30 岁这个标注答案（实际工程实现里，GBDT 是计算负梯度，用负梯度近似残差）。

#### （1）GBDT与负梯度近似残差

回归任务下，GBDT在每一轮的迭代时对每个样本都会有一个预测值，此时的损失函数为**均方差损失函数**：

![](https://www.zhihu.com/equation?tex=l%28y_%7Bi%7D%2C%20%5Chat%7By_%7Bi%7D%7D%29%3D%5Cfrac%7B1%7D%7B2%7D%28y_%7Bi%7D-%5Chat%7By_%7Bi%7D%7D%29%5E%7B2%7D)

损失函数的负梯度计算如下：

![](https://www.zhihu.com/equation?tex=-%5B%5Cfrac%7B%5Cpartial%20l%28y_%7Bi%7D%2C%20%5Chat%7By_%7Bi%7D%7D%29%7D%7B%5Cpartial%20%5Chat%7By_%7Bi%7D%7D%7D%5D%3D%28y_%7Bi%7D-%5Chat%7By_%7Bi%7D%7D%29)

![](https://img-blog.csdnimg.cn/img_convert/b93a9786c59ddf91945b3906f4b54707.png)

可以看出，当损失函数选用「**均方误差损失**」时，每一次拟合的值就是（真实值-预测值），即残差。

#### （2）GBDT训练过程

我们来借助1个简单的例子理解一下 GBDT 的训练过程。假定训练集只有 4 个人 ![](https://www.zhihu.com/equation?tex=%28A%2CB%2CC%2CD%29)，他们的年龄分别是 ![](https://www.zhihu.com/equation?tex=%2814%2C16%2C24%2C26%29)。其中，![](https://www.zhihu.com/equation?tex=A)、![](https://www.zhihu.com/equation?tex=B)分别是高一和高三学生；![](https://www.zhihu.com/equation?tex=C)、![](https://www.zhihu.com/equation?tex=D)分别是应届毕业生和工作两年的员工。

我们先看看用回归树来训练，得到的结果如下图所示：

![](https://img-blog.csdnimg.cn/img_convert/cfc752496222574423c4618259e649cd.png)

接下来改用 GBDT 来训练。由于样本数据少，我们限定叶子节点最多为 2 （即每棵树都只有一个分枝），并且限定树的棵树为 2 。最终训练得到的结果如下图所示：

![](https://img-blog.csdnimg.cn/img_convert/8e2ea81d16336b947df58cb21d5ccca7.png)

上图中的树很好理解：![](https://www.zhihu.com/equation?tex=A)、![](https://www.zhihu.com/equation?tex=B)年龄较为相近，![](https://www.zhihu.com/equation?tex=C)、![](https://www.zhihu.com/equation?tex=D)年龄较为相近，被分为左右两支，每支用平均年龄作为预测值。

![](https://img-blog.csdnimg.cn/img_convert/d27728269b48f6e51d30a29a6fe430eb.png)

上图中的树就是残差学习的过程了：

![](https://img-blog.csdnimg.cn/img_convert/cb84db1cc65145fda5f1227611e03277.png)

最终的预测过程是这样的：

综上，GBDT 需要将多棵树的得分累加得到最终的预测得分，且每轮迭代，都是在现有树的基础上，增加一棵新的树去拟合前面树的预测值与真实值之间的残差。

2.梯度提升 vs 梯度下降
--------------

下面我们来对比一下「梯度提升」与「梯度下降」。这两种迭代优化算法，都是在每1轮迭代中，利用损失函数负梯度方向的信息，更新当前模型，只不过：

*   梯度下降中，模型是以参数化形式表示，从而模型的更新等价于参数的更新。
    
    ![](https://www.zhihu.com/equation?tex=F%3DF_%7Bt-1%7D-%5Cleft.%5Crho_%7Bt%7D%20%5Cnabla_%7BF%7D%20L%5Cright%7C_%7BF%3DF_%7Bt-1%7D%7D)
    
      
    
    ![](https://www.zhihu.com/equation?tex=L%3D%5Csum_%7Bi%7D%20l%5Cleft%28y_%7Bi%7D%2C%20F%5Cleft%28x_%7Bi%7D%5Cright%29%5Cright%29)
    
*   梯度提升中，模型并不需要进行参数化表示，而是直接定义在函数空间中，从而大大扩展了可以使用的模型种类。
    
    ![](https://www.zhihu.com/equation?tex=w_%7Bt%7D%3Dw_%7Bt-1%7D-%5Cleft.%5Crho_%7Bt%7D%20%5Cnabla_%7Bw%7D%20L%5Cright%7C_%7Bw%3Dw_%7Bt-1%7D%7D)
    
      
    
    ![](https://www.zhihu.com/equation?tex=L%3D%5Csum_%7Bi%7D%20l%5Cleft%28y_%7Bi%7D%2C%20f_%7Bw%7D%5Cleft%28w_%7Bi%7D%5Cright%29%5Cright%29)
    

![](https://img-blog.csdnimg.cn/img_convert/7068332fca756feaa1d45561ff3f7a29.png)

3.GBDT优缺点
---------

下面我们来总结一下 GBDT 模型的优缺点：

![](https://img-blog.csdnimg.cn/img_convert/ef8dfeab2d8183179020e728f72857f7.png)

### 1）优点

*   预测阶段，因为每棵树的结构都已确定，可并行化计算，计算速度快。
*   适用稠密数据，泛化能力和表达能力都不错，数据科学竞赛榜首常见模型。
*   可解释性不错，鲁棒性亦可，能够自动发现特征间的高阶关系。

### 2）缺点

*   GBDT 在高维稀疏的数据集上，效率较差，且效果表现不如 SVM 或神经网络。
*   适合数值型特征，在 NLP 或文本特征上表现弱。
*   训练过程无法并行，工程加速只能体现在单颗树构建过程中。

4.随机森林 vs GBDT
--------------

对比ShowMeAI前面讲解的另外一个集成树模型算法[**随机森林**](https://www.showmeai.tech/article-detail/191)，我们来看看 GBDT 和它的异同点。

![](https://img-blog.csdnimg.cn/img_convert/4612b944868dff6c2e6a7ac1a73ca5ee.png)

### 1）相同点

*   都是集成模型，由多棵树组构成，最终的结果都是由多棵树一起决定。
*   **RF** 和 **GBDT** 在使用 **CART** 树时，可以是分类树或者回归树。

### 2）不同点

*   训练过程中，随机森林的树可以并行生成，而 **GBDT** 只能串行生成。
*   随机森林的结果是多数表决表决的，而 **GBDT** 则是多棵树累加之。
*   随机森林对异常值不敏感，而 **GBDT** 对异常值比较敏感。
*   随机森林降低模型的方差，而 **GBDT** 是降低模型的偏差。

5.Python代码应用与模型可视化
------------------

下面是我们直接使用 python 机器学习工具库 sklearn 来对数据拟合和可视化的代码：

```python
## 使用Sklearn调用GBDT模型拟合数据并可视化

import numpy as np
import pydotplus
from sklearn.ensemble import  GradientBoostingRegressor

X = np.arange(1,  11).reshape(-1,  1)
y = np.array([5.16,  4.73,  5.95,  6.42,  6.88,  7.15,  8.95,  8.71,  9.50,  9.15])

gbdt =  GradientBoostingRegressor(max_depth=4, criterion ='squared_error').fit(X, y)

from  IPython.display import  Image
from pydotplus import graph_from_dot_data
from sklearn.tree import export_graphviz

## 拟合训练5棵树
sub_tree = gbdt.estimators_[4,  0]
dot_data = export_graphviz(sub_tree, out_file=None, filled=True, rounded=True, special_characters=True, precision=2)
graph = pydotplus.graph_from_dot_data(dot_data)
Image(graph.create_png())
```

![](https://img-blog.csdnimg.cn/img_convert/09f79892cf7f82d15c228df2fe046ff0.png)

[ShowMeAI]系列教程推荐
--------------------------------------------

*   [大厂技术实现 | 推荐与广告计算解决方案](https://www.showmeai.tech/tutorials/50)
*   [大厂技术实现 | 计算机视觉解决方案](https://www.showmeai.tech/tutorials/51)
*   [大厂技术实现 | 自然语言处理行业解决方案](https://www.showmeai.tech/tutorials/52)
*   [图解Python编程：从入门到精通系列教程](https://www.showmeai.tech/tutorials/56)
*   [图解数据分析：从入门到精通系列教程](https://www.showmeai.tech/tutorials/33)
*   [图解AI数学基础：从入门到精通系列教程](https://www.showmeai.tech/tutorials/83)
*   [图解大数据技术：从入门到精通系列教程](https://www.showmeai.tech/tutorials/84)
*   [图解机器学习算法：从入门到精通系列教程](https://www.showmeai.tech/tutorials/34)
*   [机器学习实战：手把手教你玩转机器学习系列](https://www.showmeai.tech/tutorials/41)
*   [深度学习教程 | 吴恩达专项课程 · 全套笔记解读](https://www.showmeai.tech/tutorials/35)
*   [自然语言处理教程 | 斯坦福CS224n课程 · 课程带学与全套笔记解读](https://www.showmeai.tech/tutorials/36)

机器学习【算法】系列教程
------------

*   [图解机器学习 | 机器学习基础知识](https://www.showmeai.tech/article-detail/185)
*   [图解机器学习 | 模型评估方法与准则](https://www.showmeai.tech/article-detail/186)
*   [图解机器学习 | KNN算法及其应用](https://www.showmeai.tech/article-detail/187)
*   [图解机器学习 | 逻辑回归算法详解](https://www.showmeai.tech/article-detail/188)
*   [图解机器学习 | 朴素贝叶斯算法详解](https://www.showmeai.tech/article-detail/189)
*   [图解机器学习 | 决策树模型详解](https://www.showmeai.tech/article-detail/190)
*   [图解机器学习 | 随机森林分类模型详解](https://www.showmeai.tech/article-detail/191)
*   [图解机器学习 | 回归树模型详解](https://www.showmeai.tech/article-detail/192)
*   [图解机器学习 | GBDT模型详解](https://www.showmeai.tech/article-detail/193)
*   [图解机器学习 | XGBoost模型最全解析](https://www.showmeai.tech/article-detail/194)
*   [图解机器学习 | LightGBM模型详解](https://www.showmeai.tech/article-detail/195)
*   [图解机器学习 | 支持向量机模型详解](https://www.showmeai.tech/article-detail/196)
*   [图解机器学习 | 聚类算法详解](https://www.showmeai.tech/article-detail/197)
*   [图解机器学习 | PCA降维算法详解](https://www.showmeai.tech/article-detail/198)

机器学习【实战】系列教程
------------

*   [机器学习实战 | Python机器学习算法应用实践](https://www.showmeai.tech/article-detail/201)
*   [机器学习实战 | SKLearn入门与简单应用案例](https://www.showmeai.tech/article-detail/202)
*   [机器学习实战 | SKLearn最全应用指南](https://www.showmeai.tech/article-detail/203)
*   [机器学习实战 | XGBoost建模应用详解](https://www.showmeai.tech/article-detail/204)
*   [机器学习实战 | LightGBM建模应用详解](https://www.showmeai.tech/article-detail/205)
*   [机器学习实战 | Python机器学习综合项目-电商销量预估](https://www.showmeai.tech/article-detail/206)
*   [机器学习实战 | Python机器学习综合项目-电商销量预估<进阶方案>](https://www.showmeai.tech/article-detail/207)
*   [机器学习实战 | 机器学习特征工程最全解读](https://www.showmeai.tech/article-detail/208)
*   [机器学习实战 | 自动化特征工程工具Featuretools应用](https://www.showmeai.tech/article-detail/209)
*   [机器学习实战 | AutoML自动化机器学习建模](https://www.showmeai.tech/article-detail/210)

[ShowMeAI](https://www.showmeai.tech/)系列教程推荐
--------------------------------------------

[ShowMeAI](https://www.showmeai.tech/) 系列教程推荐
---------------------------------------------

*   [大厂技术实现：推荐与广告计算解决方案](https://www.showmeai.tech/tutorials/50)
*   [大厂技术实现：计算机视觉解决方案](https://www.showmeai.tech/tutorials/51)
*   [大厂技术实现：自然语言处理行业解决方案](https://www.showmeai.tech/tutorials/52)
*   [图解Python编程：从入门到精通系列教程](https://www.showmeai.tech/tutorials/56)
*   [图解数据分析：从入门到精通系列教程](https://www.showmeai.tech/tutorials/33)
*   [图解AI数学基础：从入门到精通系列教程](https://www.showmeai.tech/tutorials/83)
*   [图解大数据技术：从入门到精通系列教程](https://www.showmeai.tech/tutorials/84)
*   [图解机器学习算法：从入门到精通系列教程](https://www.showmeai.tech/tutorials/34)
*   [机器学习实战：手把手教你玩转机器学习系列](https://www.showmeai.tech/tutorials/41)
*   [深度学习教程：吴恩达专项课程 · 全套笔记解读](https://www.showmeai.tech/tutorials/35)
*   [自然语言处理教程：斯坦福CS224n课程 · 课程带学与全套笔记解读](https://www.showmeai.tech/tutorials/36)