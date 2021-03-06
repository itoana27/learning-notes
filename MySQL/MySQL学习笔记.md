### MySQL学习笔记

#### MyISAM和InnoDB的区别

|            | MyISAM | InnoDB                 |
| ---------- | ------ | ---------------------- |
| 事务支持   | 不支持 | 支持                   |
| 数据行锁定 | 不支持 | 支持                   |
| 外键       | 不支持 | 支持                   |
| 全文索引   | 支持   | 5.6.4开始支持          |
| 表空间大小 | 较小   | 较大，约为MyISAM的两倍 |

使用选择：

+ MyISAM：节约空间，速度较快
+ InnoDB：安全性高，事务处理，多表多用户处理

在物理空间的位置：

所有的数据库文件都是存在`data`目录下，本质还是文件的存储。

MySQL引擎在文件上的区别：

+ InnoDB对应文件：
  + `*.frm`文件存储表的元数据，包括表结构信息等。MySQL 8.0不再使用frm文件，而是将表的元数据一起放在ibd文件中
  + `*.ibd`：InnoDB DATA，表数据和索引文件
  + 上级目录下的ibdata1文件，表空间文件
+ MyISAM对应文件
  + `*.frm` 表结构的定义文件
  + `*.MYD`  数据文件（data）
  + `*.MYI` 索引文件（index）

设置数据库的字符集编码：

```sql
CHARSET=utf8
```

不设置的话，回事MySQL默认的字符集编码，不支持中文。

#### MySQL字段长度

+ 整数型

  如`int(11)`，11并不是限定字段最大值为11个9，`int`本身已经限制了取值范围。11表示数据在显示是显示的最小长度。当存储的字段长度超过11时，没有任何影响。当字段长度小于11时，只有在设置了`zerofill`才用0来填充，才能看到效果。换句话说，没有设置zerofill，11这个值就是无用的。

+ 字符串型`CHAR(M)`，`VARCHAR(M)`

  CHAR型的M值为这个字段的固定长度，不足的用空分填充。

  VARCHAR型的M为这个字段的最大长度，VARCHAR值保存时只保存需要的字符数，另加一个字节来记录长度。

#### 修改和删除表

```sql
ALTER TABLE 表名 RENAME AS 新表名;
ALTER TABLE 表名 ADD 字段名 约束;
ALTER TABLE 表名 MODIFY 字段名 列属性;  -- 修改约束
ALTER TABLE 表名 CHANGE 原列名 新列名 [列属性];  -- 字段重命名
ALTER TABLE 表名 DROP 字段名;  -- 删除列
DROP TABLE IF EXISTS 表名;  -- 所有的创建和删除操作尽量加上判断，以免报错
```

外键：

```sql
ALTER TABLE 表名 ADD CONSTRAINT 约束名 FOREIGN KEY(列名) REFERENCES 引用的表名(引用的列名);
```

外键不建议使用，避免数据库限制过多造成困扰。

最佳实践：

+ 数据库只是单纯的表，只用来存数据，只有行和列
+ 想使用外键用程序实现

#### DML

`DELETE` 和`TRUNCATE`的异同：

+ 都是删除数据，都不会删除表结构
+ `TRUNCATE`重新设置自增列，计数器会归零；`TRUNCATE`不会影响事务

#### 索引

索引（在MySQL中也叫做“键（key）”）是存储引擎用于快速找到一条记录的一种数据结构。

索引优化应该是对查询性能优化最有效的手段了。索引能够轻易将查询性能提高几个数量级，“最优”的索引有时比一个“好的”索引吸能要好两个数量级。

索引可以包含一个或多个列。如果索引包含多个列，那么列的顺序也十分重要，因为MySQL只能高效地使用索引最左前缀列。

##### 索引的优缺点

优点：

+ 索引大大减小了服务器需要扫描的数据量，从而大大加快数据的检索速度
+ 索引可以帮助服务器避免排序和创建临时表
+ 索引可以将随机IO变为顺序IO
+ 可以加速表和表之间的连接
+ 在使用分组和排序子句进行数据检索时，同样可以显著减少查询中分组和排序的时间

缺点：

+ 创建索引和维护索引要耗费时间，这种时间随着数量的增加而增加
+ 索引需要占用物理空间，除了数据表占用数据空间外。每一个索引还要占用一定的物理空间。如果需要建立聚簇索引，那么需要占用的空间会更大
+ 对表中的数进行增、删、改的时候，索引需要动态的维护，这样就降低了整体的维护速度
+ 如果某个数据包含许多重复的内容，为它建立索引没有太大的效果
+ 对于非常小的表，大部分情况下简单的全表扫描更高效

