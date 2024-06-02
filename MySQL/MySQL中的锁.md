# 共享锁和排它锁（Shared and Exclusive Locks）

InnoDB 有两种类型的行锁，分别是共享（S）锁和排他（X）锁

- 多个共享锁可以并存
- 只要有一个排他锁，其它锁都不能并存

# 意向锁（Intention Locks）

InnoDB 实现了多种粒度的锁，如表锁和行锁，为了提高性能，引入了意向锁，它是一种表级别的锁，如果一个事务想要加行锁，必须先提前获取表级别的意向锁，有两种类型的意向锁

- 意向共享 (IS) 锁：如果一个事务想要加共享行锁，必须先提前获取表的意向共享锁，举个例子 SELECT ... FOR SHARE 需要 IS
- 意向排他 (IX) 锁：如果一个事务想要加排他行锁，必须先提前获取表的意向排他锁，举个例子 SELECT ... FOR UPDATE 需要 IX

表级别的锁兼容性如下：
| | X | IX | S | IS |
|--| -- |--|--|--|
| X | Conflict | Conflict | Conflict | Conflict |
| IX | Conflict | Compatible | Conflict | Compatible |
| S | Conflict | Conflict | Compatible | Compatible |
| IS | Conflict | Compatible | Compatible | Compatible |

SHOW ENGINE INNODB STATUS  输出如下：

```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```
# 行锁（Record Locks）

行锁指的是在索引行上加的锁, 例如：SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE; 防止其它事务插入、更新、删除 t.c1 = 10 的行数据。行锁一定是加在索引上的行记录，在 InnoDB 中，所有记录都是存在索引上的，即使用户没有显示创建，也会有一个默认的聚簇索引

SHOW ENGINE INNODB STATUS 输出如下：

```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

# 间隙锁（Gap Locks）

间隙锁是一个范围锁。例如, SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE; 防止其它事务比如说插入 t.c1 = 15 这条记录，即使当前没有 t.c1 = 15 这条记录，因为它锁住的是一个范围。间隙锁是高性能和高并发之间做出的权衡，并且在某些事务隔离级别中使用，而在其他事务隔离级别中则不使用，间隙锁的作用仅仅只是防止其它事务在间隙中插入数据，当然这跟不同的隔离级别有关，间隙锁是可以被关闭的，比如说当隔离级别被设置为 READ COMMITTED 或者以下，举些例子深入理解：

```sql
CREATE TABLE `tb_user` (
  `id` bigint PRIMARY KEY NOT NULL,
  `name` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

INSERT INTO tb_user VALUES(10,'a'),(50,'b'),(100,'c');
```

当隔离级别设置为 READ COMMITTED 的时候

事务 A 执行

```sql
UPDATE tb_user SET name = 'd' WHERE id BETWEEN 10 AND 50;
```

事务 B 执行

```sql
INSERT INTO tb_user VALUES(20, 'd');
```

两个事务不会相互阻塞

当隔离级别设置为 REPEATABLE READ 的时候

事务 A 执行

```sql
UPDATE tb_user SET name = 'd' WHERE id BETWEEN 10 AND 50;
```

事务 B 执行

```sql
INSERT INTO tb_user VALUES(20, 'd');
```

两个事务会相互阻塞

# Next-Key Locks

 Next-Key Locks 是间隙锁和行锁的结合

(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)

InnoDB 默认的隔离级别是 REPEATABLE READ，在这种情况下，InnoDB 使用 Next-Key Locks 防止幻读

SHOW ENGINE INNODB STATUS 输出如下：

```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

# 插入意向锁（Insert Intention Locks）

插入意向锁是在间隙加的意向锁，在执行 INSERT 操作的时候会用到。例如一个索引有4和7两条数据，两个不同的事务分别插入5和6，它们都会在4到7这个间隙加上插入意向锁，但是它们之间不会相互阻塞，但是插入意向锁会和其它间隙锁相互阻塞

下面的例子会相互阻塞：

事务 A 先加间隙锁

```sql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```

事务 B 插入：

```sql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);

SHOW ENGINE INNODB STATUS 输出：

RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```

# AUTO-INC Locks

An AUTO-INC lock is a special table-level lock taken by transactions inserting into tables with AUTO_INCREMENT columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.

The innodb_autoinc_lock_mode variable controls the algorithm used for auto-increment locking. It allows you to choose how to trade off between predictable sequences of auto-increment values and maximum concurrency for insert operations.

For more information, see Section 17.6.1.6, “AUTO_INCREMENT Handling in InnoDB”.

# Predicate Locks for Spatial Indexes

InnoDB supports SPATIAL indexing of columns containing spatial data (see Section 13.4.9, “Optimizing Spatial Analysis”).

To handle locking for operations involving SPATIAL indexes, next-key locking does not work well to support REPEATABLE READ or SERIALIZABLE transaction isolation levels. There is no absolute ordering concept in multidimensional data, so it is not clear which is the “next” key.

To enable support of isolation levels for tables with SPATIAL indexes, InnoDB uses predicate locks. A SPATIAL index contains minimum bounding rectangle (MBR) values, so InnoDB enforces consistent read on the index by setting a predicate lock on the MBR value used for a query. Other transactions cannot insert or modify a row that would match the query condition.

