# 面对持续不断生成的流数据—— Amazon Kinesis Data Analytics 实现及时分析与处理
![](https://mmbiz.qpic.cn/mmbiz_gif/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLwC2aT3PiakSv6zyT1sicG0wEEEyuad3ByUxOyeyQhywwIBH52icwGvbtg/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLiacAL3N6F77xESgBhcg4C3Aicn2Ylvn6yyteY0WjT3NfQCY96gFcMZbQ/640?wx_fmt=png)

**Amazon Kinesis Data Analytics 介绍**

如今各种企业每天都在面对持续不断生成的数据需要处理，这些数据可能来自移动或 Web 应用程序生成的日志文件、网上购物数据、游戏玩家活动、社交网站信息或者是金融交易等。**能够及时地处理并分析这些流数据对企业来说至关重要**，通过良好的流数据处理和应用，企业可以快速做出业务决策，改进产品或服务的质量，提升用户的满意度。

目前，市面上已经有很多工具可以帮助企业实现流数据的处理和分析。其中，**Apache Flink**是一个用于处理数据流的流行框架和引擎，用于在无边界和有边界数据流上进行有状态的计算。Flink 能在所有常见集群环境中运行，并能以内存速度和任意规模进行计算。

-   Apache Flink

    [https://flink.apache.org](https://flink.apache.org)

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLk7aCKY0TUPDZEEiblQEiaPCK7Iolt8zoLr4c4yR9BxYfgl1DteVOkNhg/640?wx_fmt=png)

图片来自 Apache Flink 官网

Amazon Kinesis Data Analytics 是快速使用 Apache Flink 实时转换和分析流数据的简单方法，通过无服务器架构实现流数据的处理和分析。借助 Amazon Kinesis Data Analytics，您可以使用基于 Apache Flink 的开源库构建 Java、Scala 以及 Python 应用程序。

**Amazon** \***\*Kinesis\*\*\*\***Data Analytics 为您的 Apache Flink 应用程序提供底层基础设施，其核心功能包括提供计算资源、并行计算、自动伸缩和应用程序备份 \*\*（以检查点和快照的形式实现）。您可以使用高级 Flink 编程特性（如操作符、函数、源和接收器等），就像您自己在托管 Flink 基础设施时使用它们一样。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLiacAL3N6F77xESgBhcg4C3Aicn2Ylvn6yyteY0WjT3NfQCY96gFcMZbQ/640?wx_fmt=png)

📢  想要了解更多亚马逊云科技最新技术发布和实践创新，敬请关注 2021 亚马逊云科技中国峰会！点击图片报名吧～

**在 Amazon Kinesis Data Analytics**

**使用 Python**

Amazon Kinesis Data Analytics for Apache Flink 现在支持使用 Python 3.7 构建流数据分析应用程序。**这使您能够以 Python 语言在 Amazon Kinesis Data Analytics 上通过 Apache Flink v1.11 运行大数据分析**，对 Python 语言开发者来说非常方便。Apache Flink v1.11 通过 PyFlink Table API 提供对 Python 的支持，这是一个统一的关系型 API。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLsSWEGZl3P99Uk2KFpMY7YBehARMfR1Cu8q7CricbibNkNLgr27ez6UHg/640?wx_fmt=png)

图片来自 Apache Flink 官网  

此外，Apache Flink 还提供了一个用于细粒度控制状态和时间的 DataStream API，并且从 Apache Flink 1.12 版本开始就支持 Python DataStream API。有关 Apache Flink 中 API 的更多信息，请参阅**Flink 官网介绍**。

-   Flink 官网介绍

    [https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/index.html](https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/index.html)

**Amazon Kinesis Data Analytics Python**

**应用程序示例**

接下来，我们将演示如何快速上手构建 Python 版的 Amazon Kinesis Data Analytics for Flink 应用程序。示例的参考架构如下图所示，我们将发送一些测试数据到 Amazon Kinesis Data Stream，然后通过 Amazon Kinesis Data Analytics Python 应用程序的 Tumbling Window 窗口函数做基本的聚合操作，再将这些数据持久化到 Amazon S3 中；之后可以使用 Amazon Glue 和 Amazon Athena 对这些数据进行快速的查询。整个示例应用程序都采用了无服务器的架构，在可以实现快速部署和自动弹性伸缩外，还极大地减轻了运维和管理负担。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLt1O66rz3mB59joq3IvDUoGA6icQ9tkLSt6f0WCnE3ZvIHULRypLrg6A/640?wx_fmt=jpeg)

