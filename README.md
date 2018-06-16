# 时间序列数据的存储和计算

什么是时间序列（Time Series，以下简称时序）数据？从定义上来说，就是一串按时间维度索引的数据。用描述性的语言来解释什么是时序数据，简单的说，就是这类数据描述了某个被测量的主体在一个时间范围内的每个时间点上的测量值。

  对时序数据进行建模的话，会包含三个重要部分，分别是：主体，时间点和测量值。套用这套模型，你会发现你在日常工作生活中，无时无刻不在接触着这类数据。



如果你是一个股民，某只股票的股价就是一类时序数据，其记录着每个时间点该股票的股价。

如果你是一个运维人员，监控数据是一类时序数据，例如对于机器的CPU的监控数据，就是记录着每个时间点机器上CPU的实际消耗值。

 从DB-Engines的数据库类别流行度趋势榜上可以看到，时序数据库（Time Series DB）的流行度在最近的两年内，一直都是保持一个很高的增长趋势。

时间序列数据特征

有时间特征,一般是时间戳，一般精确到秒级、毫秒级、也有坚决到nano的

典型应用场景：运维、监控系统 

时间序列数据的特性

  对于时序数据的特点的分析，会从数据的写入、查询和存储这三个维度来阐述，通过对其特点的分析，来推演对时序数据库的基本要求。



数据写入的特点

写入平稳、持续、高并发高吞吐：时序数据的写入是比较平稳的，这点与应用数据不同，应用数据通常与应用的访问量成正比，而应用的访问量通常存在波峰波谷。时序数据的产生通常是以一个固定的时间频率产生，不会受其他因素的制约，其数据生成的速度是相对比较平稳的。时序数据是由每个个体独立生成，所以当个体数量众多时，通常写入的并发和吞吐量都是比较高的，特别是在物联网场景下。写入并发和吞吐量，可以简单的通过个体数量和数据生成频率来计算，例如若你有1000个个体以10秒的频率产生数据，则你平均每秒产生的并发和写入量就是100。

写多读少：时序数据上95%-99%的操作都是写操作，是典型的写多读少的数据。这与其数据特性相关，例如监控数据，你的监控项可能很多，但是你真正去读的可能比较少，通常只会关心几个特定的关键指标或者在特定的场景下才会去读数据。

实时写入最近生成的数据，无更新：时序数据的写入是实时的，且每次写入都是最近生成的数据，这与其数据生成的特点相关，因为其数据生成是随着时间推进的，而新生成的数据会实时的进行写入。数据写入无更新，在时间这个维度上，随着时间的推进，每次数据都是新数据，不会存在旧数据的更新，不过不排除人为的对数据做订正。

数据查询和分析的特点

按时间范围读取：通常来说，你不会去关心某个特定点的数据，而是一段时间的数据。所以时序数据的读取，基本都是按时间范围的读取。

最近的数据被读取的概率高：最近的数据越有可能被读取，以监控数据为例，你通常只会关心最近几个小时或最近几天的监控数据，而极少关心一个月或一年前的数据。

多精度查询：按数据点的不同密集度来区分不同的精度，例如若相邻数据点的间隔周期是10秒，则该时序数据的精度就是10秒，若相邻数据点的时间间隔周期是30秒，则该时序数据的精度就是30秒。时间间隔越短，精度越高。精度越高的数据，能够还原的历史状态更细致更准确，但其保存的数据点会越多。这个就好比相机的像素，像素越高，照片越清晰但是相片大小越大。时序数据的查询，不需要都是高精度的，这是实际的需求，也是一种取舍，同样也是成本的考虑。还是拿监控数据举例，通常监控数据会以曲线图的方式展现，由人的肉眼去识别。在这种情况下，单位长度下若展示的数据点过于密集，反而不利于观察，这是实际的需求。而另外一种取舍是，若你查询一个比较长的时间范围，比如是一个月，若查询10秒精度的数据需要返回259200个点，而若查询60秒精度的数据则只要返回43200个点，这是查询效率上的一种取舍。而成本方面的考虑，主要在存储的数据量上，存储高精度的数据需要的成本越高，通常对于历史数据，可以不需要存储很高精度的数据。总的来说，在查询和处理方面，会根据不同长度的时间范围，来获取不同精度的数据，而在存储方面，对于历史的数据，会选择降精度的数据存储。

多维分析：时序数据产生自不同的个体，这些个体拥有不同的属性，可能是同一维度的，也可能是不同维度的。还是举个监控的例子，我有个对某个集群上每台机器的网络流量的监控，此时可以查询这个集群下某台机器的网络流量，这是一个维度的查询，而同时还需要查询这个集群总的网络流量，这是另外一个维度的查询。

数据挖掘：随着大数据和人工智能技术的发展，在存储、计算能力以及云计算发展的今天，数据的高附加值的挖掘已经不再有一个很高门槛。而时序数据蕴含着很高的价值，非常值得挖掘。

数据存储的特点

数据量大：拿监控数据来举例，如果我们采集的监控数据的时间间隔是1s，那一个监控项每天会产生86400个数据点，若有10000个监控项，则一天就会产生864000000个数据点。在物联网场景下，这个数字会更大。整个数据的规模，是TB甚至是PB级的。

冷热分明：时序数据有非常典型的冷热特征，越是历史的数据，被查询和分析的概率越低。

具有时效性：时序数据具有时效性，数据通常会有一个保存周期，超过这个保存周期的数据可以认为是失效的，可以被回收。一方面是因为越是历史的数据，可利用的价值越低；另一方面是为了节省存储成本，低价值的数据可以被清理。

多精度数据存储：在查询的特点里提到时序数据出于存储成本和查询效率的考虑，会需要一个多精度的查询，同样也需要一个多精度数据的存储。

