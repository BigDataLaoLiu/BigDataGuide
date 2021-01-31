# MySQL索引篇

> - 老刘是即将找工作的研究生，自学大数据开发，一路走来，感慨颇深，网上大数据的资料良莠不齐，于是想写一份详细的大数据开发指南。这份指南把大数据的【基础知识】【框架分析】【源码理解】都用自己的话描述出来，让伙伴自学从此不求人。
> - 大数据开发指南地址如下：
>   - github：https://github.com/BigDataLaoLiu/BigDataGuide 
>   - 码云：https://gitee.com/BigDataLiu/BigDataGuide
> - 您的点赞是我持续更新的动力，禁止白嫖，看了就要有收获，有需要联系公众号：努力的老刘。

今天给大家分享的是大数据开发基础部分MySQL的索引篇，这篇内容大家一定要跟着练习，不然等于白看！

## 索引是什么？

在日常开发中常常会遇到查询比较慢的情况，我们的第一反应就是给它加索引，那索引是什么呢？官方介绍索引是帮助MySQL高效获取数据的数据结构，数据库索引好比是一本书的目录，能加快数据库的数据查询速度。

那索引的好处有哪些呢？

1. 它可以提高数据检索的效率，降低数据库的成本。
2. 通过索引对数据进行排序，降低数据排序的成本，降低CPU消耗。

任何事情都会有正反面，索引也不例外，那索引的坏处有哪些呢？

1. 索引会占据磁盘空间。
2. 索引虽然会提高查询效率，但会降低更新表的效率。
3. MySQL不仅要保存数据，还有保存或者更新对应的索引文件。

那是不是有坏处就不用索引呢？

当然不是，索引必须拿来。一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往是存储在磁盘文件上的文件中。

## 索引的分类

1. 单列索引：
   - 普通索引：add unique
   - 唯一索引：索引列中的值必须是唯一的，但允许有空值，add unique index
   - 主键索引：是一种特殊的唯一索引，不允许有空值
2. 组合索引：
   - 在表中的多个字段组合上创建的索引
   - 组合索引的使用，需要遵循最左前缀原则
   - 一般情况下，建议使用组合索引代替单列索引（主键索引除外）
3. 全文索引：只有在MyIsam、InnoDB上才能使用，而且只能在char、varchar、text类型字段上使用全文索引。
4. 空间索引：一般用不到

## 索引的使用

创建索引

```
CREATE INDEX index_name ON table(column(length)) ;
```

删除索引

```
DROP INDEX index_name ON table;
```

查看索引

```
SHOW INDEX FROM table_name \G;
```

## 索引原理（重点）

### 索引的存储结构

说索引原理之前，先说说索引存储结构。索引是在存储引擎中实现的，也就是不同的存储引擎，会使用不同索引。其中MyIsam和InnoDB只支持B+数索引，老刘先不讲B树和B+树的概念，大家自行搜索。

接下来就是索引的重点，搞清楚了非聚集索引和聚集索引，索引原理就差不多了！

### 非聚集索引(MyIsam)

它说的是B+树叶子节点只会存储数据行(数据文件)的指针，即数据和索引不在一起。它包含主键索引和辅助索引，都会存储指针的值。

**主键索引**

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-29/1611935873424-%E9%9D%9E%E8%81%9A%E9%9B%86%E4%B8%BB%E9%94%AE%E7%B4%A2%E5%BC%95.PNG)

MyIsam中B+树叶子节点存储的数据是数据的指针值，通过索引树找到对应的索引，然后通过索引中存储的记录指针，找到数据文件中对应的记录。

**辅助索引(次要索引)**

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1611936004883-%E9%9D%9E%E8%81%9A%E9%9B%86%E8%BE%85%E5%8A%A9%E7%B4%A2%E5%BC%95.PNG)

在MyIsam中，主索引和辅助索引在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key是可以重复的。

## 聚集索引(InnoDB)

- 主键索引(聚集索引)的叶子节点会存储数据行，也就是说数据和索引在一起。
- 辅助索引只会存储主键值。
- 如果没有主键，则使用唯一索引建立聚集索引；如果没有唯一索引，MySQL会按照一定规则创建聚集索引。

**主键索引**

在InnoDB中要求表必须有主键(MyIsam可以没有)，如果没有显示指定，则MySQL系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则MySQL自动为InnoDB表生成一个隐含字段作为主键类型为长整形。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1611936733196-%E8%81%9A%E9%9B%86%E4%B8%BB%E9%94%AE%E7%B4%A2%E5%BC%95.PNG)

上图是 InnoDB 主索引（同时也是数据文件）的示意图，可以看到叶节点包含了完整的数据记录，这种索引叫做聚集索引。因为 InnoDB 的数据文件本身要按主键聚集。  

