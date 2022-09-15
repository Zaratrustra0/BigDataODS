# 白话机器学习-Transformer | AI技术聚合
大抵是去年底吧，收到了几个公众号读者的信息，希望能写几篇介绍下Attention以及Transformer相关的算法的文章，当时的我也是满口答应了，但是确实最后耽误到了现在也没有写。

前一阵打算写这方面的文章，不过发现一个问题，就是如果要介绍Transformer，则必须先介绍Self Attention，亦必须介绍下Attention，以及Encoder-Decoder框架，以及GRU、LSTM、RNN和CNN，所以开始漫长的写作之旅。

终于在这个五一假期完成了最后四篇（一共八篇，从开始写到现在大概用了两个月），算是兑现了我的承诺。应该是水果。尤其是对我这种爱面子的人来说，更是充实。最近，我有了一些感悟，对很多事情有很多不同的看法和理解。以前看《易经》，觉得有点懂，现在好像完全没懂，偏离了很多。有些事情可能需要重新审视，有些想法需要重新组织。

事实上，有时我不想写这些文章。具备这种基本功的文章，可以说投入产出比很低。人们倾向于追求酷炫的技术并沉迷于他们表面上的理解。而能够打造出这些炫酷技术的，一定是基础功底扎实的人。对于算法学生来说，不仅仅是算法，还需要扎实而深厚的架构知识。

好吧，这本书又回到了主线。至此，已经完成了几篇文章的输出，列举如下。组织方法是根据学习的依赖性来构建的。喜欢的同学可以在“秃头码农”公众号上选机。学习算法系列学习。

*   《白话[机器学习](https://aitechtogether.com/tag/%e6%9c%ba%e5%99%a8%e5%ad%a6%e4%b9%a0 "机器学习")\-卷积神经网络CNN》
*   《白话机器学习-循环神经网络RNN》
*   《白话机器学习-长短期记忆网络LSTM》
*   《白话机器学习-循环神经网络概述从RNN到LSTM再到GRU》
*   《白话机器学习-Encoder-Decoder框架》
*   《白话机器学习-Attention》
*   《白话机器学习-Self Attention》
*   《白话机器学习-Transformer》

终于把这个系列写到了最后一文，五一五天假期的第四天，嗯，怎么说呢！感觉自己过的很充实。在这里炫耀下，昨天买了个mac的触摸板，比起鼠标来，还真是好用，配合上我的大显示器、机器键盘，妥妥的提升生产力，我想我还能写几百篇 哈哈。

在前一篇文章中，我们讨论了注意力——现代[深度学习](https://aitechtogether.com/tag/%e6%b7%b1%e5%ba%a6%e5%ad%a6%e4%b9%a0 "深度学习")模型中普遍存在的方法。注意力是一个有助于提高神经机器翻译应用程序性能的机制（并行化）。在这篇文章中，我们将讨论Transformer —— 利用注意力来提高训练速度的模型。Transformer在特定任务中优于谷歌神经机器翻译模型。其中最大的提升在于Transformer架构适合于并行化。事实上，谷歌Cloud建议使用Transformer作为参考模型来使用他们的云TPU产品。让我们把这个模型拆开来看看它是如何运作的。

Transformer是在Attention is all you need这篇论文中提出的，并且引起了业界的广泛的关注，仿佛你要不提下、不知道Transformer你就不是做算法的，可见其强大的影响力。

以语言机器翻译模型为例，从整体上看，机器翻译过程中，主要是输入一种语言的句子，然后输出另一种语言的对应翻译。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/c7527962-b329-4184-b40a-629ee7c480c2.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/c7527962-b329-4184-b40a-629ee7c480c2.webp)

前面的章节中，我们已经介绍过Encoder-Decoder框架，他主要包含量大组件：

*   编码组件
*   解码组件

通过编码组件和解码组件构建整个框架，并对模型进行训练和预测。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/2a69715f-58e8-4ee2-9067-fde35dd2f273.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/2a69715f-58e8-4ee2-9067-fde35dd2f273.webp)

下面我看一个例子，编码组件是一堆编码器（有六个编码器，一个叠一个——数字6没有什么神奇的，你肯定可以尝试其他的安排）。解码组件是具有相同数字的解码器的排列。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/3d2049b0-9d11-4921-8ba7-d5404beabf4c.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/3d2049b0-9d11-4921-8ba7-d5404beabf4c.webp)

