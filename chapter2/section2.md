# KairosDB

KairosDB最初是从OpenTSDB 1.x版本fork出来的一个分支，目的是在OpenTSDB的代码基础上进行二次开发来满足新的功能需求。其改造之一就是支持可插拔式的存储引擎，例如支持H2可以方便本地开发和测试，而不是像OpenTSDB一样与HBase强耦合。在其最初的几个版本中，HBase也是作为其主要的存储引擎。但是在之后的存储优化中，慢慢使用Cassandra替换了HBase，它也是第一个基于Cassandra开发的时序数据库。在最新的几个版本中，已不再支持HBase，因为其存储优化使用了Cassandra所特有而HBase没有的一些特性。



  在整体架构上，和OpenTSDB比较类似，都是采用了一个比较成熟的数据库来作为底层存储引擎。自己的主要逻辑仅仅是在存储引擎层之上很薄的一个逻辑层，这层逻辑层的部署架构是一个无状态的组件，可以很容易的水平扩展。



  在功能差异性上，它在OpenTSDB 1.x上做二次开发，也是为了对OpenTSDB的一些功能做优化，或做出一些OpenTSDB所没有的功能。我大概罗列下我看到的主要的功能差异：



可插拔式的存储引擎：OpenTSDB在早期与HBase强耦合，为了追求极致的性能，甚至自研了一个异步的HBase Client（现在作为独立的一个开源项目输出：AsyncHBase）。这样也导致其整个代码都是采用异步驱动的模式编写，不光增加了代码的复杂度和降低可阅读性，也加大了支持多种存储引擎的难度。KairosDB严格定义了存储层的API Interface，整体逻辑与存储层耦合度较低，能比较容易的扩展多种存储引擎。当然现在最新版的OpenTSDB也能够额外支持Cassandra和BigTable，但是从整体的架构上，还不能说是一个支持可插拔式存储引擎的架构。

支持多种数据类型及自定义类型的值：OpenTSDB只支持numeric的值，而KairosDB支持numeric、string类型的值，也支持自定义数值类型。在某些场景下，metric value不是一个简单的数值，例如你要统计这个时间点的TopN，对应的metric value可能是一组string值。可扩展的类型，让未来的需求扩展会变得容易。从第一第二点差异可以看出，KairosDB基于OpenTSDB的第一大改造就是将OpenTSDB的功能模型和代码架构变得更加灵活。

支持Auto-rollup：目前大部分TSDB都在朝着支持pre-aggregation和auto-rollup的方向发展，OpenTSDB是少数的不支持该feature的TSDB，在最新发布的OpenTSDB版本中，甚至都不支持多精度数据的存储。不过现在KairosDB支持的auto-rollup功能，采取的还是一个比较原始的实现方式，在下面的章节会详细讲解。

不同的存储模型：存储是TSDB核心中的核心，OpenTSDB在存储模型上使用了UID的压缩优化，来优化查询和存储。KairosDB采取了一个不同的思路，利用了Cassandra宽表的特性，这也是它从HBase转向Cassandra的一个最重要的原因，在下面的章节会详细讲解。

存储模型

  OpenTSDB的存储模型细节，可以参考这篇文章。其主要设计特点是采用了UID编码，大大节省了存储空间，并且利用UID编码的固定字节数的特性，利用HBase的Filter做了很多查询的优化。但是采用UID编码后也带来了很多的缺陷，一是需要维护metric/tagKey/tagValue到UID的映射表，所有data point的写入和读取都需要经过映射表的转换，映射表通常会缓存在TSD或者client，增加了额外的内存消耗；二是由于采用了UID编码，导致metric/tagKey/tagValue的基数是有上限的，取决于UID使用的字节数，并且在UID的分配上会有冲突，会影响写入。



本质上，OpenTSDB存储模型采用的UID编码优化，主要解决的就两个问题：



存储空间优化：UID编码解决重复的row key存储造成的冗余的存储空间问题。

查询优化：利用UID编码后TagKey和TagValue固定字节长度的特性，利用HBase的FuzzyRowFilter做特定场景的查询优化。

  KairosDB在解决这两个问题上，采取了另外一种不同的方式，使其不需要使用UID编码，也不存在使用UID编码后遗留的问题。先看下KairosDB的存储模型是怎样的，它主要由以下三张表构成：



