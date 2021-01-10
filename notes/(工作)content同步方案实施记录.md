# Content同步方案实施记录

## 方案

1. 创建呼啦宝贝->呼啦亲子content.id对应关系，用作更新或插入的参考
2. 首次同步一批数据
3. 以后每日定期同步上一天的数据

## 知识备忘

这是本文档的重点，主要记录pandas和sqlachedemy的使用方式，以免下次忘记

### pandas

官方手册：https://www.pypandas.cn/docs/reference.html

入门视频：https://www.bilibili.com/video/BV1UJ411A7Fs

核心概念：Series和DataFrame

- 读取数据库得到DataFrame

  ```python
  origin_engine = create_engine('postgresql+psycopg2://pm:ePtXwIL7Xz624bn9@pgr-wz9sl5h20qh9v67d9o.pg.rds.aliyuncs.com:1921/hulattp')
  # to_be_sync数据类型为DataFrame
  to_be_sync = pd.read_sql(""" 
          select * from content where type = 'post' and updated >= '2020-12-22 00:00:00' and updated <= '2021-01-04 23:59:59' 
      """, con=origin_engine)
  ```

- 将DataFrame整个写入数据库（注意不是更新，只适合全新的写入）

  ```python
  author_to_be_sync.to_sql('author', target_engine, if_exists='append')
  ```

- 从DataFrame获取一列，对该列的每个值转换为字符串类型，并去重，然后转换为python原生列表

  ```python
  to_be_sync['authorId'].astype(str).unique().tolist()
  ```

- 求两个DataFrame的差集，前者减去后者

  ```python
  pd.concat([baobei_author, qinzi_author, baobei_author]).drop_duplicates(subset=['id'], keep=False)
  ```

- 获取DataFrame的行数

  ```python
  author_to_be_sync.shape[0]
  ```

- 通过loc获取DataFrame的子集

  ```python
  # 获取to_be_sync中，id出现在content_sync_id_map的baobei_content_id字段的所有行
  to_be_sync.loc[to_be_sync['id'].isin(content_sync_id_map['baobei_content_id'].values)]
  # 与上面相反
  to_be_sync.loc[~to_be_sync['id'].isin(content_sync_id_map['baobei_content_id'].values)]
  ```

- 其它

  ```python
  # 删除某一列
  del baobei_contents['id']
  # 两个DataFrame做左连接
  content_sync_id_map = pd.merge(baobei_contents, qinzi_contents, how='left', on=['created'])
  # 转换为列表
  series.tolist()
  # 转换诶字典
  df.to_dict()
  ```

### sqlalchemy

sqlalchemy需要数据模型，可以直接生成：

- pip install sqlacodegen
- sqlacodegen postgresql://hulattpqinzi:test123@120.78.147.168:5432/hulattpqinzi > models.py

基本使用方式

- 插入

  content是DataFrame类型数据

  ```python
  # 插入content，获得id，同时将id插入id映射表
  Session = sessionmaker(bind=target_engine)
  session = Session()
  
  # 迭代DataFrame
  for index, row in content.iterrows():
      origin_id = row['id']
      del row['id']
  
      # 插入content
      content = Content()
      content.__dict__.update(row.to_dict())
      session.add(content)
      session.commit()
  
      # 插入映射表
      content_sync_id_map = ContentSyncIdMap(baobei_content_id=origin_id, qinzi_content_id=content.id)
      session.add(content_sync_id_map)
      session.commit()
  
      session.close()
  ```

- 更新

  content是DataFrame类型数据

  ```python
  Session = sessionmaker(bind=target_engine)
  session = Session()
  
  for index, row in content.iterrows():
      content_id = id_map.loc[id_map['baobei_content_id'] == row['id']]['qinzi_content_id'].tolist()[0]
      del row['id']
      session.query(Content).filter(Content.id == content_id).update(row.to_dict())
  
      session.commit()
      session.close()
  ```

## 测试方式

### 数据准备

将呼啦宝贝的数据导入在测试服新创建的hulattpbaobei数据库

将呼啦亲子的数据导入在测试服新创建的额hulattpqnzi数据库

```sql
-- 在测试服以postgres账户登录（登录方式：以自己账号登录，切换到root，再切换到postgres）
-- 执行如下语句
-- 创建hulattpbaobei
create user hulattpbaobei with password 'test123';
create database hulattpbaobei;
GRANT ALL PRIVILEGES ON DATABASE hulattpbaobei to hulattpbaobei;
-- 创建hulattpqinzi
create user hulattpqinzi with password 'test123';
create database hulattpqinzi;
GRANT ALL PRIVILEGES ON DATABASE hulattpqinzi to hulattpqinzi;
--  测试完成后记得删除相关database和user
drop database hulattpbaobei;
drop user hulattpbaobei;
drop database hulattpqinzi;
drop user hulattpqinzi;
```

数据搬移如下：

```bash
pg_dump --host=pgr-wz9sl5h20qh9v67d9o.pg.rds.aliyuncs.com --port=1921 --username=pm --dbname=hulattp | psql -h 120.78.147.168 -p 5432 hulattpbaobei hulattpbaobei
pg_dump --host=pgm-wz96wr9nv4t7i7ge117390.pg.rds.aliyuncs.com --port=1433 --username=hulattp --dbname=hulattp | psql -h 120.78.147.168 -p 5432 hulattpqinzi hulattpqinzi
```

别忘了使用.pgpass，文档参考：https://www.postgresql.org/docs/9.3/libpq-pgpass.html

### 代码

代码并未加入版本控制，放在阿里云函数计算中，请参考：https://fc.console.aliyun.com/fc/service/cn-shenzhen/hula-ttp/function/content-migrate

### 测试

整个同步分为三步：

- 呼啦宝贝-呼啦亲子的content表id对应关系建立、人工校对
- 初次同步
- 每日定期同步

在代码中有详细记录，不再赘述

## 问题记录

### html中的css和js静态文件无法加载导致文章无法显示

css和js路径是根据cssVersion和jsVersion由前端动态生成，因此他们需要同时存在于两遍的oss中。

解决方案：上传时同时上传新的css和js文件到呼啦宝贝和呼啦亲子两边的oss

### 非相关字段

- liked_count

  对新同步的记录，liked_count要置为0

  对需要更新的记录，liked_count字段要忽略

- favored_count

  对新同步的记录，liked_count要置为0

  对需要更新的记录，liked_count字段要忽略

- vector

  vector字段在呼啦亲子的数据库中是没有的，搬过来时要删除