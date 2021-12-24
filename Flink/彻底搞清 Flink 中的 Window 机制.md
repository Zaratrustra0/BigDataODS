# 彻底搞清 Flink 中的 Window 机制
![](https://mmbiz.qpic.cn/mmbiz_gif/Pn4Sm0RsAuiaPn8q2TmsZwA4RUjpMdPTZ3RKT9f7INA6jnnN4rn5QLB05fLOgYqaqjpmeBZo2FsxahGl5yiaP2ZA/640?wx_fmt=gif)

【CSDN 编者按】Window 是处理无限流的核心。Flink 认为 Batch 是 Streaming 的一个特例，所以 Flink 底层引擎是一个流式引擎，在上面实现了流处理和批处理。Flink 提供了非常完美的窗口机制，这是 Flink 最大的亮点之一。

作者 | 柯同学       责编 | 欧阳姝黎

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNyknDL3lnic0ZlKZUKpxbZTVIgRDZ1rytsqfZ0WysSicUWN2bqK1mVmJw/640?wx_fmt=png)

flink-window

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qkyxEibG4Uy3ibtP06LgbDfGUObnDFMEz9icU556dwCZyNgtwXH29EwH6Q/640?wx_fmt=png)

在流处理应用中，数据是连续不断的，因此我们不可能等到所有数据都到了才开始处理。当然我们可以每来一个消息就处理一次，但是有时我们需要做一些聚合类的处理，例如：在过去的 1 分钟内有多少用户点击了我们的网页。在这种情况下，我们必须定义一个窗口，用来收集最近一分钟内的数据，并对这个窗口内的数据进行计算。

Flink 认为 Batch 是 Streaming 的一个特例，所以 Flink 底层引擎是一个流式引擎，在上面实现了流处理和批处理。而窗口（window）就是从 Streaming 到 Batch 的一个桥梁。

-   一个 Window 代表有限对象的集合。一个窗口有一个最大的时间戳，该时间戳意味着在其代表的某时间点——所有应该进入这个窗口的元素都已经到达
-   Window 就是用来对一个无限的流设置一个有限的集合，在有界的数据集上进行操作的一种机制。window 又可以分为基于时间（Time-based）的 window 以及基于数量（Count-based）的 window。
-   Flink DataStream API 提供了 Time 和 Count 的 window，同时增加了基于 Session 的 window。同时，由于某些特殊的需要，DataStream API 也提供了定制化的 window 操作，供用户自定义 window。

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qOfecjttGpW3J3tvO19FFTIAqx9ibBwTIZv6fzIvOlLJUicfaicjQJpZ1w/640?wx_fmt=png)

**窗口的组成**  

* * *

### **窗口分配器**

-   ### assignWindows 将某个带有时间戳 timestamp 的元素 element 分配给一个或多个窗口，并返回窗口集合
-   getDefaultTrigger 返回跟 WindowAssigner 关联的默认触发器
-   getWindowSerializer 返回 WindowAssigner 分配的窗口的序列化器
-   窗口分配器定义如何将数据元分配给窗口。这是通过 WindowAssigner 在 window(…)（对于被 Keys 化流）或 windowAll()（对于非被 Keys 化流）调用中指定您的选择来完成的。
-   WindowAssigner 负责将每个传入数据元分配给一个或多个窗口。Flink 带有预定义的窗口分配器，用于最常见的用例  
    即翻滚窗口， 滑动窗口，会话窗口和全局窗口。
-   您还可以通过扩展 WindowAssigner 类来实现自定义窗口分配器。
-   所有内置窗口分配器（全局窗口除外）都根据时间为窗口分配数据元，这可以是处理时间或事件时间。

### **State**

-   状态，用来存储窗口内的元素，如果有 AggregateFunction，则存储的是增量聚合的中间结果。

### **窗口函数**

选择合适的计算函数，减少开发代码量提高系统性能

#### 增量聚合函数 (窗口只维护状态)

-   #### ReduceFunction
-   #### AggregateFunction
-   #### FoldFunction

#### 全量聚合函数 (窗口维护窗口内的数据)

-   #### ProcessWindowFunction
-   #### 全量计算
-   #### 支持功能更加灵活
-   #### 支持状态操作

### **触发器**

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNsRN9LNOdavic5Ms9wibA8j3YbVMIWQ0ZGLnARmxiawfS31jaZuu4zoXSg/640?wx_fmt=png)

