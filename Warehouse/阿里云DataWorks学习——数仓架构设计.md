# 阿里云DataWorks学习——数仓架构设计
ODS 层存放您从业务系统获取的最原始的数据，是其他上层数据的源数据。业务数据系统中的数据通常为非常细节的数据，经过长时间累积，且访问频率很高，是面向应用的数据。

说明 在构建 MaxCompute 数据仓库的表之前，您需要首先了解 MaxCompute 支持的数据类型版本说明。

## 数据引入层表设计

本教程中，在 ODS 层主要包括的数据有：交易系统订单详情、用户信息详情、商品详情等。这些数据未经处理，是最原始的数据。逻辑上，这些数据都是以二维表的形式存储。虽然严格的说 ODS 层不属于数仓建模的范畴，但是合理的规划 ODS 层并做好数据同步也非常重要。本教程中，使用了 6 张 ODS 表：

-   记录用于拍卖的商品信息：s_auction。
-   记录用于正常售卖的商品信息：s_sale。
-   记录用户详细信息：s_users_extra。
-   记录新增的商品成交订单信息：s_biz_order_delta。
-   记录新增的物流订单信息：s_logistics_order_delta。
-   记录新增的支付订单信息：s_pay_order_delta。

说明

-   表或字段命名尽量和业务系统保持一致，但是需要通过额外的标识来区分增量和全量表。例如，我们通过\_delta 来标识该表为增量表。
-   命名时需要特别注意冲突处理，例如不同业务系统的表可能是同一个名称。为区分两个不同的表，您可以将这两个同名表的来源数据库名称作为后缀或前缀。例如，表中某些字段的名称刚好和关键字重名了，可以通过添加\_col1 后缀解决。

## ODS 层设计规范

ODS 层表命名、数据同步任务命名、数据产出及生命周期管理及数据质量规范请参见 ODS 层设计规范。

## 建表示例

