# MySQL性能优化
---
## 一.优化的顺序（效果从高到低）
- **1**. sql 以及 索引优化。
- **2**. 数据库表结构优化。
- **3**. 系统配置优化（TCP/IP最大连接数，打开的文件数）。
- **4**. 硬件优化（成本比较高）。
---
## 二.sql 以及 索引优化
- **1**. 使用MYSQL慢查询日志对有效率问题的sql进行监控, 检查是否开启慢查日志功能，设置保存位置，打开记录未使用索引sql开关，超过1s的查询记录到日志。

``` MySQL
show vaiables like 'slow_query_log';
set global slow_query_log=on;
set global slow_query_log_file  = '/home/mysql/sql_log/mysql-slow.log';
set global log_queries_not_using_indexes=on;
set global long_query_time=1;
```

### 2.1 常用慢查日志分析工具

- **1**. mysqldumpslow
- **2**. pt-query-digest  （比较详细）

### 2.2 根据慢查日志发现有问题sql

- **1**. 查询次数多且每次查询占用时间长的sql。  
  通常为pt-query-digest分析的前几个查询。
- **2**. IO大的sql。  
  注意分析pt-query-digest分析中的Rows examine项。  
- **3**. 未命中索引的SQL。  
  注意分析Rows examine 和 Rows Send 的对比。

### 2.3.SQL查询语句的优化

通过explain查询SQL的执行计划。

![](img\20170509151706.png)

table：显示这一行的数据是关于哪张表的。  
type:这是重要的列,显示连接使用了何种类型。从最好到最差的连接类型为const、eq_reg、ref、range、index和ALL。  
possible_keys：显示可能应用在这张表中的索引。如果为空，没有可能的索引。  
key:实际使用的索引。如果为NULL，则没有使用索引。  
key_len:使用的索引的长度。在不损失精确性的情况下，长度越短越好。  
ref:显示索引的哪一列被使用了,如果可能的话，是一个常数。  
rows：MYSQL认为必须检查的用来返回请求数据的行数。  

Using filesort:看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行

Using temporary:看到这个的时候，查询需要优化了。这里，MYSQL徐哟创建一个临时表来存储接口，这通常发生在对不同的列表进行ORDER BY上，而不是GROUP BY上

### 2.3.1 优化Count()和Max()函数

- **1**.  Max()优化  

![](img\20170509152323.png)  

可以通过创建索引来加速:  （覆盖索引）
create index idx_paydate on payment(payment_date);

- **2**.  count()优化  

![](img\20170509153339.png)  

![](img\20170509153359.png)  

注意！：count(*) 和 count(id) 区别  
count(*) 包含NULL  
count(id) 不包含NULL  

### 2.3.2 子查询优化   

![](img\20170509155118.png)  

### 2.3.3 优化group by查询  

![](img\20170509155253.png)  

- **1.** 使用了临时表

![](img\20170509155411.png)  

- **2.** 优化后的sql

![](img\20170509155526.png)  


### 2.4 SQL及索引优化

### 2.4.1 如何选择合适的列建立索引

- **1.** 在where从句，group by从句，order by从句，on从句中出现的列建立索引
- **2.** 索引字段越小越好
- **3.** 离散度大的列放在联合索引的前面

### 2.4.2 索引优化SQL的方法

- **1.** 查看离散率:

![](img\20170509163059.png)  

![](img\20170509162924.png)  

- **2.** 查找冗余索引：

```Java7
select
a.TABLE_SCHEMA AS '数据名',
a.TABLE_NAME AS '表名',
a.INDEX_NAME AS '索引1',
b.INDEX_NAME AS '索引2',
a.COLUMN_NAME as '重复列名'
from STATISTICS a JOIN STATISTICS b ON
a.TABLE_SCHEMA = b.TABLE_SCHEMA
AND a.TABLE_NAME = b.TABLE_NAME
AND a.SEQ_IN_INDEX = b.SEQ_IN_INDEX
AND a.COLUMN_NAME = b.COLUMN_NAME
```

- **3.** 工具查看冗余索引：
pt-duplicate-key-checker  

### 2.4.2 索引维护的方法
- **1.** 工具查看不用的索引：  
pt-index-usage
pt-index-usage \  
-uroot -p' ' \   
mysql-slow.log  
---
## 三.表结构优化

- **1.** 最小的数据类型
- **3.** 使用简单数据类型，int比varchar类型在mysql处理上简单
- **3.** 尽可能使用not null 定义字段，设置默认值
- **3.** 尽量少用text类型