**辅助索引**

InnoDB 的辅助索引 data 域存储相应记录主键的值而不是地址。换句话说，InnoDB 的所有辅助索引都引用主键作为 data 域。  

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1611937731440-%E8%81%9A%E9%9B%86%E8%BE%85%E5%8A%A9%E7%B4%A2%E5%BC%95.PNG)

聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录，即回表查询。

```
select * from user where name='Alice'
```

根据这段SQL语句，会进行回表查询，检索两次，才会获得记录。回表性能比较低，尽量做到不回表。

## 索引使用场景

介绍完索引的相关概念后，老刘必须给讲讲哪些场景下可以使用场景，大家记几个就行。

**哪些情况下需要使用索引**

1. 主键自动建立唯一索引
2. 频繁作为查询条件的字段应该创建索引
3. 多表关联查询中，关联字段应该创建索引
4. 查询中排序的字段应该创建索引
5. 频繁查询字段
6. 查询中统计或者分组字段应该创建索引

**哪些情况下不需要创建索引**

1. 表记录太少，没必要创建索引
2. 经常进行增删改的表
3. 频繁更新的字段
4. where条件里使用频率不高的字段

**为什么推荐多使用组合索引？**

为了节省mysql索引存储空间以及提升搜索性能，能使用组合索引就不使用单列索引。

使用组合索引需要遵循最左前缀原则，什么是最左前缀原则？

1. 前缀索引：where like a%

   通配符%在右边不在左边，什么是前缀索引呢？当索引是很长的字符序列时，这个索引会很慢，占用内存。如果以name为索引，当name对应的字符串很长时，就可以用前缀索引where like a%。

2. 从左到右都有索引，不能断，直到遇到范围查询<，>，between。

## 索引失效

我们进行数据查询很慢时，可能就会存在索引失效的情况。遇到这种情况不要怕，我们可以使用explain命令对select语句的执行计划进行分析。explain出来的信息有10列，分别是  

```
id、select_type、table、type、possible_keys、key、key_len、ref、rows、Extra
```

下面老刘就使用一个案例进行这些参数进行说明，大家可以跟着老刘一起练习。这10个参数老刘只讲重要的，其他的大家自行学习。

```
--用户表
create table tuser(
id int primary key,
loginname varchar(100),
name varchar(100),
age int,
sex char(1),
dep int,
address varchar(100)
);
--部门表
create table tdep(
id int primary key,
name varchar(100)
);
--地址表
create table taddr(
id int primary key,
addr varchar(100)
);
--创建普通索引
mysql> alter table tuser add index idx_dep(dep);
--创建唯一索引
mysql> alter table tuser add unique index idx_loginname(loginname);
--创建组合索引
mysql> alter table tuser add index idx_name_age_sex(name,age,sex);
--创建全文索引
mysql> alter table taddr add fulltext ft_addr(addr);
```

### **id**

每个SELECT语句都会自动分配的一个唯一标识符，表示查询中操作表的顺序，有四种情况：

- id相同：执行顺序由上到下
- id不同：如果是子查询，id号会自增，id越大，优先级越高。
- id相同的不同的同时存在
- id列为null的就表示这是一个结果集，不需要使用它来进行查询。  

### **select_type(重要)**

表示查询类型，主要用于区别普通查询、联合查询(union、union all)、子查询等复杂查询。  

**simple**，表示不需要union操作或者不包含子查询的简单select查询。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612015016867-%E5%8F%82%E6%95%B01.PNG)

**primary**，一个需要union操作或者含有子查询的select，位于最外层的单位查询的select_type即为primary，并且只有有一个  。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612015587893-%E5%8F%82%E6%95%B02.PNG)

先执行括号里面的sql语句，再执行外面的sql语句，内层的查询就是subquery。

**subquery**，除了from字句中包含的子查询外，其他地方出现的子查询都可能是subquery。

**dependent subquery**，表示这个subquery的查询要受到外部表查询的影响。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612015835581-%E5%8F%82%E6%95%B03.PNG)

**union**，它连接的两个select查询，第一个查询是PRIMARY，除了第一个表外，第二个以后的表select_type都是union。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612015930776-%E5%8F%82%E6%95%B04.PNG)

**dependent union**，它与union一样，出现在union 或union all语句中，但是这个查询要受到外部查询的影响。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612016018466-%E5%8F%82%E6%95%B05.PNG)

**union result**，它包含union的结果集，在union和union all语句中,因为它不需要参与查询，所以id字段为null。

**derived**，from字句中出现的子查询，也叫做派生表，其他数据库中可能叫做内联视图或嵌套select。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612016152338-%E5%8F%82%E6%95%B06.PNG)

可以理解为就是from字句后面出现子查询，取个别名，叫派生表。

### table