为方便您使用，集中提供建表语句如下。更多建表信息，请参见表操作。

    CREATE TABLE IF NOT EXISTS s_auction(    id                             STRING COMMENT '商品ID',    title                          STRING COMMENT '商品名',    gmt_modified                   STRING COMMENT '商品最后修改日期',    price                          DOUBLE COMMENT '商品成交价格，单位元',    starts                         STRING COMMENT '商品上架时间',    minimum_bid                    DOUBLE COMMENT '拍卖商品起拍价，单位元',    duration                       STRING COMMENT '有效期，销售周期，单位天',    incrementnum                   DOUBLE COMMENT '拍卖价格的增价幅度',    city                           STRING COMMENT '商品所在城市',    prov                           STRING COMMENT '商品所在省份',    ends                           STRING COMMENT '销售结束时间',    quantity                       BIGINT COMMENT '数量',    stuff_status                   BIGINT COMMENT '商品新旧程度 0 全新 1 闲置 2 二手',    auction_status                 BIGINT COMMENT '商品状态 0 正常 1 用户删除 2 下架 3 从未上架',    cate_id                         BIGINT COMMENT '商品类目ID',    cate_name                        STRING COMMENT '商品类目名称',    commodity_id                     BIGINT COMMENT '品类ID',    commodity_name                    STRING COMMENT '品类名称',    umid                              STRING COMMENT '买家umid')COMMENT '商品拍卖ODS'PARTITIONED BY (ds         STRING COMMENT '格式：YYYYMMDD')LIFECYCLE 400;CREATE TABLE IF NOT EXISTS s_sale(    id                             STRING COMMENT '商品ID',    title                          STRING COMMENT '商品名',    gmt_modified                   STRING COMMENT '商品最后修改日期',    starts                         STRING COMMENT '商品上架时间',    price                          DOUBLE COMMENT '商品价格，单位元',    city                           STRING COMMENT '商品所在城市',    prov                           STRING COMMENT '商品所在省份',    quantity                       BIGINT COMMENT '数量',    stuff_status                   BIGINT COMMENT '商品新旧程度 0 全新 1 闲置 2 二手',    auction_status                 BIGINT COMMENT '商品状态 0 正常 1 用户删除 2 下架 3 从未上架',    cate_id                      BIGINT COMMENT '商品类目ID',    cate_name                    STRING COMMENT '商品类目名称',    commodity_id                 BIGINT COMMENT '品类ID',    commodity_name                STRING COMMENT '品类名称',    umid                          STRING COMMENT '买家umid')COMMENT '商品正常购买ODS'PARTITIONED BY (ds      STRING COMMENT '格式：YYYYMMDD')LIFECYCLE 400;CREATE TABLE IF NOT EXISTS s_users_extra(    id                STRING COMMENT '用户ID',    logincount        BIGINT COMMENT '登录次数',    buyer_goodnum     BIGINT COMMENT '作为买家的好评数',    seller_goodnum    BIGINT COMMENT '作为卖家的好评数',    level_type        BIGINT COMMENT '1 一级店铺 2 二级店铺 3 三级店铺',    promoted_num      BIGINT COMMENT '1 A级服务　2 B级服务　3 C级服务',    gmt_create        STRING COMMENT '创建时间',    order_id          BIGINT COMMENT '订单ID',    buyer_id          BIGINT COMMENT '买家ID',    buyer_nick        STRING COMMENT '买家昵称',    buyer_star_id     BIGINT COMMENT '买家星级 ID',    seller_id         BIGINT COMMENT '卖家ID',    seller_nick       STRING COMMENT '卖家昵称',    seller_star_id    BIGINT COMMENT '卖家星级ID',    shop_id           BIGINT COMMENT '店铺ID',    shop_name         STRING COMMENT '店铺名称')COMMENT '用户扩展表'PARTITIONED BY (ds       STRING COMMENT 'yyyymmdd')LIFECYCLE 400;CREATE TABLE IF NOT EXISTS s_biz_order_delta(    biz_order_id         STRING COMMENT '订单ID',    pay_order_id         STRING COMMENT '支付订单ID',    logistics_order_id   STRING COMMENT '物流订单ID',    buyer_nick           STRING COMMENT '买家昵称',    buyer_id             STRING COMMENT '买家ID',    seller_nick          STRING COMMENT '卖家昵称',    seller_id            STRING COMMENT '卖家ID',    auction_id           STRING COMMENT '商品ID',    auction_title        STRING COMMENT '商品标题 ',    auction_price        DOUBLE COMMENT '商品价格',    buy_amount           BIGINT COMMENT '购买数量',    buy_fee              BIGINT COMMENT '购买金额',    pay_status           BIGINT COMMENT '支付状态 1 未付款  2 已付款 3 已退款',    logistics_id         BIGINT COMMENT '物流订单ID',    mord_cod_status      BIGINT COMMENT '物流状态 0 初始状态 1 接单成功 2 接单超时3 揽收成功 4揽收失败 5 签收成功 6 签收失败 7 用户取消物流订单',    status               BIGINT COMMENT '状态 0 订单正常 1 订单不可见',    sub_biz_type         BIGINT COMMENT '业务类型 1 拍卖 2 购买',    end_time             STRING COMMENT '交易结束时间',    shop_id              BIGINT COMMENT '店铺ID')COMMENT '交易成功订单日增量表'PARTITIONED BY (ds       STRING COMMENT 'yyyymmdd')LIFECYCLE 7200;CREATE TABLE IF NOT EXISTS s_logistics_order_delta(    logistics_order_id STRING COMMENT '物流订单ID ',    post_fee           DOUBLE COMMENT '物流费用',    address            STRING COMMENT '收货地址',    full_name          STRING COMMENT '收货人全名',    mobile_phone       STRING COMMENT '移动电话',    prov               STRING COMMENT '省份',    prov_code          STRING COMMENT '省份ID',    city               STRING COMMENT '市',    city_code          STRING COMMENT '城市ID',    logistics_status   BIGINT COMMENT '物流状态1 - 未发货2 - 已发货3 - 已收货4 - 已退货5 - 配货中',    consign_time       STRING COMMENT '发货时间',    gmt_create         STRING COMMENT '订单创建时间',    shipping           BIGINT COMMENT '发货方式1，平邮2，快递3，EMS',    seller_id          STRING COMMENT '卖家ID',    buyer_id           STRING COMMENT '买家ID')COMMENT '交易物流订单日增量表'PARTITIONED BY (ds                 STRING COMMENT '日期')LIFECYCLE 7200;CREATE TABLE IF NOT EXISTS s_pay_order_delta(    pay_order_id     STRING COMMENT '支付订单ID',    total_fee        DOUBLE COMMENT '应支付总金额 （数量*单价）',    seller_id STRING COMMENT '卖家ID',    buyer_id  STRING COMMENT '买家ID',    pay_status       BIGINT COMMENT '支付状态1等待买家付款，2等待卖家发货，3交易成功',    pay_time         STRING COMMENT '付款时间',    gmt_create       STRING COMMENT '订单创建时间',    refund_fee       DOUBLE COMMENT '退款金额（包含运费）',    confirm_paid_fee DOUBLE COMMENT '已经确认收货的金额')COMMENT '交易支付订单增量表'PARTITIONED BY (ds        STRING COMMENT '日期')LIFECYCLE 7200;

