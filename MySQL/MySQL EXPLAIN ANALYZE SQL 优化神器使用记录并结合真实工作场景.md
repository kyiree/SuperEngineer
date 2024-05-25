MySQL 8.0.18 was just released, and it contains a brand new feature to analyze and understand how queries are executed: EXPLAIN ANALYZE.

接下来我分享一个真实工作场景进行 SQL 优化的过程，介绍如何使用这件神器。在客户后台管理系统中，常常使用部门做数据权限的划分，例如上级部门的用户可以看到下级部门用户的数据，但是下级部门的用户不能看到上级部门用户的数据，这个方案该怎么设计呢？性能如何呢？下面具体简介

# 前期准备工作

以用户列表为例子，上级部门的用户可以看到下级部门的用户，但是下级部门的用户不能看到上级部门的用户，所以有两个表，用户表和部门表

```sql
CREATE TABLE tb_user (
	id 												BIGINT PRIMARY KEY,  
  name 									VARCHAR        COMMENT '姓名',
  phone                 VARCHAR        COMMENT '电话',
  email                   VARCHAR        COMMENT '邮箱',
  department_id    BIGINT             COMMENT '部门id'
) COMMENT '用户表';

CREATE TABLE tb_department (
	id         BIGINT PRIMARY KEY,
  name               COMMENT '名称',
  tree_path         VARCHAR COMMENT '部门名称，例子：1-，1-2-3-'
) COMMENT '部门表';

-- 预置一个总部，其它部门只能在总部下面
INSERT INTO tb_department VALUES(1, '总部', '1-');
```

使用 Java 多线程插入数据：

```java
@Data
@TableName(value = "tb_user", autoResultMap = true)
@AllArgsConstructor
public class User {
 			private Long id;
 			private String name;
 			private String phone;
 			private String email;
 			private Long departmentId;
}

@Data
@AllArgsConstructor
@TableName(value = "tb_department", autoResultMap = true)
public class Department {
			 private Long id;
 			 private String name;
			 private String treePath;
}

void generateUser() throws InterruptedException {
			 ExecutorService threadPool = Executors.newFixedThreadPool(16);
 			 long preInsertedUsers = 0; // 0
 			 long usersToGenerate = totalUsers - preInsertedUsers;
 			 for (long i = 1; i <= usersToGenerate; i += 1000) {
 			 				long j = i;
 			 				List<User> users = new ArrayList<>();
 			 				while (j < i + 1000 && j <= usersToGenerate) {
 											users.add(new User(j, UUID.randomUUID().toString(), UUID.randomUUID().toString(), UUID.randomUUID().toString(), (long) random.nextInt(30) + 1));
							 				j++;
 							}
 			threadPool.submit(() -> userRepository.saveBatch(users));
 			} 
 
 				while (userRepository.count() != totalUsers) {
 								log.info("[{}]", userRepository.count());
 								Thread.sleep(1000 * 3);
 				}
 }
 
 void generateDepartment() throws InterruptedException {
				 ExecutorService threadPool = Executors.newFixedThreadPool(16);
 				 long preInsertedDepartments = 1;
 				 long departmentsToGenerate = totalDepartments - preInsertedDepartments;
 				 for (long i = 2; i <= departmentsToGenerate + 1; i += 1000) {
 				 				long j = i;
 								List<Department> departments = new ArrayList<>();
 								while (j < i + 1000 && j <= departmentsToGenerate + 1) {
 												departments.add(new Department(j, UUID.randomUUID().toString(), generateTreePath()));
												 j++;
 									}
 								threadPool.submit(() -> departmentRepository.saveBatch(departments));
 					}
 					while (departmentRepository.count() != totalDepartments) {
 									log.info("[{}]", departmentRepository.count());
 									Thread.sleep(1000 * 3);
 					}
 }

```

MySQL8.0.x 部署在 docker 里面，放在 VMware 虚拟机，配置为 4 核 8 g

我们计划使用下面的方式做数据权限的功能：

