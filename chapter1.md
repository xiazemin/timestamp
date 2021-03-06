# 时序数据库基本要求

综合以上对于时序数据写入、查询和存储的特点的分析，我们可以归纳总结下对于时序数据库的基本要求：



能够支撑高并发、高吞吐的写入：如上所说，时序数据具有典型的写多读少特征，其中95%-99%的操作都是写。在读和写上，首要权衡的是写的能力。由于其场景的特点，对于数据库的高并发、高吞吐写入能力有很高的要求。

交互级的聚合查询：交互级的查询延迟，并且是在数据基数（TB级）较大的情况下，也能够达到很低的查询延迟。

能够支撑海量数据存储：场景的特点决定了数据的量级，至少是TB的量级，甚至是PB级数据。

高可用：在线服务的场景下，对可用性要求也会很高。

分布式架构：写入和存储量的要求，底层若不是分布式架构基本达不到目标。

  结合时序数据的特点和时序数据库的基本要求的分析，使用基于LSM树存储引擎的NoSQL数据库（例如HBase、Cassandra或阿里云表格存储等）相比使用B+树的RDBMS，具有显著的优势。LSM树的基本原理不在这里赘述，它是为优化写性能而设计的，写性能相比B+树能提高一个数量级。但是读性能会比B+树差很多，所以极其适合写多读少的场景。目前开源的几个比较著名的时序数据库中，OpenTSDB底层使用HBase、BlueFlood和KairosDB底层使用Cassandra，InfluxDB底层是自研的与LSM类似的TSM存储引擎，Prometheus是直接基于LevelDB存储引擎。所以可以看到，主流的时序数据库的实现，底层存储基本都会采用LSM树加上分布式架构，只不过有的是直接使用已有的成熟数据库，有的是自研或者基于LevelDB自己实现。

  LSM树加分布式架构能够很好的满足时序数据写入能力的要求，但是在查询上有很大的弱势。如果是少量数据的聚合和多维度查询，勉强能够应付，但是若需要在海量数据上进行多维和聚合查询，在缺乏索引的情况下会显得比较无力。所以在开源界，也有其他的一些产品，会侧重于解决查询和分析的问题，例如Druid主要侧重解决时序数据的OLAP需求，不需要预聚合也能够提供在海量数据中的快速的查询分析，以及支持任意维度的drill down。同样的侧重分析的场景下，社区也有基于Elastic Search的解决方案。



  总之，百花齐放的时序数据库产品，各有优劣，没有最好的，只有最合适的，全凭你自己对业务需求的判断来做出选择。



时间序列数据的模型

  时序数据的数据模型主要有这么几个主要的部分组成：



主体: 被测量的主体，一个主体会拥有多个维度的属性。以服务器状态监控场景举例，测量的主体是服务器，其拥有的属性可能包括集群名、Hostname等。

测量值: 一个主体可能有一个或多个测量值，每个测量值对应一个具体的指标。还是拿服务器状态监控场景举例，测量的指标可能会有CPU使用率，IOPS等，CPU使用率对应的值可能是一个百分比，而IOPS对应的值是测量周期内发生的IO次数。

时间戳: 每次测量值的汇报，都会有一个时间戳属性来表示其时间。

目前主流时序数据库建模的方式会分为两种，按数据源建模和按指标建模，以两个例子来说明这两种方式的不同。



按数据源建模



data\_model\_by\_source



  如图所示为按数据源建模的例子，同一个数据源在某个时间点的所有指标的测量值会存储在同一行。Druid和InfluxDB采用这种模式。



按指标建模



data\_model\_by\_metric

  如图所示为按指标建模的例子，其中每行数据代表某个数据源的某个指标在某个时间点的测量值。OpenTSDB和KairosDB采用这种模式。



  这两种模型的选择没有明确的优劣，如果底层是采用列式存储且每个列上都有索引，则按数据源建模可能是一个比较干净的方式。如果底层是类似HBase或Cassandra这种的，将多个指标值存储在同一行会影响对其中某个指标的查询或过滤的效率，所以通常会选择按指标建模。



时间序列数据的处理

  这一小节会主要讲解下对时序数据的处理操作，时序数据库除了满足基本的写和存储的需求外，最重要的就是查询和分析的功能。对时序数据的处理可以简单归纳为Filter\(过滤\), Aggregation\(聚合\)、GroupBy和Downsampling\(降精度\)。为了更好的支持GroupBy查询，某些时序数据库会对数据做pre-aggregation\(预聚合\)。Downsampling对应的操作是Rollup\(汇总\)，而为了支持更快更实时的Rollup，通常时序数据库都会提供auto-rollup\(自动汇总\)。



Filter\(过滤\)

filter



  如图所示就是一个简单的Filter处理，简单的说，就是根据给定的不同维度的条件，查找符合条件的所有数据。在时序数据分析的场景，通常先是从一个高维度入手，后通过提供更精细的维度条件，对数据做更细致的查询和处理。



Aggregation\(聚合\)

  聚合是时序数据查询和分析的最基本的功能，时序数据记录的是最原始的状态变化信息，而查询和分析通常需要的不是原始值，而是基于原始值的一些统计值。Aggregation就是提供了对数据做统计的一些基本的计算操作，比较常见的有SUM（总和）、AVG（平均值）、Max（最大值）、TopN等等。例如对服务器网络流量的分析，你通常会关心流量的平均值、流量的总和或者流量的峰值。



GroupBy和Pre-aggregation\(预聚合\)

groupby



  GroupBy就是将一个低维度的时序数据转换为一个高维度的统计值的过程，如图就是一个简单的GroupBy的例子。GroupBy一般发生在查询时，查询到原始数据后做实时的计算来得到结果，这个过程有可能是很慢的，取决于其查询的原始数据的基数。主流的时序数据库提供pre-aggregation\(预聚合\)的功能来优化这一过程，数据实时写入后就会经过预聚合的运算，生成按指定规则GroupBy之后的结果，在查询时就可以直接查询结果而不需要再次计算。



Downsampling\(降精度\)，Rollup\(汇总\)和Auto-rollup\(自动汇总\)

  Downsampling就是将一个高精度的时序数据转换为一个低精度的时序数据的过程，这个过程被称作Rollup。它与GroupBy的过程比较类似，核心区别是GroupBy是基于相同的时间粒度，把同一时间层面上的不同维度的数据做聚合，转换后的结果还是相同时间粒度的数据，只不过是更高的一个维度。而Downsampling是不同时间层面上把相同维度的数据做聚合，抓换为更粗时间粒度的数据，但是还是拥有相同的维度。

downsampling

  如图就是一个简单的Downsampling的例子，将一个10秒精度的数据聚合成30秒精度的数据，统计平均值。



  Downsampling分为存储降精度和查询降精度，存储降精度的意义在于降低存储成本，特别是针对历史数据。查询降精度主要是针对较大时间范围的查询，来减少返回的数据点。不管是存储降精度还是查询降精度，都需要auto-rollup。auto-rollup就是自动的对数据做rollup，而不是等待查询时做rollup，过程与pre-aggregation类似，能有效的提高查询效率，这也是目前主流时序数据库已经提供或者正在规划的功能。目前Druid、InfluxDB和KairosDB都提供auto-rollup，OpenTSDB不提供auto-rollup但是暴露了接口支持在外部做auto-rollup后的结果导入。



