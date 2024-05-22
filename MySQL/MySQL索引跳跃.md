# 什么是 MySQL 索引跳跃（Skip Scan Range Access Method）？

答：MySQL 中的索引跳跃（Index Skip Scan）是一种查询优化技术，用于提高查询性能。它利用已存在的索引来跳过不必要的行，从而减少扫描的行数。这项技术对于多列索引特别有效，可以在用户违背最左前缀匹配的情况下利用后面的列来进行查询。下面我详细解释：

预置数据：  

```sql
-- 创建一个用户表，id 是主键，另外两个字段，sex 是性别，只有 0 和 1，0 代表女，1 代表男。
age 是用户的年龄，存在一个联合索引 index sex_age(sex, age)

CREATE TABLE tb_user(
	id PRIMARY KEY,
	sex INT, 
  age INT, 
  INDEX sex_age(sex, age)
  );
  
  -- 插入一百万条数据，age 从 0 - 99 随机分布，sex 从 0 - 1 随机分布
  ..... （此处省略）
  
  -- 最后执行，目的是统计表的静态数据，查询优化器会使用
  
  ANALYZE TABLE tb_user;
```

表、数据预置好以后，开始实验过程讲解：

执行：

```sql
EXPLAIN SELECT sex, age FROM tb_user WHERE age > 10;
```

按照联合索引最左前缀匹配的规则，上面的语句最多只能扫描整个 sex_age 索引去逐行对比条件，但是实验结果如下：

![image](https://github.com/kyiree/SuperEngineer/assets/64623867/eb40b059-7b6f-4f39-b38b-21e09bdb69b3)

可以看到 Extra 这一列有一个 “Using index for skip scan”，并且 key 使用了 “sex_age”，type 是 “range”，这就是所谓的 MySQL 索引跳跃，将全索引扫描变成了范围扫描，大大减少了扫描的行数

底层原理

InnoDB 通过统计信息（Statistics）来估算表中不同值的分布。这些统计信息在表被分析（ANALYZE TABLE）或索引被创建时生成，并在插入、删除数据时进行自动更新

所以，在我们执行查询的时候，InnoDB 已经提前知道了 tb_user 表的 sex 字段只有 0 和 1 这两个唯一值，当我们执行

```sql
SELECT sex, age FROM tb_user WHERE age > 10;
```

的时候，会转换成执行下面的语句：

```sql
SELECT sex, age FROM tb_user WHERE sex = 0 AND age > 10;
UNION ALL
SELECT sex,age FROM tb_user WHERE sex = 1 AND age > 10;
```

因为 sex 的加入，上面的语句是满足最左前缀匹配规则的，先利用 sex = xxx 走索引去定位数据，并且在 sex 相同的情况下，age 是有序的，所以就可以利用二分查找继续搜索 age > 10 这个条件，整个过程比全索引扫描快很多

参考：

https://dev.mysql.com/doc/refman/8.0/en/range-optimization.html#range-access-skip-scan

注意：官方文档说 MySQL 8.0.13 才会开始有这个特性

