举个例子，我需要实现一个查看哪些顾客存在过购买行为的需求，customer 表代表顾客表，order 代表订单表，一个 customer 对应零个或多个 order，一个 order 必须只能对应一个 customer

下面两条常见的语句都能实现需求：

语句一：
SELECT customer.* FROM customer WHERE EXISTS (SELECT 1 FROM order WHERE order.customer_id = customer.id);

语句二：
SELECT DISTINCT customer.* FROM customer JOIN order ON customer.id = order.customer_id;

对于语句一，使用 EXISTS 做半连接，在 Nested Loop Join 的内循环中，只要匹配到一条数据，就会立马中断跳出内循环，相当于一个短路操作。

而对于语句二，使用 JOIN 做全连接，会将 customer 表的所有数据和 order 表的所有数据做全匹配，所以会查询到相同的多个顾客，还要在外层去做唯一性，性能是非常差的。