##### 创建索引的准则

应该创建索引的列：

+ 主键和外键必须建立索引
+ 在经常使用在where子句的列上建立索引
+ 在经常用连接（join）的列上创建索引，可以加快连接的速度
+ 在经常需要排序（order by）的列上建立索引，因为索引已经有序，可以加快排序时间

不应创建索引的列：

+ 对于很少使用或很少参考的列上不应建立索引
+ 对于哪些只有很少数据或重复值多的列也不应建立索引
+ 对于哪些定义为text，image和bit数据类型的列不应该建立索引
+ 当该列修改次数远高于检索次数时，不应创建索引

>  B-Tree索引

B-Tree即多路搜索树，树高一层意味着多一次磁盘IO。

B-Tree的特征： 

+ 关键字分布在整棵树中
+ 任何一个关键字出现且只能出现在一个节点中
+ 搜索有可能在非叶子节点结束
+ 其搜索性能等价于在关键字全集内做一次二分查找
+ 每个节点中的key都按照从小到大的顺序排列，每个key的左子树中的所有key都小于它，而右子树中的所有key都大于它
+ 所有叶子节点都位于同一层，即根节点到每个叶子节点的长度都相同

B+Tree的特征：

+ 所有关键字都出现在叶子节点的链表中（稠密索引），且链表中的关键字恰好是有序的
+ 不可能在非叶子节点命中
+ 非叶子节点相当于是叶子节点的索引（稀疏索引），叶子节点相当于是存储（关键字）数据的数据层
+ 每一个叶子节点都包含指向下一个叶子节点的指针，从而方便叶子节点的范围遍历
+ 更适合文件索引系统

MySQL的`CREATE TABLE`语句使用`B-Tree`关键字，但底层可能使用了不同地存储结构。如InnoDB使用地是B+树。

> HASH索引

Hash索引就是采用一定的Hash算法，把键换算成新的哈希值，检索时不需要类似B+树那样从根节点到叶子节点逐级查找，只需要一次Hash算法既可立刻定位到相应的位置，速度非常快。

Hash索引仅仅能满足`=` ，`IN` 和 `!=`查询，不能使用范围查询。

由于Hash索引比较的是进行Hash运算之后的Hash值，所以它只能用于等值的过滤，不能基于范围的过滤，因为经过相应的Hash算法处理后的Hash值的大小关系并不能保证和Hash运算前一样。

##### 索引分类

按功能分：

+ 主键索引：一张表只能有一个主键索引
+ 唯一索引：数据列不允许重复，允许为NULL值
+ 普通索引
+ 全文索引：它查找的是文本中的关键词，主要用于全文检索

按列数划分：

+ 单列索引：一个索引只包含一个列
+ 组合索引：一个组合索引包含两个以上的列。查询时遵循MySQL组合索引的“最左前缀”原则，即使用where子句时条件要按照建立索引的时候字段的排列方式放置索引才会生效

物理分类：

+ 聚簇索引：聚簇索引不是单独的一种索引类型，而是一种数据存储方式。这种存储方式时是依靠B+树来实现的。聚簇索引可以理解为将数据存储与索引放到了一块，找到了索引也就找到了数据
+ 非聚簇索引：数据和索引是分开的，B+树叶子节点存放的不是数据表的行记录

虽然InnoDB和MyISAM存储引擎都默认使用B+树来存储索引，但只有InnoDB的主键索引才是聚簇索引。

InnoDB中的辅助索引以及MyISAM使用的都是非聚簇索引。每张表最多只能拥有一个聚簇索引。InnoDB必须有主键，MyISAM可以没有。如果没有显式指定主键，则选取首个为唯一且非空的列作为主键索引。如果没有满足这个条件的列，则MySQL自动为InnoDB表生成一个隐含字段作为主键。

MyISAM也使用B+树作为索引结构，但具体的实现方式却与InnoDB不同。MyISAM使用的都是非聚簇索引。MyISAM索引的叶子节点存放的式数据记录的地址，在MyISAM中，主键索引和辅助索引在结构上没有区别，只是主键索引要求key是唯一的，而辅助索引的key可以重复。

非主键索引需要访问两次索引查找，第一次找到主键值，第二次根据主键值找到行数据。

##### InnoDB索引优化