image-20210202200655485

-   EventTimeTrigger 基于事件时间的触发器，对应 onEventTime
-   ProcessingTimeTrigger  
    基于当前系统时间的触发器，对应 onProcessingTime  
    ProcessingTime 有最好的性能和最低的延迟。但在分布式计算环境中 ProcessingTime 具有不确定性，相同数据流多次运行有可能产生不同的计算结果。
-   ContinuousEventTimeTrigger
-   ContinuousProcessingTimeTrigger
-   CountTrigger
-   Trigger 确定何时窗口函数准备好处理窗口（由窗口分配器形成）。每个都有默认值。  
    如果默认触发器不符合您的需要，您可以使用指定自定义触发器。WindowAssignerTriggertrigger(…)
-   触发器界面有五种方法可以 Trigger 对不同的事件做出反应：


-   onElement() 为添加到窗口的每个数据元调用该方法。
-   onEventTime() 在注册的事件时间计时器触发时调用该方法。
-   onProcessingTime() 在注册的处理时间计时器触发时调用该方法。
-   该 onMerge() 方法与状态触发器相关，并且当它们的相应窗口合并时合并两个触发器的状态，例如当使用会话窗口时。
-   最后，该 clear() 方法在移除相应窗口时执行所需的任何动作。


-   默认触发器


-   默认触发器 GlobalWindow 是 NeverTrigger 从不触发的。因此，在使用时必须定义自定义触发器 GlobalWindow。
-   通过使用 trigger() 您指定触发器会覆盖 a 的默认触发器 WindowAssigner。例如，如果指定 a CountTrigger，TumblingEventTimeWindows 则不再根据时间进度获取窗口，  
    而是仅按计数。现在，如果你想根据时间和数量做出反应，你必须编写自己的自定义触发器。
-   event-time 窗口分配器都有一个 EventTimeTrigger 作为默认触发器。该触发器在 watermark 通过窗口末尾时出发。 

#### **触发器分类**

##### **CountTrigger**

一旦窗口中的数据元数量超过给定限制，就会触发。所以其触发机制实现在 onElement 中

##### **ProcessingTimeTrigger**

基于处理时间的触发。

##### **EventTimeTrigger**

根据 watermarks 度量的事件时间进度进行触发。

##### **PurgingTrigger**

-   另一个触发器作为参数作为参数并将其转换为清除触发器。
-   其作用是在 Trigger 触发窗口计算之后将窗口的 State 中的数据清除。

    ![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNpTYUd1apgm5GpHeKjpBX6qHV6vfGmlVdn0jJSLvtFBRicFCkd02UMSw/640?wx_fmt=png)

    image-20210202200710573
-   前两条数据先后于 20:01 和 20:02 进入窗口，此时 State 中的值更新为 3，同时到了 Trigger 的触发时间，输出结果为 3。

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgN7Bhfjbjc42hpe7oLzGGI1tVPTvjEEaRQDQs2JuZfmR49oMUHjTScSw/640?wx_fmt=png)

image-20210202200733128

-   由于 PurgingTrigger 的作用，State 中的数据会被清除。

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgN7v19XAn2rDpDCR5A5jNz5sytaA6178gtfhZ6YWFibw350ILWlrojrww/640?wx_fmt=png)

image-20210202200744793

##### **DeltaTrigger**

###### **DeltaTrigger 的应用**

-   有这样一个车辆区间测试的需求，车辆每分钟上报当前位置与车速，每行进 10 公里，计算区间内最高车速。

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNlG8aSjAdwJz1o59KZwmT0QcYLp7ibEassLDvFzES4TwPjicVuYxhWkfw/640?wx_fmt=png)

image-20210202200802480

#### **触发器原型**

-   onElement
-   onProcessingTime
-   onEventTime
-   onMerge
-   clear

#### **说明**

-   TriggerResult 可以是以下之一


-   CONTINUE 什么都不做
-   FIRE_AND_PURGE 触发计算，然后清除窗口中的元素
-   FIRE 触发计算  默认情况下，内置的触发器只返回 FIRE，不会清除窗口状态。
-   PURGE 清除窗口中的元素  


