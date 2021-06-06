2021/06/05

# Hbase技术学习

摘自：https://www.slidestalk.com/HBaseGroup/HBase_ebook28532

《2018HBase技术总结》

## 生态

### HBase生态介绍

HBase是一种分布式、对版本、面向列的开源KV数据库。稀疏的矩阵设计使得其特别适合存储非结构的化的数据。数据比列如下图所示;

![image-20210605102335410](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210605102335410.png)

急需一种系统，能够存储这些年逐年增长的数据，所以有必要在HBase之上引入各种组件，使得HBase能够支持SQL、时序、时空、图、全文检索、及复杂分析、完整的生态如下：

![image-20210605102630515](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210605102630515.png)

1、phoenix：主要提供SQL的方式来查询HBase里面的数据。一般能够在毫秒级返回，适合OLTP以及操作性分析等场景。支持构建二级索引；

2、Spqrk：很多企业使用HBase存储海量数据，一种常见的需求就是对这些数据进行离线分析，可以使用Spark来实现海量数据的离线分析需求。同时支持流计算，解决实时广澳推荐需求；

3、HGraphDB：是分布式图数据库，进行图OLTP查询，同时结合Spqrk GraphFrames实现图分析需求，通过图关联技术帮助金融机构有效识别隐藏在网络中的黑色信息，在团伙欺诈、黑中介识别等；

4、GeoMesa:目前基于NoSql数据库的时空数据引擎中功能最丰富、社区贡献人数最多的开源系统。提供高效时空索引、支持点、线、面等空间要素存储、百亿级数据实现毫秒级响应、提供轨迹查询、区域分布统计、区域查询、密度分析、聚合、等常用的时空分析功能；

5、OpenTSDB：基于HBase的分布式、可伸缩的时间序列数据库。适合做监控系统；如手机大规模集群的监控数据进行存储、查询；

6、Solor:原生的HBase只提供了Rowkey单主键，如果需要对逐渐之外的列进行查找，这时候就会有问题。幸好可以使用solor来建立二级索引充分满足我们的查询需求；

#### HBase进化—从NoSQL到NewSQL

![image-20210605104112989](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210605104112989.png)

单纯从解决大数据实际读写问题，最初的HBase系统可以去掉很多传统数据库无关的功能，比如事务、SQL表达与分析等；重点关注于分布式的扩展性、容错性、分布式缓存设计、读写性能优化。

走过了从无到有的第一阶段，我们需要让HBase更强大，更好用，门槛更低。HBase很有必要回过头来支持SQL,当然HBase SQL的实现和发展与传统单机数据库差不多，便于区别我们称之为NewSQL,这也是Phoenix的项目初衷；

![image-20210605104751428](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210605104751428.png)

黄色部分是HBase的组件，蓝色部分是Phoenix，我们可以看出Phoenix和HBase是紧密结合的。Phoenix首先要能通过SQL暴露HBase的能力，然后通过一系列的功能和特性增强HBase的能力。

##### 1、基于HBase的SQL

首先把SQL语言翻译成HBase API，并且HBase本身的功能特性必须通过SQL暴漏出来。Phoenix SQL语法即SQL-92标准+方言。

Hbase的某些特性也将通过SQL方言表达出来：

HBase的列族，把相关列放在一起减少IO,优化读写性能，Phoenix表的时候直接写成”cf.col"即可，会自动创建列族；Phoenix将insert和update合并成upsert关键字，来对应HBase的put概念；支持预分区，支持动态列；

![image-20210605105637905](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210605105637905.png)

当数据访问比较集中时，这部分数据往往会集中在某一个regionserver上，就会形成热点在成性能下降，此时需要打散数据来解决，加盐就是这样的一个技术，本质就是在原有的主键上加一个字节的前缀：

![image-20210606075831363](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606075831363.png)

###### 使用接口：

phoenix提供JDBC的方式访问，支持重客户端和轻客户端两种方式，重客户段前缀：jdbc:phoenix,轻客户端前缀jdbc:phoenix:thin。

###### 阿里实践：

1、对实时写入的并发和TPS要求很高，甚至可以达到百万级；

2、数据量较大，一般在TB到PB级别，且需要根据业务增长动态扩容；

3、有动态列的需求，根据业务调整，动态添加监控指标；

4、在线查询，甚至是毫秒级别的查询需求；

5、频繁查询最近一段时间的数据，会造成热点，可以使用加盐特性；