显示查询的表名，如果查询使用了别名，那么这里显示的是别名。

### type(重要)

它会显示很多参数类型，性能依次从好到坏显示为这样：

```
system，const，eq_ref，ref，fulltext，ref_or_null，unique_subquery，index_subquery，range，index_merge，index，ALL
```

除了all之外，其他的type都可以使用到索引，除了index_merge之外，其他的type只可以用到一个索引，优化器会选用最优索引一个，最少要索引使用到range级别。 老刘只讲这个重要的，有些内容也没搞清楚。

**system**

可遇不可求，表中只有一行数据或是空表。

**const(重要)**

使用唯一索引或主键，返回记录一定是1行记录的等值where条件。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612016789978-%E5%8F%82%E6%95%B07.PNG)

**eq_ref(重要)**  

一般是连接字段主键或者唯一性索引。

此类型通常出现在多表的 join 查询，表示对于前表的每一个结果，都只能匹配到后表的一行结果。并且查询的比较操作通常是 '='，查询效率较高。

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612016982071-%E5%8F%82%E6%95%B08.PNG)

**ref(重要)**

针对非唯一性索引，使用等值（=）查询非主键。或者是使用了最左前缀规则索引的查询。 

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612017319167-%E5%8F%82%E6%95%B09.PNG)

**range(重要)**

索引范围扫描，常见于使用>,<,is null,between ,in ,like等运算符的查询中。  

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612017434457-%E5%8F%82%E6%95%B010.PNG)

**index(重要)**

关键字：条件是出现在索引树中的节点的，可能没有完全匹配索引。

索引全表扫描，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取数据文件的查询、可以使用索引排序或者分组的查询。  

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612017561397-%E5%8F%82%E6%95%B011.PNG)

**all(重要)**

这个就是全表扫描数据文件，然后再在server层进行过滤返回符合要求的记录。  

![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612017620400-%E5%8F%82%E6%95%B012.PNG)

possible_keys、key、key_len、ref、rows就不讲了，直接讲最后一个extra。

### extra

这个列包含不适合在其他列中显示单十分重要的额外的信息，这个列可以显示的信息非常多，有几十种，这里写常见的几种。

**no tables used**

表示不带from字句的查询，使用not in()形式子查询或not exists运算符的连接查询，这种叫做反连接。一般连接查询是先查询内表，再查询外表，反连接就是先查询外表，再查询内表。  

**using filesort(重要)**

排序时无法使用到索引时，就会出现这个，常见于order by和group by语句中。

**using index(重要)**

查询时不需要回表查询，直接通过索引就可以获取查询的数据。

**using where(重要)**

通常type类型为all，记录并不是所有的都满足查询条件，通常有where条件，并且一般没索引或者索引失效。

讲完分析索引的参数后，现在老刘讲一些索引失效的情况，大家一定要用心记住，老刘也记了好几遍！

## 索引失效分析

1. 一般SQL语句查询采用全值匹配，资料上叫全值匹配我最爱。

   ![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612018355962-%E5%8F%82%E6%95%B013.PNG)

2. 最左前缀法则，对于组合索引而言，查询从索引的最左前列开始，并且不能跳过索引中的列不然就会失效。

   现在举一个带头的索引断（带头索引生效，其他索引失效）的例子：  

   ![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612018587378-%E5%8F%82%E6%95%B014.PNG)

3. 不要在索引上做计算，例如计算、函数、自动/手动类型转换，不然会导致索引失效而转向全表扫描。

   ![](https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612020530036-%E5%8F%82%E6%95%B015.PNG)

4. 范围条件右边的列失效，就是不能继续使用索引中范围条件（bettween、<、>、in等）右边的列。

5. 尽量使用覆盖索引（只查询索引的列），也就是索引列和查询列一致，减少select *。

6. 索引字段上不要使用不等，索引字段上使用（！= 或者 < >）判断时，会导致索引失效而转向全表扫描。

7. 主键索引字段上不可以判断null，索引字段上使用 is null 判断时，可使用索引。

8. 索引字段使用like以通配符开头（‘%字符串’）时，会导致索引失效而转向全表扫描。like要以通配符结束相当于范围查找，索引不会失效。

9. 索引字段是字符串时，要加单引号，否则会导致索引失效而转向全表扫描。

10. 索引字段不要使用or，否则会导致索引失效而转向全表扫描。

## 总结

这篇内容大家一定要跟着老刘练习，光看不练等于白学！尽管当前水平可能不及各位大佬，但老刘会努力变得更加优秀，让各位小伙伴自学从此不求人！

如果有相关问题，联系公众号：努力的老刘。文章都看到这了，点赞关注支持一波！

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-30/1612020883612-%E7%82%B9%E8%B5%9E.jpg" style="zoom:80%;">
</a>
</p>