-   所有的事件时间窗口分配器都有一个 EventTimeTrigger 作为默认触发器。一旦 watermark 到达窗口末尾，这个触发器就会被触发。 
-   全局窗口 (GlobalWindow) 的默认触发器是永不会被触发的 NeverTrigger。因此，在使用全局窗口时，必须自定义一个触发器。
-   通过使用 trigger() 方法指定触发器，将会覆盖窗口分配器的默认触发器。例如，如果你为 TumblingEventTimeWindows 指定 CountTrigger，  
    那么不会再根据时间进度触发窗口，而只能通过计数。目前为止，如果你希望基于时间以及计数进行触发，则必须编写自己的自定义触发器。

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qgxOukGzL6akS7ibtpEgg5xkibkgpJxVwylfcDicBABiaXlgBLsd1Afds1w/640?wx_fmt=png)

**窗口的分类**  

* * *

-   根据窗口是否调用 keyBy 算子 key 化，分为被 Keys 化 Windows 和非被 Keys 化 Windows；

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNn62hkqzePx4l09oFaTjRTfrv2suyJ4NkdThDMd4Xyr9uwYEbweick1Q/640?wx_fmt=png)

flink window 图解

-   根据窗口的驱动方式，分为时间驱动（Time Window）、数据驱动（Count Window）；
-   根据窗口的元素分配方式，分为滚动窗口（tumbling windows）、滑动窗口（sliding windows）、会话窗口（session windows）以及全局窗口（global windows）

### **被 Keys 化 Windows**

可以理解为按照原始数据流中的某个 key 进行分类，拥有同一个 key 值的数据流将为进入同一个 window，多个窗口并行的逻辑流

    stream       .keyBy(...)               <-  keyed versus non-keyed windows       .window(...)              <-  required: "assigner"      [.trigger(...)]            <-  optional: "trigger" (else default trigger)      [.evictor(...)]            <-  optional: "evictor" (else no evictor)      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)       .reduce/aggregate/fold/apply()      <-  required: "function"      [.getSideOutput(...)]      <-  optional: "output tag"

### **非被 Keys 化 Windows**

-   不做分类，每进入一条数据即增加一个窗口，多个窗口并行，每个窗口处理 1 条数据
-   WindowAll 将元素按照某种特性聚集在一起，该函数不支持并行操作，默认的并行度就是 1，所以如果使用这个算子的话需要注意一下性能问题


    stream     .windowAll(...)           <-  required: "assigner"    [.trigger(...)]            <-  optional: "trigger" (else default trigger)    [.evictor(...)]            <-  optional: "evictor" (else no evictor)    [.allowedLateness(...)]    <-  optional: "lateness" (else zero)    [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)     .reduce/aggregate/fold/apply()      <-  required: "function"    [.getSideOutput(...)]      <-  optional: "output tag"

#### **区别**

-   对于被 Key 化的数据流，可以将传入事件的任何属性用作键（此处有更多详细信息）。
-   拥有被 Key 化的数据流将允许您的窗口计算由多个任务并行执行，因为每个逻辑被 Key 化的数据流可以独立于其余任务进行处理。  
    引用相同 Keys 的所有数据元将被发送到同一个并行任务。

### **Time-Based window(基于时间的窗口)**

每一条记录来了以后会根据时间属性值采用不同的 window assinger 方法分配给一个或者多个窗口，分为滚动窗口（Tumbling windows）和滑动窗口（Sliding windows）。

-   EventTime 数据本身携带的时间，默认的时间属性；
-   ProcessingTime 处理时间；
-   IngestionTime 数据进入 flink 程序的时间；

#### **Tumbling windows（滚动窗口）**

滚动窗口下窗口之间不重叠，且窗口长度是固定的。我们可以用 TumblingEventTimeWindows 和 TumblingProcessingTimeWindows 创建一个基于 Event Time 或 Processing Time 的滚动时间窗口。

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNjjfvuibZcOHtWAMbcQ1uJR8QQP5vmRD5hQUA4OLnicCs4ia9tia3ypNq5Q/640?wx_fmt=png)

tumb-window

下面示例以滚动时间窗口（TumblingEventTimeWindows）为例，默认模式是 TimeCharacteristic.ProcessingTime 处理时间

    /** The time characteristic that is used if none other is set. */private static final TimeCharacteristic DEFAULT_TIME_CHARACTERISTIC = TimeCharacteristic.ProcessingTime;

