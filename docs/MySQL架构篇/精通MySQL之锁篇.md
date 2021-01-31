# 精通MySQL之锁篇

> - 老刘是即将找工作的研究生，自学大数据开发，一路走来，感慨颇深，网上大数据的资料良莠不齐，于是想写一份详细的大数据开发指南。这份指南把大数据的【基础知识】【框架分析】【源码理解】都用自己的话描述出来，让伙伴自学从此不求人。
> - 您的点赞是我持续更新的动力，禁止白嫖，看了就要有收获，一起加油。

今天给大家分享的是大数据开发基础部分MySQL的锁，锁在MySQL知识点中属于比较重要的部分，大家一定要好好体会老刘的话，MySQL锁篇的大纲如下：

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612080258482-%E9%94%81%E6%80%9D%E7%BB%B4%E5%AF%BC%E5%9B%BE.PNG" style="zoom: 66%;">
</a>
</p>

看完老刘这篇内容后，希望你们能够掌握以下内容：

1. MySQL的锁分类
2. 表级锁中表锁、元数据锁的原理
3. 行锁的原理、记录锁和间隙锁的使用区别、死锁的原理和死锁场景  

## MySQL锁介绍

**为什么有锁？**

多个程序对MySQL表或记录进行访问，就会产生竞态条件，为了解决这个问题，就提出了锁。

MySQL中的锁如下图：

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612081367994-%E9%94%81%E4%BB%8B%E7%BB%8D.PNG" style="zoom: 66%;">
</a>
</p>

## MySQL表级锁

表级锁是由MySQL SQL layer层实现，只要锁了一张表，只能对这张表操作。它有两种：一是表锁、二是元数据锁。

### 表锁

表锁有两种表现形式：一是表共享读锁（Table Read Lock）、二是表独占写锁（Table Write Lock）。

我们采用手动增加表锁，采用如下SQL语句：

```
lock table 表名称 read(write),表名称2 read(write)，其他;
```

删除表锁，采用如下SQL语句：

```
unlock tables;
```

查看表锁情况：

```
show open tables;
```

光说不练假把式，老刘用一个案例给各位伙伴好好讲讲表锁，大家跟着一起练。

### 表锁演示

```
--新建表
CREATE TABLE mylock (
id int(11) NOT NULL AUTO_INCREMENT,
NAME varchar(20) DEFAULT NULL,
PRIMARY KEY (id)
);
INSERT INTO mylock (id,NAME) VALUES (1, 'a');
INSERT INTO mylock (id,NAME) VALUES (2, 'b');
INSERT INTO mylock (id,NAME) VALUES (3, 'c');
INSERT INTO mylock (id,NAME) VALUES (4, 'd');
```

演示表读锁

1、我们先给表mylock加表读锁

```
session1: lock table mylock read;
```

2、加完表读锁后，session1还是能对该表进行查询

```
session1: select * from mylock;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612082917040-%E5%8F%82%E6%95%B01.PNG" style="zoom:100%;">
</a>
</p>

3、只要锁了这张表，就只能先对这张表操作，不能访问别的表，直到这张表被释放。我用session1访问表tdep就不能进行访问，如图：

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612083181704-%E5%8F%82%E6%95%B02.PNG" style="zoom:100%;">
</a>
</p>

4、虽然对这张表加锁了，但我们别的session可以对它进行访问。

```
session2：select * from mylock;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612088548329-%E5%8F%82%E6%95%B03.PNG">
</a>
</p>

5、但如果session2要对mylock表进行修改，那就不行了，表读锁不允许加写锁，只允许加读锁，这也叫表共享读。

```
session2：update mylock set name='x' where id=2;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612088692382-%E5%8F%82%E6%95%B04.PNG">
</a>
</p>

6、这个时候session1释放，session2才能进行修改。

```
session1：unlock tables;
session2：Rows matched: 1 Changed: 1 Warnings: 0 -- 修改执行完成
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612088802626-%E5%8F%82%E6%95%B05.PNG">
</a>
</p>

表读锁的内容就演示完了，现在开始演示表写锁的内容。

1、先给mylock加写锁

```
session1: lock table mylock write;
```

2、session1可以访问mylock表，但不能访问其他表。

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612088988765-%E5%8F%82%E6%95%B06.PNG">
</a>
</p>

3、session1也可以对该表进行修改，但是session2对该表进行读取和修改就不行。

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612089365669-%E5%8F%82%E6%95%B07.PNG">
</a>
</p>

4、当session1释放表写锁后，session2才能获取。

```
session1：unlock tables;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612089485679-%E5%8F%82%E6%95%B08.PNG">
</a>
</p>

```
session2：4 rows in set (41.65 sec)
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612089578984-%E5%8F%82%E6%95%B09.PNG">
</a>
</p>

