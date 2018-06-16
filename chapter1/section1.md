# 时序列数据库

![](/assets/rank.png)



InfluxDB





-	-	备注

主页	https://influxdata.com/	

编写语言	Golang	

License	MIT	

项目创建时间	2013	

最新版	v0.10.1	2016/2/18

活跃度	活跃	

文档	详细	





InfluxDB由Golang语言编写，也是由Golang编写的软件中比较著名的一个，在很多Golang的沙龙或者文章中可能都会把InfluxDB当标杆来介绍，这也间接帮助InfluxDB提高了知名度。







InfluxDB的主要特点包括下面这些：



schemaless\(无结构\)，可以是任意数量的列



可扩展（集群）



方便、强大的查询语言



Native HTTP API



集成了数据采集、存储、可视化功能



实时数据Downsampling



高效存储，使用高压缩比算法，支持retention polices







InfluxDB是TSDB中为数不多的进行了用户和角色方面实现的，提供了Cluster Admin、Database Admin和Database User三种角色。







InfluxDB的数据采集系统也支持多种协议和插件： - 行文本 - UDP - Graphite - CollectD - OpenTSDB







不过InfluxDB每次变动都较大，尤其是在存储和集群方面，追求平平安过日子，不想瞎折腾的可以考虑下。



注意：



由于InfluxDB开发太活跃了，很可能你在网上搜到的资料都是老的，会害到你，所以你需要以官方文档为主。







一句话总结：欣欣向荣、值得一试。







RRDtool





-	-	备注

主页	http://oss.oetiker.ch/rrdtool/index.en.html	

编写语言	C语言	

License	GNU GPL V2	or later

项目创建时间	16-Jul-1999	rrdtool-1.0.0

最新版	rrdtool-1.5.5	10-Nov-2015

活跃度	活跃	

文档	详细	

RRDtool全称为Round Robin Database Tool，也就是用于操作RRD的工具，简单明了的软件名。





什么是RRD呢？简单来说它就是一个循环使用的固定大小的数据库文件（其实也不太像典型的数据库）。



大体来说，RRDtool提供的主要工具如下：



创建RRD（rrdtool create）



更新RRD（rrdtool update）



画图（rrdtool graph）







这其中，画图功能是最复杂也是最强大的，甚至支持下面这些图形，这是其他TSDB中少见的：



指标比较，对两个指标值进行计算，描画出满足条件的区域



移动平均线



和历史数据进行对比



基于最小二乘法的线性预测



曲线预测







总之，它的画图功能太丰富了。





Graphite





-	-	备注

主页	http://graphite.readthedocs.org/en/latest/	

编写语言	Python	

License	Apache 2.0	

项目创建时间	2006年	

最新版	0.9.10	2012/5/31

活跃度	活跃	

文档	详细	

Graphite由Orbitz, LLC 的 Chris Davis创立于2006年，它主要有两个功能： \* 存储数值型时序列数据 \* 根据请求对数据进行可视化（画图）



相应的，它的特点为：



分布式时序列数据存储，容易扩展



功能强大的画图Web API，提供了大量的函数和输出方式



Graphite本身不带数据采集功能，但是你可以选择很多第三方插件，比如适用于collectd、Ganglia或Sensu的插件等。同时，Graphite也支持Plaintext、Pickle和AMQP这些数据输入方式。



Graphite主要由三个模块组成：



whisper：创建、更新RRD文件



carbon：以守护进程的形式运行，接收数据写入请求



carbon-cache：数据存储



carbon-relay：分区和复制，位于carbon-cache之前，类似carbon-cache的负载均衡



carbon-aggregator：数据集计，用于减轻carbon-cache的负载



graphite-web：用于读取、展示数据的Web应用



whisper使用了类似RRDtool的RRD文件格式，它也不像C/S结构的软件一样，没有服务进程，只是作为Python library使用，提供对数据的create/update/fetch操作。



如果你对它的性能比较在意，这里有一份老的数据可供参考。



Google、Etsy、GitHub、豆瓣、Instagram、Evernote和Uber等很多知名公司都是Graphite的用户。有此背景，其可信度又加一层，而且网上的资料也相当的多，值得评估一下。







一句话总结：群众基础好、可以参考。







OpenTSDB





-	-	备注

主页	http://opentsdb.net/	

编写语言	Java	

License	LGPLv2.1+ GPLv3+	

项目创建时间	2010	

最新版	2.2	

活跃度	活跃	

文档	详细	

OpenTSDB是一个分布式、可伸缩的时间序列数据库。它支持豪秒级数据采集所有metrics，支持永久存储（不需要downsampling），和InfluxDB类似，它也是无模式，以tag来实现维度的概念。



比如，这就是它的一个metric例子：









mysql.bytes\_received 1287333217 66666666 schema=foo host=db1





OpenTSDB的节点称为TSD（Time Series Daemon \(TSD\)），它没有主、从之分，消除了单点隐患，非常容易扩展。它主要以HBase作为存储系统，现在也增加了对Cassandra和Bigtable（非云端）。



OpenTSDB以数据存储和查询为主，附带了一个简单地图形界面（依赖Gnuplot），共开发、调试使用。







一句话总结：好用，我们在用。