所以如果使用 Event Time 即数据的实际产生时间，需要通过 senv.setStreamTimeCharacteristic 指定

    // 指定使用数据的实际时间senv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);DataStream<T> input = ...;// tumbling event-time windowsinput    .keyBy(<key selector>)    .window(TumblingEventTimeWindows.of(Time.seconds(5)))    .<windowed transformation>(<window function>);// tumbling processing-time windowsinput    .keyBy(<key selector>)    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))    .<windowed transformation>(<window function>);// 这里减去8小时，表示用UTC世界时间input    .keyBy(<key selector>)    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))    .<windowed transformation>(<window function>);

#### **Sliding windows（滑动窗口）**

滑动窗口以一个步长（Slide）不断向前滑动，窗口的长度固定。使用时，我们要设置 Slide 和 Size。Slide 的大小决定了 Flink 以多大的频率来创建新的窗口，Slide 较小，窗口的个数会很多。Slide 小于窗口的 Size 时，相邻窗口会重叠，一个事件会被分配到多个窗口；Slide 大于 Size，有些事件可能被丢掉。

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNmjiccAbAYx7icMJSGJJlTMtFFrD3hS15GAFBIotOAeHsPDnibznEGGlgQ/640?wx_fmt=png)

slide-window

同理，如果是滑动时间窗口，也是类似的：

    // 窗口的大小是10s，每5s滑动一次，也就是5s计算一次.timeWindow(Time.seconds(10), Time.seconds(5))

这里使用的是 timeWindow，通常使用 window，那么两者的区别是什么呢？

timeWindow 其实判断时间的处理模式是 ProcessingTime 还是 SlidingEventTimeWindows，帮我们判断好了，调用方法直接传入 (Time size, Time slide) 这两个参数就好了，如果是使用. window 方法，则需要自己来判断，就是前者写法更简单一些。

    public WindowedStream<T, KEY, TimeWindow> timeWindow(Time size, Time slide) {    if (environment.getStreamTimeCharacteristic() == TimeCharacteristic.ProcessingTime) {        return window(SlidingProcessingTimeWindows.of(size, slide));    } else {        return window(SlidingEventTimeWindows.of(size, slide));    }}

### **Count-Based window （基于计数的窗口）**

Count Window 是根据元素个数对数据流进行分组的，也分滚动（tumb）和滑动（slide）。

**Tumbling Count Window**

当我们想要每 100 个用户购买行为事件统计购买总数，那么每当窗口中填满 100 个元素了，就会对窗口进行计算，这种窗口我们称之为翻滚计数窗口（Tumbling Count Window），上图所示窗口大小为 3 个。通过使用 DataStream API，我们可以这样实现：

    // Stream of (userId, buyCnts)val buyCnts: DataStream[(Int, Int)] = ...val tumblingCnts: DataStream[(Int, Int)] = buyCnts  // key stream by sensorId  .keyBy(0)  // tumbling count window of 100 elements size  .countWindow(100)  // compute the buyCnt sum   .sum(1)

**Sliding Count Window**

当然 Count Window 也支持 Sliding Window，虽在上图中未描述出来，但和 Sliding Time Window 含义是类似的，例如计算每 10 个元素计算一次最近 100 个元素的总和，代码示例如下。

    val slidingCnts: DataStream[(Int, Int)] = vehicleCnts  .keyBy(0)  // sliding count window of 100 elements size and 10 elements trigger interval  .countWindow(100, 10)  .sum(1)

### **会话 (session) 窗口**

-   SessionWindow 中的 Gap 是一个非常重要的概念，它指的是 session 之间的间隔。
-   如果 session 之间的间隔大于指定的间隔，数据将会被划分到不同的 session 中。比如，设定 5 秒的间隔，0-5 属于一个 session，5-10 属于另一个 session

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNVUnETxwczjlOvNde5WHvibjW0vz1bsQB4J8Jvg9xYXj6P2ic0QoVgInQ/640?wx_fmt=png)

session-window

    DataStream<T> input = ...;// event-time session windows with static gapinput    .keyBy(<key selector>)    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))    .<windowed transformation>(<window function>);// event-time session windows with dynamic gapinput    .keyBy(<key selector>)    .window(EventTimeSessionWindows.withDynamicGap((element) -> {        // determine and return session gap    }))    .<windowed transformation>(<window function>);// processing-time session windows with static gapinput    .keyBy(<key selector>)    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))    .<windowed transformation>(<window function>);// processing-time session windows with dynamic gapinput    .keyBy(<key selector>)    .window(ProcessingTimeSessionWindows.withDynamicGap((element) -> {        // determine and return session gap    }))    .<windowed transformation>(<window function>);