## 数据引入层存储

为了满足历史数据分析需求，您可以在 ODS 层表中添加时间维度作为分区字段。实际应用中，您可以选择采用增量、全量存储或拉链存储的方式。

-   增量存储

    以天为单位的增量存储，以业务日期作为分区，每个分区存放日增量的业务数据。举例如下：

    说明 交易、日志等事务性较强的 ODS 表适合增量存储方式。这类表数据量较大，采用全量存储的方式存储成本压力大。此外，这类表的下游应用对于历史全量数据访问的需求较小（此类需求可通过数据仓库后续汇总后得到）。例如，日志类 ODS 表没有数据更新的业务过程，因此所有增量分区 UNION 在一起就是一份全量数据。


-   1 月 1 日，用户 A 访问了 A 公司电商店铺 B，A 公司电商日志产生一条记录 t1。1 月 2 日，用户 A 又访问了 A 公司电商店铺 C，A 公司电商日志产生一条记录 t2。采用增量存储方式，t1 将存储在 1 月 1 日这个分区中，t2 将存储在 1 月 2 日这个分区中。
-   1 月 1 日，用户 A 在 A 公司电商网购买了 B 商品，交易日志将生成一条记录 t1。1 月 2 日，用户 A 又将 B 商品退货了，交易日志将更新 t1 记录。采用增量存储方式，初始购买的 t1 记录将存储在 1 月 1 日这个分区中，更新后的 t1 将存储在 1 月 2 日这个分区中。


-   全量存储

    以天为单位的全量存储，以业务日期作为分区，每个分区存放截止到业务日期为止的全量业务数据。例如， 1 月 1 日，卖家 A 在 A 公司电商网发布了 B、C 两个商品，前端商品表将生成两条记录 t1、t2。1 月 2 日，卖家 A 将 B 商品下架了，同时又发布了商品 D，前端商品表将更新记录 t1，同时新生成记录 t3。采用全量存储方式， 在 1 月 1 日这个分区中存储 t1 和 t2 两条记录，在 1 月 2 日这个分区中存储更新后的 t1 以及 t2、t3 记录。

    说明 对于小数据量的缓慢变化维度数据，例如商品类目，可直接使用全量存储。
-   拉链存储

    拉链存储通过新增两个时间戳字段（start_dt 和 end_dt），将所有以天为粒度的变更数据都记录下来，通常分区字段也是这两个时间戳字段。

    拉链存储举例如下。

    | 商品  | start_dt | end_dt   | 卖家  | 状态  |
    | --- | -------- | -------- | --- | --- |
    | B   | 20160101 | 20160102 | A   | 上架  |
    | C   | 20160101 | 30001231 | A   | 上架  |
    | B   | 20160102 | 30001231 | A   | 下架  |

    这样，下游应用可以通过限制时间戳字段来获取历史数据。例如，用户访问 1 月 1 日数据，只需限制`start_dt<=20160101`并且 `end_dt>20160101`。

## 缓慢变化维度

MaxCompute 不推荐使用代理键，推荐使用自然键作为维度主键，主要原因有两点：

