# Explain的基本概念

查询计划的结构式计划节点树。

- 叶子节点是扫描节点，对于select有如下三种扫描节点

  - seq scan 顺序扫描
  - index scan 索引扫描
  - bitmap index scan 位图索引扫描

  对于如values之类的语句，还有其它扫描来源

- 对扫描出来的结果进行连接查询、聚集统计等操作则是由扫描节点上方的其他类型节点完成。

举一个如下的例子

```bash
hulattp=> explain select * from notification;
                           QUERY PLAN
----------------------------------------------------------------
 Seq Scan on notification  (cost=0.00..4.61 rows=161 width=105)
(1 row)
```

从左到右依次解释

- cost=0.00... 估计的启动成本
- cost=0.00...4.61 估计该步扫描的总成本，这是检索了所有可用记录的情况下评估的。但该成本可能被父节点提前中止，比如使用limit子句
- rows 该节点扫描后语句输出的行数
- width 该节点输出的结果平均每行的大小，以byte为单位

需要注意的是这里的成本单位并不一定是绝对的，依靠各种成本参数决定。其中包含了扫描磁盘的成本、CPU计算的成本等。上面同样的语句，加上条件后，扫描输出结果变少了，但扫描的总成本却增加了。这是因为此时针对每条记录都需要CPU执行比较操作，因此扫描成本有所增加。

```bash
hulattp=> explain select * from notification where id < 100;
                           QUERY PLAN
----------------------------------------------------------------
 Seq Scan on notification  (cost=0.00..5.01 rows=100 width=105)
   Filter: (id < 100)
```

另外，一个固定的查询语句如果加上一点其他条件的查询计划可能会发生巨大的变化，我没有复现，但从资料上给的可以看出一些端倪

```bash
# 这种情况下使用了bitmap index scan然后合并，最后进行输出。
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;

                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=25.08..60.21 rows=10 width=244)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   ->  BitmapAnd  (cost=25.08..25.08 rows=10 width=0)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0)
               Index Cond: (unique1 < 100)
         ->  Bitmap Index Scan on tenk1_unique2  (cost=0.00..19.78 rows=999 width=0)
               Index Cond: (unique2 > 9000)

# 加上limit后，转而直接使用了索引
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000 LIMIT 2;

                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Limit  (cost=0.29..14.48 rows=2 width=244)
   ->  Index Scan using tenk1_unique2 on tenk1  (cost=0.29..71.27 rows=10 width=244)
         Index Cond: (unique2 > 9000)
         Filter: (unique1 < 100)
```

### Nest loop

```bash
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;

                                      QUERY PLAN
--------------------------------------------------------------------------------------
 Nested Loop  (cost=4.65..118.62 rows=10 width=488)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.47 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.91 rows=1 width=244)
         Index Cond: (unique2 = t1.unique2)
```



## Explain的图形化

pgAdmin有比较形象地图形化分析，能够用比较形象的图标标识出进行了哪些索引

比如表达先bitmap_index扫描再bitmap_heap抽取数据，图表显示为

![image-20200627162957533](C:\Users\floyd\AppData\Roaming\Typora\typora-user-images\image-20200627162957533.png)

而如下的复杂SQL图形化后则明显好了很多

```sql
select notification.* from notification
left join content on notification."target_content_id" = content.id
left join comments on notification."target_comment_id" = comments.id
where content.state = 'normal' and comments.state = 'normal'
order by notification.update_time desc, notification.id desc;
```

![image-20200627163135439](C:\Users\floyd\AppData\Roaming\Typora\typora-user-images\image-20200627163135439.png)

# PostgreSQL数据库内核分析

## 看书计划

- 第五章 查询编译 187 - 282
- 第六章 查询执行 282 - 343
- 第七章 事务处理和并发控制（优先级在查询优化器之后） 343

# PostgreSQL 技术内幕 - 查询优化器深度探索

在写完本片博文后，需要的时候再看。因为它太长了。

## 问题积累

- bitmap index scan和index scan有什么区别

- 数据少了不用索引，那数据要多到什么程度才会使用索引呢？或者，我们是否能够强制使用索引以预判数据量较大时的索引使用情况呢？

- Plan Hint是什么功能：通过注释的方式手动干预SQL计划器生成的执行计划，上面是自动计划的，下面是手动要求顺序扫描干预后的。

  ```bash
  postgres=> explain select * from t1 where a = 1;
                             QUERY PLAN
  -----------------------------------------------------------------
   Index Scan using t1_i_a on t1  (cost=0.28..8.29 rows=1 width=8)
     Index Cond: (a = 1)
  (2 rows)
  
  postgres=> /*+ SeqScan(t1) */ explain select * from t1 where a = 1;
                      QUERY PLAN
  ---------------------------------------------------
   Seq Scan on t1  (cost=0.00..17.50 rows=1 width=8)
     Filter: (a = 1)
  (2 rows)
  ```

- 倒排索引是怎么回事

  

## 参考资料
1. http://www.postgres.cn/docs/9.5/using-explain.html
2. http://www.postgresonline.com/journal/archives/78-Why-is-my-index-not-being-used.html
3. https://docs.postgresql.tw/server-administration/server-configuration/query-planning#19-7-2-planner-cost-constants
4. http://mysql.taobao.org/monthly/2018/11/06/
5. http://www.postgresonline.com/journal/archives/27-Reading-PgAdmin-Graphical-Explain-Plans.html
6. 

PostgreSQL指南：内幕探索

PostgreSQL 数据库内核分析

PostgreSQL 技术内幕(查询优化器深度探索)



## 可作为概述

无论是翻看使用手册、还是看别人写的博客，读完之后只是给了我们一个答题的印象，大道理懂了，但还是似懂非懂，很是让人头疼。总结起来，大体是大部分文章要么没有介绍是什么、要么缺少怎么用。是什么这一点，比如index scan和bitmap index scan的区别，在执行上有什么不同，比如官方文档(http://www.postgres.cn/docs/9.5/planner-stats.html)。怎么用这一点，倒不是说不介绍，而是总给一个比较简单的例子，使得实际使用时不知道如何参照，如PgSQL最佳实践(http://mysql.taobao.org/monthly/2018/11/06/)，这篇文章，我不知道光看这篇文章能够得到什么，虎头蛇尾，仅介绍了概念。