### **Global Windows（全局窗口）**

![](https://mmbiz.qpic.cn/mmbiz_png/cAoClgx53VvIxKQPo8aWfI44l6pcacgNxTYCA1IsbYwPia3dh6ibziaCdHm6V8wM8xEQG0tyzW8Rk6uYsAQMDzjSA/640?wx_fmt=png)

global-window

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qA3wIF4M1rgJPLkc2t7tiatN4Txs9MPGEdYzyfH069BSHiatSpOjibBKlg/640?wx_fmt=png)

### **总结**

SlidingEventTimeWindows、SlidingProcessingTimeWindows、TumblingEventTimeWindows、TumblingProcessingTimeWindows：

-   **基于时间的滑动窗口**


-   SlidingEventTimeWindows
-   SlidingProcessingTimeWindows


-   **基于时间的翻滚窗口**


-   TumblingEventTimeWindows
-   TumblingProcessingTimeWindows


-   **基于计数的滑动窗口**


-   countWindow(100, 10)


-   **基于计数的翻滚窗口**


-   countWindow(100)


-   **会话窗口**  
    会话窗口：一条记录一个窗口


-   ProcessingTimeSessionWindows
-   EventTimeSessionWindows


-   **全局窗口 (GlobalWindows)**


-   GlobalWindow 是一个全局窗口，被实现为单例模式。其 maxTimestamp 被设置为 Long.MAX_VALUE。
-   该类内部有一个静态类定义了 GlobalWindow 的序列化器：Serializer。

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qZDicaoukpArCbp8E1Z8kCR4PoiazstRK6YZq6iaon8P2yaG3tqgjBNt9w/640?wx_fmt=png)

**延迟**  

* * *

默认情况下，当水印超过窗口末尾时，会删除延迟数据元。

但是，Flink 允许为窗口 算子指定最大允许延迟。允许延迟指定数据元在被删除之前可以延迟多少时间，并且其默认值为 0。

在水印通过窗口结束之后但在通过窗口结束加上允许的延迟之前到达的数据元，仍然添加到窗口中。

根据使用的触发器，延迟但未丢弃的数据元可能会导致窗口再次触发。就是这种情况 EventTimeTrigger。

当指定允许的延迟大于 0 时，在水印通过窗口结束后保持窗口及其内容。在这些情况下，当迟到但未掉落的数据元到达时，它可能触发窗口的另一次触发。

这些射击被称为 late firings，因为它们是由迟到事件触发的，与之相反的 main firing 是窗口的第一次射击。在会话窗口的情况下，后期点火可以进一步导致窗口的合并，因为它们可以 “桥接” 两个预先存在的未合并窗口之间的间隙。

后期触发发出的数据元应该被视为先前计算的更新结果，即，您的数据流将包含同一计算的多个结果。根据您的应用程序，您需要考虑这些重复的结果或对其进行重复数据删除。

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qVYsK9rW3PnGicfctGFTuTNMT7ZAnZriaDjnwZnQbs6fPlWEkhHFoOsfw/640?wx_fmt=png)

**窗口的使用**  

* * *

-   Flink 为每个窗口创建一个每个数据元的副本。鉴于此，翻滚窗口保存每个数据元的一个副本（一个数据元恰好属于一个窗口，除非它被延迟）  
    动窗口会每个数据元创建几个复本，如 “窗口分配器” 部分中所述。因此，尺寸为 1 天且滑动 1 秒的滑动窗口可能不是一个好主意。
-   ReduceFunction，AggregateFunction 并且 FoldFunction 可以显着降低存储要求，因为它们急切地聚合数据元并且每个窗口只存储一个值。  
    相反，仅使用 ProcessWindowFunction 需要累积所有数据元。

**Evictor**

-   它剔除元素的时机是：在触发器触发之后，在窗口被处理 (apply windowFunction) 之前
-   Flink 的窗口模型允许在窗口分配器和触发器之外指定一个可选的驱逐器 (Evictor)。可以使用 evictor(…) 方法来完成。  
    驱逐器能够在触发器触发之后，以及在应用窗口函数之前或之后从窗口中移除元素
-   默认情况下，所有内置的驱逐器在窗口函数之前使用
-   指定驱逐器可以避免预聚合 (pre-aggregation)，因为窗口内所有元素必须在应用计算之前传递给驱逐器。
-   Flink 不保证窗口内元素的顺序。这意味着虽然驱逐者可以从窗口的开头移除元素，但这些元素不一定是先到的还是后到的。

