# 📔大数据开发指南

Java大数据开发校招面试常见八股文整理，老刘2021年秋招面试总结，包括：大数据开发组件基础 | 框架 | 源码、Java基础、操作系统、数据结构、计算机网络、MySQL等面试资料，老刘会坚持将此仓库维护下去，让更多自学的人少走弯路。

👉本资料是老刘结合秋招经历总结，适合所有应届生阅读，希望对你们有用；

👉老刘的秋招offer：[普通211研究生大数据开发的秋招总结](https://mp.weixin.qq.com/s?__biz=Mzg3NTU3NDg0Mg==&mid=2247483677&idx=1&sn=6541b4c4be3f17ee98a8bbe8163dbd0d&chksm=cf3e2248f849ab5e89f08c1f5bab0f572ed3975f1178ebb76763446e760fc6905162abb4440c&token=851190291&lang=zh_CN#rd)


### 必看：说给各位伙伴的话​ :fire:

目前，这份大数据开发指南很多知识点都没有写，大家暂且把它当做一份学习路线来看，以后老刘会陆续更新，不会让大家失望的，必会让各位自学从此不求人！

## 大数据开发学习路线​ :+1:

### 一、Java基础、Linux基础 

网上资料太多了，老刘这块就忽略了！

### 二、MySQL进阶知识点

1. [mysql架构： 精通MySQL之架构篇](https://zhuanlan.zhihu.com/p/347082910)
2. [mysql索引： MySQL索引篇](https://zhuanlan.zhihu.com/p/348301974)
3. [mysql锁：精通MySQL之锁篇](https://zhuanlan.zhihu.com/p/348487133)
4. [mysql事务： MySQL事务篇](https://zhuanlan.zhihu.com/p/349418326)
5. mysql性能优化： mysql查询太慢，我们如何进行性能优化？

### 三、分布式文件系统Hadoop

1. **分布式存储系统——HDFS**（还没写，陆续更新）

   HDFS的概念、命令、编程

   核心概念Blocks（重点）

   HDFS架构（重点）

   HDFS读写流程（重点）

   Hadoop HA高可用（重点）

   文件压缩

   小文件治理（重点）

2. **分布式计算框架——MapReduce**（还没写，陆续更新）

      MapReduce编程模型（重点）

      MapReduce原理图（重点）

      MapReduce编程

      Shuffle（重点）

      自定义分区

      自定义Combiner

      MR压缩

      自定义InputFormat

      MapReduce数据倾斜（重点）

3. **资源调度系统——Yarn**（还没写，陆续更新）

      YARN介绍

      YARN架构（重点）

      YARN应用运行原理（重点）

      如何使用YARN

      YARN调度器

      YARN应用状态

### 四、分布式理论原理解析

[**带你了解分布式系统的数据一致性问题**](https://juejin.cn/post/6916095586169159694)

这是老刘发的一篇博客，里面详细讲述了分布式理论的由来发展和实现！

- **核心知识点**（这是一个完整的流程，就按这个顺序记）

  分布式的发展

  分布式事务：2PC和3PC

  分布式一致性算法：Paxos算法和ZAB协议

  鸽巢原理

  Quorum NWR机制

  CAP理论

  BASE理论

### 五、分布式协调框架ZooKeeper

第四部分给大家讲的是分布式理论，核心就是如何解决分布式数据一致性，大佬们根据ZAB协议实现了分布式协调框架ZooKeeper。

在大数据领域分布式无处不在，相当于必须要掌握的东西，大家一定要理顺所有核心知识点！

- **ZooKeeper核心知识点：（全部都是重点）**

  为什么要用ZooKeeper

  什么是ZooKeeper

  ZooKeeper命令行、java编程

  基本概念和操作

  ZooKeeper工作原理

  HDFS HA方案

  ZooKeeper集群架构、读写流程

  leader选举、ZAB算法

  ZooKeeper状态同步

### 六、分布式数据仓库Hive

1. **大数据分析利器hive的基础知识点**

   核心概念

   数据库和数据仓库的区别

   架构原理

   交互式方式

   数据类型

   DDL语法操作

2. **大数据分析利器hive的DDL操作和DML操作**

   hive中数据导入的方式

   hive中数据导出的方式

   hive创建分区表和使用方式

   hive的静态分区和动态分区

   hive中的分桶表作用

3. **大数据分析利器hive的查询操作**

   select查询语句中的基本查询

   select查询语句的分组

   select查询语句中的join

   select查询语句中的排序

4. **大数据分析利器hive的高级操作**

   hive表的数据压缩和文件存储格式

   hive的自定义UDF函数

   hive的JDBC代码操作

   hive的SerDe介绍和使用

5. **大数据分析利器hive的优化（重点）**

   如何优化

   如何避免或减轻数据倾斜

6. **hive的综合案例实战（一定要重点练习）**

### 七、分布式列式数据库HBase

1. **大数据数据库HBase的基础知识点**

   HBase是什么

   HBase表的数据模型

   HBase整体架构

   HBase shell 命令基本操作

   HBase的高级shell管理命令

2. **大数据数据库HBase的原理性知识点**

   HBase的数据存储原理

   HBase读 | 写数据流程

   HBase的flush、compact机制

   region 拆分机制

   HBase表的预分区

   region 合并

3. **大数据数据库HBase的实操知识点**

   HBase集成MapReduce

   HBase与Hive的对比

   HBase协处理器

   HBase表的rowkey设计

   HBase表的热点

   HBase的数据备份

   HBase二级索引

4. **phoenix构建二级索引详解**

### 八、分布式消息系统Kafka

1. **Kafka的基础知识点（前3条必须掌握）**

   为什么有消息系统

   Kafka核心概念

   Kafka集群架构

   kafka集群安装部署

   kafka集群启动和停止

   kafka的命令行的管理使用

   kafka的生产者和消费者api代码开发

2. **Kafka的核心知识点（必须掌握）**

   kafka分区策略

   kafka的文件存储机制

   为什么Kafka速度那么快

   Kafka的内核原理之ISR-HW-LEO机制

   producer消息发送原理

   consumer消费原理

   consumer消费者Rebalance策略

3. **Kafka的应用**

   kafka整合flume

   kafka监控工具安装和使用

### 九、大数据辅助框架

在一个完整的离线大数据处理系统中，除了hdfs+mapreduce+hive组成分析系统的核心之外，还需要数据采集、结果数据导出、任务调度等不可或缺的辅助系统，而这些辅助工具在hadoop生态体系中都有便捷的开源框架。（最后三个工具老刘只是知道没有练习过）

1. **日志采集框架Flume**

   Flume是什么

   Flume的架构

   Flume采集系统结构图

   Flume安装部署

   Flume实战

2. [**同步MySQL增量数据工具Canal**](https://juejin.cn/post/6920429751186227207)

   mysql主备复制实现原理

   mysql二进制文件格式

   Canal概念 | 工作原理 | 架构设计

   Canal同步mysql数据

   Canal的HA设计

   数据同步解决方案总结

3. **离线数据同步框架DataX**

   DataX概念

   DataX核心架构

4. **数据迁移工具Sqoop**

   sqoop的核心概念

   sqoop的架构原理

   sqoop的导入

   sqoop的导出

5. **工作流调度器Azkaban**

   为什么需要工作流调度系统

   Azkaban的核心概念

   Azkaban的架构原理

   Azkaban的安装和使用

6. **任务调度工具oozie**

7. **图形化界面工具hue**

8. **SQL查询工具Impala**

### 十、内存计算框架Spark

1. **spark的基础知识点**

   spark的核心概念

   spark集群架构和安装

   spark-shell的使用

   通过IDEA开发spark程序

2. **spark的底层抽象RDD**（这个必须熟记于心）

   RDD是什么

   RDD的五大属性

   RDD的创建方式

   RDD的算子分类

   RDD常见的算子操作说明

   RDD常用的算子操作演示和案例

3. **RDD原理剖析**

   RDD的依赖关系

   lineage（血统）

   RDD的缓存机制

   RDD的checkpoint机制

   DAG有向无环图的生成

   DAG如何划分stage

4. **spark任务提交调度和其他特性**

   spark任务的提交

   spark的运行架构

   spark中的共享变量

   spark程序的序列化

   application、job、stage、task之间的关系

5. **spark on  yarn和任务资源分配**

   spark on yarn原理和机制

   collect 算子操作剖析 

   spark任务中资源参数剖析

   spark任务的调度模式

   spark任务的分配资源策略

   spark的shuffle原理分析

6. **sparkSQL基础入门**

   sparksql概述

   sparksql的四大特性

   DataFrame概述

   读取文件构建DataFrame

   DataFrame常用操作

   DataSet概述

   通过IDEA开发程序实现把RDD转换DataFrame

7. **sparkSQL的应用案例**

   sparksql操作hivesql

   sparksql整合hive

8. **Spark调优（重中之重的重点）**

   Spark应用程序性能优化

   基于Spark内存模型调优

   数据倾斜原理和现象分析

9. **sparkStreaming实时模块**

### 十一、最新实时计算框架Flink

### 十二、后记​ :muscle:

**还没写完，陆续更新**~~~


