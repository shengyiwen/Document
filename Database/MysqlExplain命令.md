#### explain是什么

explain 命令是显示了使用mysql如何是用索引来处理select语句以及连接表的

    使用方法：explain + select 语句


#### 对结果进行解释

- id：表示执行的顺序，id越高的语句表示执行优先级越高

- table：表示当前id执行sql涉及到的表

- type: 这是最重要的列，表示连接使用了什么类型
  
从好到差以此排列的连接类型如下：

**system > constant > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > all**；

Note: 一般来说至少达到range级别，最好是使用ref及以上

- possible_keys: 显示可能使用到这张表的索引，如果为空，即没有可能使用到的索引

- key: 实际使用到的索引，如果为null，表示没有使用索引

- key_len: 使用索引的长度，在不损失精度的情况下，长度越短越好

- ref: 显示索引的哪一列被使用到了，如果可能的话，是一个常亮

- rows: mysql认为必须检查的用来返回请求数据的行数
  
- extra: 额外信息 

最坏的情况Using temporary和Using filesort 意思根本不能使用索引


#### 详细解释

1. id：包含一组数字，表示查询中执行select语句的先后顺序
   如果id相同，执行顺序由上到下；如果id不同，id值越高的优先级越高

2. select_type: 表示每个select语句的类型

- simple: 表示不包含子语句或者union

    
    eg: 【select * from a】

- primary: 查询中若包含任何复杂的子查询，则最外层即是Primary

    
    eg: 【select * from 】 (select * from b) 

- subquery：在select 或 where列表中包含了子语句，则子语句即是Subquery

    
    eg: select * from a where a.id in 【(select * from a )】

- derived：用来表示包含在from子语句中的子查询的select,mysql会递归执行并把结果放入一个临时表中，服务器内部称”派生表“，因为该临时表是从子查询中派生出来的。或者union包含在from子语句的子查询中，外层的select也被标记为dervied

    
    eg: select * from 【(select * from b)】

- union: 表示第二个select出现在union之后的

    
    eg: select * from a union 【select * from b】 

- union result: 从Union表获取的结果的select

3. type：从表中找到所需行的方法，又称访问类型
   
   常见类型： **all < index < range < ref < eq_ref < const < system <null**

性能从左到右依次递增

- ALL： full table scan： 全表扫描，遍历全表


    【 eg: select * from a】

- index: full index scan：index 与 all的区别为index类型值遍历索引树


    【eg: select id from a】

- range: 索引范围扫描，对索引的扫描开始于某一点，返回匹配域的行。带有between 或者 where语句中带有 <,>查询。当mysql 使用索引去查找一系列值的时候，例如in 或者or列表也会心安是range


    【eg: select * from a where id >10】 【eg: select * from a where id in (1, 4)】

- ref:使用非唯一索引或者唯一索引的前缀扫描，返回匹配某个单独值的记录行


    【eg: select * from a where name='asheng'】 name做了普通索引

- eq_ref: 类似ref，使用了索引是唯一索引，对于索引值表中只有一条记录，即使用了primary_key或者unique_key作为关联条件


    【eg:select * from a where id=1】

- const、system：当mysql对查询某部分进行优化，并转换为一个常量的时候，使用这种类型的访问。如将主键置于where列表中，mysql就会将该查询转换成为一个常量


    【eg:select * from (select * from a where id =1) b】
     system是const类似，当查询的表只有一行的情况下，使用system

- null: mysql在优化中分解语句，执行时甚至不访问表或者索引，例如从一个索引列中选取最小值，可以同单独的索引查询即可完成

    
    【eg: select * from a where id = (select min(id) from a)】 其中的子查询即为null type

4. possible_key：mysql能使用哪个索引查找到记录，查询涉及到的字段若存在索引，则该索引会被列出

5. key：mysql执行语句中确定使用到的索引，如没有使用索引，返回Null

6. key_len: 表示索引中使用的字节数，可通过该列计算查询中使用索引的长度（显示最大可能长度，而不是实际使用长度，根据表的定义计算出，而不是查询出的）

7. ref: 表示上述表的连接匹配条件，即哪些列或者常亮被用于查询索引列上的值

8. rows: mysql根据表统计信息以及检索选用情况，估计的找到所需记录需要读取的行数

9. extra: 包含不适合在其他列中但十分重要的额外信息

- using index：表示select中使用了覆盖索引

覆盖索引（Covering Index）
MySQL可以利用索引返回select列表中的字段，而不必根据索引再次读取数据文件
包含所有满足查询需要的数据的索引称为覆盖索引（Covering Index）
注意：如果要使用覆盖索引，一定要注意select列表中只取出需要的列，不可select *，因为如果将所有字段一起做索引会导致索引文件过大，查询性能下降

- using where

表示mysql服务器将在存储引擎检索行后再进行过滤。许多where条件里涉及索引中的列，当（并且如果）它读取索引时，就能被存储引擎检验，因此不是所有带where字句的查询都会显示"Using where"。有时"Using where"的出现就是一个暗示：查询可受益与不同的索引。

- using temporary

表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询
这个值表示使用了内部临时(基于内存的)表。一个查询可能用到多个临时表。有很多原因都会导致MySQL在执行查询期间创建临时表。两个常见的原因是在来自不同表的上使用了DISTINCT,或者使用了不同的ORDER BY和GROUP BY列。可以强制指定一个临时表使用基于磁盘的MyISAM存储引擎。这样做的原因主要有两个：


    1)内部临时表占用的空间超过min(tmp_table_size，max_heap_table_size)系统变量的限制

    2)使用了TEXT/BLOB 列

- using filesort

MySQL中无法利用索引完成的排序操作称为“文件排序”

- using join buffer

改值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。

- impossible where

这个值强调了where语句会导致没有符合条件的行。

- select tables optimized away

这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行.

- index merges

当MySQL 决定要在一个给定的表上使用超过一个索引的时候，就会出现以下格式中的一个，详细说明使用的索引以及合并的类型。
