# PostgreSQL - 看懂explain

> 对SQL优化，最直接的方式就是explain。但对explain的结果，无论是官方手册，还是别人写的博客，看完后点头如捣蒜，但对很多问题还是两眼一抹黑，说到底，就是不了解SQL优化器是如何执行SQL计划的。比如，我们并不能很好地回答以下问题
>
> - index scan、index only scan、bitmap index scan 有啥区别？
> - 为什么明明有建立索引，但PG就是不用呢？
> - 执行计划中有子查询就一定不好吗？
> - hash join、merge join、nestloop join有啥区别？
> - 。。。。。。
>
> 本文的目的，是让我们看懂SQL执行计划。毕竟，看懂了，才谈得优化。

## SQL优化器原理

一条SQL从输入到执行完毕，大致会经历如下步骤

1. 语法分析：词法分析、语法分析、语义分析
2. **查询优化：基于规则的优化、基于代价的优化**
3. 查询执行、数据存取

可以看到，语法分析的结果会经过PG进行优化，最后才是执行查询。explain看到的内容，就是查询优化的结果。而查询优化又被分为两个步骤

- 基于规则的优化：即逻辑优化，通过对关系代数表达式进行逻辑上的等价变换，以获得性能更好的结果。

- 基于代价的优化：即物理优化，逻辑优化后，实际的查询路径有很多种，PG建立了代价计算模型，计算所有可能路径的代价，选出最优路径。

  比如`select * from users where id = 1`，在扫描方式上，有顺序扫描、索引扫描等。在不同的数据量情况下，按顺序扫描和索引扫描的代价可能是不一样的，因此他的执行计划可能会随数据量的变化而变化。

### 逻辑优化

逻辑优化将语法分析生成的查询树进行逻辑等价变换，具体由如下几点。

- 子查询提升

  **相关子查询**：子查询中引用外层表的列属性，导致外层表的每一条记录，子查询都需要重新执行一次

  **非相关子查询**：子查询是独立的，与外层表没有直接的关联，子查询单独执行一次，外层表可以重复利用其结果

  通常来说，相关子查询是有必要提升的，非相关子查询由于其本来就只执行一次，因此没有太大必要提升。

  ```sql
  -- 相关子查询举例
  explain select *, (select label_id from content_to_label where content_id = content.id) as label_id from content;
  
  Seq Scan on content  (cost=0.00..3694.61 rows=1151 width=743)
    SubPlan 1
      ->  Seq Scan on content_to_label  (cost=0.00..3.10 rows=3 width=4)
            Filter: (content_id = content.id)
  
  -- 非相关子查询举例
  explain select *, (select label_id from content_to_label limit 1) as label_id from content;
  
  Seq Scan on content  (cost=0.02..126.53 rows=1151 width=743)
    InitPlan 1 (returns $0)
      ->  Limit  (cost=0.00..0.02 rows=1 width=4)
            ->  Seq Scan on content_to_label  (cost=0.00..2.68 rows=168 width=4)
  ```

  相关子查询又可以依据子查询出现的位置分为如下两种

  **子查询语句**：出现在FROM关键字后的是子查询语句

  **子连接语句**：出现在WHERE/ON等约束条件或SELECT子句中的是子连接语句

  **提升子连接**

  子连接是子查询的一种特殊情况，由于它常出现在条件中，因此通常伴随ANY、EXISTS、NOT EXISTS、IN、NOT IN等关键字，PG会对他们尝试做提升，比如如下exists子查询优化结果

  ```sql
  -- 查询那些打过标签的文章
  explain select * from content where exists(select 1 from content_to_label where content_id = content.id);
  -- 在不优化的情况下，exists子查询会像上面所示，真的是子查询。但这里优化器将子查询做了提升，提升后变成连接，通过将内表hash化，降低算法复杂度
  Hash Join  (cost=4.43..134.62 rows=59 width=739)
    Hash Cond: (content.id = content_to_label.content_id)
    ->  Seq Scan on content  (cost=0.00..126.51 rows=1151 width=739)
    ->  Hash  (cost=3.69..3.69 rows=59 width=4)
          ->  HashAggregate  (cost=3.10..3.69 rows=59 width=4)
                Group Key: content_to_label.content_id
                ->  Seq Scan on content_to_label  (cost=0.00..2.68 rows=168 width=4)
  ```

  能够被提升还有一个前提条件是子查询必须足够简单，上面同样的SQL，子查询投影改成聚集函数，就无法提升

  ```sql
  explain select * from content where exists(select sum(content_id) from content_to_label where content_id = content.id);
  
  Seq Scan on content  (cost=0.00..3714.75 rows=576 width=739)
    Filter: (SubPlan 1)
    SubPlan 1
      ->  Aggregate  (cost=3.11..3.12 rows=1 width=8)
            ->  Seq Scan on content_to_label  (cost=0.00..3.10 rows=3 width=4)
                  Filter: (content_id = content.id)
  ```

  **提升子查询**

  出现在本来在表位置的子查询，也能够提升，如下

  ```sql
  explain select * from content left join (select *, 1 from content_to_label) ctl on content.id = ctl.content_id;
  
  Hash Right Join  (cost=140.90..144.02 rows=1151 width=759)
    Hash Cond: (content_to_label.content_id = content.id)
    ->  Seq Scan on content_to_label  (cost=0.00..2.68 rows=168 width=20)
    ->  Hash  (cost=126.51..126.51 rows=1151 width=739)
          ->  Seq Scan on content  (cost=0.00..126.51 rows=1151 width=739)
  ```