```sql
SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

简单解释一下，就是根据部门数据权限分页查询用户列表，并按照用户名称倒序返回，其中

```sql
tree_path LIKE '1-%'
```

意味着当前是总部的用户查询，同理

```sql
tree_path LIKE '1-2-%
```

意味着当前是部门 id 为 2 的子部在查询，并且它的上级部门的部门 id 为 1 的部门，部门之间的上下级关系通过这样保存在数据库

# 测试实验一，预置 4500000 用户,1000 部门

执行 sql ：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.
tree_path LIKE '1-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

输出：

```sql
-> Limit: 50 row(s) (cost=2063027.23 rows=50) (actual time=5099.473..5099.523 rows=50 loops=1)
		-> Nested loop inner join (cost=2063027.23 rows=497799) (actual time=5099.472..5099.520 rows=50 loops=1)
 				-> Sort: u.`name` DESC (cost=494803.93 rows=4480638) (actual time=5099.442..5099.446 rows=50 loops=1)
 						-> Table scan on u (cost=494803.93 rows=4480638) (actual time=0.045..1065.274 rows=4544000 loops=1)
 				-> Filter: (d.tree_path like '1-%') (cost=0.25 rows=0.1) (actual time=0.001..0.001 rows=1 loops=50)
 						-> Single-row index lookup on d using PRIMARY (id=u.department_id) (cost=0.25 rows=1) (actual 
time=0.001..0.001 rows=1 loops=50)
```


分析输出：上面的语句执行耗时是 5 s 左右（第一次执行需要 10 s，后面重复执行则需要 5 s，后面的多次实验都是重复执行多次的结果），执行过程如下：先对 tb_user 表按照 name 进行排序，这个过程需要耗费 5 s 左右，然后按照顺序扫描 tb_user 表，去连接 tb_department 表，连接条件是 tb_department.id = tb_user.department_id，拿到 tb_department 的数据后，再去使用 tb_department.tree_path like '1-%' 的条件去过滤

伪代码如下：

```java
for sort(tb_user) :
		for tb_department
    		if (tb_user.department_id = tb_departmet.id)
 						if (tb_department.tree_path like '1-%') 
 								emit(row)
```

优化思路：为了避免排序，给 tb_user 的 name 字段加上索引：

```sql
ALTER TABLE tb_user ADD INDEX idx_name(name)
```

再次执行sql：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

输出：

```sql
-> Limit: 50 row(s) (cost=3733915.36 rows=50) (actual time=0.172..0.333 rows=50 loops=1)
		-> Nested loop inner join (cost=3733915.36 rows=50) (actual time=0.171..0.330 rows=50 loops=1)
				-> Index scan on u using idx_name (reverse) (cost=5.36 rows=450) (actual time=0.160..0.273 rows=50 loops=1)
 		-> Filter: (d.tree_path like '1-%') (cost=0.83 rows=0.1) (actual time=0.001..0.001 rows=1 loops=50)
				-> Single-row index lookup on d using PRIMARY (id=u.department_id) (cost=0.83 rows=1) (actual time=0.001..0.001 rows=1 loops=50)
```


可以看到，执行耗时变成了 172 ms ，执行过程如下：逆序扫描 tb_user 的 name 二级索引，然后按照顺序扫描 tb_user 表，去连接 tb_department 表， 连接条件是 tb_department.id = tb_user.department_id，拿到 tb_department 的数据后，再去使用 tb_department.tree_path like '1-%' 的条件去过滤

# 结论一：

虽然写的是子查询，但是 Nested loop inner join 表明： MySQL 优化器使用的是 join 的方式来执行

测试实验二，预置 4544000 用户，100000 部门

在实验一的数据前提，再把部门增加到 100000 个

再次执行sql：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

输出：

```sql
-> Limit: 50 row(s) (cost=1120209.86 rows=50) (actual time=0.088..0.182 rows=50 loops=1)
		-> Nested loop inner join (cost=1120209.86 rows=50) (actual time=0.088..0.179 rows=50 loops=1)
				-> Index scan on u using idx_name (reverse) (cost=5.36 rows=450) (actual time=0.078..0.127 rows=50 loops=1)
		-> Filter: (d.tree_path like '1-%') (cost=0.25 rows=0.1) (actual time=0.001..0.001 rows=1 loops=50)
				-> Single-row index lookup on d using PRIMARY (id=u.department_id) (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=50)
