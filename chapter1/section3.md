# section3



Geras





-	-	-

主页	http://1248.io/geras.php	

License	商业	





Geras是一个专注于IoT领域（当然不仅限于传感器采集到的数据）可扩展的、分布式的时序列数据库，用于帮助用户进行快速分析。Geras是一个SaaS服务，但是你也可以购买软件自己部署、托管。







Geras提供了免费版，它不对数据存储的时间和数据量做任何限制，只是不提供SLA而已。不过他们的SaaS主页我却一直没打开过，貌似该DNS记录已经不存在了，官方相关资料也不多，真怀疑这个项目已经废弃了，该产品已经演化为新产品。







Geras速度非常快，它支持对任何时间精度进行Rollup，也支持保存原始数据的精度。它专门为写操作进行了优化，可以接收海量设备的数据输入。







Geras运行在容错、分布式的数据存储之上，支持水平扩展，可以在不停止服务的前提下对系统容量进行调整。







Geras也提供了一套可视化界面，可以方便地对设备、传感器等进行管理，对metric进行图表的可视化等。







Geras支持多种技术来进行数据存储、查询和展示，比如HTTPS、JSON、RESTful、SenML、MQTT、HyperCat，设备数据可以通过HTTP POST或者轻量MQTT发布。







Akumuli





-	-	-

主页	https://github.com/Akumuli/akumuli	

编写语言	C++	

License	Apache License, Version 2.0	

项目创建时间	2014/12	

最新版	pre-release	2015/10/18

活跃度	活跃	

文档	简单	





Akumuli名称来自accumulate，是一个数值型时间序列数据库，可以存储、处理时序列数据。







它的特点如下：



基于日志结构的存储



支持无序数据



实时压缩



基于HTTP的JSON查询支持



支持面向行和面向列的存储



将最新数据放入内存存储



通过metric和tag组织时序列数据，并且可以通过tag进行join操作。



Resampling \(PAA transform\)，滑动窗口



通过基于TCP或UDP的协议接收数据（支持百万数据点每秒）



Continuous queries \(streaming\)







Akumuli由两部分组成：存储引擎和服务程序。目前只有存储引擎被实现了，而且存储引擎可以在没有服务程序的情况下作为嵌入式数据库单独使用（类似sqlite），也就是说，你得通过自己编写程序使用它的API来实现数据读写。







Akumuli的数据协议基于Redis，它的数据格式也和前面看到的不太一样，比如：









+balancers.memusage （实际没有换行）

host=machine1 （实际没有换行）

unit=Gb+20141210T074343.999999999:31





每条数据由三部分组成，第一部分称为ID，可以是数值型（以“：”开始），或者是字符串（以“+”开始），比如这个例子中，cpu host=machine1 region= europe就是ID，它由两部分组成，cpu是metric，host和region则被称为key。







数据的第二部分可以是时间戳（:1418224205），这里的格式为ISO 8601（+20141210T074343.999999）







第三部分为值部分。







类似上面的数据结构，它的query也比较容易理解：









{  

  "where": {       

     "region": \[ "europe", "us-east" \]    

   },

   "metric": "cpu",

   "range": {

     "from": "20160102T123000.000000",        

     "to":   "20160102T123010.000000" 

   }

 }

Akumuli目前还处于开发状态，不适合在生产环境下使用。







Atlas





-	-	-

主页	https://github.com/Netflix/atlas	

编写语言	Scala	

License	Apache License Version 2.0	

项目创建时间	2014/11	

最新版	v1.5.0-rc.4	2016/02

活跃度	活跃	来自于Netflix

文档	详细	





这个和波士顿动力的机器人Atlas，以及其他很多知名软件（HashiCorp、O'reilly）都重名了。







Atlas用于存储带维度信息的时序列数据，由Netflix开发，用于实时分析。Atlas使用了基于内存的存储，速度非常快。







在2011年的时候，Netflix使用自己开发的Epic工具来管理时序列数据，这是一个采用perl、RRDTool和MySQL构建的系统，但是随着数据量的增大（2M-&gt;1.2B），他们创建了这个新项目。







如果说商业智能（business intelligence）是从数据基于时间分析出趋势的话，那么Atlas可以说是一种运维智能（operational intelligence），能给用户描述出当前系统所发生的所有情况。







Atlas的设计目标主要有3点：



通用API



可扩展



维度支持







Atlas采用了类似RRDtool的查询语法，它们都通过一种对URL有好的语法来支持复杂的数据查询。







一句话点评：背靠大树好乘凉，可以试试。