- 预处理表达式
  - 常量简化
  
    即直接计算出SQL中的常量表达式
  
    ```sql
    -- 可以直接计算出101
    explain select * from content where id < 1 + 100
    
    Bitmap Heap Scan on content  (cost=5.08..123.47 rows=104 width=739)
      Recheck Cond: (id < 101)
      ->  Bitmap Index Scan on content_pkey  (cost=0.00..5.06 rows=104 width=0)
            Index Cond: (id < 101)
    ```
  
  - 谓词规范化（规约、拉平、提取公共项）
  
    对无用的约束条件去除
  
    ```sql
    -- false对于或运算是没用的，会被优化器直接去除
    explain select * from content where id < 100 or false
    
    Bitmap Heap Scan on content  (cost=5.08..123.45 rows=103 width=739)
      Recheck Cond: (id < 100)
      ->  Bitmap Index Scan on content_pkey  (cost=0.00..5.05 rows=103 width=0)
            Index Cond: (id < 100)
    ```
  
    约束条件会被拉平
  
    ```sql
    -- 约束条件进行了无谓的括号，会被拉平
    explain select * from content where id < 100 or (id < 1000 or id > 2000)
    
    Seq Scan on content  (cost=0.00..135.14 rows=978 width=739)
      Filter: ((id < 100) OR (id < 1000) OR (id > 2000))
    ```
  
    约束条件经过逻辑运算后，会被提取公共项
  
    ```sql
    -- 约束条件提取公共项，只剩下id > 1 and id < 2
    explain select * from content where (id > 1 and id < 2 and id < 100) or (id > 1 and id < 2);
    
    Index Scan using content_pkey on content  (cost=0.28..8.30 rows=1 width=739)
      Index Cond: ((id > 1) AND (id < 2))
    ```
  
- 处理HAVING子句

  HAVING子句的优化主要是将部分条件转变为普通的过滤条件，从而减少原始数据的大小。

  ```sql
  -- 统计发过发过10篇以上内容且用户id>10的用户
  explain select "authorId" from content group by "authorId" having count(1) > 10 and "authorId" > 100;
  
  HashAggregate  (cost=132.85..133.50 rows=65 width=4)
    Group Key: "authorId"
    Filter: (count(1) > 10)
    ->  Seq Scan on content  (cost=0.00..129.39 rows=692 width=4)
          Filter: ("authorId" > 100)
  ```