6、维度比较多，需要用到二级索引，以加速在线查询。

#### 重庆新能源汽车

架构图：

![image-20210606081802015](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606081802015.png)

左侧实时采集数据发送到平台，验证完之后写入到Kafka消息队列中，流式计算服务器从Kafka消息队列中提取原始数据，对数据进行解析、存储、转发等操作。HBase集群负责存储数据，MySQL负责存储组织关系数据。同时将超过一定时间的数据转存到OSS存储介质中，降低成本。Spark会对各项数据进行分析。

##### HBase使用难点

##### Row key设计

1、设备ID+时间戳:某一批次的设备故障造成单点region写入压力过大；

2、subString(MD5(设备ID),x)+设备ID+时间戳：此种方式将所有车辆打散到每个region中去，避免热点和数据倾斜问题，式中的x一般取5左右。

##### 多语言连接问题

团队使用python开发系统，可是HBase使用JAVA语言编写，原生提供了JAVA API，并未对python语言提供独立的API。我们使用**happybase连接HBase**，因为它提供了更为Pythonic的接口服务。另外，我们也使用QueryServer作为python组件和Phoenix连接的纽带。

#### 滴滴出行 李扬

##### 1、业务类型

Hbase原生对离线任务支持好，又因为LSM树是一个优秀的高吞吐数据库结构，所以同时也对接了很多线上业务。离线业务通常是数仓的定时大批量处理任务，对一段时间内的数据进行处理并产生结果，对任务完成的时间要求并不是很敏感，并且处理逻辑复杂，如天级别报表、安全和用户行为分析、模型训练等。

##### 2、多语言支持

HBase提供了多语言解决方案，并且由于滴滴各业务线RD所使用的开发语言各有偏好，HBase Java native API 、Thrift Server(主要应用于C++、PHP、Python)、JAVAJDBC、Phoenix QueryServer、Map reduce job。

##### 3、场景

近期订单会落在Redis上，超过一定时间不用时，就会落在HBase上。

![image-20210606094136017](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606094136017.png)

### HBase基本知识介绍及典型案例分析

吴阳平 阿里巴巴 HBase业务架构师

##### 1、HBase基本知识

分布式、多版本、面向列的开源数据库。特点是支持上亿行、百万列、支持强一致性、并且具有高扩展、高可用等特点。

组成部分：Rowkey、coloumn Family、column、Version number、value。

在物理层面上，所有的数据其实是存放在Region里面的，而且Region又由RegionServer管理。

![image-20210606094855973](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606094855973.png)

2、HBase的读写流程

物理视图：数据是以kv形式存储的，每个kv只存储一个cell里面的数据，不同的CF的数据是存在不同的文件里的。

HBase的写过程：

先将数据写到WAl中；

WAl存放在HDFS之上；

每次Put、delete操作的数据均追加到WAL末端；

持久化到WAL之后，再写到MemStore中；

两者写完返回ACK客户端；

![image-20210606095557353](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606095557353.png)

既然写数都是先写WAL，再写Memstore，而Memstore是内存结构总会写满的，将memstore数据从内存刷写到磁盘的操作成为flush；

![image-20210606095805699](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606095805699.png)

每次的Flush操作都将一个Memstore的数据写到一个Hfile里面，所以上图中HDFS上有许多个HFile文件，所以HBase会隔一定的时间将HFile进行合并，根据合并范围不同分为MinorCompaction 和 major compaction：

![image-20210606100054931](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606100054931.png)

HBase的读操作更加复杂，其读取Blockcache、memstore以及HFile。

![image-20210606100248336](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606100248336.png)

**如何查找RegionServer呢？**

![image-20210606100440517](C:\Users\jinjunfei\AppData\Roaming\Typora\typora-user-images\image-20210606100440517.png)

##### 3、Rowkey设计

###### Rowkey的作用：

读写数据时通过RowKey找到对应的Region

MemStore中的数据按Row Key字典顺序排序

HFile中的数据按照Row Key字典序排序

所以，底层的HFile最终是按照RowKey进行切分的，所以我们的设计原则是结合业务特点，考虑高频查询，尽可能的将数据打散到整个集群。

1、Salting：原理是将固定长度的随机数放在行键的起始处；

2、Hashing：进行哈希计算，取hash的部分字符串和原来的Rowkey进行拼接；实际应用中一般选md5计算

3、Reversing：原理是反转一段固定长度或者全部的键；牺牲了排序的特性；