以下示例是在由光环新网运营的亚马逊云科技中国（北京）区域上进行。  

**创建 Amazon Kinesis Data Stream**

示例将在控制台上创建 Amazon Kinesis Data Stream，首先选择到 Amazon Kinesis 服务 - 数据流，然后点击 “创建数据流”。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcL5WBibzWab63Ots7oZ7795AYRiamGAn4xuZBQBnI7p1zEZrwqmPh6SR0A/640?wx_fmt=jpeg)

输入数据流名称，如 “kda-input-stream“；数据流容量中的分区数设置为 1，注意这里是为了演示，请根据实际情况配置合适的容量。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLu8nqtkyvDIgEF70C42C6fcva5t2m604VafL4H8LjOSsRJ40IGVFwWg/640?wx_fmt=png)

点击创建数据流，等待片刻，数据流创建完成。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLvLI8JZ5WJsicS73GmbpYyQblWTQvia6FwMaK6Ay7JOGiafR4vlxJgRV3Q/640?wx_fmt=png)

稍后，我们将像这个 Amazon Kinesis 数据流发送示例数据。

**创建 Amazon S3 存储桶**

示例将在控制台上创建 Amazon S3 存储桶，首先选择到 Amazon Kinesis 服务，然后点击 “创建存储桶”。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLuDnVrhobSfeLOGn2luZroe3P5EWKeNnvLHXBNIl0AldDXWReibaqS8Q/640?wx_fmt=png)

输入存储桶名称，如 “kda-pyflink-\*\*\*”，这个名称稍后我们在 Amazon Kinesis 应用程序中用到。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLloGtVzuyUtNShWp83AW8FjNX9uM3PgTfCRFc2Z0VhWVQeq7h8IcOcA/640?wx_fmt=png)

保持其它配置不变，点击 “创建存储桶”。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLib6k9fiahHKtpgQqDFnSFIGhic5qGziaqQxGDvrUzCz4QX5QyDyspCziblg/640?wx_fmt=png)

稍等片刻，可以看到已成功创建存储桶。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLKRDKtZaTGkQm5qg0JHASKg4RxWG60f60xFvGU3jaI3xMP3N1xML9ng/640?wx_fmt=png)

**发送示例数据到**

**Amazon Kinesis Data Stream**