- GroupBy键值消除

  GroupBy子句需要借助排序或哈希实现，如果能减少它后面的字段，就能降低损耗。典型的是如果group by中出现了一个主键和多个不相关的字段，则仅保留主键即可，因为主键唯一不可重复，没有必要再对其他字段进行排序或hash操作了

  ```sql
  explain select * from content group by id, type, "authorId";
  
  HashAggregate  (cost=129.39..140.90 rows=1151 width=739)
    Group Key: id
    ->  Seq Scan on content  (cost=0.00..126.51 rows=1151 width=739)
  ```

- 外连接消除

  如果两个表是内连接，则他们之间的顺序可以任意交换，会方便谓词下推。而对于外连接，则不会那么方便。如果能够将外连接转换为内连接，则查询过程会简化。能够被转换为内连接的情况如下

  ```sql
  -- 常规左外连接
  explain select * from content left outer join content_to_label ctl on content.id = ctl.content_id;
  
  Hash Right Join  (cost=140.90..144.02 rows=1151 width=755)
    Hash Cond: (ctl.content_id = content.id)
    ->  Seq Scan on content_to_label ctl  (cost=0.00..2.68 rows=168 width=16)
    ->  Hash  (cost=126.51..126.51 rows=1151 width=739)
          ->  Seq Scan on content  (cost=0.00..126.51 rows=1151 width=739)
  
  -- 做左外连接，条件上限制可空侧的表格，消除外连接
  explain select * from content left outer join content_to_label ctl on content.id = ctl.content_id where ctl.content_id is not null;
  
  Hash Join  (cost=140.90..144.02 rows=168 width=755)
    Hash Cond: (ctl.content_id = content.id)
    ->  Seq Scan on content_to_label ctl  (cost=0.00..2.68 rows=168 width=16)
          Filter: (content_id IS NOT NULL)
    ->  Hash  (cost=126.51..126.51 rows=1151 width=739)
          ->  Seq Scan on content  (cost=0.00..126.51 rows=1151 width=739)
  ```

- 谓词下推

  

- 消除无用连接

  

### 物理优化

- cost计算模型

## 查询树介绍

SQL优化器最终输出计划树，执行器依照该树从低向上执行。

所有节点中，最重要的是扫描节点和连接节点。

### 扫描

- Seq Scan
- 索引扫描
  - Index Scan
  - Index Only Scan
  - BitmapIndex Scan
- Bitmap Scan
  - BitmapHeap Scan
  - BitmapIndex Scan
- TID Scan

### 连接

- Nestlooped Join
- Hash Join
- Merge Join
- Semi Join
- Anti-Semi Join

### 其它

- Materialize

- CTE

  通用表达式对应的是WITH语句，它的作用和子查询类似，但具有一次求值，多次使用的特点，并不会多次执行，因此一般不会被优化。
  
- Param

  在部分子查询的计划树中我们会看到`returns $0`这样的内容。其原理是：子查询每次执行结果都会返回并存储在一个Param中，父查询通过向Param传参启动子查询，如果发现之前同样的参数已经执行过了，则直接获取之前执行获得的结果，从而节省运行时间。

  ```sql
  Seq Scan on content  (cost=0.02..126.53 rows=1151 width=743)
    InitPlan 1 (returns $0)
      ->  Limit  (cost=0.00..0.02 rows=1 width=4)
            ->  Seq Scan on content_to_label  (cost=0.00..2.68 rows=168 width=4)
  ```

  

## Explain怎么看 - 直接看





## Explain怎么看 - 可视化



## 总结



## 参考资料

1. [Using explain - PostgreSQL](https://www.postgresql.org/docs/10/using-explain.html)
2. [Why my index not being used](http://www.postgresonline.com/journal/archives/78-Why-is-my-index-not-being-used.html)
3. [PgSQL · 最佳实践 · EXPLAIN 使用浅析](http://mysql.taobao.org/monthly/2018/11/06/)
4. [使用PgAmin查看explain结果](http://www.postgresonline.com/journal/archives/27-Reading-PgAdmin-Graphical-Explain-Plans.html)
5. 《数据库系统概念 - 第六版》（参考连接部分）
6. 《PostgreSQL技术内幕——查询优化深度探索》
7. 《PostgreSQL指南——内幕探索》