通过这个例子，大家要明白读锁就是表共享读锁，自己可以用，其他人也可以访问；写锁就是表独占写锁，意思就是只有自己能够访问，
别的人不能访问。

### 元数据锁

什么是元数据锁？

元数据，英语缩写是MDL，在 MySQL 5.5 版本中引入了 MDL，当对一个表做增删改查操作的时候，自动加 MDL 读锁；当要对表做结
构变更操作的时候，自动加 MDL 写锁。  

**注意：这个锁是自动提交的，要先开始事务，然后进行增删改查时，会自动加MDL读锁。**

现在开始元数据锁的演示，大家跟着老刘一起练习。

1、session1开启事务，给表自动加读锁。

```
session1: begin;
          select * from mylock;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612090464652-%E5%8F%82%E6%95%B010.PNG">
</a>
</p>

2、session2对该表进行修改时就会造成阻塞。

```
session2: alter table mylock add f int;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612090564496-%E5%8F%82%E6%95%B011.PNG">
</a>
</p>

3、session1提交事务 或者 rollback 释放读锁 。

4、session2修改完成。

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612090640128-%E5%8F%82%E6%95%B012.PNG">
</a>
</p>

## MySQL行级锁

MySQL的行级锁是由存储引擎实现的，利用存储引擎锁住索引项来实现，主要讲InnoDB行级锁。

InnoDB行级锁，按锁范围分为三种：

- 记录锁：锁定索引中的一条记录。
- 间隙锁：锁的是缝，要么锁住索引记录中间的值，要么锁住第一个索引记录前面的值或者最后一个索引记录后面的值。
- Next-key locks：记录锁和间隙锁的组合(可以不看)

InnoDB行级锁，按功能分：

- 共享读锁(S)：手动添加，允许一个事务去读一行，其他事务可以读数据，但不能修改数据。

  ```
  SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE -- 共享读锁 手动添加
  ```

- 排他写锁(X)：自动添加，指的是一个事务在一行数据加上排他锁后，**其他事务不能再在其上加其他的锁**。

InnoDB也实现了表级锁，也就是意向锁，意向锁是mysql内部使用的，不需要用户干预。  

### 两阶段锁(2PL)

两阶段锁讲的是锁操作分为两个阶段：加锁阶段和解锁阶段。

加锁阶段：只加锁，不放锁。

解锁阶段：只放锁，不加锁。

### 行锁演示

InnoDB行锁是通过给索引上的索引项加锁来实现的，因此只有通过索引条件检索的数据，InnoDB才会使用行级锁，否则，InnoDB将使用表锁！  

**行读锁**

1、我们利用session1给id=1的行加读锁，使用索引。

```
session1: begin;
          select * from mylock where ID=1 lock in share mode; 
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612094009379-%E5%8F%82%E6%95%B013.PNG">
</a>
</p>

2、由于行锁锁定的是行，所以利用session2修改别的行例如id=2是可以的，修改id=1就不行了。

```
session2：update mylock set name='M' where id=2;
session2：update mylock set name='M' where id=1;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612094042569-%E5%8F%82%E6%95%B014.PNG">
</a>
</p>

3、当session1提交后，session2才可以修改。

```
session1: commit; 
session2：update mylock set name='M' where id=1; 
Query OK, 1 row affected (0.00 sec)
Rows matched: 1 Changed: 1 Warnings: 0
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612094213633-%E5%8F%82%E6%95%B016.PNG">
</a>
</p>

关于行锁，我们要注意使用索引加行锁 ，未锁定的行可以访问。

**行读锁升级为表锁**

mylock表中id加了索引，name没有加索引，当我们对name加行读锁时，就会出现行读锁升级为表锁。

1、session1开启事务

```
session1: begin;
```

2、手动加name='c'的行读锁,未使用索引

```
select * from mylock where name='c' lock in share mode;
```

3、session2修改阻塞，未用索引行锁升级为表锁

```
update mylock set name='N' where id=2;
```

4、session1提交事务或者 rollback 释放读锁

```
commit;
```

5、session2就会修改成功

```
update mylock set name='N' where id=2;
```

**行写锁**

1、session1开启事务并且手动加id=1的行写锁。

```
session1: begin;
          select * from mylock where id=1 for update;
