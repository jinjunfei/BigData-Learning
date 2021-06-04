2021/06/04

# Hadoop学习笔记

摘自：https://easyai.tech/blog/bigdata-what-is-hadoop/

#### 1、Hadoop的发展

刚开始做的搜索引擎，google和Nutch随着时间推移，都面临搜索对象“体积不断增大的问题。因为互联网搜索引擎需要存储大量的网页，并不断地优化搜索算法。

在此过程中，谷歌找到许多好办法发表了学术论文，2003年发表第一篇介绍了谷歌分布式文件系统GFS，这是谷歌为了存储海量搜索数据而设计的专用文件系统。2004年，Doug Cutting基于Google的GFS论文，实现了分布式文件存储系统，命名为NDFS.

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9amy9l7ej30eq050jrp.jpg)

2004年，Google又发表了一篇学术论文，介绍了MapReduce编程模型。这个模型基于大规模数据的并行计算。2005年，Doug Cutting 又基于MapReduce在Nutch搜索引擎上也实现了此功能。

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9an60l99j30ea04wjrq.jpg)

2006年，雅虎公司招安了Doug Cutting,2004年之前雅虎是以Google作为自家搜索引擎的，2004年开始，雅虎放弃谷歌，开始自己研发，加盟Yahoo之后，Doug Cutting 将NDFS和MapReduce进行了升级改造，并重新命名为Hadoop，大名鼎鼎的大数据框架系统由此诞生。

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9ano2eenj30gq0b6dgj.jpg)

hadoop这个名字，实际上他儿子黄色玩具大象的名字；![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9anufpp9j30g704omxj.jpg)

2006年，Google又发表了论文介绍了自己的Big Table,这是一种分布式数据库系统，一种用来处理海量数据的分关系型数据库，Doug Cutting 在自己的Hadoop生态系统中引入了HBase.

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9ao8m768j30dv04qgly.jpg)

2008年，Hadoop正式成为Apache基金会的顶级项目，同年二月Yahoo宣布建成了一个拥有一万个内核的Hadoop集群，并将自己的搜索引擎产品全部部署在上面，7月Hadoop打破世界纪录，成为最快排序1TB数据的系统，用时209秒，此后进入高速发展期。

#### 2、Hadoop的核心架构

Hadoop的核心架构就是HDFS和MapReduce。HDFS为海量数据提供了存储，而MapReduce为海量数据提供了计算框架。

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9ap6xl9rj30ct04f0t3.jpg)

##### HDFS

整个HDFS有三个重要角色，NameNode、DataNode和Client.典型的主从架构，用TCP/IP通信。

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9apg63l5j30a8057mxa.jpg)

NameNode:是Master主节点，可以看作是分布式文件系统中的管理者，主要负责管理文件系统中的命名空间，集群配置信息和存储块的复制等。NameNode会将文件系统的Meta-data存储在内存中，这些信息包括了文件信息，每一个文件对应的文件块的信息和每一个文件块在DataNode中的信息等；

DataNode:是Slave节点，是文件存储的基本单元，他将Block存储在本地文件系统中，保存了Block的Meta-data，同时周期性的将所有存在Block信息发送给NameNode.

Client,切分文件；访问HDFS,与NameNode交互，获取文件位置信息；与DataNode交互，读取和写入数据。

Block,BLock是HDFS中的基本读写单元，HDFS中的文件都是被切割为block进行存储的，这些块被复制到多个DataNode中，块的大小通常为64MB,复制的块数量在创建文件时由Client决定。

#### 3、HDFS的读写流程

##### 写入流程

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9apv3190j30kh0a40ts.jpg)

1、由用户client提出请求，例如需要写入200MB的数据；

2、client制定计划：将数据按照64MB为块进行切分，所有的块保存三份；

3、client将大文件分成块block；

4、针对第一个块，client告诉NameNode，将64MB的块复制3份；

5、NameNode告诉Client三个DataNode的地址，并且将他们据Client的距离进行了排序；

6、Client把数据和清单发送给第一个DataNode

7、第一个DataNode将数据复制给第二个DataNode;

8、第二个DataNode将数据复制给第三个DataNode;

9、如果某一个块的所有数据都已写入，就会向NameNode反馈已完成；

10、针对第二个Block,同样进行如上操作；

11、所有的Block都完成后、关闭文件。NameNode会将数据持久化到磁盘上。

##### 读取流程

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9ar31np8j30l609t0tn.jpg)

1、用户向client提出读取请求；

2、client向namenode请求这个文件所有信息；

3、NameNode将给client这个文件的块列表，以及存储各个块的数据节点清单（按照距离排序）

4、client从距离最近的数据节点下载所需的块；

#### MapReduce

MapReduce其实是一种编程模型，主要分为两部分：Map(映射)和Reduce(归约)

当你向MapReduce框架中提交一个计算作业时，他会首先把作业拆分为若干个Map任务，然后分配到不同的节点上去执行，每一个Map任务处理输入数据的一部分，当Map任务执行完毕后，它会生成一些中间文件，这些中间文件将会作为Reduce的输入数据，Reduce任务的主要目标是把前面若干个Map的输出汇总到一起并输出。

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9arno4r0j30u00axdhm.jpg)

经典例子wordCount

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9artje8oj30u00b4wg5.jpg)

1、Hadoop将输入数据分为若干个分片，并将每个split交给一个map任务处理。

2、Mapping之后，相当于得出这个task里面每个单词以及它出现的次数；

3、shuffle拖移将相同的词放在一起，并对他们进行排序，分成若干个分片；

4、根据这些分片，进行reduce（规约）

5、统计出reduce task的结果，输出到文件；

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9asylk85j30u00eiq5b.jpg)

jobTracker用于调度和管理其它的TaskTracker。JobTracker可以运行于集群中的任一台计算机上。

TaskTracker负责执行任务，必须运行于DataNode上；

#### 4、Hadoop2.0版本

2012年5月，Hadoop推出了2.0版本，2.0版本在HDFS之上，增加了YARN（资源管理框架），为各类应用程序提供资源管理和调度。

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9atg5y5jj30l606kaax.jpg)

#### 5、Hadoop生态圈

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9atuj449j30u00e0mzo.jpg)

在整个Hadoop架构中，计算框架起到承上启下作用，一方面可以操作HDFS中的数据，另一方面可以被封装，提供Hive、Pig这样的上层组件的调用；

HBase:来源于Google的BigTable;是一个高可靠性、高性能、面向列、可伸缩性的分布式数据库。

Hive：是一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，通过类似SQL的语句快速实现简单的MapReduce统计，不必开发专门的MapReduce应用，十分适合数据仓库的统计分析；

Pig:是一个基于Hadoop的大规模数据分析工具，他提供SQL-LIKE的语言叫做Pig Latin，该语言的编译器会把类SQL的数据分析请求转换为一系列的经过优化的MapReduce运算；

zooKeeper:来源于Google的Chubby，它主要用来解决分布式应用中经常遇到的一些数据管理问题，简化分布式应用协调及其管理的难度；

Ambari：Hadoop 的管理工具，可以快捷地监控、部署、管理集群；

Sqoop:用于在Hadoop于传统数据库之间进行数据传递；

Mahout:一个可以扩展的机器学习和数据挖掘库；

![img](https://easy-ai.oss-cn-shanghai.aliyuncs.com/img/006tNc79gy1fz9auuxok6j30u00cj40z.jpg)

#### 6、Hadoop的优点和应用

高可靠性

高扩展性

高效性

高容错性

低成本