DataPoints: 存储所有原始数据点，每个数据点也是由metric、tags、timestamp和value构成。该表中一行数据的时间跨度是三周，也就是说三周内的所有数据点都存储在同一行，而OpenTSDB内的行的时间跨度只有一个小时。RowKey的组成与OpenTSDB类似，结构为&lt;metric&gt;&lt;timestamp&gt;&lt;tagk1&gt;&lt;tagv1&gt;&lt;tagk2&gt;tagv2&gt;...&lt;tagkn&gt;&lt;tagvn&gt;，不同的是metric, tag key和tag value都存储原始值，而不是UID。

RowKeyIndex: 该表存储所有metric对应DataPoints表内所有row key的映射，也就是说同一个metric上写入的所有的row key，都会存储在同一行内，并且按时间排序。该表主要被用于查询，在根据tag key或者tag value做过滤时，会先从这张表过滤出要查询的时间段内所有符合条件的row key，后在DataPoints表内查询数据。

StringIndex: 该表就三行数据，每一行分别存储所有的metric、tag key和tag value。

  KairosDB采取的存储模型，是利用了Cassandra宽表的特性。HBase的底层文件存储格式中，每一列会对应一个KeyValue，Key为该行的RowKey，所以HBase中一行中的每一列，都会重复的存储相同的RowKey，这也是为何采用了UID编码后能大大节省存储空间的主要原因，也是为何有了UID编码后还能采用compaction策略（将一行中所有列合并为一列）来进一步压缩存储空间的原因。而Cassandra的底层文件存储格式与HBase不同，它一行数据不会为每一列都重复的存储RowKey，所以它不需要使用UID编码。Cassandra内降低存储空间的一个优化方案就是缩减行数，这也是为何它一行存储三周数据而不是一个小时数据的原因。要进一步了解两种设计方案的原因，可以看下HBase文件格式以及Cassandra文件格式。



  利用Cassandra的宽表特性，即使不采用UID编码，存储空间上相比采用UID编码的OpenTSDB，也不会差太多。可以看下官方的解释：



For one we do not use id’s for strings. The string data \(metric names and tags\) are written to row keys and the appropriate indexes. Because Cassandra has much wider rows there are far fewer keys written to the database. The space saved by using id’s is minor and by not using id’s we avoid having to use any kind of locks across the cluster.  

As mentioned the Cassandra has wider rows. The default row size in OpenTSDB HBase is 1 hour. Cassandra is set to 3 weeks.

在查询优化上，采取的也是和OpenTSDB不一样的优化方式。先看下KairosDB内查询的整个流程：



根据查询条件，找出所有DataPoints表里的row key



如果有自定义的plugin，则从plugin中获取要查询的所有row key。（通过Plugin可以扩展使用外部索引系统来对row key进行索引，例如使用ElasticSearch）

如果没有自定义的plugin，则在RowKeyIndex表里根据metric和时间范围，找出所有的row key。（根据列名的范围来缩小查询范围，列名的范围是\(metric+startTime, metric+endTime\)\)

根据row key，从DataPoints表里找出所有的数据

  相比OpenTSDB直接在数据表上进行扫描来过滤row key的方式，KairosDB利用索引表无疑会大大减少扫描的数据量。在metric下tagKey和tagValue组合有限的情况下，会大大的提高查询效率。并且KairosDB还提供了QueryPlugin的方式，能够扩展利用外部组件来对row key进行索引，例如可以利用ElasticSearch，或者其他的索引系统，毕竟通过索引的方式，才是最优的查询方案，这也是Heroic相比KairosDB最大的一个改进的地方。



Auto-rollup

  KairosDB的官方文档中有关于auto-rollup如何配置的章节，但是在讨论组内，其关于auto-rollup的说明如下:



    First off Kairos does not do any aggregation on ingest.  Ingest is direct to the storage on purpose - performance.



    Kairos aggregation is done after the fact at query time.  The rollups are queries that are ran and the results are saved back as a new metric.  Right now the rollups are all configured on a per kairos node basis.  We plan on changing this in the future.



    Right now Kairos does not share any state with other Kairos nodes.  They have very little state on the node \(except for rollups\).



    As for consistency it is up to you on how much you want or how important the data is to you.  

  总结来说，目前KairosDB提供的auto-rollup方案，还是比较简单的实现。就是一个可配置的单机组件，能够定时启动，把已经写入的数据读出后进行aggregation后再次写入，确实非常的原始，可用性和性能都比较低。



  但是有总比没有好，支持auto-rollup一定是所有TSDB的趋势，也是能拉开功能差异和提高核心竞争力的关键功能。