### **内置的 Evitor**

-   ### **TimeEvitor**


-   以毫秒为单位的时间间隔作为参数，对于给定的窗口，找到元素中的最大的时间戳 max_ts，并删除时间戳小于 max_ts - interval 的所有元素。
-   本质上是将罪行的元素选出来


-   **CountEvitor**


-   保持窗口内元素数量符合用户指定数量，如果多于用户指定的数量，从窗口缓冲区的开头丢弃剩余的元素。


-   **DeltaEvitor**


-   使用 DeltaFunction 和 一个阈值，计算窗口缓冲区中的最后一个元素与其余每个元素之间的 delta 值，并删除 delta 值大于或等于阈值的元素。
-   通过定义的 DeltaFunction 和 Threshold , 计算窗口中元素和最新元素的 Delta 值，将 Delta 值超过 Threshold 的元素删除

## **watermark**

-   watermark 是一种衡量 Event Time 进展的机制，它是数据本身的一个隐藏属性。
-   watermark Apache Flink 为了处理 EventTime 窗口计算提出的一种机制, 本质上也是一种时间戳，  
    由 Apache Flink Source 或者自定义的 Watermark 生成器按照需求 Punctuated 或者 Periodic 两种方式生成的一种系统 Event，  
    与普通数据流 Event 一样流转到对应的下游算子，接收到 Watermark Event 的算子以此不断调整自己管理的 EventTime clock。  
    算子接收到一个 Watermark 时候，框架知道不会再有任何小于该 Watermark 的时间戳的数据元素到来了，所以 Watermark 可以看做是告诉 Apache Flink 框架数据流已经处理到什么位置 (时间维度) 的方式。
-   通常基于 Event Time 的数据，自身都包含一个 timestamp.watermark 是用于处理乱序事件的，而正确的处理乱序事件，通常用 watermark 机制结合 window 来实现。
-   waterMark 的触发时间机制 (waterMark>= window_end_time)


-   当第一次触发之后，以后所有到达的该窗口的数据 (迟到数据) 都会触发该窗口
-   定义允许延迟，所以 waterMark

    =window_end_time+allowedLateness 是窗口被关闭，数据被丢弃
-   对于 out-of-order 的数据，Flink 可以通过 watermark 机制结合 window 的操作，来处理一定范围内的乱序数据,（新进来的数据）晚于前面进来的数据，但是该数据所在窗口没有被触发，  
    这个时候数据还是有效的——EventTime&lt;WaterMark 的
-   对于 out-of-order 的数据，延迟太多
-   注意，如果不定义允许最大迟到时间，并且在有很多数据迟到的情况下，会严重影响正确结果，只要 Event Time &lt; watermark 时间就会触发窗口，也就是说迟到的每一条数据都会触发  
    该窗口

### **产生方式**

-   **Punctuated**


-   数据流中每一个递增的 EventTime 都会产生一个 Watermark（其实是根据某个计算条件来做判断）。
-   在实际的生产中 Punctuated 方式在 TPS 很高的场景下会产生大量的 Watermark 在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择 Punctuated 的方式进行 Watermark 的生成。
-   每个事件都会携带事件，可以根据该时间产生一个 watermark 或者可以根据事件携带的其他标志——业务的结束标志


-   Periodic - 周期性的（一定时间间隔或者达到一定的记录条数）产生一个 Watermark。

    在实际的生产中 Periodic 的方式必须结合时间和积累条数两个维度继续周期性产生 Watermark，否则在极端情况下会有很大的延时。

### **背景**