1.  MaxCompute 是分布式计算引擎，生成全局唯一的代理键工作量非常大。当遇到大数据量情况下，这项工作就会更加复杂，且没有必要。
2.  使用代理键会增加 ETL 的复杂性，从而增加 ETL 任务的开发和维护成本。

在不使用代理键的情况下，缓慢变化维度可以通过快照方式处理。

快照方式下数据的计算周期通常为每天一次。基于该周期，处理维度变化的方式为每天一份全量快照。

例如商品维度，每天保留一份全量商品快照数据。任意一天的事实表均可以取到当天的商品信息，也可以取到最新的商品信息，通过限定日期，采用自然键进行关联即可。该方式的优势主要有以下两点：

-   处理缓慢变化维度的方式简单有效，开发和维护成本低。
-   使用方便，易于理解。数据使用方只需要限定日期即可取到当天的快照数据。任意一天的事实快照与任意一天的维度快照通过维度的自然键进行关联即可。

该方法的弊端主要是存储空间的极大浪费。例如某维度每天的变化量占总体数据量比例很低，极端情况下，每天无变化，这种情况下存储浪费严重。该方法主要实现了通过牺牲存储获取 ETL 效率的优化和逻辑上的简化。请避免过度使用该方法，且必须要有对应的数据生命周期制度，清除无用的历史数据。

## 数据同步加载与处理

ODS 的数据需要由各数据源系统同步到 MaxCompute，才能用于进一步的数据开发。本教程建议您使用 DataWorks 数据集成功能完成数据同步，详情请参见概述。在使用数据集成的过程中，建议您遵循以下规范：

-   一个系统的源表只允许同步到 MaxCompute 一次，保持表结构的一致性。
-   数据集成仅用于离线全量数据同步，实时增量数据同步需要您使用数据传输服务 DTS 实现，详情请参见数据传输服务 DTS。
-   数据集成全量同步的数据直接进入全量表的当日分区。
-   ODS 层的表建议以统计日期及时间分区表的方式存储，便于管理数据的存储成本和策略控制。
-   数据集成可以自适应处理源系统字段的变更：


-   如果源系统字段的目标表在 MaxCompute 上不存在，可以由数据集成自动添加不存在的表字段。
-   如果目标表的字段在源系统不存在，数据集成填充 NULL。

公共维度汇总层（DIM）基于维度建模理念，建立整个企业的一致性维度。

公共维度汇总层（DIM）主要由维度表（维表）构成。维度是逻辑概念，是衡量和观察业务的角度。维表是根据维度及其属性将数据平台上构建的表物理化的表，采用宽表设计的原则。因此，构建公共维度汇总层（DIM）首先需要定义维度。

## 定义维度

在划分数据域、构建总线矩阵时，需要结合对业务过程的分析定义维度。以本教程中 A 电商公司的营销业务板块为例，在交易数据域中，我们重点考察确认收货（交易成功）的业务过程。

在确认收货的业务过程中，主要有商品和收货地点（本教程中，假设收货和购买是同一个地点）两个维度所依赖的业务角度。从商品角度可以定义出以下维度：

-   商品 ID
-   商品名称
-   商品价格
-   商品新旧程度：0 - 全新、1 - 闲置、 2 - 二手
-   商品类目 ID
-   商品类目名称
-   品类 ID
-   品类名称
-   买家 ID
-   商品状态：0 - 正常、1 - 用户删除、2 - 下架、3 - 从未上架
-   商品所在城市
-   商品所在省份

从地域角度，可以定义出以下维度：

-   买家 ID
-   城市 code
-   城市名称
-   省份 code
-   省份名称

作为维度建模的核心，在企业级数据仓库中必须保证维度的唯一性。以 A 公司的商品维度为例，有且只允许有一种维度定义。例如，省份 code 这个维度，对于任何业务过程所传达的信息都是一致的。

## 设计维表

完成维度定义后，您就可以对维度进行补充，进而生成维表了。维表的设计需要注意：

-   建议维表单表信息不超过 1000 万条。
-   维表与其他表进行 Join 时，建议您使用 Map Join。
-   避免过于频繁的更新维表的数据。

