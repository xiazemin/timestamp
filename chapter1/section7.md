#  LSM tree 替换 B tree

对于 90% 以上场景都是写入的时序数据库，B tree 很明显是不合适的。



业界主流都是采用 LSM tree 替换 B tree，比如 Hbase, Cassandra 等 nosql 中。这里我们详细介绍一下。



LSM tree 包括内存里的数据结构和磁盘上的文件两部分。分别对应 Hbase 里的 MemStore 和 HLog；对应 Cassandra 里的 MemTable 和 sstable。



LSM tree 操作流程如下：



数据写入和更新时首先写入位于内存里的数据结构。为了避免数据丢失也会先写到 WAL 文件中。



内存里的数据结构会定时或者达到固定大小会刷到磁盘。这些磁盘上的文件不会被修改。



随着磁盘上积累的文件越来越多，会定时的进行合并操作，消除冗余数据，减少文件数量。





p4-Hbase LSM tree 结构介绍（注 1）



可以看到 LSM tree 核心思想就是通过内存写和后续磁盘的顺序写入获得更高的写入性能，避免了随机写入。但同时也牺牲了读取性能，因为同一个 key 的值可能存在于多个 HFile 中。为了获取更好的读取性能，可以通过 bloom filter 和 compaction 得到，这里限于篇幅就不详细展开。



分布式存储

时序数据库面向的是海量数据的写入存储读取，单机是无法解决问题的。所以需要采用多机存储，也就是分布式存储。



分布式存储首先要考虑的是如何将数据分布到多台机器上面，也就是 分片（sharding）问题。下面我们就时序数据库分片问题展开介绍。分片问题由分片方法的选择和分片的设计组成。



分片方法

时序数据库的分片方法和其他分布式系统是相通的。



哈希分片：这种方法实现简单，均衡性较好，但是集群不易扩展。



一致性哈希：这种方案均衡性好，集群扩展容易，只是实现复杂。代表有 Amazon 的 DynamoDB 和开源的 Cassandra。



范围划分：通常配合全局有序，复杂度在于合并和分裂。代表有 Hbase。



分片设计

分片设计简单来说就是以什么做分片，这是非常有技巧的，会直接影响写入读取的性能。



结合时序数据库的特点，根据 metric+tags 分片是比较好的一种方式，因为往往会按照一个时间范围查询，这样相同 metric 和 tags 的数据会分配到一台机器上连续存放，顺序的磁盘读取是很快的。再结合上面讲到的单机存储内容，可以做到快速查询。



进一步我们考虑时序数据时间范围很长的情况，需要根据时间范围再将分成几段，分别存储到不同的机器上，这样对于大范围时序数据就可以支持并发查询，优化查询速度。



如下图，第一行和第三行都是同样的 tag（sensor=95D8-7913;city= 上海），所以分配到同样的分片，而第五行虽然也是同样的 tag，但是根据时间范围再分段，被分到了不同的分片。第二、四、六行属于同样的 tag（sensor=F3CC-20F3;city= 北京）也是一样的道理。







p5- 时序数据分片说明



6. 真实案例

下面我以一批开源时序数据库作为说明。



InfluxDB:

非常优秀的时序数据库，但只有单机版是免费开源的，集群版本是要收费的。从单机版本中可以一窥其存储方案：在单机上 InfluxDB 采取类似于 LSM tree 的存储结构 TSM；而分片的方案 InfluxDB 先通过+（事实上还要加上 retentionPolicy）确定 ShardGroup，再通过+的 hash code 确定到具体的 Shard。



这里 timestamp 默认情况下是 7 天对齐，也就是说 7 天的时序数据会在一个 Shard 中。







p6-Influxdb TSM 结构图（注 2）



Kairosdb:

底层使用 Cassandra 作为分布式存储引擎，如上文提到单机上采用的是 LSM tree。



Cassandra 有两级索引：partition key 和 clustering key。其中 partition key 是其分片 ID，使用的是一致性哈希；而 clustering key 在一个 partition key 中保证有序。



Kairosdb 利用 Cassandra 的特性，将++&lt;数据类型&gt;+作为 partition key，数据点时间在 timestamp 上的偏移作为 clustering key，其有序性方便做基于时间范围的查询。



partition key 中的 timestamp 是 3 周对齐的，也就是说 21 天的时序数据会在一个 clustering key 下。3 周的毫秒数是 18 亿正好小于 Cassandra 每行列数 20 亿的限制。



OpenTsdb:

底层使用 Hbase 作为其分布式存储引擎，采用的也是 LSM tree。



Hbase 采用范围划分的分片方式。使用 row key 做分片，保证其全局有序。每个 row key 下可以有多个 column family。每个 column family 下可以有多个 column。







上图是 OpenTsdb 的 row key 组织方式。不同于别的时序数据库，由于 Hbase 的 row key 全局有序，所以增加了可选的 salt 以达到更好的数据分布，避免热点产生。再由与 timestamp 间的偏移和数据类型组成 column qualifier。



他的 timestamp 是小时对齐的，也就是说一个 row key 下最多存储一个小时的数据。并且需要将构成 row key 的 metric 和 tags 都转成对应的 uid 来减少存储空间，避免 Hfile 索引太大。