+ InnoDB主键不宜定义太大，因为辅助索引也会包含主键列，如果主键定义太大，其他索引也很大
+ InnoDB中尽量不使用非单调字段作为主键，因为InnoDB数据本身是一棵B+树，非单调的主键会造成插入新纪录时数据文件为了维持B+树的特性而频繁的分裂调整。使用自增字段是一个很好的选择

##### 索引操作

```sql
-- 创建普通索引 
CREATE INDEX index_name ON table_name(col_name);

-- 创建唯一索引
CREATE UNIQUE INDEX index_name ON table_name(col_name);

-- 创建普通组合索引
CREATE INDEX index_name ON table_name(col_name_1,col_name_2);

-- 创建唯一组合索引
CREATE UNIQUE INDEX index_name ON table_name(col_name_1,col_name_2);

-- 修改表结构创建索引
ALTER TABLE table_name ADD INDEX index_name(col_name);

-- 创建表结构时直接指定索引
CREATE TABLE table_name (
    ID INT NOT NULL,
    col_name VARCHAR (16) NOT NULL,
    INDEX index_name (col_name)
);

-- 直接删除索引
DROP INDEX index_name ON table_name;

-- 修改表结构删除索引
ALTER TABLE table_name DROP INDEX index_name;

-- 查看索引信息（包括索引结构等）
show index from table_name;
```

##### 最左前缀原则

在MySQL上建立联合索引时会遵循最左前缀匹配原则，即：有索引`index1:(a, b, c)`，使用下列三种查询条件：

```sql
where a='1';
where a='1' and b='1';
where a='1' and b='1' and c='1';
```

都会走索引，而使用下列条件：

```sql
where b='1';
where c='1';
where b='1' and c='1';
```

不会走索引。使用：

```sql
where a='1' and c='1';
```

也会走索引，但只会走a的索引，即跟

```sql
where a='1'
```

走一样的索引。

且a，b，c三个条件的顺序不重要，MySQL查询优化器会调整顺序，b，c，a和a，b，c的效果一样。

##### 回表查询与覆盖索引

什么是回表查询？

先定位主键值，在根据主键值定位行记录，就是回表查询。InnoDB的辅助索引查询就是回表查询。

什么是覆盖索引？

覆盖索引是一个非聚簇索引，只需在这一棵索引树上就能获取SQL所需的列数据，无需回表。

如何实现索引覆盖？

将被查询的字段，建立到联合索引里去。

##### 全文索引

MySQL 5.6 以前的版本，只有MyISAM存储引擎支持全文索引；

MySQL 5.6 及以后的版本，MyISAM和InnoDB均支持全文索引。

只有CHAR、VARCHAR、TEXT数据类型可以建立全文索引。

创建全文索引：

```sql
-- 建表的时候
FULLTEXT KEY keyname(colume1);

-- 在已存在的表上创建
create fulltext index keyname on xxtable(colume1);

alter table xxtable add fulltext index keyname (colume1);
```

使用全文索引：

全文索引有独特的语法格式，需要配合`match()`和`against()`函数使用。

+ match() 函数中指定的列必须是设置为全文索引的列
+ against() 函数标识需要模糊查询的关键字

```sql
select * from fulltext_test where match(words,artical) against('a');
select * from fulltext_test where match(words,artical) against('aa');
select * from fulltext_test where match(words,artical) against('aaa');
select * from fulltext_test where match(words,artical) against('aaaa');
```

全文索引关键字具有阈值，可通过

```sql
show variables like '%ft%';
```

查看。InnoDB的全文索引关键字最小索引长度为3。

##### 索引失效

+ `like %xx` 

  如果搜索键值以通配符`%`开头，则放弃使用索引，直接全表扫描；若只是以`%`结尾则不影响。

+ 对索引列进行运算

  如果查询条件中含有函数或表达式，将导致索引失效而进行全表扫描。

+ `or` 条件索引问题

  `or` 的条件除了同时是主键的时候，索引才会生效，其余情况下，索引都失效。

+ 数据类型不一致（隐式类型转换导致的索引失效）

  如果列是字符串类型，传入条件必须用引号引起来，不然索引失效。

+ `!=` 问题

  普通索引使用`!=` 索引失效，主键索引没有影响。where语句中索引列使用了负向查询，可能会导致索引失效。

  负向查询包括：`NOT`、`!=`、`<>`、`NOT IN`、`NOT LIKE`。

+ 联合索引违背最左前缀原则

+ order by问题

  `order by`对主键索引排序会用到索引，其他索引失效。