```

可以看到，执行耗时 88 ms

# 测试实验三，预置 4544000 用户，1000000 部门

在实验二的数据前提，再把部门增加到 100000 个

再次执行 sql：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

可以看到，执行耗时 88 ms

# 结论二

通过不断地增加部门的数据量，发现执行时间并不会增加，认真分析 EXPLAIN ANALYZE 的执行结果，因为是 ORDER BY u.name DESC LIMIT 0,50 的缘故，所 以整个过程如下：MySQL 在 tb_user 的 name 二级索引倒序查找（reverse），然后拿着这条数据去全表扫描 tb_department 表，又因为是 like '1-%'， 且所有部门都在总部底下，所以只需要遍历 tb_user 的 name 的二级索引最后 50 条数据即可，所以整个过程时间复杂度只跟部门表的大小有关，百万以内的全表扫描也不是问题

# 测试实验四，查询子部用户

基于结论二，查询总部用户速度会很快，但是查询子部用户应该会慢

执行 sql：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-2-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

输出

```sql
-> Limit: 50 row(s) (cost=3222593.07 rows=50) (actual time=31530.847..31530.847 rows=0 loops=1)
		-> Nested loop inner join (cost=3222593.07 rows=50) (actual time=31530.847..31530.847 rows=0 loops=1)
				-> Index scan on u using idx_name (reverse) (cost=5.03 rows=450) (actual time=0.145..28004.565 rows=4544000 loops=1)
				-> Filter: (d.tree_path like '1-2-%') (cost=0.72 rows=0.1) (actual time=0.001..0.001 rows=0 loops=4544000)
						-> Single-row index lookup on d using PRIMARY (id=u.department_id) (cost=0.72 rows=1) (actual time=0.001..0.001 rows=1 loops=4544000)
```

可以看到，执行耗时 31 s，很关键的一个点：Single-row index lookup on d using PRIMARY (id=u.department_id) (cost=0.72 rows=1) (actual time=0.001..0.001 rows=1 loops=4544000)，分析上面的分析结果，全索引扫描 tb_user 的 name，然后每条数据去全表扫描 tb_department，算法没有改变，依然是 Nested loop join

鉴于此，给 tb_department 的 tree_path 加上索引：alter table tb_department add index idx_tree_path(tree_path)

再次执行 sql：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-2-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

输出：

```sql
-> Limit: 50 row(s) (actual time=1254.099..1254.099 rows=0 loops=1)
		-> Sort: u.`name` DESC, limit input to 50 row(s) per chunk (actual time=1254.098..1254.098 rows=0 loops=1)
				-> Stream results (cost=495549.39 rows=448064) (actual time=1254.091..1254.091 rows=0 loops=1)
						-> Inner hash join (u.department_id = d.id) (cost=495549.39 rows=448064) (actual time=1254.090..1254.090 rows=0 loops=1)
 								-> Table scan on u (cost=92290.76 rows=4480638) (actual time=0.034..1020.373 rows=4544000 loops=1)
 								-> Hash
 										-> Filter: (d.tree_path like '1-2-%') (cost=1.21 rows=1) (actual time=0.014..0.016 rows=1 loops=1)
 												-> Covering index range scan on d using idx_tree_path over ('1-2-' <= tree_path <= '1-2-????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????') (cost=1.21 rows=1) (actual time=0.012..0.014 rows=1 loops=1)