接下来，我们将使用一段 Python 程序向 Amazon Kinesis 数据流发送数据。创建 kda-input-stream.py 文件，并复制以下内容到这个文件，注意修改 STREAM_NAME 为您刚刚创建的 Amazon Kinesis 数据流名称，profile_name 配置为对应的用户信息。

    import datetimeimport jsonimport randomimport boto3STREAM_NAME = "kda-input-stream"def get_data():    return {        'event_time': datetime.datetime.now().isoformat(),        'ticker': random.choice(['AAPL', 'AMZN', 'MSFT', 'INTC', 'TBV']),        'price': round(random.random() * 100, 2)}def generate(stream_name, kinesis_client):    while True:        data = get_data()        print(data)        kinesis_client.put_record(            StreamName=stream_name,            Data=json.dumps(data),            PartitionKey="partitionkey")if __name__ == '__main__':    session = boto3.Session(profile_name='<your profile>')    generate(STREAM_NAME, session.client('kinesis', region_name='cn-n

执行以下代码，开始向 Amazon Kinesis 数据流发送数据。

    $ python kda-input-stream.py

**编写 Pyflink 代码**

接下来，我们编写 PyFlink 代码。创建 kda-pyflink-demo.py 文件，并复制以下内容到这个文件。

    # -*- coding: utf-8 -*-"""kda-pyflink-demo.py~~~~~~~~~~~~~~~~~~~1. 创建 Table Environment2. 创建源 Kinesis Data Stream3. 创建目标 S3 Bucket4. 执行窗口函数查询5. 将结果写入目标"""from pyflink.table import EnvironmentSettings, StreamTableEnvironmentfrom pyflink.table.window import Tumbleimport osimport json# 1. 创建 Table Environmentenv_settings = (    EnvironmentSettings.new_instance().in_streaming_mode().use_blink_planner().build())table_env = StreamTableEnvironment.create(environment_settings=env_settings)statement_set = table_env.create_statement_set()APPLICATION_PROPERTIES_FILE_PATH = "/etc/flink/application_properties.json"def get_application_properties():    if os.path.isfile(APPLICATION_PROPERTIES_FILE_PATH):        with open(APPLICATION_PROPERTIES_FILE_PATH, "r") as file:            contents = file.read()            properties = json.loads(contents)            return properties    else:        print('A file at "{}" was not found'.format(APPLICATION_PROPERTIES_FILE_PATH))def property_map(props, property_group_id):    for prop in props:        if prop["PropertyGroupId"] == property_group_id:            return prop["PropertyMap"]def create_source_table(table_name, stream_name, region, stream_initpos):    return """ CREATE TABLE {0} (                ticker VARCHAR(6),                price DOUBLE,                event_time TIMESTAMP(3),                WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND              )              PARTITIONED BY (ticker)              WITH (                'connector' = 'kinesis',                'stream' = '{1}',                'aws.region' = '{2}',                'scan.stream.initpos' = '{3}',                'format' = 'json',                'json.timestamp-format.standard' = 'ISO-8601'              ) """.format(        table_name, stream_name, region, stream_initpos    )def create_sink_table(table_name, bucket_name):    return """ CREATE TABLE {0} (                ticker VARCHAR(6),                price DOUBLE,                event_time TIMESTAMP(3),                WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND              )              PARTITIONED BY (ticker)              WITH (                  'connector'='filesystem',                  'path'='s3a://{1}/',                  'format'='csv',                  'sink.partition-commit.policy.kind'='success-file',                  'sink.partition-commit.delay' = '1 min'              ) """.format(        table_name, bucket_name)def count_by_word(input_table_name):    # 使用 Table API    input_table = table_env.from_path(input_table_name)    tumbling_window_table = (        input_table.window(            Tumble.over("1.minute").on("event_time").alias("one_minute_window")        )        .group_by("ticker, one_minute_window")        .select("ticker, price.avg as price, one_minute_window.end as event_time")    )    return tumbling_window_tabledef main():    # KDA 应用程序属性键    input_property_group_key = "consumer.config.0"    sink_property_group_key = "sink.config.0"    input_stream_key = "input.stream.name"    input_region_key = "aws.region"    input_starting_position_key = "flink.stream.initpos"    output_sink_key = "output.bucket.name"    # 输入输出数据表    input_table_name = "input_table"    output_table_name = "output_table"    # 获取 KDA 应用程序属性    props = get_application_properties()    input_property_map = property_map(props, input_property_group_key)    output_property_map = property_map(props, sink_property_group_key)    input_stream = input_property_map[input_stream_key]    input_region = input_property_map[input_region_key]    stream_initpos = input_property_map[input_starting_position_key]    output_bucket_name = output_property_map[output_sink_key]    # 2. 创建源 Kinesis Data Stream    table_env.execute_sql(        create_source_table(            input_table_name, input_stream, input_region, stream_initpos        )    )    # 3. 创建目标 S3 Bucket    create_sink = create_sink_table(        output_table_name, output_bucket_name    )    table_env.execute_sql(create_sink)    # 4. 执行窗口函数查询    tumbling_window_table = count_by_word(input_table_name)    # 5. 将结果写入目标    tumbling_window_table.execute_insert(output_table_name).wait()    statement_set.execute()if __name__ == "__main__":    main()

因为应用程序要使用到 Amazon Kinesis Flink SQL Connector，这里需要将对应的**amazon-kinesis-sql-connector-flink-2.0.3.jar**下载下来。

-   amazon-kinesis-sql-connector-flink-2.0.3.jar

    [https://repo1.maven.org/maven2/software/amazon/kinesis/amazon-kinesis-sql-connector-flink/2.0.3/amazon-kinesis-sql-connector-flink-2.0.3.jar](https://repo1.maven.org/maven2/software/amazon/kinesis/amazon-kinesis-sql-connector-flink/2.0.3/amazon-kinesis-sql-connector-flink-2.0.3.jar)

将 kda-pyflink-demo.py 和 amazon-kinesis-sql-connector-flink-2.0.3.jar 打包成 zip 文件，例如 kda-pyflink-demo.zip/；然后，将这个 zip 包上传到刚刚创建的 Amazon S3 存储桶中。进入刚刚创建的 Amazon S3 存储桶，点击 “上传”。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLIruwdoklvmL8XXEjSwZSsicEmLOB9qDNI4wMdcXD6eVDfh2OIhncn4Q/640?wx_fmt=jpeg)

选择刚刚打包好的 zip 文件，然后点击 “上传”。  

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLWytbbWHg6CLofQeQ1GlvrAYGShn2jWcicpvA4pCmuicE08canm1kKf7w/640?wx_fmt=png)

**创建 Python Amazon Kinesis Data Analytics**

**应用程序**

首先选择到 Amazon Kinesis 服务 - Data Analytics，然后点击 “创建应用程序”

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLkUKoUTFLJUR4BxOiavkvibPrEn3FDUv5YY6T8iaicvz4dgJg8e3NalvnPQ/640?wx_fmt=png)

