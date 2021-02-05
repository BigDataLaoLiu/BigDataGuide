# mysql查询太慢，我们如何进行性能优化？

> - 老刘是即将找工作的研究生，自学大数据开发，一路走来，感慨颇深，网上大数据的资料良莠不齐，于是想写一份详细的大数据开发指南。这份指南把大数据的【基础知识】【框架分析】【源码理解】都用自己的话描述出来，让伙伴自学从此不求人。
> - 您的点赞是我持续更新的动力，禁止白嫖，看了就要有收获，一起加油。

今天给大家分享的是MySQL性能优化，也是大数据开发指南MySQL的最后一部分。性能优化对于老刘来说，是必须掌握的一个手段，如何让自己变得更加优秀，这块内容还是好好看看！

本篇内容相对简洁，核心内容在SQL优化经验总结，通过这篇mysql的性能优化，大家能够掌握如下内容：

1. 会使用和分析慢查询日志
2. 会使用和分析profile
3. SQL优化经验总结

## 如何进行性能分析？

一般进行性能分析，分如下三步：

1. 首先需要使用**慢查询日志**功能，去获取所有查询时间比较长的SQL语句
2. 其次**查看执行计划**查看有问题的SQL的执行计划 explain
3. 最后可以使用**show profile**查看有问题的SQL的性能使用情况  

### 慢查询日志分析

首先我们要使用慢查询日志，因为它收集了查询时间比较长的SQL语句，但使用之前必须开启慢查询日志，在配置文件my.cnf（一般为/etc/my.cnf）中的[mysqld] 增加如下参数：

```
slow_query_log=ON
long_query_time=3
slow_query_log_file=/var/lib/mysql/slow-log.log
```

增加这些参数之后，重启MySQL，可以进行查询慢查询日志是否开启。

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-2-6/1612607647167-%E6%85%A2%E6%9F%A5%E8%AF%A2%E6%97%A5%E5%BF%97.PNG" style="zoom:80%;">
</a>
</p>

### 分析慢查询日志的工具  

分析慢查询日志的工具有很多，老刘分享几种工具，详细的用法大家自行查询。

1. mysqldumpslow是MySQL自带的慢查询日志工具，我们可以使用mysqldumpslow工具搜索慢查询日志中的SQL语句。  
2. percona-toolkit是一组高级命令行工具的集合，可以查看当前服务的摘要信息，磁盘检测，分析慢查询日志，查找重复索引，实现表同步等等(有空单独写一篇关于percona-toolkit的入门博客)。

### explain查看有问题的SQL语句

当SQL查询速度比较慢的时候，我们可以用explain查看这个SQL语句的相关情况，[这部分内容已经在精通MySQL之索引篇讲过，大家可以去看看。](https://juejin.cn/post/6923773896248262664#heading-8)

### show profile查看有问题的SQL语句

Query Profiler是MySQL自带的一种query诊断分析工具，通过它可以分析出一条SQL语句的硬件性能瓶颈在什么地方。比如CPU，IO等，以及该SQL执行所耗费的时间等。不过该工具只有在MySQL 5.0.37以及以上版本中才有实现。默认的情况下，MYSQL的该功能没有打开，需要自己手动启动。  

## SQL优化经验总结

由于老刘还是研究生以及还没工作，所以在SQL性能优化这块只能总结出别人的经验分享给大家，老刘本篇主要想做的事情也是分享一些优秀工程师总结的SQL优化知识点，前面的内容写的相对简洁，希望大家不要埋怨！

1. 任何地方都不要使用 select * from t，用具体的字段列表代替“*“，不要返回用不到的任何字段。

2. 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有必要。  

3. 并不是所有索引对查询都有效，SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引，如一表中有字段sex，male、female几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用。  

4. 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。  

5. 尽可能的使用 varchar 代替 char ，因为首先变长字段存储空间小，可以节省存储空间， 其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。  

6. 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，然后 drop table ，这样可以避免系统表的较长时间锁定。  

7. 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where和order by相关的列上建立索引。  

8. 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描。

   例如： select * from t where num is null

   我们可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：select * from t where num=0。

9. 索引字段上不要使用不等，索引字段上使用（！= 或者 < >）判断时，会导致索引失效而转向全表扫描。

10. 应尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描。

    例如： select * from t where num=10 or num=20 

    我们可以这样查询：select * from t where num=10 union all select * from t where num=20  

11. 应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。

    例如：select * from t where num/2=100 

    我们应该改为: select * from t where num=100*2  

12. 应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。

    例如：select * from t where substring(name,1,3)='abc' -- name以abc开头的id

    我们应该改为: select * from t where name like 'abc%'

13. 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。  

14. 很多时候用 exists 代替 in 是一个好的选择。

    例如：select num from a where num in(select num from b)

    我们应该这样替换：select num from a where exists(select 1 from b where num=a.num)  

## 总结

本文作为大数据开发指南MySQL的最后一篇简洁明练的讲述了一些SQL性能优化的技巧，希望大家能够跟着老刘的文章，好好捋捋思路，争取能够用自己的话把这些知识点讲述出来！

尽管当前水平可能不及各位大佬，但老刘会努力变得更加优秀，让各位小伙伴自学从此不求人！

> 大数据开发指南地址如下：
>
> - github：https://github.com/BigDataLaoLiu/BigDataGuide
> - 码云：https://gitee.com/BigDataLiu/BigDataGuide
>
> 如果有相关问题，联系公众号：努力的老刘。文章都看到这了，点赞关注支持一波！

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612020883612-%E7%82%B9%E8%B5%9E.jpg" style="zoom:80%;">
</a>
</p>