# 一网打尽Flink中的时间、窗口和流Join
首先，我们会学习如何定义时间属性，时间戳和水位线。然后我们将会学习底层操作 process function，它可以让我们访问时间戳和水位线，以及注册定时器事件。接下来，我们将会使用 Flink 的 window API，它提供了通常使用的各种窗口类型的内置实现。我们将会学到如何进行用户自定义窗口操作符，以及窗口的核心功能：assigners（分配器）、triggers（触发器）和 evictors（清理器）。最后，我们将讨论如何基于时间来做流的联结查询，以及处理迟到事件的策略。

### 时间操作

#### 1 设置时间属性

如果我们想要在分布式流处理应用程序中定义有关时间的操作，彻底理解时间的语义是非常重要的。当我们指定了一个窗口去收集某 1 分钟内的数据时，这个长度为 1 分钟的桶中，到底应该包含哪些数据？在 DataStream API 中，我们将使用时间属性来告诉 Flink：当我们创建窗口时，我们如何定义时间。时间属性是 StreamExecutionEnvironment 的一个属性，有以下值：

> ProcessingTime

机器时间在分布式系统中又叫做 “墙上时钟”。

当操作符执行时，此操作符看到的时间是操作符所在机器的机器时间。Processing-time window 的触发取决于机器时间，窗口包含的元素也是那个机器时间段内到达的元素。通常情况下，窗口操作符使用 processing time 会导致不确定的结果，因为基于机器时间的窗口中收集的元素取决于元素到达的速度快慢。使用 processing time 会为程序提供极低的延迟，因为无需等待水位线的到达。

如果要追求极限的低延迟，请使用 processing time。

> EventTime

当操作符执行时，操作符看的当前时间是由流中元素所携带的信息决定的。流中的每一个元素都必须包含时间戳信息。而系统的逻辑时钟由水位线 (Watermark) 定义。我们之前学习过，时间戳要么在事件进入流处理程序之前已经存在，要么就需要在程序的数据源（source）处进行分配。当水位线宣布特定时间段的数据都已经到达，事件时间窗口将会被触发计算。即使数据到达的顺序是乱序的，事件时间窗口的计算结果也将是确定性的。窗口的计算结果并不取决于元素到达的快与慢。

当水位线超过事件时间窗口的结束时间时，窗口将会闭合，不再接收数据，并触发计算。

> IngestionTime

当事件进入 source 操作符时，source 操作符所在机器的机器时间，就是此事件的 “摄入时间”（IngestionTime），并同时产生水位线。IngestionTime 相当于 EventTime 和 ProcessingTime 的混合体。一个事件的 IngestionTime 其实就是它进入流处理器中的时间。

IngestionTime 没什么价值，既有 EventTime 的执行效率（比较低），有没有 EventTime 计算结果的准确性。

下面的例子展示了如何设置事件时间。

    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment;env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);DataStream<SensorReading> sensorData = env.addSource(...);如果要使用processing time，将TimeCharacteristic.EventTime替换为TimeCharacteristic.ProcessingTIme就可以了。

1.1 指定时间戳和产生水位线

如果使用事件时间，那么流中的事件必须包含这个事件真正发生的时间。使用了事件时间的流必须携带水位线。

时间戳和水位线的单位是毫秒，记时从 1970-01-01T00:00:00Z 开始。到达某个操作符的水位线就会告知这个操作符：小于等于水位线中携带的时间戳的事件都已经到达这个操作符了。时间戳和水位线可以由 SourceFunction 产生，或者由用户自定义的时间戳分配器和水位线产生器来生成。

Flink 暴露了 TimestampAssigner 接口供我们实现，使我们可以自定义如何从事件数据中抽取时间戳。一般来说，时间戳分配器需要在 source 操作符后马上进行调用。

因为时间戳分配器看到的元素的顺序应该和 source 操作符产生数据的顺序是一样的，否则就乱了。这就是为什么我们经常将 source 操作符的并行度设置为 1 的原因。

也就是说，任何分区操作都会将元素的顺序打乱，例如：并行度改变，keyBy() 操作等等。

所以最佳实践是：在尽量接近数据源 source 操作符的地方分配时间戳和产生水位线，甚至最好在 SourceFunction 中分配时间戳和产生水位线。当然在分配时间戳和产生水位线之前可以对流进行 map 和 filter 操作是没问题的，也就是说必须是窄依赖。

以下这种写法是可以的。

    DataStream<T> stream = env  .addSource(...)  .map(...)  .filter(...)  .assignTimestampsAndWatermarks(...)

下面的例子展示了首先 filter 流，然后再分配时间戳和水位线。

    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment;// 从调用时刻开始给env创建的每一个stream追加时间特征env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);DataStream<SensorReading> readings = env  .addSource(new SensorSource)  .filter(r -> r.temperature > 25)  .assignTimestampsAndWatermarks(new MyAssigner());

MyAssigner 有两种类型

-   AssignerWithPeriodicWatermarks
-   AssignerWithPunctuatedWatermarks

以上两个接口都继承自 TimestampAssigner。

1.2 周期性的生成水位线

周期性的生成水位线：系统会周期性的将水位线插入到流中（水位线也是一种特殊的事件!）。默认周期是 200 毫秒，也就是说，系统会每隔 200 毫秒就往流中插入一次水位线。

这里的 200 毫秒是机器时间！

可以使用 ExecutionConfig.setAutoWatermarkInterval() 方法进行设置。

    val env = StreamExecutionEnvironment.getExecutionEnvironmentenv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)// 每隔5秒产生一个水位线env.getConfig.setAutoWatermarkInterval(5000)

上面的例子产生水位线的逻辑：每隔 5 秒钟，Flink 会调用 AssignerWithPeriodicWatermarks 中的 getCurrentWatermark() 方法。如果方法返回的时间戳大于之前水位线的时间戳，新的水位线会被插入到流中。这个检查保证了水位线是单调递增的。如果方法返回的时间戳小于等于之前水位线的时间戳，则不会产生新的水位线。

例子，自定义一个周期性的时间戳抽取