在设计维表时，您需要从下列方面进行考虑：

-   维表中数据的稳定性。例如 A 公司电商会员通常不会出现消亡，但会员数据可能在任何时候更新，此时要考虑创建单个分区存储全量数据。如果存在不会更新的记录，您可能需要分别创建历史表与日常表。日常表用于存放当前有效的记录，保持表的数据量不会膨胀；历史表根据消亡时间插入对应分区，使用单个分区存放分区对应时间的消亡记录。
-   是否需要垂直拆分。如果一个维表存在大量属性不被使用，或由于承载过多属性字段导致查询变慢，则需考虑对字段进行拆分，创建多个维表。
-   是否需要水平拆分。如果记录之间有明显的界限，可以考虑拆成多个表或设计成多级分区。
-   核心的维表产出时间通常有严格的要求。

设计维表的主要步骤如下：

1.  完成维度的初步定义，并保证维度的一致性。
2.  确定主维表（中心事实表，本教程中采用星型模型）。此处的主维表通常是数据引入层（ODS）表，直接与业务系统同步。例如，s_auction 是与前台商品中心系统同步的商品表，此表即是主维表。
3.  确定相关维表。数据仓库是业务源系统的数据整合，不同业务系统或者同一业务系统中的表之间存在关联性。根据对业务的梳理，确定哪些表和主维表存在关联关系，并选择其中的某些表用于生成维度属性。以商品维度为例，根据对业务逻辑的梳理，可以得到商品与类目、卖家、店铺等维度存在关联关系。
4.  确定维度属性，主要包括两个阶段。第一个阶段是从主维表中选择维度属性或生成新的维度属性；第二个阶段是从相关维表中选择维度属性或生成新的维度属性。以商品维度为例，从主维表（s_auction）和类目 、卖家、店铺等相关维表中选择维度属性或生成新的维度属性。

-   尽可能生成丰富的维度属性。
-   尽可能多地给出富有意义的文字性描述。
-   区分数值型属性和事实。
-   尽量沉淀出通用的维度属性。

## 公共维度汇总层（DIM）维表规范

公共维度汇总层（DIM）维表命名规范：dim_{业务板块名称 / pub}_{维度定义}\[\_{自定义命名标签}]，所谓 pub 是与具体业务板块无关或各个业务板块都可公用的维度，如时间维度。举例如下：

-   公共区域维表 dim_pub_area
-   A 公司电商板块的商品全量表 dim_asale_itm

## 建表示例

本例中，最终的维表建表语句如下所示。

    CREATE TABLE IF NOT EXISTS dim_asale_itm(    item_id                            BIGINT COMMENT '商品ID',    item_title                      STRING COMMENT '商品名称',    item_price                     DOUBLE COMMENT '商品成交价格_元',    item_stuff_status              BIGINT COMMENT '商品新旧程度_0全新1闲置2二手',    cate_id                          BIGINT COMMENT '商品类目ID',    cate_name                        STRING COMMENT '商品类目名称',    commodity_id                      BIGINT COMMENT '品类ID',    commodity_name                  STRING COMMENT '品类名称',    umid                           STRING COMMENT '买家ID',    item_status                    BIGINT COMMENT '商品状态_0正常1用户删除2下架3未上架',    city                           STRING COMMENT '商品所在城市',    prov                           STRING COMMENT '商品所在省份')COMMENT '商品全量表'PARTITIONED BY (ds        STRING COMMENT '日期,yyyymmdd');CREATE TABLE IF NOT EXISTS dim_pub_area(    buyer_id       STRING COMMENT '买家ID',    city_code      STRING COMMENT '城市code',    city_name      STRING COMMENT '城市名称',    prov_code      STRING COMMENT '省份code',    prov_name      STRING COMMENT '省份名称')COMMENT '公共区域维表'PARTITIONED BY (ds             STRING COMMENT '日期分区，格式yyyymmdd')LIFECYCLE 3600;

明细粒度事实层以业务过程驱动建模，基于每个具体的业务过程特点，构建最细粒度的明细层事实表。您可以结合企业的数据使用特点，将明细事实表的某些重要维度属性字段做适当冗余，即宽表化处理。

