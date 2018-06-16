# OpenTSDB

OpenTSDB是一个分布式、可伸缩的时序数据库，支持高达每秒百万级的写入能力，支持毫秒级精度的数据存储，不需要降精度也可以永久保存数据。其优越的写性能和存储能力，得益于其底层依赖的HBase，HBase采用LSM树结构存储引擎加上分布式的架构，提供了优越的写入能力，底层依赖的完全水平扩展的HDFS提供了优越的存储能力。OpenTSDB对HBase深度依赖，并且根据HBase底层存储结构的特性，做了很多巧妙的优化。关于存储的优化，我在这篇文章中有详细的解析。在最新的版本中，还扩展了对BigTable和Cassandra的支持。



架构

image

  如图是OpenTSDB的架构，核心组成部分就是TSD和HBase。TSD是一组无状态的节点，可以任意的扩展，除了依赖HBase外没有其他的依赖。TSD对外暴露HTTP和Telnet的接口，支持数据的写入和查询。TSD本身的部署和运维是很简单的，得益于它无状态的设计，不过HBase的运维就没那么简单了，这也是扩展支持BigTable和Cassandra的原因之一吧。



数据模型

  OpenTSDB采用按指标建模的方式，一个数据点会包含以下组成部分：



metric：时序数据指标的名称，例如sys.cpu.user，stock.quote等。

timestamp：秒级或毫秒级的Unix时间戳，代表该时间点的具体时间。

tags：一个或多个标签，也就是描述主体的不同的维度。Tag由TagKey和TagValue组成，TagKey就是维度，TagValue就是该维度的值。

value：该指标的值，目前只支持数值类型的值。

存储模型

  OpenTSDB底层存储的优化思想，可以参考这篇文章，简单总结就是以下这几个关键的优化思路：



对数据的优化：为Metric、TagKey和TagValue分配UniqueID，建立原始值与UniqueID的索引，数据表存储Metric、TagKey和TagValue对应的UniqueID而不是原始值。

对KeyValue数的优化：如果对HBase底层存储模型十分了解的话，就知道行中的每一列在存储时对应一个KeyValue，减少行数和列数，能极大的节省存储空间以及提升查询效率。

对查询的优化：利用HBase的Server Side Filter来优化多维查询，利用Pre-aggregation和Rollup来优化GroupBy和降精度查询。

UIDTable

  接下来看一下OpenTSDB在HBase上的几个关键的表结构的设计，首先是tsdb-uid表，结构如下：



image



Metric、TagKey和TagValue都会被分配一个相同的固定长度的UniqueID，默认是三个字节。tsdb-uid表使用两个ColumnFamily，存储了Metric、TagKey和TagValue与UniqueID的映射和反向映射，总共是6个Map的数据。



从图中的例子可以解读出：



TagKey为'host'，对应的UniqueID为'001'

TagValue为'static'，对应的UniqueId为'001'

Metric为'proc.loadavg.1m'，对应的UniqueID为'052'

  为每一个Metric、TagKey和TagValue都分配UniqueID的好处，一是大大降低了存储空间和传输数据量，每个值都只需要3个字节就可以表示，这个压缩率是很客观的；二是采用固定长度的字节，可以很方便的从row key中解析出所需要的值，并且能够大大减少Java堆内的内存占用（bytes相比String能节省很多的内存占用），降低GC的压力。

  不过采用固定字节的UID编码后，对于UID的个数是有上限要求的，3个字节最多只允许有16777216个不同的值，不过在大部分场景下都是够用的。当然这个长度是可以调整的，不过不支持动态更改。



DataTable

第二张关键的表是数据表，结构如下：



image



  该表中，同一个小时内的数据会存储在同一行，行中的每一列代表一个数据点。如果是秒级精度，那一行最多会有3600个点，如果是毫秒级精度，那一行最多会有3600000个点。

  这张表设计的精妙之处在于row key和qualifier（列名）的设计，以及对整行数据的compaction策略。row key格式为：



