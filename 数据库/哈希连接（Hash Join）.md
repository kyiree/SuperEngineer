举个例子，我需要实现一个查看哪些顾客存在过购买行为的需求，customer 表代表顾客表，order 代表订单表，一个 customer 对应零个或多个 order，一个 order 必须只能对应一个 customer。

SELECT customer.* FROM customer WHERE EXISTS (SELECT 1 FROM order WHERE order.customer_id = customer.id);

假设 customer 有 m 条，order 有 n 条

如果使用 Nest Loop Join 算法，上面 SQL 语句的 IO 耗时会是 m * log(n)*io_factor

如果使用 Hash Join 算法，上面 SQL 语句的 IO 耗时会是 m + C，C 是一个常量

我来讲解一下这个过程：首先全表扫描 order 表，构建一个内存的 BitMap，将 order.customer_id % BitMap.size() 映射进去，BitMap 赋值完成后，紧接着扫描 customer 表，如果 customer.id % BitMap.size() 在相应的 BitMap 数组索引中值为 1，那么就代表顾客存在过购买行为。这个过程只需要 IO 扫描一次 order 表，把所有的 order 信息记录到内存中，采用 BitMap 来压缩信息。后续基于 customer 的条件比较都只需要在内存中进行而不用多次重复 IO 扫描 order 表
