# BERT！BERT！BERT!
从ELMO说起的预训练语言模型
---------------

我们先来看一张图：

![](https://img-blog.csdnimg.cn/img_convert/824c838c065d4aff50b402ebf64a02ff.png)

从图中可以看到，ELMO其实是NLP模型发展的一个转折点，从ELMO开始，Pre-training+finetune的模式开始崭露头角并逐渐流行起来。最初的ELMO也只是**为了解决word2vec不能表达”一词多义“的问题**提出来的，它所代表的动态词向量的思想更是被不少任务拿来借鉴。它最大的贡献还是提出的预训练+微调的模式，这种迁移学习的思想几乎是它作为NLP模型发展转折点的一个重要原因。

我们简单地看一下ELMO的模型结构（我加了红色标注便于直观理解）：

![](https://img-blog.csdnimg.cn/img_convert/c2d72dc261f2dc41ea5ee3706c088a47.png)

ELMO有一个很明显的特征就是它使用的是**双层双向的LSTM**，最后网络会产生一个动态词向量，它来自于不同的层输出的组合，**由于不同层学习到的特征不一样，最后通过权重的调整向量表征的含义侧重点也不一样**。与CV中的模型类似，越靠近输入层的层级学习到的特征越简单，比如W1可能是一些词性或字的特征；越靠近输出层的层级学习到的特征越高级，比如W3可能就是学习到一些句子级的特征。动态词向量偏重于哪一层取决于下游的任务，举个例子，序列标注任务多偏重于底层级一般效果就会更好一点。

ELMO在预训练阶段得到的网络参数再拿到具体任务中进行微调，这种迁移学习的思想**大大降低了一些任务对大数据量的依赖性，在节约成本的同时效果也很不错**，这为后面的[BERT](https://so.csdn.net/so/search?q=BERT&spm=1001.2101.3001.7020)的出现铺平了道路，也慢慢引领了一个时代的潮流。

> ELMO的缺点在其他文章中也有提及：ELMO 使用了 LSTM ，特征提取能力没有Transformer好；ELMO采取双向拼接这种融合特征的能力可能比 Bert 一体化的融合特征方式弱。但这都是事后视角了，了解即可。

BERT详解
------

### BERT的总体结构

先来看看BERT的结构框架:

![](http://pennlee-aliyun.oss-cn-beijing.aliyuncs.com/img/image-20201217014110062.png)

上面就是BERT的模型结构图了，是不是和ELMO长的有点像，**模型的核心由BERT Encoder组成，BERT Encoder由多层BERT Layer组成，每一层的BERT Layer其实都是Transformer中的Encoder Block**。再来回顾一下图：

![](http://pennlee-aliyun.oss-cn-beijing.aliyuncs.com/img/image-20201217015348332.png)

说到这里其实我们就已经了解到了BERT的一个大体形貌了，对于Transformer中的Encoder Block有点生疏的话可以再去回顾一下：[Transformer 看这一篇就够了](https://blog.csdn.net/rongsenmeng2835/article/details/110511294)，相信可以回忆起来的。

> **由于BERT使用的是Transformer中的Encoder Block，所以其本质还是一个特征提取器，只不过由于网络较深和Self-Attention机制的帮衬，它的特征提取/模型学习能力更强。也正是由于它只是Encoder的缘故，所以BERT也不具备生成能力，没法单独用BERT来解决NLG的问题，只能解决NLU的问题。** 

### BERT的输入

还是老规矩，对着图说：

![](https://img-blog.csdnimg.cn/img_convert/bde1cefd1ed3686810d421b60c111dd2.png)

如图所示，BERT模型输入的Embedding由Token Embeddings、Segment Embeddings和Position Embeddings相加而成，如下图。我们分别来看一下这三部分：

*   **Token Embeddings**：最传统的词向量，与之前我们一起学习的语言模型的输入一样，它是将token嵌入到一个高维空间中表示的结果。对这一部分的Embeddings，有几点需要注意：

> *   token的选取与之前的Transformer不太一样，在这里采用的一种新的token形式，是一种子词粒度的表现形式，称之为subword。说的大白话一点就是中文还是字符级别的token，英文不再按照传统的空格分词，而是将部分单词再拆分成字根的形式，比如playing拆成play+ing。**这样做最大的好处就是字典变小了，计算量也变小了，同时也在一定程度上解决了OOV的问题。** 
>     
>     > 常用的subword的方法可参见：[深入理解NLP Subword算法：BPE、WordPiece、ULM](https://zhuanlan.zhihu.com/p/86965595)
>     
> *   输入token时，第一个token是一个\[CLS\]的特殊字符，可以用于下游的任务，比如分类任务。句子和句子的中间，以及句子的末尾会有一个特殊的符号\[SEP\]；
>     

*   **Segment Embeddings**：用来区别两种句子，因为预训练不光做LM还要做以两个句子为输入的分类任务，即主要给NSP任务使用。
    
*   **Position Embeddings**：和Transformer不一样，0-10的位置编码只能表示绝对位置，而Transformer中的正余弦函数还可以表示相对位置。（为什么不用正余弦了？因为BERT模型学习能力强。）
    

对于BERT输入部分还有几点需要补充：

1.  上图中标注了各个Embeddings对应的维度，768是BERT-base的词向量维度，BERT-Large与BERT-base的参数对比贴在这：
    
    > 【L：网络层数，H：隐藏层维度，A：Multi-Head Attention 多头头数】
    > 
    > Bert-base： (L=12, H=768，A=12, Total Parameters=110M).（使用GPU内存：7G+）
    > 
    > BERT-large： (L=24, H=1024，A=16, Total Parameters=340M).（使用GPU内存：32G+）
    
2.  BERT中输入的tokens的长度不能超过512，即一个句子最大的长度加上\[CLS\]和\[SEP\]不能超过512，这里可能就涉及到句子截取等工作。
    
3.  一个思考：\[CLS\]或\[SEP\]多几个或者少几个可以吗？
    
    > 可以，因为网络用到的Transformer中有Self-Attention，每一个位置都可以学习到其他位置词的信息，所以在这一点上他们是一样的。多几个或少几个都是可以的，甚至可以一个token后面跟一个\[SEP\]，训练完后拿出来做NER任务。
    

### How to Pre-training

BERT的预训练过程主要分为两个部分：MLM（Mask Language Model，完形填空）和NSP（Next Sentence Prediction，句对预测）。接下来我们分别来看这两部分：

#### MLM

MLM本质还是一个语言模型，那么既然是一个语言模型，那它干的事必然就是【预测一句话合理性的概率】，换句话说就是**利用上下文预测可能出现的词**。由于LM是单向的，要不从左到右要不从右到左，很难做到结合上下文语义。**为了改进LM，实现双向的学习，MLM通过对输入文本序列随机的mask，然后通过上下文来预测这个mask应该是什么词，至此解决了双向的问题**。所以人们称之为完形填空，不同于之前LM是自回归问题，这里的**MLM是一个自编码问题**。目的就是学习语料中的各种特征。

> 那么这个mask又是怎么做的呢？
> 
> 具体的做法是，对输入句子中的token以15%的概率进行随机选取（如选中"bed"这个token），再将选取出来的token以80%的概率替换成\[MASK\]，10%的概率替换成替换成其他token（如“dad”），最后10%保持不变(依然是“bed”)。
> 
> 这样大费周章地做的**目的是**什么呢？
> 
> 其实是**提高了模型的泛化能力，因为后面的微调阶段并不会做mask的操作，为了减少预训练和微调阶段输入分布不一致的问题导致的模型表达能力差的现象，故采用了这种策略。** 

#### NSP

NSP任务主要做的就是一件事：预测句子1与句子2挨在一起的概率。构建数据的方法是，对于句子1，句子2以50%的概率为句子1相连的下一句，以50%的概率在语料库里随机抽取一句。以此构建了一半正样本一半负样本。再用输出的CLS进行二分类，以此来判断输入的两个句子是不是前后相连的关系。

### How to finetune

之前说过，迁移学习的思想是要将Pre-training阶段的主要成果应用在 finetune阶段，finetune充分应用了大规模预训练模型的优势，只在下游任务上再进行一些微调训练，就可以达到非常不错的效果。

下面是BERT原文[BERT:Pre-trainingofDeepBidirectionalTransformersfor LanguageUnderstanding](https://arxiv.org/abs/1810.04805)中提及的微调阶段的四种不同类型的下游任务：

![](https://img-blog.csdnimg.cn/img_convert/8d89ad6c45c1762ad583a8dacfadfff1.png)

在这里简述一下四种任务对应微调阶段的基本方法：

*   **句子对匹配（sentence pair classification）**：文本匹配类任务的输入是两个不同的句子，最终其实要实现的仍然是一个二分类的问题，若要用BERT实现，基本代码和流程与分类问题一致，全连接层输出维度是２。但实际工程应用上，直接采用BERT来做文本匹配问题的话最终效果不一定会好，一般解决文本匹配问题可以采用一些类似孪生网络的结构去解决，这块可以另外再去探究。
*   **文本分类（single sentence classification）**：文本分类是BERT最擅长做的事情了，最基本的做法就是将预训练的BERT加载后，同时在输出\[CLS\]的基础上加一个全连接层来做分类，全连接层输出的维度就是我们要分类的类别数。当然也可以在分类之前加一个其他的网络层以达到对应的目的。
*   **抽取式问答（question answering）**：注意由于BERT没有生成能力，所以只能做抽取式的问答。可以这样理解：这个回答其实是在一篇文章中找答案的过程，通过预测答案开始与结束位置在文中的id来进行训练。
*   **序列标注（single sentence tagging）**：由于一般的分词任务、词性标注和命名体识别任务都属于序列标注问题。这类问题因为输入句子的每一个token都需要预测它们的标签，所以序列标注是一个单句多label分类任务，BERT模型的所有输出（除去特殊符号）都要给出一个预测结果。

### BERT微调模型的设计

针对不同的任务我们可以继续在bert的预训练模型基础上加一些网络的设计，比如文本分类上加一些cnn；比如在序列标注上加一些crf等等。

下面只记录几点提要，具体使用还需深究代码。

BERT+CNN：由于BERT中Attention能捕捉长距离的语义特征，而CNN中的滤波器与卷积操作更能捕捉局部特征，故BERT+CNN（+池化层）也是一个能很好互补的组合。相比之下就不太建议加RNN或者Attention之类的结构了，因为BERT里就带了类似的结构了。

BERT＋CRF：NER问题很好的一种解决方案。

### BERT的输出

之前我们看到了BERT的输入是三种Embeddings的组合，那BERT模型的输出是什么呢。通过下图能够看出会有两种输出，一个对应的是红色框，也就是对应的\[CLS\]的输出，输出的shape是\[batch size，hidden size\]；另外一个对应的是蓝色框，是所有输入的token对应的输出，它的shape是\[batch size，seq length，hidden size\]，这其中不仅仅有\[CLS\]对于的输出，还有其他所有token对应的输出。

![](https://img-blog.csdnimg.cn/img_convert/6b026a37608a5f5fd3477d690e1182c8.png)

在使用代码上就要考虑到底是使用第一种还是第二种作为输出了。大部分情况是是会选择\[CLS\]的输出，再进行微调的操作。不过有的时候使用所有token的输出也会有一些意想不到的效果。

### 其他一些疑问与细节

#### **预训练模型为何会有输入长度512的限制？**

> BERT模型要求输入句子的长度不能超过512，同时还要考虑\[CLS\]这些特殊符号的存在，实际文本的长度会更短。**究其原因，随着文本长度的不断增加，计算所需要的显存也会成线性增加，运行时间也会随着增长。所以输入文本的长度是需要加以控制的**。

> 在实际的任务中我们的输入文本一般会有两个方面，要不就是特别长，比如文本摘要、阅读理解任务，它们的输入文本是有可能超过512；另外一种就是一些短文本任务，如短文本分类任务。

#### **长文本该如何处理？**

> 说到长文本处理，最直接的方法就是**截断**。
> 
> 由于 Bert 支持最大长度为 512 个token，那么如何截取文本也成为一个很关键的问题。[How to Fine-Tune BERT for Text Classification?](https://arxiv.org/abs/1905.05583)中给出了几种解决方法
> 
> *   head-only： 保存前 510 个 token （留两个位置给 \[CLS\] 和 \[SEP\] ）
> *   tail-only： 保存最后 510 个token
> *   head + tail ： 选择前128个 token 和最后382个 token
> 
> 作者是在IMDB和Sogou News数据集上做的试验，发现head+tail效果会更好一些。**但是在实际的问题中，我们还是要人工的筛选一些数据观察数据的分布情况，视情况选择哪种截断的方法。** 
> 
> 除了上述截断的方法之外，还可以采用sliding window的方式做。
> 
> 用划窗的方式对长文本切片，分别放到BERT里，得到相对应的CLS，然后对CLS进行融合，融合的方式也比较多，可以参考以下方式：
> 
> *   max pooling最大池化
> *   avg pooling平均池化
> *   attention注意力融合
> *   transformer等

#### **短文本该如何处理？**

> 在遇到一些短文本的NLP任务时，我们需要根据输入文本长度的分布情况重新选取max\_sequence\_length，最大输入文本长度的取值可以通过正态分布得出。

#### 什么是warmup？为什么要在设置学习率的时候采用的warmup策略？

> `warmup`是一种学习率优化方法（最早出现在ResNet论文中）。在模型训练之初选用较小的学习率，训练一段时间之后（如：10epoches或10000steps）使用预设的学习率进行训练，或逐渐减小。（BERT原文中前10000步会增长到1e-4, 之后再线性下降。）
> 
> 如下图的Linner Warmup：
> 
> ![](https://img-blog.csdnimg.cn/img_convert/85b5eff17568b921e0171f49f55452be.png)
> 
> 感性分析为何要使用warmup策略：
> 
> 1.  刚开始模型对数据完全不了解，这个时候步子太大，loss容易跑飞，此时需要使用小学习率摸着石头过河；
> 2.  对数据了解了一段时间之后，可以使用大学习率朝着目标大步向前；
> 3.  快接近目标时，使用小学习率进行精调探索，此时步子太大，容易错过目标点；

#### 不同学习率的设置

> 在 fine-tune阶段使用过大的学习率，会打乱 pretrain 阶段学习到的句子信息，造成“灾难性遗忘”。BERT没有下游微调结构的，是直接用BERT去fine-tune时，BERT模型的训练和微调学习率取2e-5和5e-5效果会好一些。那如果微调的时候接了更多的结构，比如BERT+TextCNN，BERT+BiLSTM+CRF，此种情况下BERT的fine-tune学习率可以设置为5e-5, 3e-5, 2e-5。而下游任务结构的学习率可以设置为1e-4，让其比bert的学习更快一些。至于这么做的原因也很简单：BERT本体是已经预训练过的，即本身就带有权重，所以用小的学习率很容易fine-tune到最优点，而下接结构是从零开始训练，用小的学习率训练不仅学习慢，而且也很难与BERT本体训练同步。为此，我们将下游任务网络结构的学习率调大，争取使两者在训练结束的时候同步：当BERT训练充分时，下游任务结构也能够训练充分。

#### BERT与GPT的区别？BERT可以做生成任务吗？

> BERT与GPT最大的区别是：BERT用的是Transformer中的Encoder部分，而GPT用的是Transformer中的Decoder部分。
> 
> BERT加上Transformer中的Decoder部分就可以做生成任务了。

#### BERT中的weight decay权重衰减是什么？作用？

> **权重衰减等价于L2范数正则化。正则化通过为模型损失函数添加惩罚项使得学习的模型参数值较小，是常用的过拟合的常用手段。** 
> 
> 权重衰减并不是所有的权重参数都需要衰减，比如bias，和LayerNorm.weight就不需要衰减。

### BERT不同层的含义

[BERT Rediscovers the Classical NLP Pipeline](https://arxiv.org/abs/1905.05950)中对于BERT不同层所学习到的不同信心做了一定的研究，总体来说会有以下发现：

*   BERT会在较低层编码更多语法信息，在较高层编码更多语义信息（**像句法分析、实体识别这些比较简单的NLP任务更倾向于使用BERT底层的信息。而像关系分类等复杂的NLP任务更倾向于BERT的高层信息**）。
*   语法信息的体现比较局部化（Locolizable），语义信息的体现在各层中分布的比较均匀。
*   该文章定性地分析证明在较低层产生的一些有歧义的决定可以在较高层被修正。

### BERT模型的迁移

#### 迁移特性

[《Linguistic Knowledge and Transferability of Contextual Representations》](https://arxiv.org/pdf/1903.08855.pdf)一文所做的工作提到部分关于BERT模型迁移特性的结论：

对于难度比较低的任务，仅仅需要BERT的前几层就能得到更好的迁移学习能力。所以适当减少层数是有助于获得更好的迁移学习基础。在不去微调预训练模型的基础上，合理的选择层是有必要的。如果对于BERT-base（12层）来说，最佳的迁移层数经常在6-12层之间，对于比较难的任务，需要的层数会多一些。

#### 迁移策略

那拿到一个BERT预训练模型后，我们会有两种选择：

1.  **把BERT当做特征提取器或者句向量，不在下游任务中微调。** 
2.  **把BERT做为下游业务的主要模型，在下游任务中微调。** 

基于BERT模型的改进型模型简介
----------------

> **Roberta模型：**更具鲁棒性的、优化后的BERT模型。采用**动态Mask**策略，Roberta是把数据复制10份，每一份中采用不同的静态Mask操作，使得预训练的每条数据有了10种不同的Mask数据。在Roberta中，采用的是full-sentence的形式，每一个训练样本都是从一个文档中连续sample出来的，并且不采用NSP损失。从结论中发现效果会更好，说明NSP任务不是必须的。

> **BERT-WWM：**核心是**全词Mask**。主要更改了原预训练阶段的训练样本生成策略。在全词Mask中，如果一个完整的词的部分WordPiece子词被Mask，则同属该词的其他部分也会被Mask，即全词Mask。（例如`“这个模型真好用”`可能会被Mask成`"这个模[MASK]真好用"`，而在WWM下会被Mask成`“这个[MASK] [MASK]真好用”`）。需要注意的是，这里的Mask指的是广义的Mask（替换成\[MASK\]；保持原词汇；随机替换成另外一个词），并非只局限于单词替换成\[MASK\]标签的情况。

芝麻街的初心
------

自ELMO出来之后，后续又有例如BERT这样革命性的模型出现，再到后来，更多优秀的模型也被提出来，他们中有好多都像BERT致敬ELMO一样致敬了前面的模型，起的名字好多都来自于动画节目《芝麻街》，以下列举了这几个模型对应的人物形象以及论文链接（看了半天BERT着实是好看啊，哈哈）。致敬一群群推动NLP发展的创造者们，希望自己也能保持一颗孩童时代的好奇心，保持出发时的初心，一路向前。加油！

* * *

> 除文中已提及文章外其他参考文章：
> 
> [The Bright Future of ACL/NLP](https://www.msra.cn/wp-content/uploads/2019/08/ACL-MingZhou.pdf)
> 
> [Deep contextualized word representations](https://arxiv.org/abs/1802.05365)
> 
> [ELMo原理解析及简单上手使用](https://zhuanlan.zhihu.com/p/51679783)
> 
> [关于ELMo你不知道的一些细节](https://zhuanlan.zhihu.com/p/89894807)
> 
> [深入理解NLP Subword算法：BPE、WordPiece、ULM](https://zhuanlan.zhihu.com/p/86965595)
> 
> [Bert输入输出是什么](https://zhuanlan.zhihu.com/p/248017234?utm_source=wechat_session)
> 
> [BERT 详解](https://zhuanlan.zhihu.com/p/103226488)