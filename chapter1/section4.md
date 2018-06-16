# section4

Blueflood





-	-	-

主页	http://blueflood.io/	

编写语言	Java	

License	Apache License, Version 2.0	

项目创建时间	2013	

最新版	1.0.2123	2016/01/02

活跃度	活跃	来自Rackspace

文档	比较详细	





Blueflood由Rackspace的Cloud Monitoring团队创建，用于管理Cloud Monitoring系统产生的metric数据。如果你还不熟悉Rackspace这个公司的话，可以去网上了解一下，它也是比较大的云计算公司。







除了自己使用，Rackspace还提供了免费的Blueflood-as-a-Service，此外还有一些大公司也在准备使用Blueflood。





Blueflood特点如下：



built on top of Cassandra



基于HTTP API/JSON的数据采集（ingest）和读取



支持数值、布尔和字符串metric data



多租户



水平扩展







Blueflood主要由以下模块构成：



Ingest - 采集/写入数据



Rollup - 聚合计算



Query - 处理用户查询







不过遗憾的是Blueflood现在没有一个类似tag的多维度方案。







一句话点评：背靠大树好乘凉，可以试试。







Gnocchi





-	-	-

主页	http://docs.openstack.org/developer/gnocchi/	

编写语言	Python	

License	Apache License Version 2.0	

项目创建时间	2014	

最新版	2.0.0	

活跃度	活跃	来自OpenStack

文档	一般丰富	





Gnocchi是OpenStack项目的一部分，但它也能独立工作。







Gnocchi是一个多租户的时间序列、metric和资源（resource）数据库。它提供了一组HTTP REST API来创建和管理数据。







Gnocchi特点如下：



HTTP RES接口



水平扩展



Metri聚合



支持批处理



支持归档功能



按metric值查找（这个在TSDB很少见）



多租户支持



对Grafan的支持



支持Statsd协议







Gnocchi后端存储支持一下4种方式：



File



Swift



Ceph \(推荐方式\)



InfluxDB \(实验功能\)







前三种方式基于一个名为Carbonara的库，这个库负责对时间序列数据的管理，因为这三种存储方案本身并不关心数据的特点。InfluxDB前面已经说过，本身就是一个TSDB。





Newts





-	-	-

主页	https://github.com/OpenNMS/newts	

编写语言	Java	

License	Apache License Version 2.0	

项目创建时间	2014	1.0 release

最新版	1.3.3	2016/01

活跃度	一般	

文档	一般	





Newts是基于Cassandra的时间序列数据库。正如其名所示，它是一个“New-fangled Timeseries Data Store”。它的开发者是OpenNMS，这也是一个知名的开源网络监控和管理平台。







Newts基于Cassandra，对写优化、完全分布式，吞吐量非常高；通过将类似的metric分组存储，让写和读更高效。





SiteWhere





-	-	-

主页	http://www.sitewhere.org/	

编写语言	Java	

License	CPAL 1.0	

最新版	1.6.1	

活跃度	活跃	

文档	详细	





SiteWhere是一个开源的IoT开放平台，帮助用户快速将IoT应用推向市场。SiteWhere提供了一套完整的设备管理解决方案，通过MQTT、AMQP、Stomp和其他协议连接设备，支持自注册、REST和批处理方式注册设备。







SiteWhere也提供了可扩展的大数据解决方案，底层采用经优化的MongoDB和HBase，为存储设备事件提供了时序列数据库，且经过Hortonworks和Cloudera的测试。







SiteWhere还内嵌了Siddhi用于Complex Event Processing （CEP），提供了和Azure EventHub、Apache Solr以及Twilio的集成，以及Android和Arduino平台开发用的SDK。







SiteWhere的部署方式也很灵活，支持公有云、私有机房，Ubuntu Juju和Docker的部署方式。







SiteWhere也支持多租户，不同的租户数据分开存储，还能为不同的租户提供不同的处理引擎，租户的启动、停止互不影响。







SiteWhere的service provider interfaces（SPIs）提供了平台的核心对象模型，第三方可以对平台进行扩展。







建议你接着看一下它的 System Overview \(http://documentation.sitewhere.org/overview.html\) 和 System Architecture \(http://documentation.sitewhere.org/architecture.html\) 以对这个系统有更深入的了解。







作为IoT平台，SiteWhere算是这几个中不错的。







一句话点评：开源、功能强大、一体化方案。







TempoIQ





-	-	-

主页	https://www.tempoiq.com/	

License	商业	





TempoIQ也是一个IoT平台服务，它能帮助用户快速、灵活的的创建IoT应用，提供了实时的数据分析、报警、仪表盘、报告功能。2014年7月从TempoDB改名为TempoIQ，它的很多组件都有IQ结尾，代表智商很高？







TempoIQ主要由以下几个部分组成：







CloudIQ



TempoIQ采用第四代CoDA（Context Delivery Architecture）架构，用户可以不必关心复杂的基础设施就可以部署IoT应用。







ConnectIQ



基于数据事件的API，用于连接任何设备，支持灵活的事件数据模型：REST API、HTTPs和MQTT。它不关心设备来源，可以及时进行数据流处理，不需要提前安装和配置。







DataIQ



对所有IoT数据和分析报告进行安全、持久的存储，并提供报告下载功能。







AnalyzeIQ



用户可以创建分析流（custom analytics streams）来洞悉IoT数据实时状态，自动将数据存储到DataIQ。并可以创建报警来持续监控分析流，当有超预期的变动或者达到致命条件时进行实时报警。







ViewIQ



用户使用ViewIQ可以创建实时的IoT仪表盘、应用和可视化组件，而且不需要任何编码工作。







Riak TS





-	-	-

主页	http://basho.com/products/riak-ts/	

编写语言	Erlang	

License	商业	

最新版	1.1.0	2016/01/14

活跃度	

商业支持









Riak作为NoSQL和K/V存储可能更有名，而Riak TS是一个为时序列和为存储IoT数据进行了优化的NoSQL数据库软件。







在官方主页上写道：“Riak TS is engineered to be faster than Cassandra”。







由于其非开源性，网上（包括官网）详细资料都不是特别多。







Cyanite





-	-	-

主页	http://cyanite.io/	

编写语言	Clojure	

License	MIT	

项目创建时间	2013年	

最新版	

还没有正式release（GitHub上）

活跃度	活跃	

文档	详细	





Cyanite是一个用于接收和存储时序列数据的守护进程，它的设计目标是兼容Graphite生态系统。







Cyanite默认使用Apache Cassandra来存储时序列数据，它的特点如下：







可扩展性



基于Cassandra，Cyanite可以实现高可用、具有弹性，以及低延迟的存储。







兼容性



由于Graphite已经成为事实上管理时序列数据的标准，不管是使用graphite-web还是Grafana。Cyanite将会尽可能的保持与这些生态系统的兼容以提供简单地交互模式。







Cyanite是一个典型的流式处理模型，它接收数据，对数据进行标准化，然后进行输出。其交互图如下所示：







Cyanite的工作流程如下：



每条输入都会产生规范化的metrics，并添加到消息队列



核心处理引擎从队列取出数据并处理。



通过存储和索引组件进行时序列数据的存储和metric名的索引



API组件用于处理引擎和其他查询、存储模块的交互







Cyanite的输入模块支持Carbon（Graphite组件），而Kafka,和Pickle则还在计划中。







索引（Index）模块则用于对metric名进行索引和查询。这一模块有两种实现方式：



memory用于在内存中存储反向索引



elasticsearch用于在elasticsearch中存储metric名（path-names）







一句话总结：新生事物、值得关注。



