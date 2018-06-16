# section2



KDB+







-	-	备注

主页	http://kx.com/	

License	商业	

所有TSDB中，估计就数这个最酷了，我说的是域名，只有两个字母，猥琐地想一下，域名就值很多钱 ：-）。







kdb+是一个面向列的时序列数据看，以及专门为其设计的查询语言q（和他们的域名一样简短）。Kdb+混合使用了流、内存和实时分析，速度很快，支持分析10亿级别的记录以及快速访问TB级别的历史数据。







不过这是一个商业产品，但是也提供了免费版本（貌似还限制在32位）。







KairosDB





-	-	备注

主页	http://kairosdb.github.io/	

编写语言	Java	

License	Apache License 2.0	

项目创建时间	2013	

最新版	1.1.1	2015/12/08

活跃度	活跃	

文档	详细	

KairosDB是一个OpenTSDB的fork，不过是基于Cassandra存储的。由于Cassandra的行比HBase宽，所以KairosDB的Cassandra的默认行大小为3星期，而OpenTSDB的HBase则为1小时。







KairosDB支持通过Telnet、Rest、Graphite等协议写入数据，你也可以通过编写插件自己实现数据写入。







KairosDB也提供了基于Web API的查询接口，支持数据聚合、持过滤和分组等功能。







同时KairosDB提供了一个供开发用的Web UI，图形绘制引擎使用了 Flot。







和OpenTSDB类似，KairosDB 也提供了插件机制，你可以使用插件完成如下工作：



添加数据点（data point）监听器



添加新的数据存储服务



添加新的协议处理程序



添加自定义系统监视服务







Druid





-	-	备注

主页	http://druid.io/	

编写语言	Java	

License	Apache License 2.0	

项目创建时间	2011	

最新版	Druid 0.9.0-RC2	2016/02/23

活跃度	活跃	

文档	详细	

Druid是一个快速、近实时的海量数据OLAP系统，并且是开源的。Druid诞生于Metamarkets，后来一些核心人员创立了IMPLY公司，进行Druid相关的产品开发。







Druid会按时间来进行分区（segment），并且是面向列存储的。它的主要特性如下：



支持嵌套数据的列式存储



层级查询



二级索引



实时数据摄取



分布式容错架构







根据去年底druid.io发布的技术白皮书，现在生产环境下最大的集群规模如下：



&gt;3M EVENTS / SECOND SUSTAINED \(200B+ EVENTS/DAY\)



10 – 100K EVENTS / SECOND / CORE



&gt;500TB OF SEGMENTS \(&gt;50 TRILLION RAW EVENTS\)



&gt;5000 CORES \(&gt;400 NODES, &gt;100TB RAM\)



QUERY LATENCY \(500MS AVERAGE\)



90% &lt; 1S 95% &lt; 2S 99% &lt; 10S



3+ trillion events/month



3M+ events/sec through Druid’s real-time ingestion



100+ PB of raw data



50+ trillion events







Druid企业用户比较多，比如OneAPM、Netflix和Paypal等。具体可以参考 http://druid.io/druid-powered.html 。







Druid架构比较复杂，因此对部署和运维也有一定的负担，比如需要的机器多、机器配置要高（尤其是内存）。





一句话总结：好用，我们在用。







Prometheus





-	-	备注

主页	http://prometheus.io/	

编写语言	Golang	

License	Apache License 2.0	

项目创建时间	2012	

最新版	0.17.0rc2	2016-02-05

活跃度	活跃	

文档	详细	

Prometheus是一个开源的服务监控系统和时序列数据库，由社交音乐平台SoundCloud在2012年开发，最近也变得很流行，最新版本为0.17.0rc2。







Prometheus从各种输入源采集metric，进行计算后显示结果，或者根据指定条件出发报警。







和其他监控系统相比，Prometheus的特点包括：



多维数据模型（时序列数据由metric名和一组key/value组成）



灵活的查询语言



不依赖分布式存储，单台服务器即可工作



通过基于HTTP的pull方式采集是序列数据



可以通过中间网关进行时序列数据推送



多种可视化和仪表盘支持







由于Prometheus采用了类似OpenTSDB 和 InfluxDB的key/value维度机制，所以如果你对任一种TSDB有了解的话，学习起来会简单些。







一句话总结：貌似比较火，何不试一试？







Pinot





-	-	备注

主页	https://github.com/linkedin/pinot/wiki	

编写语言	Java	

License	Apache License 2.0	

项目创建时间	2014/08	

最新版	0.016	

活跃度	活跃	

文档	详细	

Pinot是一个开源的实时、分布式OLAP数据存储方案。它来自Linkedin，虽然Linkedin最近估价表现很差，但是他们创建的各种软件、中间件实在太多了。这一点我们做软件的都应该向Linkedin表示感谢。







Pinot就像是一个Druid的copy，不过两者的灵感都来源于SenseiDB（Sensei在日语里为老师的意思，写成汉字为“先生”）。







Pinot也像Druid一样，能加载offline数据（Hadoop文件）和实时数据（Kafka）。Pinot从设计上就面向水平扩展。







Pinot主要特点：



面向列



插拔式索引引擎：排序索引、位图索引和反向索引



根据查询语句和segment信息对查询/执行计划进行优化



从Kafka实时数据摄取（ingestion）



从Hadoop进行批量摄取



类似SQL的查询语言，支持聚合、过滤、分组、排序和唯一处理。



支持多值字段



水平扩展和容错







Pinot的特点和Druid很像，两者可互为参考。





一句话总结：背靠大树好乘凉。