```

可以看到，执行耗时 1.254 s，更重要的是，算法改变了，不再是 Nested loop join，使用了 Hash join，意味着把整个 tb_department 表缓存到内存里面 了，但是需要全表扫描 tb_user 表并且需要对结果进行排序，所以整个过程耗时超过 1 s

优化思路：去掉 tb_user 的 idx_name(name) 索引，换成 idx_department_id(department_id) 索引

再次执行 sql：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-2-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

输出：

```sql
-> Limit: 50 row(s) (actual time=0.029..0.029 rows=0 loops=1)
		-> Sort: u.`name` DESC, limit input to 50 row(s) per chunk (actual time=0.028..0.028 rows=0 loops=1)
				-> Stream results (cost=664.73 rows=603) (actual time=0.025..0.025 rows=0 loops=1)
						-> Nested loop inner join (cost=664.73 rows=603) (actual time=0.025..0.025 rows=0 loops=1)
								-> Filter: (d.tree_path like '1-2-%') (cost=1.21 rows=1) (actual time=0.013..0.015 rows=1 loops=1)
										-> Covering index range scan on d using idx_tree_path over ('1-2-' <= tree_path <= '1-2-????????????????????????????????????????????????????????????????????????????????????????????????????????????????') (cost=1.21 rows=1) (actual time=0.011..0.013 rows=1 loops=1)
								-> Index lookup on u using idx_department_name (department_id=d.id) (cost=663.52 rows=603) (actual time=0.008..0.008 rows=0 loops=1)
```

可以看到，执行耗时 只需要 29 ms，算法改变了，又变回了 Nested loop join，首先走 tb_department 的 tree_path 索引去过滤 d.tree_path like '1-2- %' 的条件，获得上一步过滤出来的 tb_department 后使用 Nest loop join 去连接 tb_user，连接条件为 tb_department.id = tb_user.department_id， 用小表去驱动大表，并把结果暂存起来，最后将之前的结果做一个排序返回，整个过程不会涉及全表扫描，都是走索引的方式，扫描 tb_department 的时候甚至用了覆盖索引 Covering index，所以整个过程速度非常快

伪代码如下：

```java
function executeQuery() {
		// Limit the result to 50 rows
 		limit = 50
 
 		// Step 1: Filter tb_department rows where tree_path like '1-2-%'
 		filtered_departments = []
 		for row in tb_department_index_scan('idx_tree_path', '1-2-%') {
 				if row.tree_path like '1-2-%' {
 						filtered_departments.append(row)
 				}
 		}
 		// Step 2: Perform a nested loop inner join
    result = []
 		for d in filtered_departments {
 				// Step 2.1: Use index lookup on tb_user using idx_department_name
 				for u in tb_user_index_lookup('idx_department_name', d.id) {
 						// Combine the rows from tb_user and tb_department
 						joined_row = join(u, d)
 						result.append(joined_row)
 						// Stream results to the next step
 						streamResult(joined_row)
 				}
 		}
 
 		// Step 3: Sort the results by u.name in descending order
 		sorted_result = sort(result, 'u.name', DESC)
 
 		// Step 4: Limit the output to 50 rows
 		final_result = limitResult(sorted_result, 50)

 		return final_result
}
```


# 结论三

对于 sql：

```sql
EXPLAIN ANALYZE SELECT * FROM tb_user u WHERE u.department_id IN (SELECT id FROM tb_department d WHERE d.tree_path LIKE '1-2-%') ORDER BY u.name DESC LIMIT 0, 50 ;
```

最终优化方案是，给 tb_user 加上 index(department_id)、给 tb_department 加上 index(tree_path)

# 总结
在没有开天眼的情况下，通过 EXPLAIN ANALYZE 一步一步的去进行 SQL 语句的优化，并且了解 SQL 执行的底层原理，是非常有效的一种操作，日常工作要精通这个神器的使用，EXPLAIN 和 EXPLAIN ANALYZE 一个很大的区别就是，EXPLAIN 只是一个预估的分析，而 EXPLAIN ANALYZE 则是会真正去执行 SQL，并将整个执行过程返回，每一步耗时多少、扫描了多少行、使用了什么方式都会返回
在没有开天眼的情况下，通过 EXPLAIN ANALYZE 一步一步的去进行 SQL 语句的优化，并且了解 SQL 执行的底层原理，是非常有效的一种操作，日常工作要精通这个神器的使用，EXPLAIN 和 EXPLAIN ANALYZE 一个很大的区别就是，EXPLAIN 只是一个预估的分析，而 EXPLAIN ANALYZE 则是会真正去执行 SQL，并将整个执行过程返回，每一步耗时多少、扫描了多少行、使用了什么方式都会返回