输入应用程序名称，例如 “kda-pyflink-demo”；运行时选择 Apache Flink，保持默认 1.11 版本。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLTFNUY34uK8wxtWrB5IoLz8iblCukn4IdRTGdibuVxEwVKIJhhGzfGGyQ/640?wx_fmt=png)

访问权限保持默认，例如 “创建 / 更新 Amazon IAM 角色 kinesis-analytics-kda-pyflink-demo-cn-north-1”；应用程序设置的模板选择 “开发”，注意这里是为了演示，可以根据实际情况选择 “生产”。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLpsZvz2fFUI7SibSfQrSdEbyOJNUO37iaaTGX5jbwDf799ZAsl8MdLicXw/640?wx_fmt=png)

点击 “创建应用程序”，稍等片刻，应用程序创建完成。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLnwFKYuJ9ia0yGFjDd0xRzicMuBKcNJwzVMPQwZeaGIrJchgDuoK8JLTg/640?wx_fmt=png)

根据提示，我们继续配置应用程序，点击 “配置”；代码位置配置为刚刚创建的 Amazon S3 中的 zip 包位置。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLGa8WnYg9YAtv8vf5hYoLJYsibLzwXTFn6j1riaaWZYtz7xnCDAXRMJ0Q/640?wx_fmt=png)

然后展开属性配置。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLJKgHxx4MlkyYYFhemsSaqTpIpLbJwqCGz1L2L42fBK0icRnTMGfeGjQ/640?wx_fmt=png)

创建属性组，设置组名为 “consumer.config.0”，并配置以下键值对：

input.stream.name 为刚刚创建的 Amazon Kinesis 数据流，例如 kda-input-stream

aws.region 为当前区域，这里是 cn-north-1

flink.stream.initpos 设置读取流的位置，配置为 LATEST

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLzmxxH4x3gpsUrIeicCQT9tQibLhdATSb6MmcDPEMyiaQaWibSKN36Uznjw/640?wx_fmt=png)

创建属性组，设置组名为 “sink.config.0”，并配置以下键值对：

output.bucket.name 为刚刚创建的 Amazon S3 存储桶，例如 kda-pyflink-shtian

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLSOeO9OPCbsZ8icy725tXehxCjibNe7eScZlUQnNaibwHByGJBaKmWUpuw/640?wx_fmt=png)

创建属性组，设置组名为 “kinesis.analytics.flink.run.options”，并配置以下键值对：

python 为刚刚创建的 PyFlink 程序，kda-pyflink-demo.py

jarfile 为 Amazon Kinesis Connector 的名称，这里是 amazon-kinesis-sql-connector-flink-2.0.3.jar

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLVuuB6GxWGuqibsV2icf1VvTh3TPXcpiaib3AbJcRzoGZzQUQ4icUM5gtt7Q/640?wx_fmt=png)

然后点击 “更新”，刷新应用程序配置

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLyYKIEWvhndftDLshD2LeibbqSeJD7StibSMCMdw3u5ibsvoZRoKicAa0hw/640?wx_fmt=png)

接下来，配置应用程序使用的 Amazon IAM 角色的权限。进入到 Amazon IAM 界面，选择角色，然后找到刚刚新创建的角色。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLP7uDN9onBZ0KbCqaTrl5cfB9H4MKXwPQVHUibF1FEicXYzfySumBpLow/640?wx_fmt=png)

然后，展开附加策略，并点击 “编辑策略”。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcL1GmngVtMmEK3J37RUwbIeLSxP8xb6gYzMkKZ2KeLIliaZ8EMicjtBpHg/640?wx_fmt=png)