公共汇总粒度事实层（DWS）和明细粒度事实层（DWD）的事实表作为数据仓库维度建模的核心，需紧绕业务过程来设计。通过获取描述业务过程的度量来描述业务过程，包括引用的维度和与业务过程有关的度量。度量通常为数值型数据，作为事实逻辑表的依据。事实逻辑表的描述信息是事实属性，事实属性中的外键字段通过对应维度进行关联。

事实表中一条记录所表达的业务细节程度被称为粒度。通常粒度可以通过两种方式来表述：一种是维度属性组合所表示的细节程度，一种是所表示的具体业务含义。

作为度量业务过程的事实，通常为整型或浮点型的十进制数值，有可加性、半可加性和不可加性三种类型：

-   可加性事实是指可以按照与事实表关联的任意维度进行汇总。
-   半可加性事实只能按照特定维度汇总，不能对所有维度汇总。例如库存可以按照地点和商品进行汇总，而按时间维度把一年中每个月的库存累加则毫无意义。
-   完全不可加性，例如比率型事实。对于不可加性的事实，可分解为可加的组件来实现聚集。

事实表相对维表通常更加细长，行增加速度也更快。维度属性可以存储到事实表中，这种存储到事实表中的维度列称为维度退化，可加快查询速度。与其他存储在维表中的维度一样，维度退化可以用来进行事实表的过滤查询、实现聚合操作等。

明细粒度事实层（DWD）通常分为三种：事务事实表、周期快照事实表和累积快照事实表，详情请参见数仓建设指南。

-   事务事实表用来描述业务过程，跟踪空间或时间上某点的度量事件，保存的是最原子的数据，也称为原子事实表。
-   周期快照事实表以具有规律性的、可预见的时间间隔记录事实。
-   累积快照事实表用来表述过程开始和结束之间的关键步骤事件，覆盖过程的整个生命周期，通常具有多个日期字段来记录关键时间点。当累积快照事实表随着生命周期不断变化时，记录也会随着过程的变化而被修改。

## 明细粒度事实表设计原则

明细粒度事实表设计原则如下所示：

-   通常，一个明细粒度事实表仅和一个维度关联。
-   尽可能包含所有与业务过程相关的事实 。
-   只选择与业务过程相关的事实。
-   分解不可加性事实为可加的组件。
-   在选择维度和事实之前必须先声明粒度。
-   在同一个事实表中不能有多种不同粒度的事实。
-   事实的单位要保持一致。
-   谨慎处理 Null 值。
-   使用退化维度提高事实表的易用性。

