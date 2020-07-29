# PostgreSQL - 看懂explain

> 对SQL优化，最直接的方式就是explain。但对explain的结果，无论是官方手册，还是别人写的博客，看完后点头如捣蒜，但对很多问题还是两眼一抹黑，说到底，就是不了解SQL优化器是如何执行SQL计划的。比如，我们并不能很好地回答以下问题
>
> - index scan、index only scan、bitmap index scan 有啥区别？
> - 为什么明明有建立索引，但PG就是不用呢？
> - 执行计划中有子查询就一定不好吗？
> - hash join、merge join、nestloop join有啥区别？
> - 。。。。。。
>
> 本文的目的，是让读者看懂SQL执行计划。毕竟，看懂了，才谈得上去优化它。

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
- UNION ALL优化
- 预处理表达式
  - 常量简化
  - 谓词规范（规约、拉平、提取公共项）
- 处理HAVING子句
- GroupBy键值消除
- 外连接消除
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