编码器在结构上都是相同的（但它们不共享权重）。每个都分为两个子层：

[![](https://aitechtogether.com/wp-content/uploads/2022/05/39c82002-9200-4449-ab61-f1a484fbc5da.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/39c82002-9200-4449-ab61-f1a484fbc5da.webp)

解码器有两层：

*   Self-Attention层：编码器的输入首先经过一个Self-Attention层——本层可以帮助编码器在对某个特定单词进行编码时查看输入句子中的其他单词与本单词的相关性，并且进行加权计算。
*   Feed Forward Neural Network：Self-Attention层的输出到前馈神经网络，完全相同的前馈网络独立地应用于每个位置。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/50b16ca2-4167-4c5a-8381-a30691ffda58.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/50b16ca2-4167-4c5a-8381-a30691ffda58.webp)

现在我们已经看到了模型的主要组成部分，让我们开始研究各种矢量/张量，以及它们如何在这些组成部分之间流动，从而将训练模型的输入转换为输出。

与一般的NLP应用程序一样，我们首先使用嵌入算法将每个输入单词转换为一个向量。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/3842015b-e6cd-4e89-b06b-75b72818f042.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/3842015b-e6cd-4e89-b06b-75b72818f042.webp)

嵌入只发生在最底部的编码器中。所有编码器的共同抽象是，它们接收一个大小为512的向量列表——在底部的编码器中，这个词是“嵌入”，但在其他编码器中，它将是直接下面的编码器的输出。

在输入序列中嵌入单词后，每个单词都会流经两层编码器。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/66be2847-13d1-454e-a278-8557ea25d407.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/66be2847-13d1-454e-a278-8557ea25d407.webp)

在这里，我们开始看到Transformer的一个关键属性，即每个位置的单词在编码器中都通过自己的路径流动。在自我注意层中，这些路径之间存在依赖关系。然而，前馈层没有这些依赖关系，因此各种路径可以在流经前馈层时并行执行。

正如我们已经提到的，编码器接收向量列表作为输入。它处理这个向量，将这些向量传递给“自我注意”层，然后进入前馈神经网络，并将输出发送到下一个可以是多层的编码器。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/16a3c5fa-e116-4e37-986b-5d21f0a8d0d1.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/16a3c5fa-e116-4e37-986b-5d21f0a8d0d1.webp)

上一篇文章已经介绍过Self Attention，这里从另外的角度进行下阐述。其实注意力机制在现实生活中是大量存在的，而且人类社会也在一直使用。让我们提炼一下它是如何工作的。

假设以下句子是我们要翻译的输入句子：

“这只动物没有过马路，因为它太累了。”

这句话中的“它”指的是什么？它指的是街道还是动物？这对人类来说是一个简单的问题，但对算法来说却不是那么简单。

使用self-[attention](https://aitechtogether.com/tag/attention "attention")机制时，模型在处理“it”这个词时，self-attention让它把“it”和“animal”联系起来。

当模型处理每个单词（输入序列中的每个位置）时，self-attention 允许它查看输入序列中的其他位置以寻找有助于更好地编码该单词的线索。

如果你对RNN很熟悉，想想如何维护一个隐藏状态让RNN将它之前处理过的单词/向量的表示与当前正在处理的单词/向量结合起来。自我关注是Transformer用来将其他相关词汇的“理解”融入到我们当前处理的词汇中，同样可以达到这个目标，同时由于其算法的特点可以无视距离，并且实现并行化的操作，提升计算性能。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/914bc097-0d92-4e0e-be88-9971b49b27bd.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/914bc097-0d92-4e0e-be88-9971b49b27bd.webp)

本节我们再复习下如何计算Self-Attention，本节不做过细的介绍，一律使用矩阵计算。

计算自注意力的第一步是从每个输入向量到编码器创建三个向量（在这种情况下，在训练期间根据三个参数矩阵计算）。

这些新的向量比嵌入向量的维度要小。它们的维数为64，而嵌入向量和编码器输入输出向量的维数为512。向量的维度可以由大家自行定义，这里使用64。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/21374173-36dd-41cf-9edc-3e5b9ac137d0.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/21374173-36dd-41cf-9edc-3e5b9ac137d0.webp)

