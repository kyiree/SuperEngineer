mysql 的比较规则是为了满足不同用户需求而设置的。例如使用 utf8mb4_general_ci 作为字段的比较规则它是不区分大小写的，比如查询 select * from user where name = ‘tom‘ 和 select * from user where name = ‘TOM’ 这两条语句是一样的

而使用 utf8mb4_bin 作为比较规则则会区分大小写，上面两条语句查出的结果不一样

此外，索引创建的时候，key 之间的比较也是按照字段的比较规则，所以要注意如果是唯一索引并且比较规则不区分大小写，那么不能同时存储 TOM 和 tom这两个单词