```

**2、这里有一个特别重要的知识点，很多人会弄错！**

**排他锁锁住一行数据后，其他事务就不能读取和修改该行数据，其实不是这样的！**

排他锁指的是一个事务在一行数据加上排他锁后，其他事务不能再在其上加其他的锁。MySQL InnoDB引擎默认的修改数据语句：update，delete，insert都会自动给数据加上排他锁，**select语句默认不会加任何锁类型**，如果加排他锁可以使用select …for update语句，加共享锁可以使用select … lock in share mode语句。所以其他事务是不能修改加过排他锁的数据行，其他事务也不能通过for update和lock in share mode锁的方式查询数据，**但可以直接通过select …from…查询数据，因为普通查询没有任何锁机制。**

所以session2可以访问id=1的数据行。

```
session2: select * from mylock where id=1 ;
```

3、但是不能给它继续加锁

```
session2: select * from mylock where id=1 lock in share mode ;
```

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612096900830-%E5%8F%82%E6%95%B017.PNG">
</a>
</p>

4、session1提交事务或者rollback释放写锁，session2才会执行成功

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612096983192-%E5%8F%82%E6%95%B018.PNG">
</a>
</p>

### 间隙锁

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612097231518-%E9%97%B4%E9%9A%99%E9%94%811.PNG">
</a>
</p>

根据检索条件向左寻找最靠近检索条件的记录值A，作为左区间，向右寻找最靠近检索条件的记录值B作为右区间，即锁定的间隙为（A，B）。根据图中where number=5的话，那么间隙锁的区间范围为（4,11）；

间隙锁防止两种情况

1. 防止插入间隙内的数据
2. 防止已有数据更新为间隙内的数据 

间隙情况：

1. id、number均在间隙内
2. id、number均在间隙外
3. id在间隙内、number在间隙外
4. id在间隙外，number在间隙内
5. id、number为边缘数据  

**非唯一索引等值**  

```
update news set number=3 where number=4;
```

检索条件number=4,向左取得最靠近的值2作为左区间，向右取得最靠近的5作为右区间，因此，session 1的间隙锁的范围（2，4），（4，5），即记录（id=1,number=2）和记录（id=3,number=4）之间间隙会被锁定，记录（id=3,number=4）和记录（id=6,number=5）之间间隙被锁定。

当我们添加数据时，结果如下：

```
insert into news value(2,3);
均在间隙内，阻塞
insert into news value(7,8);
均在间隙外，成功
insert into news value(2,8);
id在间隙内，number在间隙外，成功
insert into news value(4,8);
id在间隙内，number在间隙外，成功
insert into news value(7,3);
id在间隙外，number在间隙内，阻塞
insert into news value(7,2);
id在间隙外，number为上边缘数据，阻塞
insert into news value(2,2);
id在间隙内，number为上边缘数据，阻塞
insert into news value(7,5);
id在间隙外，number为下边缘数据，成功
insert into news value(4,5);
id在间隙内，number为下边缘数据，阻塞
```

我们可以得到只要number（where后面的）在间隙里（2 3 4），不包含最后一个数（5）则不管id是多少都会阻塞。 如果是下边缘数据需要看id是否在间隙内。 

**主键索引范围**  

由于主键不能重复，所以id无边缘数据。

```
update news set number=3 where id>1 and id <6;
```

```
insert into news value(2,3);
均在间隙内，阻塞
insert into news value(7,8);
均在间隙外，成功
insert into news value(2,8);
id在间隙内，number在间隙外，阻塞
insert into news value(4,8);
id在间隙内，number在间隙外，阻塞
insert into news value(7,3);
id在间隙外，number在间隙内，成功
```

我们可以得到只要id（在where后面的）在间隙里(2 4 5)，则不管number是多少都会阻塞。  

**非唯一索引无穷大**  

```
update news set number=3 where number=13 ;
```

```
insert into news value(11,5);
执行成功
insert into news value(12,11);
执行成功
insert into news value(14,11);
阻塞
insert into news value(15,12);
阻塞
```

检索条件number=13,向左取得最靠近的值11作为左区间，向右由于没有记录因此取得无穷大作为右区间，因此，session 1的间隙锁的范围（11，无穷大），当id和number同时满足 ，才会阻塞。

### 死锁  

两个 session 互相等等待对方的资源释放之后，才能释放自己的资源,造成了死锁，主要是顺序出现问题。

我们session1先给id=1加锁，session2再给id=2加锁，此时session1想再给id=2加锁，但session2已经给它加锁了，就会造成死锁。

如何解决死锁？

MySQL默认会主动探知死锁，并回滚某一个影响最小的事务，等待另一个事务执行完成之后，再重新执行该事务。

## 总结

本文作为大数据开发指南MySQL的第三篇详细介绍了MySQL锁的内容，希望大家能够跟着老刘的文章，好好捋捋思路，争取能够用自己的话把这些知识点讲述出来！

尽管当前水平可能不及各位大佬，但老刘会努力变得更加优秀，让各位小伙伴自学从此不求人！

> 大数据开发指南地址如下：
>
> - github：https://github.com/BigDataLaoLiu/BigDataGuide
> - 码云：https://gitee.com/BigDataLiu/BigDataGuide
>
> 如果有相关问题，联系公众号：努力的老刘。文章都看到这了，点赞关注支持一波！

<p align="center">
<a>
   <img src="https://gitee.com/BigDataLiu/big-data-map/raw/master/2021-1-31/1612100626519-%E7%82%B9%E8%B5%9E.jpg">
</a>
</p>