什么是“查询”、“键”和“值”向量？

它们是用于计算和思考注意力的抽象概念。一旦你阅读了下面的注意力是如何计算的，你就会知道几乎所有你需要知道的关于每个向量的作用。

计算自我注意力的第二步是计算分数。假设我们在计算这个例子中第一个单词“Thinking”的自我注意力。我们需要将输入句子中的每个单词与这个单词进行比较。分数决定了当我们在某个位置编码一个单词时，将多少注意力放在输入句子的其他部分。

分数是通过查询向量与我们打分的单词的关键向量的点积来计算的。如果我们处理位置1的单词的自我注意，第一个分数就是q1和k1的点积。第二个分数是q1和k2的点积。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/21551acc-dbad-4fd4-ab9b-9abe048beffc.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/21551acc-dbad-4fd4-ab9b-9abe048beffc.webp)

第三步和第四步是将分数除以8（论文中使用的关键向量维数的平方根- 64），据说这样可以拥有更稳定的梯度。然后通过softmax操作传递结果。Softmax将这些分数归一化，使它们都是正的，加起来等于1。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/f11d500c-ba0b-4771-9ee2-1c887306725f.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/f11d500c-ba0b-4771-9ee2-1c887306725f.webp)

这个softmax分数决定了在这个位置每个单词被歧途单词影响的程度。显然，在这个位置的单词将拥有最高的softmax分数，但有时关注与当前单词相关的另一个单词是非常有用的。

第五步是将每个值向量乘以softmax得分（准备将它们相加）。这里的直觉是保留我们想要关注的单词的值，并淹没不相关的单词（例如，通过将它们乘以很小的数字，如0.001），然后进行按元素相加，生成最终的表征向量。

第六步是对加权值向量求和。这会在这个位置产生自注意力层的输出（对于第一个单词）。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/7835a952-0fb0-4e1c-ad87-bd84438a38e6.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/7835a952-0fb0-4e1c-ad87-bd84438a38e6.webp)

第一步是计算Query、Key和Value矩阵。我们通过将我们的嵌入投射到一个矩阵X中，并将其乘以我们训练过的权重矩阵(WQ, WK, WV)来实现。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/f2bf966e-ccee-4744-862b-f63b8e090085.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/f2bf966e-ccee-4744-862b-f63b8e090085.webp)

最后，第二步到第六步可以浓缩成一个公式来计算self-attention层的输出。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/afbb9a10-2a09-4f3e-b992-9e5eb79b9718.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/afbb9a10-2a09-4f3e-b992-9e5eb79b9718.webp)

本文进一步细化了self-attention层，增加了“multi-head”的注意力机制。这通过两种方式提高了注意力层的性能：

*   它扩展了模型关注不同位置的能力。是的，在上面的例子中，z1包含了一些其他的编码，但是它可以被实际的单词本身支配。如果我们在翻译像“The animal didn ‘t cross The street because It was too tired”这样的句子，我们会想知道“It”指的是哪个词，这是很有用的。
*   它给了注意层多个“表示子空间”。正如我们接下来将看到的，对于多头部注意，我们不仅有一个，而且有多个查询/键/值权重矩阵集（Transformer使用8个注意头部，因此我们最终为每个编码器/解码器提供8个注意头部集）。每个集合都是随机初始化的。然后，经过训练，每个集合用于将输入嵌入（或来自较低编码器/解码器的向量）投影到不同的表示子空间，通过不同的子空间挖掘更多的相互之间的关系。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/5c5cbe76-6860-49ff-baf1-5a3462cc607f.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/5c5cbe76-6860-49ff-baf1-5a3462cc607f.webp)

如果我们做同样的自我注意计算，只是用不同的权重矩阵进行8次不同的计算，我们就得到了8个不同的Z矩阵。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/afc6f389-aba3-4233-9780-15d4d8a311ed.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/afc6f389-aba3-4233-9780-15d4d8a311ed.webp)

这给我们留下了一点挑战。前馈层期望的不是8个矩阵——它期望的是单个矩阵(每个单词对应一个向量)。所以我们需要一种方法把这8个压缩成一个矩阵。