补充最后两段 Amazon IAM 策略，允许该角色可以访问 Amazon Kinesis 数据流和 Amazon S3 存储桶，注意需要替换为您的亚马逊云科技中国区账号。

    {    "Version": "2012-10-17",    "Statement": [        {            "Sid": "ReadCode",            "Effect": "Allow",            "Action": [                "s3:GetObject",                "s3:GetObjectVersion"            ],            "Resource": [                "arn:aws-cn:s3:::kda-pyflink-shtian/kda-pyflink-demo.zip"            ]        },        {            "Sid": "ListCloudwatchLogGroups",            "Effect": "Allow",            "Action": [                "logs:DescribeLogGroups"            ],            "Resource": [                "arn:aws-cn:logs:cn-north-1:012345678901:log-group:*"            ]        },        {            "Sid": "ListCloudwatchLogStreams",            "Effect": "Allow",            "Action": [                "logs:DescribeLogStreams"            ],            "Resource": [                "arn:aws-cn:logs:cn-north-1:012345678901:log-group:/aws/kinesis-analytics/kda-pyflink-demo:log-stream:*"            ]        },        {            "Sid": "PutCloudwatchLogs",            "Effect": "Allow",            "Action": [                "logs:PutLogEvents"            ],            "Resource": [                "arn:aws-cn:logs:cn-north-1:012345678901:log-group:/aws/kinesis-analytics/kda-pyflink-demo:log-stream:kinesis-analytics-log-stream"            ]        },        {            "Sid": "ReadInputStream",            "Effect": "Allow",            "Action": "kinesis:*",            "Resource": "arn:aws-cn:kinesis:cn-north-1:012345678901:stream/kda-input-stream"        },        {            "Sid": "WriteObjects",            "Effect": "Allow",            "Action": [                "s3:Abort*",                "s3:DeleteObject*",                "s3:GetObject*",                "s3:GetBucket*",                "s3:List*",                "s3:ListBucket",                "s3:PutObject"            ],            "Resource": [                "arn:aws-cn:s3:::kda-pyflink-shtian",                "arn:aws-cn:s3:::kda-pyflink-shtian/*"            ]        }    ]}

回到 Amazon Kinesis Data Analytics 应用程序界面，点击 “运行”。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLaCqJVn51sHuq3Lkx807aVpvOfLUQDSkYISB7AZcxuUgI0VjlGu4LkQ/640?wx_fmt=png)

点击 “打开 Apache Flink 控制面板”，跳转到 Flink 的界面。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLShhR0bicmtbIeB3JEYwIsicxqIWxDw65RicdvJascZkxM2ZYibiagjgCxUw/640?wx_fmt=jpeg)

点击查看正在运行的任务。  

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLCBJBWzbLwTJdr9sQE20A55ib1yWXCQicB6dgV2JnoQUtrXZ1lqaAKZcw/640?wx_fmt=jpeg)

您可以根据需求进一步查看详细信息。下面，我们到 Amazon S3 中验证数据是否已经写入，进入到创建的存储桶，可以看到数据已经成功写入。  

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcL8s3XWJ6rz44mHOsR7hZIkLjibmWodyXsh3xSo1QZ5jErgCYboWKFhbA/640?wx_fmt=png)

**使用 Amazon Glue 对数据进行爬取**

进入到 Amazon Glue 服务界面，选择爬网程序，点击 “添加爬网程序 “，输入爬网程序名称。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLhBSh0gzL8FtJWibPaJ9YMgtheI8sy1VtXa3f5I55RTna0dO6YY8vpBQ/640?wx_fmt=png)

保持源类型不变，添加数据存储为创建的 Amazon S3 存储桶输出路径。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLZGoc0fMYbOTHmKM621MfsoVy3RbThsWgibJ8M79n1nEuEjq0Ouz5hKQ/640?wx_fmt=png)

选择已有角色或者创建一个新的角色。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLJm98dMISiazojnEKz65ecibpERczU25pKLJMDepxXHvJbkQhKiblGZa7w/640?wx_fmt=jpeg)

选择默认数据库，可以根据需求添加表前缀。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLbxraAdEiaZ1TmvsmbwBubQjWZR3zgwg77LdfwDYiarP92lFffPhS314g/640?wx_fmt=png)

创建完成后，点击执行。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLK6nnoCIDkgEEuWhGjlNeyz5SC9NzibQiasXCpT4rwPvFgIBBDYsibhO8w/640?wx_fmt=jpeg)

爬取成功后，可以在数据表中查看到详细信息。  

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLBkalVM1dB07esP4ZP29fSvd751f5EzZLRA3Hic1MxlhAEXXeVaRt6Pg/640?wx_fmt=png)

然后，可以切换到 Amazon Athena 服务来查询结果。