-   流处理从事件产生，到流经 source，再到 operator，中间是有一个过程和时间的。虽然大部分情况下，流到 operator 的数据都是按照事件产生的时间顺序来的
-   但是也不排除由于网络、背压等原因，导致乱序的产生（out-of-order 或者说 late element）。 
-   对于 late element，我们又不能无限期的等下去，必须要有个机制来保证一个特定的时间后，必须触发 window 去进行计算了
-   它表示当达到 watermark 到达之后, 在 watermark 之前的数据已经全部达到 (即使后面还有延迟的数据

### **解决的问题**

-   Watermark 的时间戳可以和 Event 中的 EventTime 一致，也可以自己定义任何合理的逻辑使得 Watermark 的时间戳不等于 Event 中的 EventTime，  
    Event 中的 EventTime 自产生那一刻起就不可以改变了，不受 Apache Flink 框架控制，  
    而 Watermark 的产生是在 Apache Flink 的 Source 节点或实现的 Watermark 生成器计算产生 (如上 Apache Flink 内置的 Periodic Watermark 实现),  
    Apache Flink 内部对单流或多流的场景有统一的 Watermark 处理。
-   默认情况下小于 watermark 时间戳的 event 会被丢弃吗

### **多流 waterMark**

-   在实际的流计算中往往一个 job 中会处理多个 Source 的数据，对 Source 的数据进行 GroupBy 分组，那么来自不同 Source 的相同 key 值会 shuffle 到同一个处理节点，  
    并携带各自的 Watermark，Apache Flink 内部要保证 Watermark 要保持单调递增，多个 Source 的 Watermark 汇聚到一起时候可能不是单调自增的
-   Apache Flink 内部实现每一个边上只能有一个递增的 Watermark， 当出现多流携带 Eventtime 汇聚到一起 (GroupBy or Union) 时候，  
    Apache Flink 会选择所有流入的 Eventtime 中最小的一个向下游流出。从而保证 watermark 的单调递增和保证数据的完整性

### **理解**

-   默认情况下 watermark 已经触发过得窗口，即使有新数据（迟到）落进去不会被计算 ，迟到的意思

    watermark>=window_n_end_time && window_n_start_time&lt;=vent_time&lt;window_n_end_time(即数据属于这个窗口)
-   允许迟到  
    watermark>=window_n_end_time && watermark&lt;window_n_end_time+lateness && window_n_start_time&lt;=vent_time&lt;window_n_end_time  
    在 watermark 大于窗口结束时间不超过特定延迟范围时，落在此窗口内的数据是有效的，可以触发窗口。

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1q4xOcqPNS5Z6kCXdn3fesF3hsl5Zxgntcw5ImO64d58B91WguHpZ1yg/640?wx_fmt=png)

**窗口聚合**  

* * *

-   增量聚合


-   窗口内来一条数据就计算一次


-   全量聚合


-   一次计算整个窗口里的所有元素（可以进行排序，一次一批可以针对外部链接）
-   使用
-   窗口之后调用 apply , 创建的元素里面方法的参数是一个迭代器

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qqAMiagSzJ8icbpo8M4jok0ficoITFTCLluKBDbnFaZgXbKibpSicjbfJxRg/640?wx_fmt=png)

**常用的一些方法**  

* * *

-   window
-   timeWindow 和 countWind
-   process 和 apply

AssignerWithPeriodicWatermarks 或接口 AssignerWithPunctuatedWatermarks。  
简而言之，前一个接口将会周期性发送 Watermark，而第二个接口根据一些到达数据的属性，例如一旦在流中碰到一个特殊的 element 便发送 Watermark。

### **自定义窗口**

-   Window Assigner：负责将元素分配到不同的 window。
-   Trigger 即触发器，定义何时或什么情况下 Fire 一个 window。
-   对于 CountWindow，我们可以直接使用已经定义好的 Trigger：CountTrigger trigger(CountTrigger.of(2))
-   Evictor（可选） 驱逐者，即保留上一 window 留下的某些元素。
-   最简单的情况，如果业务不是特别复杂，仅仅是基于 Time 和 Count，我们其实可以用系统定义好的 WindowAssigner 以及 Trigger 和 Evictor 来实现不同的组合：

![](https://mmbiz.qpic.cn/mmbiz_png/Pn4Sm0RsAugJkibeIkBwI7feN33tcTm1qBXWPDdbyLYO54KaZweOaKx6ibvRsoyIZf7CmicjoYiauOOSdD1kXvXYTA/640?wx_fmt=png)

**window 出现数据倾斜**  

* * *

-   window 产生数据倾斜指的是数据在不同的窗口内堆积的数据量相差过多。本质上产生这种情况的原因是数据源头发送的数据量速度不同导致的。出现这种情况一般通过两种方式来解决：
-   在数据进入窗口前做预聚合；
-   重新设计窗口聚合的 key； 
    [https://mp.weixin.qq.com/s/pnB-LUssXncIMxQgUf1w8w](https://mp.weixin.qq.com/s/pnB-LUssXncIMxQgUf1w8w)