怎么做呢?我们将这些矩阵连接然后将它们乘以一个附加的权重矩阵WO。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/c1331cb0-d98f-4186-aeee-5b981a87f5f8.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/c1331cb0-d98f-4186-aeee-5b981a87f5f8.webp)

我们将整个流程组合在一张图中，如下图所示。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/811347f8-3709-498a-a763-0ac5a9110581.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/811347f8-3709-498a-a763-0ac5a9110581.webp)

正如我们目前所描述的，该模型缺少的一件事是解释输入序列中单词的顺序。Self—Attention无视距离，无视空间，但是位置信息还是非常重要的，下面我们看看如何解决这个问题。

为了解决这个问题，转换器在每个输入嵌入中添加一个向量。这些向量遵循模型学习的特定模式，这有助于模型确定每个单词的位置，或序列中不同单词之间的距离。这里的直觉是，将这些值添加到嵌入中，一旦它们被投射到Q/K/V向量中，在点积注意期间，就可以在嵌入向量之间提供有意义的距离。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/3c97ffd6-ea80-4f55-bd71-2c183dc5ce7a.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/3c97ffd6-ea80-4f55-bd71-2c183dc5ce7a.webp)

如果我们假设嵌入的维度是4，实际的位置编码应该是这样的:

[![](https://aitechtogether.com/wp-content/uploads/2022/05/038b50e8-3b52-4abf-beda-3612f4e3b712.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/038b50e8-3b52-4abf-beda-3612f4e3b712.webp)

这个模型会是什么样子？

在下面的图中，每一行对应于一个向量的位置编码。所以第一行就是我们在输入序列中嵌入第一个单词时要加上的向量。每行包含512个值—每个值都在1到-1之间。我们用颜色标记了它们，这样图案就清晰可见了。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/41eee6cb-e421-4aad-afc5-9efc54788314.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/41eee6cb-e421-4aad-afc5-9efc54788314.webp)

位置编码公式在Transformer的论文中描述（第3.5节）。您可以在get\_timing\_signal\_1d()https://github.com/tensorflow/tensor2tensor/blob/23bd23b9830059fbc349381b70d9429b5c40a139/tensor2tensor/layers/common\_attention.py中看到用于生成位置编码的代码。这不是位置编码的唯一可行方法。然而，它的优势在于能够缩放到序列中看不见的长度(例如，如果我们训练过的模型被要求翻译一个比我们训练集中的任何句子都长的句子)。

2020年7月更新：上面显示的位置编码来自Transformer的[transformer](https://aitechtogether.com/tag/transformer "transformer")2transformer实现。本文所示的方法略有不同，它不是直接串联，而是交织两个信号。下图显示了它的样子。下面是生成它的代码：https://github.com/jalammar/jalammar.github.io/blob/master/notebookes/transformer/transformer\_positional\_encoding\_graph.ipynb

[![](https://aitechtogether.com/wp-content/uploads/2022/05/bd38095b-ac64-4d3b-8169-8a58adb52c74.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/bd38095b-ac64-4d3b-8169-8a58adb52c74.webp)

介绍到这里，大家可能有个疑问，那就是Transformer的Loss是如何计算的？下面就一起探讨下：

假设我们正在训练我们的模型，这是我们训练阶段的第一步，我们正在用一个简单的例子来训练它——把“merci”翻译成“thanks”。

这意味着，我们希望输出是表示“谢谢”一词的概率分布。但由于模型没有经过训练，目前这种情况不太可能发生。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/58f851be-273b-4c2a-be5f-8d059403d4c7.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/58f851be-273b-4c2a-be5f-8d059403d4c7.webp)

如何比较两个概率分布?我们只是简单地用一个减去另一个。要了解更多细节，请看交叉熵和Kullback-Leibler散度。

但请注意，这是一个过于简化的示例。更现实地说，我们会使用比一个单词更长的句子。例如-输入:“je suis étudiant”，期望输出:“i am a student”。这实际上意味着，我们希望我们的模型连续输出概率分布，其中：

*   每个概率分布由宽度为vocab\_size的向量表示（在我们的示例中为6，但更实际的是词库的大小，可能比较大如30000或50000）
*   第一个概率分布在与单词“ i ”相关的单元格上的概率最高
*   第二个概率分布在与单词” am “相关的单元格中概率最高
*   以此类推，直到第5个输出分布指示‘’符号，它也有一个与10,000个元素词汇表相关联的单元格。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/e394c867-7375-42d8-9b59-e674cf213f4c.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/e394c867-7375-42d8-9b59-e674cf213f4c.webp)

在足够大的数据集上训练模型足够长的时间后，可能生成的概率分布如下：

[![](https://aitechtogether.com/wp-content/uploads/2022/05/cdfb3a2b-b2b8-4677-97fd-6c056f53ecc8.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/cdfb3a2b-b2b8-4677-97fd-6c056f53ecc8.webp)

现在，由于该模型每次产生一个输出，我们可以假设该模型从该概率分布中选择了概率最高的单词，而丢弃了其余的。这是一种方法（称为贪婪解码）。另一种方法是，抓住最前面的两个单词（例如，‘I’和‘a’），然后在下一步中，运行模型两次：一次假设第一个输出位置是单词‘I’，另一次假设第一个输出位置是单词‘a’，并且考虑到位置#1和#2，保留产生较少错误的版本。我们对2号和3号位置重复这个动作。这个方法被称为“beam search”，在我们的例子中，beam\_size为2（这意味着在任何时候，内存中都保留着两个部分假设。还有top\_beams，这两个都是可以试验的超参数。

1.  Attention Is All You Need (arxiv.org)[\[0\]](https://arxiv.org/abs/1706.03762)
2.  Depthwise Separable Convolutions for Neural Machine Translation[\[0\]](https://arxiv.org/abs/1706.03059)
3.  One Model To Learn Them All[\[0\]](https://arxiv.org/abs/1706.05137)
4.  Discrete Autoencoders for Sequence Models[\[0\]](https://arxiv.org/abs/1801.09797)
5.  Generating Wikipedia by Summarizing Long Sequences[\[0\]](https://arxiv.org/abs/1801.10198)
6.  Image Transformer[\[0\]](https://arxiv.org/abs/1802.05751)
7.  Training Tips for the Transformer Model[\[0\]](https://arxiv.org/abs/1804.00247)
8.  Self-Attention with Relative Position Representations[\[0\]](https://arxiv.org/abs/1803.02155)
9.  Fast Decoding in Sequence Models using Discrete Latent Variables[\[0\]](https://arxiv.org/abs/1803.03382)
10.  Adafactor: Adaptive Learning Rates with Sublinear Memory Cost[\[0\]](https://arxiv.org/abs/1804.04235)
11.  https://jalammar.github.io/illustrated-transformer/

> 介绍：杜宝坤，互联网行业从业者，十五年老兵。精通搜广推架构与算法，并且从0到1带领团队构建了京东的联邦学习解决方案9N-FL，同时主导了联邦学习框架与联邦开门红业务。  
> 个人比较喜欢学习新东西，乐于钻研技术。基于从全链路思考与决策技术规划的考量，研究的领域比较多，从工程架构、大数据到机器学习算法与算法框架、隐私计算均有涉及。欢迎喜欢技术的同学和我交流，邮箱：baokun06@163.com

很长一段时间以来，我一直在写自己的博客。由于涉及到很多技术领域，所以对高并发高性能、分布式、传统机器学习算法和框架、深度学习算法和框架、密码安全、隐私计算、联邦学习、大数据等非常感兴趣. 参与。曾主导零售联合学习等多个重大项目，并多次在社区分享。此外，他坚持写原创博客，多篇文章阅读量过万。公众号的光头码农可以根据话题持续阅读。我已经按照学习路线对章节进行了排序。话题是公众号下方红色标记的话题。您可以单击它查看该主题下的许多文章。文章，比如下图（题目分为：1、隐私计算2、联邦学习3、机器学习框架4、机器学习算法5、高性能计算6、广告算法7、程序寿命），知乎账号是同样关注 专利就行了。

[![](https://aitechtogether.com/wp-content/uploads/2022/05/69a32f06-3bbf-455e-81b7-df8b79d47464.webp)
](https://aitechtogether.com/wp-content/uploads/2022/05/69a32f06-3bbf-455e-81b7-df8b79d47464.webp)

一切缘法如梦泡、露、电，应如此观。