scala version

    class PeriodicAssigner extends AssignerWithPeriodicWatermarks[SensorReading] {  val bound = 60 * 1000 // 延时为1分钟  var maxTs = Long.MinValue + bound + 1 // 观察到的最大时间戳  override def getCurrentWatermark: Watermark {    new Watermark(maxTs - bound - 1)  }  override def extractTimestamp(r: SensorReading, previousTS: Long) {    maxTs = maxTs.max(r.timestamp)    r.timestamp  }}

java version

    .assignTimestampsAndWatermarks(  // generate periodic watermarks  new AssignerWithPeriodicWatermarks[(String, Long)] {    val bound = 10 * 1000L // 最大延迟时间    var maxTs = Long.MinValue + bound + 1 // 当前观察到的最大时间戳    // 用来生成水位线    // 默认200ms调用一次    override def getCurrentWatermark: Watermark = {      println("generate watermark!!!" + (maxTs - bound - 1) + "ms")      new Watermark(maxTs - bound - 1)    }    // 每来一条数据都会调用一次    override def extractTimestamp(t: (String, Long), l: Long): Long = {      println("extract timestamp!!!")      maxTs = maxTs.max(t._2) // 更新观察到的最大事件时间      t._2 // 抽取时间戳    }  })

如果我们事先得知数据流的时间戳是单调递增的，也就是说没有乱序。我们可以使用 assignAscendingTimestamps，方法会直接使用数据的时间戳生成水位线。

scala version

    val stream = ...val withTimestampsAndWatermarks = stream.assignAscendingTimestamps(e => e.timestamp)

java version

    .assignTimestampsAndWatermarks(        WatermarkStrategy                .<SensorReading>forMonotonousTimestamps()                .withTimestampAssigner(new SerializableTimestampAssigner<SensorReading>() {                    @Override                    public long extractTimestamp(SensorReading r, long l) {                        return r.timestamp;                    }                }))

如果我们能大致估算出数据流中的事件的最大延迟时间，可以使用如下代码：

最大延迟时间就是当前到达的事件的事件时间和之前所有到达的事件中最大时间戳的差。

scala version

    .assignTimestampsAndWatermarks(  // 水位线策略；默认200ms的机器时间插入一次水位线  // 水位线 = 当前观察到的事件所携带的最大时间戳 - 最大延迟时间  WatermarkStrategy    // 最大延迟时间设置为5s    .forBoundedOutOfOrderness[(String, Long)](Duration.ofSeconds(5))    .withTimestampAssigner(new SerializableTimestampAssigner[(String, Long)] {      // 告诉系统第二个字段是时间戳，时间戳的单位是毫秒      override def extractTimestamp(element: (String, Long), recordTimestamp: Long): Long = element._2    }))

java version

    .assignTimestampsAndWatermarks(        WatermarkStrategy                .<Tuple2<String, Long>>forBoundedOutOfOrderness(Duration.ofSeconds(5))                .withTimestampAssigner(new SerializableTimestampAssigner<Tuple2<String, Long>>() {                    @Override                    public long extractTimestamp(Tuple2<String, Long> element, long recordTimestamp) {                        return element.f1;                    }                }))

以上代码设置了最大延迟时间为 5 秒。

1.3 如何产生不规则的水位线

有时候输入流中会包含一些用于指示系统进度的特殊元组或标记。Flink 为此类情形以及可根据输入元素生成水位线的情形提供了 AssignerWithPunctuatedWatermarks 接口。该接口中的 checkAndGetNextWatermark() 方法会在针对每个事件的 extractTimestamp() 方法后立即调用。它可以决定是否生成一个新的水位线。如果该方法返回一个非空、且大于之前值的水位线，算子就会将这个新水位线发出。

    class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks[SensorReading] {  val bound = 60 * 1000  // 每来一条数据就调用一次  // 紧跟`extractTimestamp`函数调用  override def checkAndGetNextWatermark(r: SensorReading, extractedTS: Long) {    if (r.id == "sensor_1") {      // 抽取的时间戳 - 最大延迟时间      new Watermark(extractedTS - bound)    } else {      null    }  }  // 每来一条数据就调用一次  override def extractTimestamp(r: SensorReading, previousTS: Long) {    r.timestamp  }}

现在我们已经知道如何使用 TimestampAssigner 来产生水位线了。现在我们要讨论一下水位线会对我们的程序产生什么样的影响。

水位线用来平衡延迟和计算结果的正确性。水位线告诉我们，在触发计算（例如关闭窗口并触发窗口计算）之前，我们需要等待事件多长时间。基于事件时间的操作符根据水位线来衡量系统的逻辑时间的进度。

完美的水位线永远不会错：时间戳小于水位线的事件不会再出现。在特殊情况下 (例如非乱序事件流)，最近一次事件的时间戳就可能是完美的水位线。启发式水位线则相反，它只估计时间，因此有可能出错，即迟到的事件(其时间戳小于水位线标记时间) 晚于水位线出现。针对启发式水位线，Flink 提供了处理迟到元素的机制。

设定水位线通常需要用到领域知识。举例来说，如果知道事件的迟到时间不会超过 5 秒，就可以将水位线标记时间设为收到的最大时间戳减去 5 秒。另一种做法是，采用一个 Flink 作业监控事件流，学习事件的迟到规律，并以此构建水位线生成模型。

如果最大延迟时间设置的很大，计算出的结果会更精确，但收到计算结果的速度会很慢，同时系统会缓存大量的数据，并对系统造成比较大的压力。如果最大延迟时间设置的很小，那么收到计算结果的速度会很快，但可能收到错误的计算结果。不过 Flink 处理迟到数据的机制可以解决这个问题。上述问题看起来很复杂，但是恰恰符合现实世界的规律：大部分真实的事件流都是乱序的，并且通常无法了解它们的乱序程度 (因为理论上不能预见未来)。水位线是唯一让我们直面乱序事件流并保证正确性的机制; 否则只能选择忽视事实，假装错误的结果是正确的。

> 思考题一：实时程序，要求实时性非常高，并且结果并不一定要求非常准确，那么应该怎么办？

回答：直接使用处理时间。

> 思考题二：如果要进行时间旅行，也就是要还原以前的数据集当时的流的状态，应该怎么办？

回答：使用事件时间。使用 Hive 将数据集先按照时间戳升序排列，再将最大延迟时间设置为 0。

#### 2 处理函数

我们之前学习的转换算子是无法访问事件的时间戳信息和水位线信息的。而这在一些应用场景下，极为重要。例如 MapFunction 这样的 map 转换算子就无法访问时间戳或者当前事件的事件时间。

基于此，DataStream API 提供了一系列的 Low-Level 转换算子。可以访问时间戳、水位线以及注册定时事件。还可以输出特定的一些事件，例如超时事件等。Process Function 用来构建事件驱动的应用以及实现自定义的业务逻辑 (使用之前的 window 函数和转换算子无法实现)。例如，Flink-SQL 就是使用 Process Function 实现的。

Flink 提供了 8 个 Process Function：

    ProcessFunctionKeyedProcessFunctionCoProcessFunctionProcessJoinFunctionBroadcastProcessFunctionKeyedBroadcastProcessFunctionProcessWindowFunctionProcessAllWindowFunction

我们这里详细介绍一下 KeyedProcessFunction。

KeyedProcessFunction 用来操作 KeyedStream。KeyedProcessFunction 会处理流的每一个元素，输出为 0 个、1 个或者多个元素。所有的 Process Function 都继承自 RichFunction 接口，所以都有 open()、close() 和 getRuntimeContext() 等方法。而 KeyedProcessFunction\[KEY, IN, OUT]还额外提供了两个方法:

processElement(v: IN, ctx: Context, out: Collector\[OUT]), 流中的每一个元素都会调用这个方法，调用结果将会放在 Collector 数据类型中输出。Context 可以访问元素的时间戳，元素的 key，以及 TimerService 时间服务。Context 还可以将结果输出到别的流 (side outputs)。

onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector\[OUT]) 是一个回调函数。当之前注册的定时器触发时调用。参数 timestamp 为定时器所设定的触发的时间戳。Collector 为输出结果的集合。OnTimerContext 和 processElement 的 Context 参数一样，提供了上下文的一些信息，例如 firing trigger 的时间信息 (事件时间或者处理时间)。

2.1 时间服务和定时器

Context 和 OnTimerContext 所持有的 TimerService 对象拥有以下方法:

-   currentProcessingTime(): Long 返回当前处理时间
-   currentWatermark(): Long 返回当前水位线的时间戳
-   registerProcessingTimeTimer(timestamp: Long): Unit 会注册当前 key 的 processing time 的 timer。当 processing time 到达定时时间时，触发 timer。
-   registerEventTimeTimer(timestamp: Long): Unit 会注册当前 key 的 event time timer。当水位线大于等于定时器注册的时间时，触发定时器执行回调函数。
-   deleteProcessingTimeTimer(timestamp: Long): Unit 删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行。
-   deleteEventTimeTimer(timestamp: Long): Unit 删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行。

当定时器 timer 触发时，执行回调函数 onTimer()。processElement() 方法和 onTimer() 方法是同步（不是异步）方法，这样可以避免并发访问和操作状态。

针对每一个 key 和 timestamp，只能注册一个定期器。也就是说，每一个 key 可以注册多个定时器，但在每一个时间戳只能注册一个定时器。KeyedProcessFunction 默认将所有定时器的时间戳放在一个优先队列中。在 Flink 做检查点操作时，定时器也会被保存到状态后端中。

举个例子说明 KeyedProcessFunction 如何操作 KeyedStream。

下面的程序展示了如何监控温度传感器的温度值，如果温度值在一秒钟之内 (processing time) 连续上升，报警。

scala version

    val warnings = readings  .keyBy(r => r.id)  .process(new TempIncreaseAlertFunction)

      class TempIncrease extends KeyedProcessFunction[String, SensorReading, String] {    // 懒加载；    // 状态变量会在检查点操作时进行持久化，例如hdfs    // 只会初始化一次，单例模式    // 在当机重启程序时，首先去持久化设备寻找名为`last-temp`的状态变量，如果存在，则直接读取。不存在，则初始化。    // 用来保存最近一次温度    // 默认值是0.0    lazy val lastTemp: ValueState[Double] = getRuntimeContext.getState(      new ValueStateDescriptor[Double]("last-temp", Types.of[Double])    )    // 默认值是0L    lazy val timer: ValueState[Long] = getRuntimeContext.getState(      new ValueStateDescriptor[Long]("timer", Types.of[Long])    )    override def processElement(value: SensorReading, ctx: KeyedProcessFunction[String, SensorReading, String]#Context, out: Collector[String]): Unit = {      // 使用`.value()`方法取出最近一次温度值，如果来的温度是第一条温度，则prevTemp为0.0      val prevTemp = lastTemp.value()      // 将到来的这条温度值存入状态变量中      lastTemp.update(value.temperature)      // 如果timer中有定时器的时间戳，则读取      val ts = timer.value()      if (prevTemp == 0.0 || value.temperature < prevTemp) {        ctx.timerService().deleteProcessingTimeTimer(ts)        timer.clear()      } else if (value.temperature > prevTemp && ts == 0) {        val oneSecondLater = ctx.timerService().currentProcessingTime() + 1000L        ctx.timerService().registerProcessingTimeTimer(oneSecondLater)        timer.update(oneSecondLater)      }    }    override def onTimer(timestamp: Long, ctx: KeyedProcessFunction[String, SensorReading, String]#OnTimerContext, out: Collector[String]): Unit = {      out.collect("传感器ID是 " + ctx.getCurrentKey + " 的传感器的温度连续1s上升了！")      timer.clear()    }  }

java version

    DataStream<String> warings = readings    .keyBy(r -> r.id)    .process(new TempIncreaseAlertFunction());

看一下 TempIncreaseAlertFunction 如何实现, 程序中使用了 ValueState 这样一个状态变量, 后面会详细讲解。

        public static class TempIncreaseAlertFunction extends KeyedProcessFunction<String, SensorReading, String> {        private ValueState<Double> lastTemp;        private ValueState<Long> currentTimer;        @Override        public void open(Configuration parameters) throws Exception {            super.open(parameters);            lastTemp = getRuntimeContext().getState(                    new ValueStateDescriptor<>("last-temp", Types.DOUBLE)            );            currentTimer = getRuntimeContext().getState(                    new ValueStateDescriptor<>("current-timer", Types.LONG)            );        }        @Override        public void processElement(SensorReading r, Context ctx, Collector<String> out) throws Exception {            // 取出上一次的温度            Double prevTemp = 0.0;            if (lastTemp.value() != null) {                prevTemp = lastTemp.value();            }            // 将当前温度更新到上一次的温度这个变量中            lastTemp.update(r.temperature);            Long curTimerTimestamp = 0L;            if (currentTimer.value() != null) {                curTimerTimestamp = currentTimer.value();            }            if (prevTemp == 0.0 || r.temperature < prevTemp) {                // 温度下降或者是第一个温度值，删除定时器                ctx.timerService().deleteProcessingTimeTimer(curTimerTimestamp);                // 清空状态变量                currentTimer.clear();            } else if (r.temperature > prevTemp && curTimerTimestamp == 0) {                // 温度上升且我们并没有设置定时器                long timerTs = ctx.timerService().currentProcessingTime() + 1000L;                ctx.timerService().registerProcessingTimeTimer(timerTs);                // 保存定时器时间戳                currentTimer.update(timerTs);            }        }        @Override        public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {            super.onTimer(timestamp, ctx, out);            out.collect("传感器id为: "                    + ctx.getCurrentKey()                    + "的传感器温度值已经连续1s上升了。");            currentTimer.clear();        }    }

2.2 将事件发送到侧输出

大部分的 DataStream API 的算子的输出是单一输出，也就是某种数据类型的流。除了 split 算子，可以将一条流分成多条流，这些流的数据类型也都相同。process function 的 side outputs 功能可以产生多条流，并且这些流的数据类型可以不一样。一个 side output 可以定义为 OutputTag\[X]对象，X 是输出流的数据类型。process function 可以通过 Context 对象发射一个事件到一个或者多个 side outputs。

例子

scala version

    object SideOutputExample {  val output = new OutputTag[String]("side-output")  def main(args: Array[String]): Unit = {    val env = StreamExecutionEnvironment.getExecutionEnvironment    env.setParallelism(1)    val stream = env.addSource(new SensorSource)    val warnings = stream      .process(new FreezingAlarm)    warnings.print() // 打印主流    warnings.getSideOutput(output).print() // 打印侧输出流    env.execute()  }  class FreezingAlarm extends ProcessFunction[SensorReading, SensorReading] {    override def processElement(value: SensorReading, ctx: ProcessFunction[SensorReading, SensorReading]#Context, out: Collector[SensorReading]): Unit = {      if (value.temperature < 32.0) {        ctx.output(output, "传感器ID为：" + value.id + "的传感器温度小于32度！")      }      out.collect(value)    }  }}

java version

    public class SideOutputExample {    private static OutputTag<String> output = new OutputTag<String>("side-output"){};    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setParallelism(1);        DataStream<SensorReading> stream = env.addSource(new SensorSource());        SingleOutputStreamOperator<SensorReading> warnings = stream                .process(new ProcessFunction<SensorReading, SensorReading>() {                    @Override                    public void processElement(SensorReading value, Context ctx, Collector<SensorReading> out) throws Exception {                        if (value.temperature < 32) {                            ctx.output(output, "温度小于32度！");                        }                        out.collect(value);                    }                });        warnings.print();        warnings.getSideOutput(output).print();        env.execute();    }}

2.3 CoProcessFunction

对于两条输入流，DataStream API 提供了 CoProcessFunction 这样的 low-level 操作。CoProcessFunction 提供了操作每一个输入流的方法: processElement1() 和 processElement2()。类似于 ProcessFunction，这两种方法都通过 Context 对象来调用。这个 Context 对象可以访问事件数据，定时器时间戳，TimerService，以及 side outputs。CoProcessFunction 也提供了 onTimer() 回调函数。下面的例子展示了如何使用 CoProcessFunction 来合并两条流。

scala version

    object SensorSwitch {  def main(args: Array[String]): Unit = {    val env = StreamExecutionEnvironment.getExecutionEnvironment    env.setParallelism(1)    val stream = env.addSource(new SensorSource).keyBy(r => r.id)    val switches = env.fromElements(("sensor_2", 10 * 1000L)).keyBy(r => r._1)    stream      .connect(switches)      .process(new SwitchProcess)      .print()    env.execute()  }  class SwitchProcess extends CoProcessFunction[SensorReading, (String, Long), SensorReading] {    lazy val forwardSwitch = getRuntimeContext.getState(      new ValueStateDescriptor[Boolean]("switch", Types.of[Boolean])    )    override def processElement1(value: SensorReading, ctx: CoProcessFunction[SensorReading, (String, Long), SensorReading]#Context, out: Collector[SensorReading]): Unit = {      if (forwardSwitch.value()) {        out.collect(value)      }    }    override def processElement2(value: (String, Long), ctx: CoProcessFunction[SensorReading, (String, Long), SensorReading]#Context, out: Collector[SensorReading]): Unit = {      forwardSwitch.update(true)      ctx.timerService().registerProcessingTimeTimer(ctx.timerService().currentProcessingTime() + value._2)    }    override def onTimer(timestamp: Long, ctx: CoProcessFunction[SensorReading, (String, Long), SensorReading]#OnTimerContext, out: Collector[SensorReading]): Unit = {      forwardSwitch.clear()    }  }}

java version

    public class SensorSwitch {    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setParallelism(1);        KeyedStream<SensorReading, String> stream = env                .addSource(new SensorSource())                .keyBy(r -> r.id);        KeyedStream<Tuple2<String, Long>, String> switches = env                .fromElements(Tuple2.of("sensor_2", 10 * 1000L))                .keyBy(r -> r.f0);        stream                .connect(switches)                .process(new SwitchProcess())                .print();        env.execute();    }    public static class SwitchProcess extends CoProcessFunction<SensorReading, Tuple2<String, Long>, SensorReading> {        private ValueState<Boolean> forwardingEnabled;        @Override        public void open(Configuration parameters) throws Exception {            super.open(parameters);            forwardingEnabled = getRuntimeContext().getState(                    new ValueStateDescriptor<>("filterSwitch", Types.BOOLEAN)            );        }        @Override        public void processElement1(SensorReading value, Context ctx, Collector<SensorReading> out) throws Exception {            if (forwardingEnabled.value() != null && forwardingEnabled.value()) {                out.collect(value);            }        }        @Override        public void processElement2(Tuple2<String, Long> value, Context ctx, Collector<SensorReading> out) throws Exception {            forwardingEnabled.update(true);            ctx.timerService().registerProcessingTimeTimer(ctx.timerService().currentProcessingTime() + value.f1);        }        @Override        public void onTimer(long timestamp, OnTimerContext ctx, Collector<SensorReading> out) throws Exception {            super.onTimer(timestamp, ctx, out);            forwardingEnabled.clear();        }    }}

### 窗口操作

#### 1 窗口操作符

窗口操作是流处理程序中很常见的操作。窗口操作允许我们在无限流上的一段有界区间上面做聚合之类的操作。而我们使用基于时间的逻辑来定义区间。窗口操作符提供了一种将数据放进一个桶，并根据桶中的数据做计算的方法。例如，我们可以将事件放进 5 分钟的滚动窗口中，然后计数。

无限流转化成有限数据的方法：使用窗口。

1.1 定义窗口操作符

Window 算子可以在 keyed stream 或者 nokeyed stream 上面使用。

创建一个 Window 算子，需要指定两个部分：

-   window assigner 定义了流的元素如何分配到 window 中。window assigner 将会产生一条 WindowedStream(或者 AllWindowedStream，如果是 nonkeyed DataStream 的话)
-   window function 用来处理 WindowedStream(AllWindowedStream) 中的元素。

下面的代码说明了如何使用窗口操作符。

    stream  .keyBy(...)  .window(...)  // 指定window assigner  .reduce/aggregate/process(...) // 指定window functionstream  .windowAll(...) // 指定window assigner  .reduce/aggregate/process(...) // 指定window function

我们的学习重点是 Keyed WindowedStream。

1.2 内置的窗口分配器

窗口分配器将会根据事件的事件时间或者处理时间来将事件分配到对应的窗口中去。窗口包含开始时间和结束时间这两个时间戳。

所有的窗口分配器都包含一个默认的触发器：

-   对于事件时间：当水位线超过窗口结束时间，触发窗口的求值操作。
-   对于处理时间：当机器时间超过窗口结束时间，触发窗口的求值操作。

> 需要注意的是：当处于某个窗口的第一个事件到达的时候，这个窗口才会被创建。Flink 不会对空窗口求值。

Flink 创建的窗口类型是 TimeWindow，包含开始时间和结束时间，区间是左闭右开的，也就是说包含开始时间戳，不包含结束时间戳。

> 滚动窗口 (tumbling windows)

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFEck2pAmBg60J1kDeXD2cYkvDMKplAPicjRblia2dnapPSFGYM7gic39uw/640?wx_fmt=png)

    DataStream<SensorReading> sensorData = ...DataStream<T> avgTemp = sensorData  .keyBy(r -> r.id)  // group readings in 1s event-time windows  .window(TumblingEventTimeWindows.of(Time.seconds(1)))  .process(new TemperatureAverager);DataStream<T> avgTemp = sensorData  .keyBy(r -> r.id)  // group readings in 1s processing-time windows  .window(TumblingProcessingTimeWindows.of(Time.seconds(1)))  .process(new TemperatureAverager);// 其实就是之前的// shortcut for window.(TumblingEventTimeWindows.of(size))DataStream<T> avgTemp = sensorData  .keyBy(r -> r.id)  .timeWindow(Time.seconds(1))  .process(new TemperatureAverager);

默认情况下，滚动窗口会和 1970-01-01-00:00:00.000 对齐，例如一个 1 小时的滚动窗口将会定义以下开始时间的窗口：00:00:00，01:00:00，02:00:00，等等。

> 滑动窗口 (sliding window)

对于滑动窗口，我们需要指定窗口的大小和滑动的步长。当滑动步长小于窗口大小时，窗口将会出现重叠，而元素会被分配到不止一个窗口中去。当滑动步长大于窗口大小时，一些元素可能不会被分配到任何窗口中去，会被直接丢弃。

下面的代码定义了窗口大小为 1 小时，滑动步长为 15 分钟的窗口。每一个元素将被分配到 4 个窗口中去。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFwKWPRoGNco2HO3Repoa7a1Jkcvs9Y1VnKV8z13cXYiaPZECiaomaEuQg/640?wx_fmt=png)

    DataStream<T> slidingAvgTemp = sensorData  .keyBy(r -> r.id)  .window(    SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(15))  )  .process(new TemperatureAverager);DataStream<T> slidingAvgTemp = sensorData  .keyBy(r -> r.id)  .window(    SlidingProcessingTimeWindows.of(Time.hours(1), Time.minutes(15))  )  .process(new TemperatureAverager);DataStream<T> slidingAvgTemp = sensorData  .keyBy(r -> r.id)  .timeWindow(Time.hours(1), Time.minutes(15))  .process(new TemperatureAverager);

> 会话窗口 (session windows)

会话窗口不可能重叠，并且会话窗口的大小也不是固定的。不活跃的时间长度定义了会话窗口的界限。不活跃的时间是指这段时间没有元素到达。下图展示了元素如何被分配到会话窗口。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFtMLn6099Rx8VLoKCTFUH4x9eUpIO67vebQ34LSwREm8qHsajl606ZQ/640?wx_fmt=png)

    DataStream<T> sessionWindows = sensorData  .keyBy(r -> r.id)  .window(EventTimeSessionWindows.withGap(Time.minutes(15)))  .process(...);DataStream<T> sessionWindows = sensorData  .keyBy(r -> r.id)  .window(ProcessingTimeSessionWindows.withGap(Time.minutes(15)))  .process(...);

由于会话窗口的开始时间和结束时间取决于接收到的元素，所以窗口分配器无法立即将所有的元素分配到正确的窗口中去。相反，会话窗口分配器最开始时先将每一个元素分配到它自己独有的窗口中去，窗口开始时间是这个元素的时间戳，窗口大小是 session gap 的大小。接下来，会话窗口分配器会将出现重叠的窗口合并成一个窗口。

1.3 调用窗口计算函数

window functions 定义了窗口中数据的计算逻辑。有两种计算逻辑：

-   增量聚合函数 (Incremental aggregation functions)：当一个事件被添加到窗口时，触发函数计算，并且更新 window 的状态 (单个值)。最终聚合的结果将作为输出。ReduceFunction 和 AggregateFunction 是增量聚合函数。
-   全窗口函数 (Full window functions)：这个函数将会收集窗口中所有的元素，可以做一些复杂计算。ProcessWindowFunction 是 window function。

> ReduceFunction

例子: 计算每个传感器 15s 窗口中的温度最小值

scala version

    val minTempPerWindow = sensorData  .map(r => (r.id, r.temperature))  .keyBy(_._1)  .timeWindow(Time.seconds(15))  .reduce((r1, r2) => (r1._1, r1._2.min(r2._2)))

java version

    DataStream<Tuple2<String, Double>> minTempPerwindow = sensorData    .map(new MapFunction<SensorReading, Tuple2<String, Double>>() {        @Override        public Tuple2<String, Double> map(SensorReading value) throws Exception {            return Tuple2.of(value.id, value.temperature);        }    })    .keyBy(r -> r.f0)    .timeWindow(Time.seconds(5))    .reduce(new ReduceFunction<Tuple2<String, Double>>() {        @Override        public Tuple2<String, Double> reduce(Tuple2<String, Double> value1, Tuple2<String, Double> value2) throws Exception {            if (value1.f1 < value2.f1) {                return value1;            } else {                return value2;            }        }    })

> AggregateFunction

先来看接口定义

    public interface AggregateFunction<IN, ACC, OUT>  extends Function, Serializable {  // create a new accumulator to start a new aggregate  ACC createAccumulator();  // add an input element to the accumulator and return the accumulator  ACC add(IN value, ACC accumulator);  // compute the result from the accumulator and return it.  OUT getResult(ACC accumulator);  // merge two accumulators and return the result.  ACC merge(ACC a, ACC b);}

IN 是输入元素的类型，ACC 是累加器的类型，OUT 是输出元素的类型。

例子

    val avgTempPerWindow: DataStream[(String, Double)] = sensorData  .map(r => (r.id, r.temperature))  .keyBy(_._1)  .timeWindow(Time.seconds(15))  .aggregate(new AvgTempFunction)// An AggregateFunction to compute the average temperature per sensor.// The accumulator holds the sum of temperatures and an event count.class AvgTempFunction  extends AggregateFunction[(String, Double),    (String, Double, Int), (String, Double)] {  override def createAccumulator() = {    ("", 0.0, 0)  }  override def add(in: (String, Double), acc: (String, Double, Int)) = {    (in._1, in._2 + acc._2, 1 + acc._3)  }  override def getResult(acc: (String, Double, Int)) = {    (acc._1, acc._2 / acc._3)  }  override def merge(acc1: (String, Double, Int),    acc2: (String, Double, Int)) = {    (acc1._1, acc1._2 + acc2._2, acc1._3 + acc2._3)  }}

> ProcessWindowFunction

一些业务场景，我们需要收集窗口内所有的数据进行计算，例如计算窗口数据的中位数，或者计算窗口数据中出现频率最高的值。这样的需求，使用 ReduceFunction 和 AggregateFunction 就无法实现了。这个时候就需要 ProcessWindowFunction 了。

先来看接口定义

    public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window>  extends AbstractRichFunction {  // Evaluates the window  void process(KEY key, Context ctx, Iterable<IN> vals, Collector<OUT> out)    throws Exception;  // Deletes any custom per-window state when the window is purged  public void clear(Context ctx) throws Exception {}  // The context holding window metadata  public abstract class Context implements Serializable {    // Returns the metadata of the window    public abstract W window();    // Returns the current processing time    public abstract long currentProcessingTime();    // Returns the current event-time watermark    public abstract long currentWatermark();    // State accessor for per-window state    public abstract KeyedStateStore windowState();    // State accessor for per-key global state    public abstract KeyedStateStore globalState();    // Emits a record to the side output identified by the OutputTag.    public abstract <X> void output(OutputTag<X> outputTag, X value);  }}

process() 方法接受的参数为：

-   window 的 key
-   Iterable 迭代器包含窗口的所有元素
-   Collector 用于输出结果流。

Context 参数和别的 process 方法一样。而 ProcessWindowFunction 的 Context 对象还可以访问 window 的元数据 (窗口开始和结束时间)，当前处理时间和水位线，per-window state 和 per-key global state，side outputs。

-   per-window state: 用于保存一些信息，这些信息可以被 process() 访问，只要 process 所处理的元素属于这个窗口。
-   per-key global state: 同一个 key，也就是在一条 KeyedStream 上，不同的 window 可以访问 per-key global state 保存的值。

例子：计算 5s 滚动窗口中的最低和最高的温度。输出的元素包含了 (流的 Key, 最低温度, 最高温度, 窗口结束时间)。

    val minMaxTempPerWindow: DataStream[MinMaxTemp] = sensorData  .keyBy(_.id)  .timeWindow(Time.seconds(5))  .process(new HighAndLowTempProcessFunction)case class MinMaxTemp(id: String, min: Double, max: Double, endTs: Long)class HighAndLowTempProcessFunction  extends ProcessWindowFunction[SensorReading,    MinMaxTemp, String, TimeWindow] {  override def process(key: String,                       ctx: Context,                       vals: Iterable[SensorReading],                       out: Collector[MinMaxTemp]): Unit = {    val temps = vals.map(_.temperature)    val windowEnd = ctx.window.getEnd    out.collect(MinMaxTemp(key, temps.min, temps.max, windowEnd))  }}

我们还可以将 ReduceFunction/AggregateFunction 和 ProcessWindowFunction 结合起来使用。ReduceFunction/AggregateFunction 做增量聚合，ProcessWindowFunction 提供更多的对数据流的访问权限。如果只使用 ProcessWindowFunction(底层的实现为将事件都保存在 ListState 中)，将会非常占用空间。分配到某个窗口的元素将被提前聚合，而当窗口的 trigger 触发时，也就是窗口收集完数据关闭时，将会把聚合结果发送到 ProcessWindowFunction 中，这时 Iterable 参数将会只有一个值，就是前面聚合的值。

例子

    input  .keyBy(...)  .timeWindow(...)  .reduce(    incrAggregator: ReduceFunction[IN],    function: ProcessWindowFunction[IN, OUT, K, W])input  .keyBy(...)  .timeWindow(...)  .aggregate(    incrAggregator: AggregateFunction[IN, ACC, V],    windowFunction: ProcessWindowFunction[V, OUT, K, W])

我们把之前的需求重新使用以上两种方法实现一下。

    case class MinMaxTemp(id: String, min: Double, max: Double, endTs: Long)val minMaxTempPerWindow2: DataStream[MinMaxTemp] = sensorData  .map(r => (r.id, r.temperature, r.temperature))  .keyBy(_._1)  .timeWindow(Time.seconds(5))  .reduce(    (r1: (String, Double, Double), r2: (String, Double, Double)) => {      (r1._1, r1._2.min(r2._2), r1._3.max(r2._3))    },    new AssignWindowEndProcessFunction  )class AssignWindowEndProcessFunction  extends ProcessWindowFunction[(String, Double, Double),    MinMaxTemp, String, TimeWindow] {    override def process(key: String,                       ctx: Context,                       minMaxIt: Iterable[(String, Double, Double)],                       out: Collector[MinMaxTemp]): Unit = {    val minMax = minMaxIt.head    val windowEnd = ctx.window.getEnd    out.collect(MinMaxTemp(key, minMax._2, minMax._3, windowEnd))  }}

1.4 自定义窗口操作符

Flink 内置的 window operators 分配器已经已经足够应付大多数应用场景。尽管如此，如果我们需要实现一些复杂的窗口逻辑，例如：可以发射早到的事件或者碰到迟到的事件就更新窗口的结果，或者窗口的开始和结束决定于特定事件的接收。

DataStream API 暴露了接口和方法来自定义窗口操作符。

-   自定义窗口分配器
-   自定义窗口计算触发器 (trigger)
-   自定义窗口数据清理功能 (evictor)

当一个事件来到窗口操作符，首先将会传给 WindowAssigner 来处理。WindowAssigner 决定了事件将被分配到哪些窗口。如果窗口不存在，WindowAssigner 将会创建一个新的窗口。

如果一个 window operator 接受了一个增量聚合函数作为参数，例如 ReduceFunction 或者 AggregateFunction，新到的元素将会立即被聚合，而聚合结果 result 将存储在 window 中。如果 window operator 没有使用增量聚合函数，那么新元素将被添加到 ListState 中，ListState 中保存了所有分配给窗口的元素。

新元素被添加到窗口时，这个新元素同时也被传给了 window 的 trigger。trigger 定义了 window 何时准备好求值，何时 window 被清空。trigger 可以基于 window 被分配的元素和注册的定时器来对窗口的所有元素求值或者在特定事件清空 window 中所有的元素。

当 window operator 只接收一个增量聚合函数作为参数时：

当 window operator 只接收一个全窗口函数作为参数时：

当 window operator 接收一个增量聚合函数和一个全窗口函数作为参数时：

evictor 是一个可选的组件，可以被注入到 ProcessWindowFunction 之前或者之后调用。evictor 可以清除掉 window 中收集的元素。由于 evictor 需要迭代所有的元素，所以 evictor 只能使用在没有增量聚合函数作为参数的情况下。

下面的代码说明了如果使用自定义的 trigger 和 evictor 定义一个 window operator：

    stream  .keyBy(...)  .window(...) [.trigger(...)] [.evictor(...)]  .reduce/aggregate/process(...)

注意：每个 WindowAssigner 都有一个默认的 trigger。

> 窗口生命周期

当 WindowAssigner 分配某个窗口的第一个元素时，这个窗口才会被创建。所以不存在没有元素的窗口。

一个窗口包含了如下状态：

-   Window content

分配到这个窗口的元素 增量聚合的结果 (如果 window operator 接收了 ReduceFunction 或者 AggregateFunction 作为参数)。

-   Window object

WindowAssigner 返回 0 个，1 个或者多个 window object。window operator 根据返回的 window object 来聚合元素。每一个 window object 包含一个 windowEnd 时间戳，来区别于其他窗口。

-   触发器的定时器：一个触发器可以注册定时事件，到了定时的时间可以执行相应的回调函数，例如：对窗口进行求值或者清空窗口。
-   触发器中的自定义状态：触发器可以定义和使用自定义的、per-window 或者 per-key 状态。这个状态完全被触发器所控制。而不是被 window operator 控制。

当窗口结束时间来到，window operator 将删掉这个窗口。窗口结束时间是由 window object 的 end timestamp 所定义的。无论是使用 processing time 还是 event time，窗口结束时间是什么类型可以调用 WindowAssigner.isEventTime() 方法获得。

> 窗口分配器 (window assigners)

WindowAssigner 将会把元素分配到 0 个，1 个或者多个窗口中去。我们看一下 WindowAssigner 接口：

    public abstract class WindowAssigner<T, W extends Window>    implements Serializable {  public abstract Collection<W> assignWindows(    T element,    long timestamp,    WindowAssignerContext context);  public abstract Trigger<T, W> getDefaultTriger(    StreamExecutionEnvironment env);  public abstract TypeSerializer<W> getWindowSerializer(    ExecutionConfig executionConfig);  public abstract boolean isEventTime();  public abstract static class WindowAssignerContext {    public abstract long getCurrentProcessingTime();  }}

WindowAssigner 有两个泛型参数：

-   T: 事件的数据类型
-   W: 窗口的类型

下面的代码创建了一个自定义窗口分配器，是一个 30 秒的滚动事件时间窗口。

    class ThirtySecondsWindows    extends WindowAssigner[Object, TimeWindow] {  val windowSize: Long = 30 * 1000L  override def assignWindows(    o: Object,    ts: Long,    ctx: WindowAssigner.WindowAssignerContext  ): java.util.List[TimeWindow] = {    val startTime = ts - (ts % windowSize)    val endTime = startTime + windowSize    Collections.singletonList(new TimeWindow(startTime, endTime))  }  override def getDefaultTrigger(    env: environment.StreamExecutionEnvironment  ): Trigger[Object, TimeWindow] = {      EventTimeTrigger.create()  }  override def getWindowSerializer(    executionConfig: ExecutionConfig  ): TypeSerializer[TimeWindow] = {    new TimeWindow.Serializer  }  override def isEventTime = true}

增量聚合示意图

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFe3uSA8MqQIx3BcbyWicfnJSzHOofnscLhyR5hyNoTFJG0YWdE1HmnSw/640?wx_fmt=png)

全窗口聚合示意图

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFZgBYzJ8t56kTP1EMja4fU5EEvDnicquD0siaaCDUgOTib8wYwV71aK7Xg/640?wx_fmt=png)

增量聚合和全窗口聚合结合使用的示意图

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFkgGCLBfhyHl37pVIaJoGI3ick5GwialJnVOT0wmXKukDJIHGjkRh8zuA/640?wx_fmt=png)

> 触发器 (Triggers)

触发器定义了 window 何时会被求值以及何时发送求值结果。触发器可以到了特定的时间触发也可以碰到特定的事件触发。例如：观察到事件数量符合一定条件或者观察到了特定的事件。

默认的触发器将会在两种情况下触发

-   处理时间：机器时间到达处理时间
-   事件时间：水位线超过了窗口的结束时间

触发器可以访问流的时间属性以及定时器，还可以对 state 状态编程。所以触发器和 process function 一样强大。

例如我们可以实现一个触发逻辑：当窗口接收到一定数量的元素时，触发器触发。再比如当窗口接收到一个特定元素时，触发器触发。还有就是当窗口接收到的元素里面包含特定模式 (5 秒钟内接收到了两个同样类型的事件)，触发器也可以触发。在一个事件时间的窗口中，一个自定义的触发器可以提前(在水位线没过窗口结束时间之前) 计算和发射计算结果。这是一个常见的低延迟计算策略，尽管计算不完全，但不像默认的那样需要等待水位线没过窗口结束时间。

每次调用触发器都会产生一个 TriggerResult 来决定窗口接下来发生什么。TriggerResult 可以取以下结果：

-   CONTINUE：什么都不做
-   FIRE：如果 window operator 有 ProcessWindowFunction 这个参数，将会调用这个 ProcessWindowFunction。如果窗口仅有增量聚合函数 (ReduceFunction 或者 AggregateFunction) 作为参数，那么当前的聚合结果将会被发送。窗口的 state 不变。
-   PURGE：窗口所有内容包括窗口的元数据都将被丢弃。
-   FIRE_AND_PURGE：先对窗口进行求值，再将窗口中的内容丢弃。

TriggerResult 可能的取值使得我们可以实现很复杂的窗口逻辑。一个自定义触发器可以触发多次，可以计算或者更新结果，可以在发送结果之前清空窗口。

接下来我们看一下 Trigger API：

    public abstract class Trigger<T, W extends Window>    implements Serializable {  TriggerResult onElement(    long timestamp,    W window,    TriggerContext ctx);  public abstract TriggerResult onProcessingTime(    long timestamp,    W window,    TriggerContext ctx);  public abstract TriggerResult onEventTime(    long timestamp,    W window,    TriggerContext ctx);  public boolean canMerge();  public void onMerge(W window, OnMergeContext ctx);  public abstract void clear(W window, TriggerContext ctx);}public interface TriggerContext {  long getCurrentProcessingTime();  long getCurrentWatermark();  void registerProcessingTimeTimer(long time);  void registerEventTimeTimer(long time);  void deleteProcessingTimeTimer(long time);  void deleteEventTimeTimer(long time);  <S extends State> S getPartitionedState(    StateDescriptor<S, ?> stateDescriptor);}public interface OnMergeContext extends TriggerContext {  void mergePartitionedState(    StateDescriptor<S, ?> stateDescriptor  );}

这里要注意两个地方：清空 state 和 merging 合并触发器。

当在触发器中使用 per-window state 时，这里我们需要保证当窗口被删除时 state 也要被删除，否则随着时间的推移，window operator 将会积累越来越多的数据，最终可能使应用崩溃。

当窗口被删除时，为了清空所有状态，触发器的 clear() 方法需要需要删掉所有的自定义 per-window state，以及使用 TriggerContext 对象将处理时间和事件时间的定时器都删除。

下面的例子展示了一个触发器在窗口结束时间之前触发。当第一个事件被分配到窗口时，这个触发器注册了一个定时器，定时时间为水位线之前一秒钟。当定时事件执行，将会注册一个新的定时事件，这样，这个触发器每秒钟最多触发一次。

scala version

    class OneSecondIntervalTrigger    extends Trigger[SensorReading, TimeWindow] {  override def onElement(    SensorReading r,    timestamp: Long,    window: TimeWindow,    ctx: Trigger.TriggerContext  ): TriggerResult = {    val firstSeen: ValueState[Boolean] = ctx      .getPartitionedState(        new ValueStateDescriptor[Boolean](          "firstSeen", classOf[Boolean]        )      )    if (!firstSeen.value()) {      val t = ctx.getCurrentWatermark       + (1000 - (ctx.getCurrentWatermark % 1000))      ctx.registerEventTimeTimer(t)      ctx.registerEventTimeTimer(window.getEnd)      firstSeen.update(true)    }    TriggerResult.CONTINUE  }  override def onEventTime(    timestamp: Long,    window: TimeWindow,    ctx: Trigger.TriggerContext  ): TriggerResult = {    if (timestamp == window.getEnd) {      TriggerResult.FIRE_AND_PURGE    } else {      val t = ctx.getCurrentWatermark       + (1000 - (ctx.getCurrentWatermark % 1000))      if (t < window.getEnd) {        ctx.registerEventTimeTimer(t)      }      TriggerResult.FIRE    }  }  override def onProcessingTime(    timestamp: Long,    window: TimeWindow,    ctx: Trigger.TriggerContext  ): TriggerResult = {    TriggerResult.CONTINUE  }  override def clear(    window: TimeWindow,    ctx: Trigger.TriggerContext  ): Unit = {    val firstSeen: ValueState[Boolean] = ctx      .getPartitionedState(        new ValueStateDescriptor[Boolean](          "firstSeen", classOf[Boolean]        )      )    firstSeen.clear()  }}

java version

    public class TriggerExample {    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);        env.setParallelism(1);        env                .socketTextStream("localhost", 9999)                .map(new MapFunction<String, Tuple2<String, Long>>() {                    @Override                    public Tuple2<String, Long> map(String s) throws Exception {                        String[] arr = s.split(" ");                        return Tuple2.of(arr[0], Long.parseLong(arr[1]) * 1000L);                    }                })                .assignTimestampsAndWatermarks(                        WatermarkStrategy.<Tuple2<String, Long>>forMonotonousTimestamps()                        .withTimestampAssigner(new SerializableTimestampAssigner<Tuple2<String, Long>>() {                            @Override                            public long extractTimestamp(Tuple2<String, Long> stringLongTuple2, long l) {                                return stringLongTuple2.f1;                            }                        })                )                .keyBy(r -> r.f0)                .timeWindow(Time.seconds(5))                .trigger(new OneSecondIntervalTrigger())                .process(new ProcessWindowFunction<Tuple2<String, Long>, String, String, TimeWindow>() {                    @Override                    public void process(String s, Context context, Iterable<Tuple2<String, Long>> iterable, Collector<String> collector) throws Exception {                        long count = 0L;                        for (Tuple2<String, Long> i : iterable) count += 1;                        collector.collect("窗口中有 " + count + " 条元素");                    }                })                .print();        env.execute();    }    public static class OneSecondIntervalTrigger extends Trigger<Tuple2<String, Long>, TimeWindow> {        // 来一条调用一次        @Override        public TriggerResult onElement(Tuple2<String, Long> r, long l, TimeWindow window, TriggerContext ctx) throws Exception {            ValueState<Boolean> firstSeen = ctx.getPartitionedState(                    new ValueStateDescriptor<Boolean>("first-seen", Types.BOOLEAN)            );            if (firstSeen.value() == null) {                // 4999 + (1000 - 4999 % 1000) = 5000                System.out.println("第一条数据来的时候 ctx.getCurrentWatermark() 的值是 " + ctx.getCurrentWatermark());                long t = ctx.getCurrentWatermark() + (1000L - ctx.getCurrentWatermark() % 1000L);                ctx.registerEventTimeTimer(t);                ctx.registerEventTimeTimer(window.getEnd());                firstSeen.update(true);            }            return TriggerResult.CONTINUE;        }        // 定时器逻辑        @Override        public TriggerResult onEventTime(long ts, TimeWindow window, TriggerContext ctx) throws Exception {            if (ts == window.getEnd()) {                return TriggerResult.FIRE_AND_PURGE;            } else {                System.out.println("当前水位线是：" + ctx.getCurrentWatermark());                long t = ctx.getCurrentWatermark() + (1000L - ctx.getCurrentWatermark() % 1000L);                if (t < window.getEnd()) {                    ctx.registerEventTimeTimer(t);                }                return TriggerResult.FIRE;            }        }        @Override        public TriggerResult onProcessingTime(long l, TimeWindow timeWindow, TriggerContext triggerContext) throws Exception {            return TriggerResult.CONTINUE;        }        @Override        public void clear(TimeWindow timeWindow, TriggerContext ctx) throws Exception {            ValueState<Boolean> firstSeen = ctx.getPartitionedState(                    new ValueStateDescriptor<Boolean>("first-seen", Types.BOOLEAN)            );            firstSeen.clear();        }    }}

清理器 (EVICTORS)

evictor 可以在 window function 求值之前或者之后移除窗口中的元素。

我们看一下 Evictor 的接口定义：

    public interface Evictor<T, W extends Window>    extends Serializable {  void evictBefore(    Iterable<TimestampedValue<T>> elements,    int size,    W window,    EvictorContext evictorContext);  void evictAfter(    Iterable<TimestampedValue<T>> elements,    int size,    W window,    EvictorContext evictorContext);  interface EvictorContext {    long getCurrentProcessingTime();    long getCurrentWatermark();  }}

evictBefore() 和 evictAfter() 分别在 window function 计算之前或者之后调用。Iterable 迭代器包含了窗口所有的元素，size 为窗口中元素的数量，window object 和 EvictorContext 可以访问当前处理时间和水位线。可以对 Iterator 调用 remove() 方法来移除窗口中的元素。

evictor 也经常被用在 GlobalWindow 上，用来清除部分元素，而不是将窗口中的元素全部清空。

### 数据流操作

#### 1 基于时间的双流 Join

数据流操作的另一个常见需求是对两条数据流中的事件进行联结（connect）或 Join。Flink DataStream API 中内置有两个可以根据时间条件对数据流进行 Join 的算子：基于间隔的 Join 和基于窗口的 Join。本节我们会对它们进行介绍。

如果 Flink 内置的 Join 算子无法表达所需的 Join 语义，那么你可以通过 CoProcessFunction、BroadcastProcessFunction 或 KeyedBroadcastProcessFunction 实现自定义的 Join 逻辑。

> 注意，你要设计的 Join 算子需要具备高效的状态访问模式及有效的状态清理策略。

1.1 基于间隔的 Join

基于间隔的 Join 会对两条流中拥有相同键值以及彼此之间时间戳不超过某一指定间隔的事件进行 Join。

下图展示了两条流（A 和 B）上基于间隔的 Join，如果 B 中事件的时间戳相较于 A 中事件的时间戳不早于 1 小时且不晚于 15 分钟，则会将两个事件 Join 起来。Join 间隔具有对称性，因此上面的条件也可以表示为 A 中事件的时间戳相较 B 中事件的时间戳不早于 15 分钟且不晚于 1 小时。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFOhEvqhiaLcY9cDpuLB8637OmTDGmic7iaSoKYJLPeDzzu4n6ibEibbSODTw/640?wx_fmt=png)

基于间隔的 Join 目前只支持事件时间以及 INNER JOIN 语义（无法发出未匹配成功的事件）。下面的例子定义了一个基于间隔的 Join。

    input1  .intervalJoin(input2)  .between(<lower-bound>, <upper-bound>) // 相对于input1的上下界  .process(ProcessJoinFunction) // 处理匹配的事件对

Join 成功的事件对会发送给 ProcessJoinFunction。下界和上界分别由负时间间隔和正时间间隔来定义，例如 between(Time.hour(-1), Time.minute(15))。在满足下界值小于上界值的前提下，你可以任意对它们赋值。例如，允许出现 B 中事件的时间戳相较 A 中事件的时间戳早 1～2 小时这样的条件。

基于间隔的 Join 需要同时对双流的记录进行缓冲。对第一个输入而言，所有时间戳大于当前水位线减去间隔上界的数据都会被缓冲起来；对第二个输入而言，所有时间戳大于当前水位线加上间隔下界的数据都会被缓冲起来。注意，两侧边界值都有可能为负。上图中的 Join 需要存储数据流 A 中所有时间戳大于当前水位线减去 15 分钟的记录，以及数据流 B 中所有时间戳大于当前水位线减去 1 小时的记录。不难想象，如果两条流的事件时间不同步，那么 Join 所需的存储就会显著增加，因为水位线总是由 “较慢” 的那条流来决定。

例子：每个用户的点击 Join 这个用户最近 10 分钟内的浏览

scala version

    object IntervalJoinExample {  def main(args: Array[String]): Unit = {    val env = StreamExecutionEnvironment.getExecutionEnvironment    env.setParallelism(1)    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)    /*    A.intervalJoin(B).between(lowerBound, upperBound)    B.intervalJoin(A).between(-upperBound, -lowerBound)     */    val stream1 = env      .fromElements(        ("user_1", 10 * 60 * 1000L, "click"),        ("user_1", 16 * 60 * 1000L, "click")      )      .assignAscendingTimestamps(_._2)      .keyBy(r => r._1)    val stream2 = env      .fromElements(        ("user_1", 5 * 60 * 1000L, "browse"),        ("user_1", 6 * 60 * 1000L, "browse")      )      .assignAscendingTimestamps(_._2)      .keyBy(r => r._1)    stream1      .intervalJoin(stream2)      .between(Time.minutes(-10), Time.minutes(0))      .process(new ProcessJoinFunction[(String, Long, String), (String, Long, String), String] {        override def processElement(in1: (String, Long, String), in2: (String, Long, String), context: ProcessJoinFunction[(String, Long, String), (String, Long, String), String]#Context, collector: Collector[String]): Unit = {          collector.collect(in1 + " => " + in2)        }      })      .print()    stream2      .intervalJoin(stream1)      .between(Time.minutes(0), Time.minutes(10))      .process(new ProcessJoinFunction[(String, Long, String), (String, Long, String), String] {        override def processElement(in1: (String, Long, String), in2: (String, Long, String), context: ProcessJoinFunction[(String, Long, String), (String, Long, String), String]#Context, collector: Collector[String]): Unit = {          collector.collect(in1 + " => " + in2)        }      })      .print()    env.execute()  }}

java version

    public class IntervalJoinExample {    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);        env.setParallelism(1);        KeyedStream<Tuple3<String, Long, String>, String> stream1 = env                .fromElements(                        Tuple3.of("user_1", 10 * 60 * 1000L, "click")                )                .assignTimestampsAndWatermarks(                        WatermarkStrategy                                .<Tuple3<String, Long, String>>forMonotonousTimestamps()                                .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, Long, String>>() {                                    @Override                                    public long extractTimestamp(Tuple3<String, Long, String> stringLongStringTuple3, long l) {                                        return stringLongStringTuple3.f1;                                    }                                })                )                .keyBy(r -> r.f0);        KeyedStream<Tuple3<String, Long, String>, String> stream2 = env                .fromElements(                        Tuple3.of("user_1", 5 * 60 * 1000L, "browse"),                        Tuple3.of("user_1", 6 * 60 * 1000L, "browse")                )                .assignTimestampsAndWatermarks(                        WatermarkStrategy                                .<Tuple3<String, Long, String>>forMonotonousTimestamps()                                .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, Long, String>>() {                                    @Override                                    public long extractTimestamp(Tuple3<String, Long, String> stringLongStringTuple3, long l) {                                        return stringLongStringTuple3.f1;                                    }                                })                )                .keyBy(r -> r.f0);        stream1                .intervalJoin(stream2)                .between(Time.minutes(-10), Time.minutes(0))                .process(new ProcessJoinFunction<Tuple3<String, Long, String>, Tuple3<String, Long, String>, String>() {                    @Override                    public void processElement(Tuple3<String, Long, String> stringLongStringTuple3, Tuple3<String, Long, String> stringLongStringTuple32, Context context, Collector<String> collector) throws Exception {                        collector.collect(stringLongStringTuple3 + " => " + stringLongStringTuple32);                    }                })                .print();        env.execute();    }}

1.2 基于窗口的 Join

顾名思义，基于窗口的 Join 需要用到 Flink 中的窗口机制。其原理是将两条输入流中的元素分配到公共窗口中并在窗口完成时进行 Join（或 Cogroup）。

下面的例子展示了如何定义基于窗口的 Join。

    input1.join(input2)  .where(...)       // 为input1指定键值属性  .equalTo(...)     // 为input2指定键值属性  .window(...)      // 指定WindowAssigner  [.trigger(...)]   // 选择性的指定Trigger  [.evictor(...)]   // 选择性的指定Evictor  .apply(...)       // 指定JoinFunction

下图展示了 DataStream API 中基于窗口的 Join 是如何工作的。

![](https://mmbiz.qpic.cn/mmbiz_png/UdK9ByfMT2M06nPLKJeDgLDPznk6C7OFwae58a3LYY32KtwE77wtaxTl6qM9icbQ57HuicbXiaxiaJ1KWRXWKMNDeA/640?wx_fmt=png)

两条输入流都会根据各自的键值属性进行分区，公共窗口分配器会将二者的事件映射到公共窗口内（其中同时存储了两条流中的数据）。当窗口的计时器触发时，算子会遍历两个输入中元素的每个组合（叉乘积）去调用 JoinFunction。同时你也可以自定义触发器或移除器。由于两条流中的事件会被映射到同一个窗口中，因此该过程中的触发器和移除器与常规窗口算子中的完全相同。

除了对窗口中的两条流进行 Join，你还可以对它们进行 Cogroup，只需将算子定义开始位置的 join 改为 coGroup() 即可。Join 和 Cogroup 的总体逻辑相同，二者的唯一区别是：Join 会为两侧输入中的每个事件对调用 JoinFunction；而 Cogroup 中用到的 CoGroupFunction 会以两个输入的元素遍历器为参数，只在每个窗口中被调用一次。

注意，对划分窗口后的数据流进行 Join 可能会产生意想不到的语义。例如，假设你为执行 Join 操作的算子配置了 1 小时的滚动窗口，那么一旦来自两个输入的元素没有被划分到同一窗口，它们就无法 Join 在一起，即使二者彼此仅相差 1 秒钟。

scala version

    object TwoWindowJoinExample {  def main(args: Array[String]): Unit = {    val env = StreamExecutionEnvironment.getExecutionEnvironment    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)    env.setParallelism(1)    val stream1 = env      .fromElements(        ("a", 1000L),        ("a", 2000L)      )      .assignAscendingTimestamps(_._2)    val stream2 = env      .fromElements(        ("a", 3000L),        ("a", 4000L)      )      .assignAscendingTimestamps(_._2)    stream1      .join(stream2)      // on A.id = B.id      .where(_._1)      .equalTo(_._1)      .window(TumblingEventTimeWindows.of(Time.seconds(5)))      .apply(new JoinFunction[(String, Long), (String, Long), String] {        override def join(in1: (String, Long), in2: (String, Long)): String = {          in1 + " => " + in2        }      })      .print()    env.execute()  }}

java version

    public class TwoWindowJoinExample {    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setParallelism(1);        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);        DataStream<Tuple2<String, Long>> stream1 = env                .fromElements(                        Tuple2.of("a", 1000L),                        Tuple2.of("b", 1000L),                        Tuple2.of("a", 2000L),                        Tuple2.of("b", 2000L)                )                .assignTimestampsAndWatermarks(                        WatermarkStrategy                                .<Tuple2<String, Long>>forMonotonousTimestamps()                                .withTimestampAssigner(                                        new SerializableTimestampAssigner<Tuple2<String, Long>>() {                                            @Override                                            public long extractTimestamp(Tuple2<String, Long> stringLongTuple2, long l) {                                                return stringLongTuple2.f1;                                            }                                        }                                )                );        DataStream<Tuple2<String, Long>> stream2 = env                .fromElements(                        Tuple2.of("a", 3000L),                        Tuple2.of("b", 3000L),                        Tuple2.of("a", 4000L),                        Tuple2.of("b", 4000L)                )                .assignTimestampsAndWatermarks(                        WatermarkStrategy                                .<Tuple2<String, Long>>forMonotonousTimestamps()                                .withTimestampAssigner(                                        new SerializableTimestampAssigner<Tuple2<String, Long>>() {                                            @Override                                            public long extractTimestamp(Tuple2<String, Long> stringLongTuple2, long l) {                                                return stringLongTuple2.f1;                                            }                                        }                                )                );        stream1                .join(stream2)                .where(r -> r.f0)                .equalTo(r -> r.f0)                .window(TumblingEventTimeWindows.of(Time.seconds(5)))                .apply(new JoinFunction<Tuple2<String, Long>, Tuple2<String, Long>, String>() {                    @Override                    public String join(Tuple2<String, Long> stringLongTuple2, Tuple2<String, Long> stringLongTuple22) throws Exception {                        return stringLongTuple2 + " => " + stringLongTuple22;                    }                })                .print();        env.execute();    }}

#### 2 处理迟到的元素

水位线可以用来平衡计算的完整性和延迟两方面。除非我们选择一种非常保守的水位线策略 (最大延时设置的非常大，以至于包含了所有的元素，但结果是非常大的延迟)，否则我们总需要处理迟到的元素。

迟到的元素是指当这个元素来到时，这个元素所对应的窗口已经计算完毕了 (也就是说水位线已经没过窗口结束时间了)。这说明迟到这个特性只针对事件时间。

DataStream API 提供了三种策略来处理迟到元素

-   直接抛弃迟到的元素
-   将迟到的元素发送到另一条流中去
-   可以更新窗口已经计算完的结果，并发出计算结果。

2.1 抛弃迟到元素

抛弃迟到的元素是 event time window operator 的默认行为。也就是说一个迟到的元素不会创建一个新的窗口。

process function 可以通过比较迟到元素的时间戳和当前水位线的大小来很轻易的过滤掉迟到元素。

2.2 重定向迟到元素

迟到的元素也可以使用侧输出 (side output) 特性被重定向到另外的一条流中去。迟到元素所组成的侧输出流可以继续处理或者 sink 到持久化设施中去。

例子：

scala version

    val readings = env  .socketTextStream("localhost", 9999, '\n')  .map(line => {    val arr = line.split(" ")    (arr(0), arr(1).toLong * 1000)  })  .assignAscendingTimestamps(_._2)val countPer10Secs = readings  .keyBy(_._1)  .timeWindow(Time.seconds(10))  .sideOutputLateData(    new OutputTag[(String, Long)]("late-readings")  )  .process(new CountFunction())val lateStream = countPer10Secs  .getSideOutput(    new OutputTag[(String, Long)]("late-readings")  )lateStream.print()

实现 CountFunction:

    class CountFunction extends ProcessWindowFunction[(String, Long),  String, String, TimeWindow] {  override def process(key: String,                       context: Context,                       elements: Iterable[(String, Long)],                       out: Collector[String]): Unit = {    out.collect("窗口共有" + elements.size + "条数据")  }}

java version

    public class RedirectLateEvent {    private static OutputTag<Tuple2<String, Long>> output = new OutputTag<Tuple2<String, Long>>("late-readings"){};    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setParallelism(1);        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);        DataStream<Tuple2<String, Long>> stream = env                .socketTextStream("localhost", 9999)                .map(new MapFunction<String, Tuple2<String, Long>>() {                    @Override                    public Tuple2<String, Long> map(String s) throws Exception {                        String[] arr = s.split(" ");                        return Tuple2.of(arr[0], Long.parseLong(arr[1]) * 1000L);                    }                })                .assignTimestampsAndWatermarks(                        WatermarkStrategy.                                // like scala: assignAscendingTimestamps(_._2)                                <Tuple2<String, Long>>forMonotonousTimestamps()                                .withTimestampAssigner(new SerializableTimestampAssigner<Tuple2<String, Long>>() {                                    @Override                                    public long extractTimestamp(Tuple2<String, Long> value, long l) {                                        return value.f1;                                    }                                })                );        SingleOutputStreamOperator<String> lateReadings = stream                .keyBy(r -> r.f0)                .timeWindow(Time.seconds(5))                .sideOutputLateData(output) // use after keyBy and timeWindow                .process(new ProcessWindowFunction<Tuple2<String, Long>, String, String, TimeWindow>() {                    @Override                    public void process(String s, Context context, Iterable<Tuple2<String, Long>> iterable, Collector<String> collector) throws Exception {                        long exactSizeIfKnown = iterable.spliterator().getExactSizeIfKnown();                        collector.collect(exactSizeIfKnown + " of elements");                    }                });        lateReadings.print();        lateReadings.getSideOutput(output).print();        env.execute();    }}

下面这个例子展示了 ProcessFunction 如何过滤掉迟到的元素然后将迟到的元素发送到侧输出流中去。

scala version

    val readings: DataStream[SensorReading] = ...val filteredReadings: DataStream[SensorReading] = readings  .process(new LateReadingsFilter)// retrieve late readingsval lateReadings: DataStream[SensorReading] = filteredReadings  .getSideOutput(new OutputTag[SensorReading]("late-readings"))/** A ProcessFunction that filters out late sensor readings and  * re-directs them to a side output */class LateReadingsFilter    extends ProcessFunction[SensorReading, SensorReading] {  val lateReadingsOut = new OutputTag[SensorReading]("late-readings")  override def processElement(      SensorReading r,      ctx: ProcessFunction[SensorReading, SensorReading]#Context,      out: Collector[SensorReading]): Unit = {    // compare record timestamp with current watermark    if (r.timestamp < ctx.timerService().currentWatermark()) {      // this is a late reading => redirect it to the side output      ctx.output(lateReadingsOut, r)    } else {      out.collect(r)    }  }}

java version

    public class RedirectLateEvent {    private static OutputTag<String> output = new OutputTag<String>("late-readings"){};    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setParallelism(1);        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);        SingleOutputStreamOperator<Tuple2<String, Long>> stream = env                .socketTextStream("localhost", 9999)                .map(new MapFunction<String, Tuple2<String, Long>>() {                    @Override                    public Tuple2<String, Long> map(String s) throws Exception {                        String[] arr = s.split(" ");                        return Tuple2.of(arr[0], Long.parseLong(arr[1]) * 1000L);                    }                })                .assignTimestampsAndWatermarks(                        WatermarkStrategy.                                <Tuple2<String, Long>>forMonotonousTimestamps()                                .withTimestampAssigner(new SerializableTimestampAssigner<Tuple2<String, Long>>() {                                    @Override                                    public long extractTimestamp(Tuple2<String, Long> value, long l) {                                        return value.f1;                                    }                                })                )                .process(new ProcessFunction<Tuple2<String, Long>, Tuple2<String, Long>>() {                    @Override                    public void processElement(Tuple2<String, Long> stringLongTuple2, Context context, Collector<Tuple2<String, Long>> collector) throws Exception {                        if (stringLongTuple2.f1 < context.timerService().currentWatermark()) {                            context.output(output, "late event is comming!");                        } else {                            collector.collect(stringLongTuple2);                        }                    }                });        stream.print();        stream.getSideOutput(output).print();        env.execute();    }}

2.3 使用迟到元素更新窗口计算结果

由于存在迟到的元素，所以已经计算出的窗口结果是不准确和不完全的。我们可以使用迟到元素更新已经计算完的窗口结果。

如果我们要求一个 operator 支持重新计算和更新已经发出的结果，就需要在第一次发出结果以后也要保存之前所有的状态。但显然我们不能一直保存所有的状态，肯定会在某一个时间点将状态清空，而一旦状态被清空，结果就再也不能重新计算或者更新了。而迟到的元素只能被抛弃或者发送到侧输出流。

window operator API 提供了方法来明确声明我们要等待迟到元素。当使用 event-time window，我们可以指定一个时间段叫做 allowed lateness。window operator 如果设置了 allowed lateness，这个 window operator 在水位线没过窗口结束时间时也将不会删除窗口和窗口中的状态。窗口会在一段时间内 (allowed lateness 设置的) 保留所有的元素。

当迟到元素在 allowed lateness 时间内到达时，这个迟到元素会被实时处理并发送到触发器 (trigger)。当水位线没过了窗口结束时间 + allowed lateness 时间时，窗口会被删除，并且所有后来的迟到的元素都会被丢弃。

Allowed lateness 可以使用 allowedLateness() 方法来指定，如下所示：

    val readings: DataStream[SensorReading] = ...val countPer10Secs: DataStream[(String, Long, Int, String)] = readings  .keyBy(_.id)  .timeWindow(Time.seconds(10))  // process late readings for 5 additional seconds  .allowedLateness(Time.seconds(5))  // count readings and update results if late readings arrive  .process(new UpdatingWindowCountFunction)  /** A counting WindowProcessFunction that distinguishes between  * first results and updates. */class UpdatingWindowCountFunction    extends ProcessWindowFunction[SensorReading,      (String, Long, Int, String), String, TimeWindow] {  override def process(      id: String,      ctx: Context,      elements: Iterable[SensorReading],      out: Collector[(String, Long, Int, String)]): Unit = {    // count the number of readings    val cnt = elements.count(_ => true)    // state to check if this is    // the first evaluation of the window or not    val isUpdate = ctx.windowState.getState(      new ValueStateDescriptor[Boolean](        "isUpdate",        Types.of[Boolean]))    if (!isUpdate.value()) {      // first evaluation, emit first result      out.collect((id, ctx.window.getEnd, cnt, "first"))      isUpdate.update(true)    } else {      // not the first evaluation, emit an update      out.collect((id, ctx.window.getEnd, cnt, "update"))    }  }}

java version

    public class UpdateWindowResultWithLateEvent {    public static void main(String[] args) throws Exception {        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();        env.setParallelism(1);        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);        DataStreamSource<String> stream = env.socketTextStream("localhost", 9999);        stream                .map(new MapFunction<String, Tuple2<String, Long>>() {                    @Override                    public Tuple2<String, Long> map(String s) throws Exception {                        String[] arr = s.split(" ");                        return Tuple2.of(arr[0], Long.parseLong(arr[1]) * 1000L);                    }                })                .assignTimestampsAndWatermarks(                        WatermarkStrategy.<Tuple2<String, Long>>forBoundedOutOfOrderness(Duration.ofSeconds(5))                        .withTimestampAssigner(new SerializableTimestampAssigner<Tuple2<String, Long>>() {                            @Override                            public long extractTimestamp(Tuple2<String, Long> stringLongTuple2, long l) {                                return stringLongTuple2.f1;                            }                        })                )                .keyBy(r -> r.f0)                .timeWindow(Time.seconds(5))                .allowedLateness(Time.seconds(5))                .process(new UpdateWindowResult())                .print();        env.execute();    }    public static class UpdateWindowResult extends ProcessWindowFunction<Tuple2<String, Long>, String, String, TimeWindow> {        @Override        public void process(String s, Context context, Iterable<Tuple2<String, Long>> iterable, Collector<String> collector) throws Exception {            long count = 0L;            for (Tuple2<String, Long> i : iterable) {                count += 1;            }            // 可见范围比getRuntimeContext.getState更小，只对当前key、当前window可见            // 基于窗口的状态变量，只能当前key和当前窗口访问            ValueState<Boolean> isUpdate = context.windowState().getState(                    new ValueStateDescriptor<Boolean>("isUpdate", Types.BOOLEAN)            );            // 当水位线超过窗口结束时间时，触发窗口的第一次计算！            if (isUpdate.value() == null) {                collector.collect("窗口第一次触发计算！一共有 " + count + " 条数据！");                isUpdate.update(true);            } else {                collector.collect("窗口更新了！一共有 " + count + " 条数据！");            }        }    }}

 [https://mp.weixin.qq.com/s/ptQjJlaoVPZw6lPWGMoW1w](https://mp.weixin.qq.com/s/ptQjJlaoVPZw6lPWGMoW1w)