![](https://mmbiz.qpic.cn/mmbiz_png/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLEdse8Cud9tyTb3dIqA7iacQDibIvQla1PWTib6lUZWVecTBx6zhFdX1hQ/640?wx_fmt=png)

注：如果出现 Amazon Glue 爬网程序或者 Amazon Athena 查询权限错误，可能是由于开启 Lake Formation 导致，可以参考**文档**授予角色相应的权限。

-   文档

    [https://docs.aws.amazon.com/lake-formation/latest/dg/granting-catalog-permissions.html](https://docs.aws.amazon.com/lake-formation/latest/dg/granting-catalog-permissions.html)

**小结**

本文首先介绍了在亚马逊云科技平台上使用 Apache Flink 的快速方式 – Amazon Kinesis Data Analytics for Flink，然后通过一个无服务器架构的示例演示了如何在 Amazon Kinesis Data Analytics for Flink 通过 PyFlink 实现 Python 流数据处理和分析，并通过 Amazon Glue 和 Amazon Athena 对数据进行即席查询。Amazon Kinesis Data Analytics for Flink 对 Python 的支持也已经在在光环新网运营的亚马逊云科技中国（北京）区域及西云数据运营的亚马逊云科技中国（宁夏）区域上线，欢迎使用。

**参考资料**

1.  [https://aws.amazon.com/solutions/implementations/aws-streaming-data-solution-for-amazon-kinesis/](https://aws.amazon.com/solutions/implementations/aws-streaming-data-solution-for-amazon-kinesis/)
2.  [https://docs.aws.amazon.com/lake-formation/latest/dg/granting-catalog-permissions.html](https://docs.aws.amazon.com/lake-formation/latest/dg/granting-catalog-permissions.html)
3.  [https://docs.aws.amazon.com/kinesisanalytics/latest/java/examples-python-s3.html](https://docs.aws.amazon.com/kinesisanalytics/latest/java/examples-python-s3.html)
4.  [https://ci.apache.org/projects/flink/flink-docs-release-1.11/](https://ci.apache.org/projects/flink/flink-docs-release-1.11/)

相关阅读

\[

推出 Amazon Kinesis Data Analytics Studio —— 

与流数据快速交互

点击图片查看原文

]([https://mp.weixin.qq.com/s?\_\_biz=Mzg4NjU5NDUxNg==&mid=2247491154&idx=1&sn=4cd236d18398351b30b0ad399c0deaff&scene=21#wechat_redirect](https://mp.weixin.qq.com/s?__biz=Mzg4NjU5NDUxNg==&mid=2247491154&idx=1&sn=4cd236d18398351b30b0ad399c0deaff&scene=21#wechat_redirect) "[https://mp.weixin.qq.com/s?\_\_biz=Mzg4NjU5NDUxNg==&mid=2247491154&idx=1&sn=4cd236d18398351b30b0ad399c0deaff&scene=21#wechat_redirect"](https://mp.weixin.qq.com/s?__biz=Mzg4NjU5NDUxNg==&mid=2247491154&idx=1&sn=4cd236d18398351b30b0ad399c0deaff&scene=21#wechat_redirect"))

**本篇作者**

![](https://mmbiz.qpic.cn/mmbiz_jpg/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLicEcQqGLTge4hVtzTEibE5kiaqF2j4ybT8ACPuvreIY4xdDpFHianyByeg/640?wx_fmt=jpeg)

**史天**

_亚马逊云科技解决方案架构师_

拥有丰富的云计算、大数据和机器学习经验，目前致力于数据科学、机器学习、无服务器等领域的研究和实践。译有《机器学习即服务》《基于 Kubernetes 的 DevOps 实践》《Prometheus 监控实战》等。

![](https://mmbiz.qpic.cn/mmbiz_gif/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLHqeXSvVp2IfALeFpxEB0qqgwJibRP8zqvuRd5j6tlm12G9LsuUmKvrA/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_gif/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLB2qAJkIDhK4StVicHC88BJbHbxolQrNlMAibk1iawibez2GqPt39y1qKCQ/640?wx_fmt=gif)

**听说，点完下面 4 个按钮**

**就不会碰到 bug 了！**

![](https://mmbiz.qpic.cn/mmbiz_gif/kbAJm9Dy4QScibSpgNJb4q4xox8A3jTcLB2qAJkIDhK4StVicHC88BJbHbxolQrNlMAibk1iawibez2GqPt39y1qKCQ/640?wx_fmt=gif) 
 [https://mp.weixin.qq.com/s/fOxx37LCsP4NvcQwG1t0Rg](https://mp.weixin.qq.com/s/fOxx37LCsP4NvcQwG1t0Rg)