BlueFlood

  上面主要分析了KairosDB，第一个基于Cassandra构建的TSDB，那干脆继续分析下其他基于Cassandra构建的TSDB。



  BlueFlood也是一个基于Cassandra构建的TSDB，从这个PPT介绍上可以看到整体架构上核心组成部分主要有三个：



Ingest module: 处理数据写入。

Rollup module: 做自动的预聚合和降精度。

Query module: 处理数据查询。

  相比KairosDB，其在数据模型上与其他的TSDB有略微差异，主要在：



引入了租户的维度：这是一个创新，如果你是做一个服务化的TSDB，那租户这个维度是必需的。

不支持Tag：这一点上，是比较让我差异的地方。在大多数TSDB都基本上把Tag作为模型的不可缺少部分的情况下，BlueFlood在模型上居然不支持Tag。不过这有可能是其没有想好如何优化Tag维度查询的一种取舍，既然没想好怎么优化，那干脆就先不支持，反正未来再去扩展Tag是可以完全兼容的。BlueFlood当前已经利用ElasticSearch去构建metric的索引，我相信它未来的方案，应该也是基于ElasticSearch去构建Tag的索引，在这个方案完全支持好后，应该才会去引入Tag。

  模型上的不足，BlueFlood不需要去考虑Tag查询如何优化，把精力都投入到了其他功能的优化上，例如auto-rollup。它在auto-rollup的功能支持上，甩了KairosDB和OpenTSDB几条街。来看看它的Auto-rollup功能的特点：



仅支持固定的Interval：5min，20min，60min，4hour，1day。

提供分布式的Rollup Service：rollup任务可以分布式的调度，rollup的数据是通过离线的批量扫描获取。

从它14年的介绍PPT上，还可以看到它在未来规划的几个功能点：



ElasticSearch Indexer and discovery: 目前这个已经实现，但是仅支持metric的索引，未来引入Tag后，可能也会用于Tag的索引。

Cloud files exporter for rollups: 这种方式对离线计算更加优化，rollup的大批量历史数据读取就不会影响在线的业务。

Apache Kafka exporter for rollups: 这种方式相比离线计算更进一步，rollup可以用流计算来做，实时性更加高。

  总结来说，如果你不需要Tag的支持，并且对Rollup有强需求，那BlueFlood相比KairosDB会是一个更好的选择，反之还是选择KairosDB。



Heroic

  第三个要分析的基于Cassandra的TSDB是Heroic，它在DB-Engines上的排名是第19，虽然比BlueFlood和KairosDB都落后，但是我认为它的设计实现却是最好的一个。关于它的起源，可以看下这篇文章或者这个PPT，都是宝贵的经验教训。



  Spotify在决定研发Heroic之前，在OpenTSDB、InfluxDB、KairosDB等TSDB中选用KairosDB来替换他们老的监控系统的底层。但是很快就遇到了KairosDB在查询方面的问题，最主要还是KairosDB对metric和tag没有索引，在metric和tag基数达到一定数量级后，查询会变的很慢。所以Spotify研发Heroic的最大动机就是解决KairosDB的查询问题，采用的解决方案是使用ElasticSearch来作为索引优化查询引擎，而数据的写入和数据表的Schema则完全与KairosDB一致。



简单总结下它的特点：



完整的数据模型，完全遵循metric2.0的规范。

数据存储模型与KairosDB一致，使用ElasticSearch优化查询引擎。（这是除了InfluxDB外，其他TSDB如KairosDB、OpenTSDB、BlueFlood等现存最大的问题，是其核心竞争力之一）

不支持auto-rollup，这是它的缺陷之一。

  如果你需要TSDB支持完整的数据模型，且希望得到高效的索引查询，那Heroic会是你的选择。