明细粒度事实表整体设计流程如下图所示。![](https://mmbiz.qpic.cn/mmbiz_png/cv9Nw4dcXR6vb09lxbCPJ6rdtibljFoK8rWIFZfRl82tTxtciaib4L6Dxrld6TCSeP6XIYMSYJjPLHhpDia15NwFyg/640?wx_fmt=png)

在一致性度量中已定义好了交易业务过程及其度量。明细事实表注意针对业务过程进行模型设计。明细事实表的设计可以分为四个步骤：选择业务过程、确定粒度、选择维度、确定事实（度量）。粒度主要是在维度未展开的情况下记录业务活动的语义描述。在您建设明细事实表时，需要选择基于现有的表进行明细层数据的开发，清楚所建表记录存储的是什么粒度的数据。

## 明细粒度事实层（DWD）规范

通常您需要遵照的命名规范为：dwd_{业务板块 / pub}_{数据域缩写}_{业务过程缩写}\[_{自定义表命名标签缩写}] \_{单分区增量全量标识}，pub 表示数据包括多个业务板块的数据。单分区增量全量标识通常为：i 表示增量，f 表示全量。例如：dwd_asale_trd_ordcrt_trip_di（A 电商公司航旅机票订单下单事实表，日刷新增量）及 dwd_asale_itm_item_df（A 电商商品快照事实表，日刷新全量）。

本教程中，DWD 层主要由三个表构成：

-   交易商品信息事实表：dwd_asale_trd_itm_di。
-   交易会员信息事实表：ods_asale_trd_mbr_di。
-   交易订单信息事实表：dwd_asale_trd_ord_di。

DWD 层数据存储及生命周期管理规范请参见 CDM 明细层设计规范。

## 建表示例

本教程中充分使用了维度退化以提升查询效率，建表语句如下所示。

    CREATE TABLE IF NOT EXISTS dwd_asale_trd_itm_di(    item_id              BIGINT COMMENT '商品ID',    item_title           STRING COMMENT '商品名称',    item_price           DOUBLE COMMENT '商品价格',    item_stuff_status    BIGINT COMMENT '商品新旧程度_0全新1闲置2二手',    item_prov            STRING COMMENT '商品省份',    item_city            STRING COMMENT '商品城市',    cate_id              BIGINT COMMENT '商品类目ID',    cate_name            STRING COMMENT '商品类目名称',    commodity_id         BIGINT COMMENT '品类ID',    commodity_name       STRING COMMENT '品类名称',    buyer_id             BIGINT COMMENT '买家ID',)COMMENT '交易商品信息事实表'PARTITIONED BY (ds     STRING COMMENT '日期')LIFECYCLE 400;CREATE TABLE IF NOT EXISTS ods_asale_trd_mbr_di(    order_id         BIGINT COMMENT '订单ID',    bc_type          STRING COMMENT '业务分类',    buyer_id         BIGINT COMMENT '买家ID',    buyer_nick       STRING COMMENT '买家昵称',    buyer_star_id    BIGINT COMMENT '买家星级ID',    seller_id        BIGINT COMMENT '卖家ID',    seller_nick      STRING COMMENT '卖家昵称',    seller_star_id   BIGINT COMMENT '卖家星级ID',    shop_id          BIGINT COMMENT '店铺ID',    shop_name        STRING COMMENT '店铺名称')COMMENT '交易会员信息事实表'PARTITIONED BY (ds     STRING COMMENT '日期')LIFECYCLE 400;CREATE TABLE IF NOT EXISTS dwd_asale_trd_ord_di(    order_id              BIGINT COMMENT '订单ID',    pay_order_id          BIGINT COMMENT '支付订单ID',    pay_status            BIGINT COMMENT '支付状态_1未付款2已付款3已退款',    succ_time             STRING COMMENT '订单交易结束时间',    item_id               BIGINT COMMENT '商品ID',    item_quantity         BIGINT COMMENT '购买数量',    confirm_paid_amt      DOUBLE COMMENT '订单已经确认收货的金额',    logistics_id          BIGINT COMMENT '物流订单ID',    mord_prov             STRING COMMENT '收货人省份',    mord_city             STRING COMMENT '收货人城市',    mord_lgt_shipping     BIGINT COMMENT '发货方式_1平邮2快递3EMS',    mord_address          STRING COMMENT '收货人地址',    mord_mobile_phone     STRING COMMENT '收货人手机号',    mord_fullname         STRING COMMENT '收货人姓名',    buyer_nick            STRING COMMENT '买家昵称',    buyer_id              BIGINT COMMENT '买家ID')COMMENT '交易订单信息事实表'PARTITIONED BY (ds       STRING COMMENT '日期')LIFECYCLE 400;

公共汇总粒度事实层以分析的主题对象作为建模驱动，基于上层的应用和产品的指标需求构建公共粒度的汇总指标事实表。公共汇总层的一个表通常会对应一个派生指标。

## 公共汇总事实表设计原则

聚集是指针对原始明细粒度的数据进行汇总。DWS 公共汇总层是面向分析对象的主题聚集建模。在本教程中，最终的分析目标为：最近一天某个类目（例如：厨具）商品在各省的销售总额、该类目 Top10 销售额商品名称、各省用户购买力分布。因此，我们可以以最终交易成功的商品、类目、买家等角度对最近一天的数据进行汇总。

注意

-   聚集是不跨越事实的。聚集是针对原始星形模型进行的汇总。为获取和查询与原始模型一致的结果，聚集的维度和度量必须与原始模型保持一致，因此聚集是不跨越事实的。
-   聚集会带来查询性能的提升，但聚集也会增加 ETL 维护的难度。当子类目对应的一级类目发生变更时，先前存在的、已经被汇总到聚集表中的数据需要被重新调整。

此外，进行 DWS 层设计时还需遵循以下原则：

-   数据公用性：需考虑汇总的聚集是否可以提供给第三方使用。您可以判断，基于某个维度的聚集是否经常用于数据分析中。如果答案是肯定的，就有必要把明细数据经过汇总沉淀到聚集表中。
-   不跨数据域。数据域是在较高层次上对数据进行分类聚集的抽象。数据域通常以业务过程进行分类，例如交易统一划到交易域下， 商品的新增、修改放到商品域下。
-   区分统计周期。在表的命名上要能说明数据的统计周期，例如\_1d 表示最近 1 天，td 表示截至当天，nd 表示最近 N 天。

## 公共汇总事实表规范

公共汇总事实表命名规范：dws_{业务板块缩写 / pub}_{数据域缩写}_{数据粒度缩写}\[_{自定义表命名标签缩写}]\_{统计时间周期范围缩写}。

-   关于统计实际周期范围缩写，缺省情况下，离线计算应该包括最近一天（\_1d），最近 N 天（\_nd）和历史截至当天（\_td）三个表。如果出现\_nd 的表字段过多需要拆分时，只允许以一个统计周期单元作为原子拆分。即一个统计周期拆分一个表，例如最近 7 天（\_1w）拆分一个表。不允许拆分出来的一个表存储多个统计周期。
-   对于小时表（无论是天刷新还是小时刷新），都用\_hh 来表示。
-   对于分钟表（无论是天刷新还是小时刷新），都用\_mm 来表示。

举例如下：

-   dws_asale_trd_byr_subpay_1d（A 电商公司买家粒度交易分阶段付款一日汇总事实表）
-   dws_asale_trd_byr_subpay_td（A 电商公司买家粒度分阶段付款截至当日汇总表）
-   dws_asale_trd_byr_cod_nd（A 电商公司买家粒度货到付款交易汇总事实表）
-   dws_asale_itm_slr_td（A 电商公司卖家粒度商品截至当日存量汇总表）
-   dws_asale_itm_slr_hh（A 电商公司卖家粒度商品小时汇总表）--- 维度为小时
-   dws_asale_itm_slr_mm（A 电商公司卖家粒度商品分钟汇总表）--- 维度为分钟

DWS 层数据存储及生命周期管理规范请参见 CDM 汇总层设计规范。

## 建表示例

满足业务需求的 DWS 层建表语句如下。

    CREATE TABLE IF NOT EXISTS dws_asale_trd_byr_ord_1d(    buyer_id                BIGINT COMMENT '买家ID',    buyer_nick              STRING COMMENT '买家昵称',    mord_prov               STRING COMMENT '收货人省份',    cate_id                 BIGINT COMMENT '商品类目ID',    cate_name               STRING COMMENT '商品类目名称',    confirm_paid_amt_sum_1d DOUBLE COMMENT '最近一天订单已经确认收货的金额总和')COMMENT '买家粒度所有交易最近一天汇总事实表'PARTITIONED BY (ds         STRING COMMENT '分区字段YYYYMMDD')LIFECYCLE 36000;CREATE TABLE IF NOT EXISTS dws_asale_trd_itm_ord_1d(    item_id                 BIGINT COMMENT '商品ID',    item_title               STRING COMMENT '商品名称',    cate_id                 BIGINT COMMENT '商品类目ID',    cate_name               STRING COMMENT '商品类目名称',    mord_prov               STRING COMMENT '收货人省份',    confirm_paid_amt_sum_1d DOUBLE COMMENT '最近一天订单已经确认收货的金额总和')COMMENT '商品粒度交易最近一天汇总事实表'PARTITIONED BY (ds         STRING COMMENT '分区字段YYYYMMDD')LIFECYCLE 36000;

 [https://mp.weixin.qq.com/s/JLUdxAJLefHgqiTO6gqq0g](https://mp.weixin.qq.com/s/JLUdxAJLefHgqiTO6gqq0g)
