# 【AI实战】训练第一个AI模型：MNIST手写数字识别模型
![](https://oscimg.oschina.net/oscnet/df37242c0e75538192040a8805b89ce4351.jpg)

在上篇文章中，我们已经把 AI 的基础环境搭建好了（见文章：[Ubuntu + conda + tensorflow + GPU + pycharm 搭建 AI 基础环境](https://my.oschina.net/u/876354/blog/1924805)），接下来将基于 tensorflow 训练第一个 AI 模型：MNIST 手写数字识别模型。  
MNIST 是一个经典的手写数字数据集，来自美国国家标准与技术研究所，由不同人手写的 0 至 9 的数字构成，由 60000 个训练样本集和 10000 个测试样本集构成，每个样本的尺寸为 28x28，以二进制格式存储，如下图所示：  
![](https://oscimg.oschina.net/oscnet/62a35c6835c3221cb2928bf04c28f4e138a.jpg)
   
MNIST 手写数字识别模型的主要任务是：**输入一张手写数字的图像，然后识别图像中手写的是哪个数字。** 

  
该模型的目标明确、任务简单，数据集规范、统一，数据量大小适中，在普通的 PC 电脑上都能训练和识别，堪称是深度学习领域的 “Hello World!”，学习 AI 的入门必备模型。

**0、AI 建模主要步骤**  
在构建 AI 模型时，一般有以下主要步骤：准备数据、数据预处理、划分数据集、配置模型、训练模型、评估优化、模型应用，如下图所示：  
![](https://oscimg.oschina.net/oscnet/9c4d61b2595dac34bbc6f012e04621f1a8f.jpg)
   
下面将按照主要步骤进行介绍。  
【注意】由于 MNIST 数据集太经典了，很多深度学习书籍在介绍该入门模型案例时，基本上就是直接下载获取数据，然后就进行模型训练，最后得出一个准确率出来。但这样的入门案例学习后，当要拿自己的数据来训练模型，却往往不知该如何处理数据、如何训练、如何应用。在本文，将分两种情况进行介绍：（1）使用 MNIST 数据（本案例），（2）使用自己的数据。

  
下面将针对模型训练的各个主要环节进行介绍，便于读者快速迁移去训练自己的数据模型。

**1、准确数据**  
准备数据是训练模型的第一步，基础数据可以是网上公开的数据集，也可以是自己的数据集。视觉、语音、语言等各种类型的数据在网上都能找到相应的数据集。  
**（1）使用 MNIST 数据（本案例）**  
MNIST 数据集由于非常经典，已集成在 tensorflow 里面，可以直接加载使用，也可以从 MNIST 的官网上（http://yann.lecun.com/exdb/mnist/） 直接下载数据集，代码如下：

```
from tensorflow.examples.tutorials.mnist import input_data


data_dir='/home/roger/data/work/tensorflow/data/mnist'

mnist = input_data.read_data_sets(data_dir, one_hot=True) 
```

集成或下载的 MNIST 数据集已经是打好标签了，直接使用就行。

  
**（2）使用自己的数据**  
如果是使用自己的数据集，在准备数据时的重要工作是 “标注数据”，也就是对数据进行打标签，主要的标注方式有：  
① 整个文件打标签。例如 MNIST 数据集，每个图像只有 1 个数字，可以从 0 至 9 建 10 个文件夹，里面放相应数字的图像；也可以定义一个规则对图像进行命名，如按标签 + 序号命名；还可以在数据库里面创建一张对应表，存储文件名与标签之间的关联关系。如下图：  
![](https://oscimg.oschina.net/oscnet/68a5fa875c11c8d082e774ec5ca1f7f2bc3.jpg)
   
② 圈定区域打标签。例如 ImageNet 的物体识别数据集，由于每张图片上有各种物体，这些物体位于不同位置，因此需要圈定某个区域进行标注，目前比较流行的是 VOC2007、VOC2012 数据格式，这是使用 xml 文件保存图片中某个物体的名称（name）和位置信息（xmin,ymin,xmax,ymax）。  
如果图片很多，一张一张去计算位置信息，然后编写 xml 文件，实在是太耗时耗力了。所幸，有一位大神开源了一个数据标注工具 labelImg（https://github.com/tzutalin/labelImg），只要在界面上画框标注，就能自动生成 VOC 格式的 xml 文件了，非常方便，如下图所示：  
![](https://oscimg.oschina.net/oscnet/b39be86d04be933653680700702924dc30d.jpg)
   
③ 数据截段打标签。针对语音识别、文字识别等，有些是将数据截成一段一段的语音或句子，然后在另外的文件中记录对应的标签信息。

**2、数据预处理**  
在准备好基础数据之后，需要根据模型需要对基础数据进行相应的预处理。  
**（1）使用 MNIST 数据（本案例）**  
由于 MNIST 数据集的尺寸统一，只有黑白两种像素，无须再进行额外的预处理，直接拿来建模型就行。  
**（2）使用自己的数据**  
而如果是要训练自己的数据，根据模型需要一般要进行以下预处理：  
 ![](https://oscimg.oschina.net/oscnet/2e7ffa9a92c9c36c3df43cc271d5f267841.jpg)
  
**a. 统一格式：** 即统一基础数据的格式，例如图像数据集，则全部统一为 jpg 格式；语音数据集，则全部统一为 wav 格式；文字数据集，则全部统一为 UTF-8 的纯文本格式等，方便模型的处理；  
**b. 调整尺寸：** 根据模型的输入要求，将样本数据全部调整为统一尺寸。例如 LeNet 模型是 32x32，AlexNet 是 224x224，VGG 是 224x224 等；  
**c. 灰度化：** 根据模型需要，有些要求输入灰度图像，有些要求输入 RGB 彩色图像；  
**d. 去噪平滑：** 为提升输入图像的质量，对图像进行去噪平滑处理，可使用中值滤波器、高斯滤波器等进行图像的去噪处理。如果训练数据集的图像质量很好了，则无须作去噪处理；  
**e. 其它处理：** 根据模型需要进行直方图均衡化、二值化、腐蚀、膨胀等相关的处理；  
**f. 样本增强：** 有一种观点认为神经网络是靠数据喂出来的，如果能够增加训练数据的样本量，提供海量数据进行训练，则能够有效提升算法的质量。常见的样本增强方式有：水平翻转图像、随机裁剪、平移变换，颜色、光照变换等，如下图所示：

![](https://oscimg.oschina.net/oscnet/d00c298a95e844188c3ec4a128106605096.jpg)

**3、划分数据集**  
在训练模型之前，需要将样本数据划分为训练集、测试集，有些情况下还会划分为训练集、测试集、验证集。  
**（1）使用 MNIST 数据（本案例）**  
本案例要训练模型的 MNIST 数据集，已经提供了训练集、测试集，代码如下：

```
 train_xdata = mnist.train.images
test_xdata = mnist.test.images


train_labels = mnist.train.labels
test_labels = mnist.test.labels
```

**（2）使用自己的数据**  
如果是要划分自己的数据集，可使用 scikit-learn 工具进行划分，代码如下：

```
from sklearn.cross_validation import train_test_split




X_train,X_test,y_train,y_test=train_test_split(X_data,y_labels,test_size=0.25,random_state=33)
```

**4、配置模型**  
接下来是选择模型、配置模型参数，建议先阅读深度学习经典模型的文章（见文章：[大话卷积神经网络模型](https://my.oschina.net/u/876354/blog/1620906)），便于快速掌握深度学习模型的相关知识。  
**（1）选择模型**  
本案例将采用 LeNet 模型来训练 MNIST 手写数字模型，LeNet 是一个经典卷积神经网络模型，结构简单，针对 MNIST 这种简单的数据集可达到比较好的效果，LeNet 模型的原理介绍请见文章（[大话 CNN 经典模型：LeNet](https://my.oschina.net/u/876354/blog/1632862)），网络结构图如下：  
![](https://oscimg.oschina.net/oscnet/84e967f4c179074ef393f3dd24671c67f12.jpg)
   
**（2）设置参数**  
在训练模型时，一般要设置的参数有：

```
step_cnt=10000    
batch_size = 100    
learning_rate = 0.001    
```

除此之外还有卷积层权重和偏置、池化层权重、全联接层权重和偏置、优化函数等等，根据模型需要进行设置。

**5、训练模型**  
接下来便是根据选择好的模型，构建网络，然后开始训练。  
**（1）构建模型**  
本案例按照 LeNet 的网络模型结构，构建网络模型，网络结果如下  
![](https://oscimg.oschina.net/oscnet/0f750c7dc321fdc81a024063adeb812ceb8.jpg)
   
代码如下：

```
 x = tf.placeholder("float", shape=[None, 784])

y_ = tf.placeholder("float", shape=[None, 10])

x_image = tf.reshape(x, [-1, 28, 28, 1])


keep_prob = tf.placeholder(tf.float32)



conv1_weights = tf.get_variable("conv1_weights", [5, 5, 1, 32], initializer=tf.truncated_normal_initializer(stddev=0.1))
conv1_biases = tf.get_variable("conv1_biases", [32], initializer=tf.constant_initializer(0.0))
conv1 = tf.nn.conv2d(x_image, conv1_weights, strides=[1, 1, 1, 1], padding='SAME')
relu1 = tf.nn.relu(tf.nn.bias_add(conv1, conv1_biases))



pool1 = tf.nn.max_pool(relu1, ksize=[1, 2, 2, 1], strides=[1, 2, 2, 1], padding='SAME')



conv2_weights = tf.get_variable("conv2_weights", [5, 5, 32, 64], initializer=tf.truncated_normal_initializer(stddev=0.1))
conv2_biases = tf.get_variable("conv2_biases", [64], initializer=tf.constant_initializer(0.0))
conv2 = tf.nn.conv2d(pool1, conv2_weights, strides=[1, 1, 1, 1], padding='SAME')
relu2 = tf.nn.relu(tf.nn.bias_add(conv2, conv2_biases))



pool2 = tf.nn.max_pool(relu2, ksize=[1, 2, 2, 1], strides=[1, 2, 2, 1], padding='SAME')


fc1_weights = tf.get_variable("fc1_weights", [7 * 7 * 64, 1024],
                              initializer=tf.truncated_normal_initializer(stddev=0.1))
fc1_baises = tf.get_variable("fc1_baises", [1024], initializer=tf.constant_initializer(0.1))
pool2_vector = tf.reshape(pool2, [-1, 7 * 7 * 64])
fc1 = tf.nn.relu(tf.matmul(pool2_vector, fc1_weights) + fc1_baises)


fc1_dropout = tf.nn.dropout(fc1, keep_prob)


fc2_weights = tf.get_variable("fc2_weights", [1024, 10],
                              initializer=tf.truncated_normal_initializer(stddev=0.1))  
fc2_biases = tf.get_variable("fc2_biases", [10], initializer=tf.constant_initializer(0.1))
fc2 = tf.matmul(fc1_dropout, fc2_weights) + fc2_biases


y_conv = tf.nn.softmax(fc2)
```

**（2）训练模型**  
在训练模型时，需要选择优化器，也就是说要告诉模型以什么策略来提升模型的准确率，一般是选择交叉熵损失函数，然后使用优化器在反向传播时最小化损失函数，从而使模型的质量在不断迭代中逐步提升。  
代码如下：

```
 cross_entropy = tf.reduce_mean(-tf.reduce_sum(y_ * tf.log(y_conv), reduction_indices=[1]))

train_step = tf.train.AdamOptimizer(learning_rate).minimize(cross_entropy)

correct_prediction = tf.equal(tf.argmax(y_conv, 1), tf.argmax(y_, 1))

accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))

saver=tf.train.Saver()
with tf.Session() as sess:
    tf.global_variables_initializer().run()

    for step in range(step_cnt):
        batch = mnist.train.next_batch(batch_size)
        if step % 100 == 0:
            
            train_accuracy = accuracy.eval(feed_dict={x: batch[0], y_: batch[1], keep_prob: 1.0})
            print("step %d, training accuracy %g" % (step, train_accuracy))
            saver.save(sess,model_dir+'/my_mnist_model.ctpk',global_step=step)
        train_step.run(feed_dict={x: batch[0], y_: batch[1], keep_prob: 0.8})

    
    print("test accuracy %g" % accuracy.eval(feed_dict={x: mnist.test.images, y_: mnist.test.labels, keep_prob: 1.0}))
```

训练的结果如下，由于 MNIST 数据集比较简单，模型训练很快就达到 99% 的准确率，如下图所示：  
![](https://oscimg.oschina.net/oscnet/d085bed5a7e25a7cb032491dbae9d9f7bf2.jpg)
   
模型训练后保存的结果如下图所示：

![](https://oscimg.oschina.net/oscnet/07001c5e613ba06d453e0dc09adf8f6950d.jpg)

**6、评估优化**  
在使用训练数据完成模型的训练之后，再使用测试数据进行测试，了解模型的泛化能力，代码如下

```
 test_acc=accuracy.eval(feed_dict={x: test_xdata, y_: test_labels, keep_prob: 1.0})
print("test accuracy %g" %test_acc)
```

模型测试结果如下：  
![](https://oscimg.oschina.net/oscnet/5aa7696b891a6940773071bb929775cc113.jpg)
   
**7、模型应用**  
模型训练完成后，将模型保存起来，当要实际应用时，则通过加载模型，输入图像进行应用。代码如下：

```
 saver = tf.train.Saver()
with tf.Session() as sess:
    saver.restore(sess, tf.train.latest_checkpoint(model_dir))

    
    test_len=len(mnist.test.images)
    test_idx=random.randint(0,test_len-1)
    x_image=mnist.test.images[test_idx]
    y=np.argmax(mnist.test.labels[test_idx])

    
    y_conv = tf.argmax(y_conv,1)
    pred=sess.run(y_conv,feed_dict={x:[x_image], keep_prob: 1.0})

    print('正确：',y,'，预测：',pred[0])
```

使用模型进行测试的结果如下图：  
![](https://oscimg.oschina.net/oscnet/94c6c1e2e54ac1d71f4470e896ae8f616d4.jpg)

至此，一个完整的模型训练和应用的过程就介绍完了。  
**接下来还会有更多 AI 实战的精彩内容，敬请期待！**

**获取完整源代码**

想要阅读本案例的 完整代码，请关注本人公众号 “大数据与人工智能 Lab”（BigdataAILab），然后回复 “**代码**” 关键字可查看 **完整源代码**。

![](https://static.oschina.net/uploads/space/2018/0213/155533_IdYn_876354.jpg)

![](https://oscimg.oschina.net/oscnet/480d26aef8f0225d786abce5deaf342ac84.jpg)

**推荐相关阅读**

*   [【AI 实战】快速掌握 TensorFlow（一）：基本操作](https://my.oschina.net/u/876354/blog/1930175)
*   [【AI 实战】快速掌握 TensorFlow（二）：计算图、会话](https://my.oschina.net/u/876354/blog/1930490)
*   [【AI 实战】快速掌握 TensorFlow（三）：激励函数](https://my.oschina.net/u/876354/blog/1937296)
*   [【AI 实战】快速掌握 TensorFlow（四）：损失函数](https://my.oschina.net/u/876354/blog/1940819)
*   [【AI 实战】搭建基础环境](https://my.oschina.net/u/876354/blog/1924805)
*   [【AI 实战】训练第一个模型](https://my.oschina.net/u/876354/blog/1926060)
*   [【AI 实战】编写人脸识别程序](https://my.oschina.net/u/876354/blog/1926679)
*   [【AI 实战】动手训练目标检测模型（SSD 篇）](http://my.oschina.net/u/876354/blog/1927351)
*   [【AI 实战】动手训练目标检测模型（YOLO 篇）](https://my.oschina.net/u/876354/blog/1927881)

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
*   [27 种深度学习经典模型](https://my.oschina.net/u/876354/blog/1924779)
*   [浅说 “迁移学习”](https://my.oschina.net/u/876354/blog/1614883)
*   [什么是 “强化学习”](https://my.oschina.net/u/876354/blog/1614879)
*   [AlphaGo 算法原理浅析](https://my.oschina.net/u/876354/blog/1594849)
*   [大数据究竟有多少个 V](https://my.oschina.net/u/876354/blog/1604254)
*   [Apache Hadoop 2.8 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/993836)
*   [Apache Hive 2.1.1 安装配置超详细教程](https://my.oschina.net/u/876354/blog/1057639)
*   [Apache HBase 1.2.6 完全分布式集群搭建超详细教程](https://my.oschina.net/u/876354/blog/1163018)
*   [离线安装 Cloudera Manager 5 和 CDH5（最新版 5.13.0）超详细教程](https://my.oschina.net/u/876354/blog/1605320)

关注本人公众号 “大数据与人工智能 Lab”（BigdataAILab），获取更多信息**。** 

![](https://static.oschina.net/uploads/space/2018/0213/155533_IdYn_876354.jpg)