&lt;metric&gt;&lt;timestamp&gt;&lt;tagk1&gt;&lt;tagv1&gt;&lt;tagk2&gt;tagv2&gt;...&lt;tagkn&gt;&lt;tagvn&gt;



  其中metric、tagk和tagv都是用uid来表示，由于uid固定字节长度的特性，所以在解析row key的时候，可以很方便的通过字节偏移来提取对应的值。Qualifier的取值为数据点的时间戳在这个小时的时间偏差，例如如果你是秒级精度数据，第30秒的数据对应的时间偏差就是30，所以列名取值就是30。列名采用时间偏差值的好处，主要在于能大大节省存储空间，秒级精度的数据只要占用2个字节，毫秒精度的数据只要占用4个字节，而若存储完整时间戳则要6个字节。整行数据写入后，OpenTSDB还会采取compaction的策略，将一行内的所有列合并成一列，这样做的主要目的是减少KeyValue数目。



查询优化

  HBase仅提供简单的查询操作，包括单行查询和范围查询。单行查询必须提供完整的RowKey，范围查询必须提供RowKey的范围，扫描获得该范围下的所有数据。通常来说，单行查询的速度是很快的，而范围查询则是取决于扫描范围的大小，扫描个几千几万行问题不大，但是若扫描个十万上百万行，那读取的延迟就会高很多。

  OpenTSDB提供丰富的查询功能，支持任意TagKey上的过滤，支持GroupBy以及降精度。TagKey的过滤属于查询的一部分，GroupBy和降精度属于对查询后的结果的计算部分。在查询条件中，主要的参数会包括：metric名称、tag key过滤条件以及时间范围。上面一章中指出，数据表的rowkey的格式为：&lt;metric&gt;&lt;timestamp&gt;&lt;tagk1&gt;&lt;tagv1&gt;&lt;tagk2&gt;tagv2&gt;...&lt;tagkn&gt;&lt;tagvn&gt;，从查询的参数上可以看到，metric名称和时间范围确定的话，我们至少能确定row key的一个扫描范围。但是这个扫描范围，会把包含相同metric名称和时间范围内的所有的tag key的组合全部查询出来，如果你的tag key的组合有很多，那你的扫描范围是不可控的，可能会很大，这样查询的效率基本是不能接受的。



我们具体看一下OpenTSDB对查询的优化措施：



Server side filter

HBase提供了丰富和可扩展的filter，filter的工作原理是在server端扫描得到数据后，先经过filter的过滤后再将结果返回给客户端。Server side filter的优化策略无法减少扫描的数据量，但是可以大大减少传输的数据量。OpenTSDB会将某些条件的tag key filter转换为底层HBase的server side filter，不过该优化带来的效果有限，因为影响查询最关键的因素还是底层范围扫描的效率而不是传输的效率。

减少范围查询内扫描的数据量

要想真正提高查询效率，还是得从根本上减少范围扫描的数据量。注意这里不是减小查询的范围，而是减少该范围内扫描的数据量。这里用到了HBase一个很关键的filter，即FuzzyRowFilter，FuzzyRowFilter能够根据指定的条件，在执行范围扫描时，动态的跳过一定数据量。但不是所有OpenTSDB提供的查询条件都能够应用该优化，需要符合一定的条件，具体要符合哪些条件就不在这里说明了，有兴趣的可以去了解下FuzzyRowFilter的原理。

范围查询优化成单行查询

这个优化相比上一条，更加的极端。优化思路非常好理解，如果我能够知道要查询的所有数据对应的row key，那就不需要范围扫描了，而是单行查询就行了。这里也不是所有OpenTSDB提供的查询条件都能够应用该优化，同样需要符合一定的条件。单行查询要求给定确定的row key，而数据表中row key的组成部分包括metric名称、timestamp以及tags，metric名称和timestamp是能够确定的，如果tags也能够确定，那我们就能拼出完整的row key。所以很简单，如果要能够应用此优化，你必须提供所有tag key对应的tag value才行。

  以上就是OpenTSDB对HBase查询的一些优化措施，但是除了查询，对查询后的数据还需要进行GroupBy和降精度。GroupBy和降精度的计算开销也是非常可观的，取决于查询后的结果的数量级。对GroupBy和降精度的计算的优化，几乎所有的时序数据库都采用了同样的优化措施，那就是pre-aggregation和auto-rollup。思路就是预先进行计算，而不是查询后计算。不过OpenTSDB在已发布的最新版本中，还未支持pre-aggregation和rollup。而在开发中的2.4版本中，也只提供了半吊子的方案，它只提供了一个新的接口支持将pre-aggregation和rollup的结果进行写入，但是对数据的pre-aggregation和rollup的计算还需要用户自己在外层实现。

