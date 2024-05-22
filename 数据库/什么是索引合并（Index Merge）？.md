将多个单索引的结果进行聚合然后再回表，例如 tb_user 这个表里面有 age 、name、phone 这三个字段，并且具有 index(age)、index(name)，这两个索引。

假如执行 SELECT * FROM tb_user WHERE age > 30 AND name = '张三' AND phone = '123';

这个搜索条件有两个单字段索引，通常的做法是选择其中某一个索引查询到主键 id 后再回表查询完整信息去对比其它搜索条件。

索引合并指的是（可利用多线程），同时扫描 index(age) 和 index(name)，然后将它们得到的主键 ids 取一个交集，这样就可以减少回表的次数，同时也能够利用多个索引
