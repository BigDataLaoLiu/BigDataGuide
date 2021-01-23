
> 老刘是一名即将找工作的研二学生，写博客一方面是总结大数据开发的知识点，一方面是希望能够帮助伙伴让自学从此不求人。由于老刘是自学大数据开发，博客中肯定会存在一些不足，还希望大家能够批评指正，让我们一起进步！
- [背景](#背景)
- [mysql 主备复制实现原理](#mysql-主备复制实现原理)
- [Canal 核心知识点](#canal-核心知识点)
  - [Canal 的工作原理](#canal-的工作原理)
  - [Canal 概念](#canal-概念)
  - [Canal 架构](#canal-架构)
- [Canal 同步 MySQL 增量数据](#canal-同步-mysql-增量数据)
  - [开启 mysql binlog](#开启-mysql-binlog)
  - [Canal 实时同步](#canal-实时同步)
- [Canal 的 HA 机制设计](#canal-的-ha-机制设计)
- [数据同步方案总结](#数据同步方案总结)
  - [DataX (处理离线数据)](#datax-处理离线数据)
  - [Flume（处理实时数据）](#flume处理实时数据)
  - [Logstash（处理离线数据）](#logstash处理离线数据)
  - [Sqoop（处理离线数据）](#sqoop处理离线数据)
- [总结](#总结)
## 背景

大数据领域数据源有业务库的数据，也有移动端埋点数据、服务器端产生的日志数据。我们在对数据进行采集时根据下游对数据的要求不同，我们可以使用不同的采集工具来进行。今天老刘给大家讲的是同步mysql增量数据的工具Canal，本篇文章的大纲如下：

1. Canal 的概念
2. mysql 中主备复制实现原理
3. Canal 如何从 MySQL 中同步数据
4. Canal 的 HA 机制设计
5. 各种数据同步解决方法的简单总结

老刘争取用这一篇文章让大家直接上手 Canal 这个工具，不再花别的时间来学习。

## mysql 主备复制实现原理

由于 Canal 是用来同步 mysql 中增量数据的，所以老刘先讲 mysql 的主备复制原理，之后再讲 Canal 的核心知识点。



根据这张图，老刘把 mysql 的主备复制原理分解为如下流程：

1. 主服务器首先必须启动二进制日志 binlog，用来记录任何修改了数据库数据的事件。
2. 主服务器将数据的改变记录到二进制 binlog 日志。
3. 从服务器会将主服务器的二进制日志复制到其本地的中继日志（Relaylog）中。这一步细化的说就是首先从服务器会启动一个工作线程 I/O 线程，I/O 线程会跟主库建立一个普通的客户单连接，然后在主服务器上启动一个特殊的二进制转储（binlog dump）线程，这个 binlog dump 线程会读取主服务器上二进制日志中的事件，然后向 I/O 线程发送二进制事件，并保存到从服务器上的中继日志中。
4. 从服务器启动 SQL 线程，从中继日志中读取二进制日志，并且在从服务器本地会再执行一次数据修改操作，从而实现从服务器数据的更新。

那么 mysql 主备复制实现原理就讲完了，大家看完这个流程，能不能猜到 Canal 的工作原理？

## Canal 核心知识点

### Canal 的工作原理

Canal 的工作原理就是它模拟 MySQL slave 的交互协议，把自己伪装为 MySQL slave，向 MySQL master 发动 dump 协议。MySQL master 收到 dump 请求后，就会开始推送 binlog 给 Canal。最后 Canal 就会解析 binlog 对象。

### Canal 概念

Canal，美[kəˈnæl]，是这样读的，意思是水道/管道/渠道，主要用途就是用来同步 MySQL 中的增量数据（可以理解为实时数据），是阿里巴巴旗下的一款纯 Java 开发的开源项目。

### Canal 架构

![](https://imgkr2.cn-bj.ufileos.com/915ba506-7df4-4acb-a964-9643b38af366.PNG?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=8SODrh%252FJhniwd9cqFtsaDfBCtq0%253D&Expires=1611314417)

server 代表一个 canal 运行实例，对应于一个 JVM。
instance 对应于一个数据队列，1 个 canal server 对应 1..n 个 instance
instance 下的子模块：

1. EventParser:数据源接入，模拟 salve 协议和 master 进行交互，协议解析
2. EventSink：Parser 和 Store 链接器，进行数据过滤，加工，分发的工作
3. EventStore：数据存储
4. MetaManager: 增量订阅&消费信息管理器

到现在 Canal 的基本概念就讲完了，那接下来就要讲 Canal 如何同步 mysql 的增量数据。

## Canal 同步 MySQL 增量数据

### 开启 mysql binlog

我们用 Canal 同步 mysql 增量数据的前提是 mysql 的 binlog 是开启的，阿里云的 mysql 数据库是默认开启 binlog 的，但是如果我们是自己安装的 mysql 需要手动开启 binlog 日志功能。

先找到 mysql 的配置文件：

```
etc/my.cnf

server-id=1
log-bin=mysql-bin
binlog-format=ROW
```

这里有一个知识点是关于 binlog 的格式，老刘给大家讲讲。

binlog 的格式有三种：STATEMENT、ROW、MIXED

1. ROW 模式（一般就用它）

   日志会记录每一行数据被修改的形式，不会记录执行 SQL 语句的上下文相关信息，只记录要修改的数据，哪条数据被修改了，修改成了什么样子，只有 value，不会有 SQL 多表关联的情况。

   优点：它仅仅只需要记录哪条数据被修改了，修改成什么样子了，所以它的日志内容会非常清楚地记录下每一行数据修改的细节，非常容易理解。

   缺点：ROW 模式下，特别是数据添加的情况下，所有执行的语句都会记录到日志中，都将以每行记录的修改来记录，这样会产生大量的日志内容。

2. STATEMENT 模式

   每条会修改数据的 SQL 语句都会被记录下来。

   缺点：由于它是记录的执行语句，所以，为了让这些语句在 slave 端也能正确执行，那他还必须记录每条语句在执行过程中的一些相关信息，也就是上下文信息，以保证所有语句在 slave 端被执行的时候能够得到和在 master 端执行时候相同的结果。

   但目前例如 step()函数在有些版本中就不能被正确复制，在存储过程中使用了 last-insert-id()函数，可能会使 slave 和 master 上得到不一致的 id，就是会出现数据不一致的情况，ROW 模式下就没有。

3. MIXED 模式

   以上两种模式都使用。

### Canal 实时同步

1. 首先我们要配置环境，在 conf/example/instance.properties 下：

   ```
   ## mysql serverId
   canal.instance.mysql.slaveId = 1234
   #position info，需要修改成自己的数据库信息
   canal.instance.master.address = 127.0.0.1:3306
   canal.instance.master.journal.name =
   canal.instance.master.position =
   canal.instance.master.timestamp =
   #canal.instance.standby.address =
   #canal.instance.standby.journal.name =
   #canal.instance.standby.position =
   #canal.instance.standby.timestamp =
   #username/password，需要修改成自己的数据库信息
   canal.instance.dbUsername = canal
   canal.instance.dbPassword = canal
   canal.instance.defaultDatabaseName =
   canal.instance.connectionCharset = UTF-8
   #table regex
   canal.instance.filter.regex = .\*\\\\..\*
   ```

   其中，canal.instance.connectionCharset 代表数据库的编码方式对应到 java 中的编码类型，比如 UTF-8，GBK，ISO-8859-1。

2. 配置完后，就要启动了
   ```
   sh bin/startup.sh
   关闭使用 bin/stop.sh
   ```
3. 观察日志

   一般使用 cat 查看 canal/canal.log、example/example.log

4. 启动客户端

   在 IDEA 中业务代码，mysql 中如果有增量数据就拉取过来，在 IDEA 控制台打印出来

   在 pom.xml 文件中添加：

   ```
   <dependency>
     <groupId>com.alibaba.otter</groupId>
     <artifactId>canal.client</artifactId>
     <version>1.0.12</version>
   </dependency>
   ```

   添加客户端代码：

   ```
   public class Demo {
    public static void main(String[] args) {
        //创建连接
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("hadoop03", 11111),
                "example", "", "");
        connector.connect();
        //订阅
        connector.subscribe();
        connector.rollback();
        int batchSize = 1000;
        int emptyCount = 0;
        int totalEmptyCount = 100;
        while (totalEmptyCount > emptyCount) {
            Message msg = connector.getWithoutAck(batchSize);
            long id = msg.getId();
            List<CanalEntry.Entry> entries = msg.getEntries();
            if(id == -1 || entries.size() == 0){
                emptyCount++;
                System.out.println("emptyCount : " + emptyCount);
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }else{
                emptyCount = 0;
                printEntry(entries);
            }
            connector.ack(id);
        }
    }
    // batch -> entries -> rowchange - rowdata -> cols
    private static void printEntry(List<CanalEntry.Entry> entries) {
        for (CanalEntry.Entry entry : entries){
            if(entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONBEGIN ||
                    entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONEND){
                continue;
            }
            CanalEntry.RowChange rowChange = null;
            try {
                rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
            } catch (InvalidProtocolBufferException e) {
                e.printStackTrace();
            }
            CanalEntry.EventType eventType = rowChange.getEventType();
            System.out.println(entry.getHeader().getLogfileName()+" __ " +
                    entry.getHeader().getSchemaName() + " __ " + eventType);
            List<CanalEntry.RowData> rowDatasList = rowChange.getRowDatasList();
            for(CanalEntry.RowData rowData : rowDatasList){
                for(CanalEntry.Column column: rowData.getAfterColumnsList()){
                    System.out.println(column.getName() + " - " +
                            column.getValue() + " - " +
                            column.getUpdated());
                }
            }
        }
    }
   }
   ```
5. 在mysql中写数据，客户端就会把增量数据打印到控制台。

## Canal 的 HA 机制设计

在大数据领域很多框架都会有 HA 机制，Canal 的 HA 分为两部分，Canal server 和 Canal client 分别有对应的 HA 实现：

1. canal server：为了减少对 mysql dump 的请求，不同 server 上的 instance 要求同一时间只能有一个处于 running，其他的处于 standby 状态。
2. canal client：为了保证有序性，一份 instance 同一时间只能由一个 canal client 进行 get/ack/rollback 操作，否则客户端接收无法保证有序。

整个 HA 机制的控制主要是依赖了 ZooKeeper 的几个特性，ZooKeeper 这里就不讲了。

Canal Server：

1. canal server 要启动某个 canal instance 时都先向 ZooKeeper 进行一次尝试启动判断（创建 EPHEMERAL 节点，谁创建成功就允许谁启动）。
2. 创建 ZooKeeper 节点成功后，对应的 canal server 就启动对应的 canal instance，没有创建成功的 canal instance 就会处于 standby 状态。
3. 一旦 ZooKeeper 发现 canal server 创建的节点消失后，立即通知其他的 canal server 再次进行步骤 1 的操作，重新选出一个 canal server 启动 instance。
4. canal client 每次进行 connect 时，会首先向 ZooKeeper 询问当前是谁启动了 canal instance，然后和其建立连接，一旦连接不可用，会重新尝试 connect。
5. canal client 的方式和 canal server 方式类似，也是利用 ZooKeeper 的抢占 EPHEMERAL 节点的方式进行控制。

Canal HA 的配置，并把数据实时同步到 kafka 中。

1. 修改 conf/canal.properties 文件
   ```
   canal.zkServers = hadoop02:2181,hadoop03:2181,hadoop04:2181
   canal.serverMode = kafka
   canal.mq.servers = hadoop02:9092,hadoop03:9092,hadoop04:9092
   ```
2. 配置 conf/example/example.instance
   ```
   canal.instance.mysql.slaveId = 790 /两台canal server的slaveID唯一
   canal.mq.topic = canal_log //指定将数据发送到kafka的topic
   ```

## 数据同步方案总结

讲完了 Canal 工具，现在给大家简单总结下目前常见的数据采集工具，不会涉及架构知识，只是简单总结，让大家有个印象。

常见的数据采集工具有：DataX、Flume、Canal、Sqoop、LogStash 等。

### DataX (处理离线数据)

DataX 是阿里巴巴开源的一个异构数据源离线同步工具，异构数据源离线同步指的是将源端数据同步到目的端，但是端与端的数据源类型种类繁多，在没有 DataX 之前，端与端的链路将组成一个复杂的网状结构，非常零散无法把同步核心逻辑抽象出来。

![](https://imgkr2.cn-bj.ufileos.com/9908b5c8-5c11-4e7e-8735-c128c114a08b.PNG?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=f7RlW9ZYA2Sg4nq8g8tYx5KyMVg%253D&Expires=1611318590)

为了解决异构数据源同步问题，DataX 将复杂的网状的同步链路变成了星型数据链路，DataX 作为中间传输载体负责连接各种数据源。

所以，当需要接入一个新的数据源的时候，只需要将此数据源对接到 DataX，就可以跟已有的数据源做到无缝数据同步。

![](https://imgkr2.cn-bj.ufileos.com/71c5a50f-2d33-4c3b-9434-bcf1bb188b95.PNG?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=qA7z9%252FOkNQ4qWHDDVFuMQtwj%252FPo%253D&Expires=1611318772)

DataX本身作为离线数据同步框架，采用Framework+plugin架构构建。将数据源读取和写入抽象成为Reader/Writer插件，纳入到整个同步框架中。

1. Reader: 它为数据采集模块，负责采集数据源的数据，将数据发送给Framework。
2. Writer: 它为数据写入模块，负责不断向Framework取数据，并将数据写入到目的端。
3. Framework：它用于连接Reader和Writer，作为两者的数据传输通道，并处理缓冲、并发、数据转换等问题。

DataX的核心架构如下图：

![](https://imgkr2.cn-bj.ufileos.com/ca599c1b-4f42-488a-8f2c-a318b0c2b31a.PNG?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=%252Fjinx%252Fcz9ZSTgf16Phb0JAAi1iw%253D&Expires=1611322055)

核心模块介绍：

1. DataX完成单个数据同步的作业，我们把它称之为Job，DataX接收到一个Job之后，将启动一个进程来完成整个作业同步过程。
2. DataX Job启动后，会根据不同的源端切分策略，将Job切分成多个小的Task(子任务)，以便于并发执行。
3. 切分多个Task之后，DataX Job会调用Scheduler模块，根据配置的并发数据量，将拆分成的Task重新组合，组装成TaskGroup(任务组)。每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，默认单个任务组的并发数量为5。
4. 每一个Task都由TaskGroup负责启动，Task启动后，会固定启动Reader->Channel->Writer的线程来完成任务同步工作。
5. DataX作业运行完成之后，Job监控并等待多个TaskGroup模块任务完成，等待所有TaskGroup任务完成后Job成功退出。否则，异常退出。

### Flume（处理实时数据）

![](https://imgkr2.cn-bj.ufileos.com/9199e4f1-8fb9-4cc2-be9b-77ed9d704bc7.PNG?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=aoyiBfceKbfluLk5cuOtMMarfWw%253D&Expires=1611322722)

Flume主要应用的场景是同步日志数据，主要包含三个组件：Source、Channel、Sink。

Flume最大的优点就是官网提供了丰富的Source、Channel、Sink，根据不同的业务需求，我们可以在官网查找相关配置。另外，Flume还提供了自定义这些组件的接口。

### Logstash（处理离线数据）

![](https://imgkr2.cn-bj.ufileos.com/c7bfadd6-26cf-4ac1-a304-051c0b5a298f.PNG?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=TCDLeZeLf%252Ft1XKTovAtNvQxbw4I%253D&Expires=1611322912)

Logstash就是一根具备实时数据传输能力的管道，负责将数据信息从管道的输入端传输到管道的输出端；与此同时这根管道还可以让你根据自己的需求在中间加上过滤网，Logstash提供了很多功能强大的过滤网来满足各种应用场景。

Logstash是由JRuby编写，使用基于消息的简单架构，在JVM上运行。在管道内的数据流称之为event，它分为inputs阶段、filters阶段、outputs阶段。

### Sqoop（处理离线数据）

![](https://imgkr2.cn-bj.ufileos.com/fdc7c36c-60fb-4fdf-b916-0f10917722b2.PNG?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=O2yJk2gigOTOql5lz8sdoLe3Akc%253D&Expires=1611323159)

Sqoop是Hadoop和关系型数据库之间传送数据的一种工具，它是用来从关系型数据库如MySQL到Hadoop的HDFS从Hadoop文件系统导出数据到关系型数据库。Sqoop底层用的还是MapReducer，用的时候一定要注意数据倾斜。

## 总结

老刘本篇文章主要讲述了Canal工具的核心知识点及其数据采集工具的对比，其中数据采集工具只是大致讲了讲概念和应用，目的也是让大家有个印象。老刘敢做保证看完这篇文章基本等于入门，剩下的就是练习了。

好啦，同步mysql增量数据的工具Canal的内容就讲完了，尽管当前水平可能不及各位大佬，但老刘会努力变得更加优秀，让各位小伙伴自学从此不求人！

如果有相关问题，联系公众号：努力的老刘。文章都看到这了，点赞关注支持一波！

