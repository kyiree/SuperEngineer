# 1 string

# 2 list

# 3 set

# 4 zset

redis的有序集合（zset）主要是根据分数（score）来进行排序的，这是它的核心特性之一。每个元素在有序集合中都有一个与之关联的分数，这个分数用于确定元素在集合中的排序位置。当有序集合进行操作如插入、删除或查找元素时，redis会首先根据分数进行排序。如果多个元素有相同的分数，那么这些元素会根据它们的元素值（通常是字符串）在字典序中的排列进行二级排序。

Zset 类型的底层数据结构是由 (listpack) 或 (zskiplist + 哈希) 实现的：

- 如果有序集合的元素个数小于 xxx 个，并且每个元素的值小于 xxx 字节时，redis 会使用 listpack 作为 zset 类型的底层数据结构，这个 xxx 可以配置。
- 如果有序集合的元素不满足上面的条件，redis 会使用 zskiplist + 哈希作为 zset 类型的底层数据结构。在哈希中，key 是元素，值的对应的分数。在 zskiplist 中，将元素和分数放到 zskiplistNode 里，元素在 zskiplist 里按照分数进行排序。（哈希的目的是根据元素查找分数，zskiplist 的目的是根据分数查找元素，两个是相反的操作）

## 4.1 当 zset 底层使用 listpack 时，是怎么插入的？

listpack 存储元素是按照（元素1，分数1，元素2，分数2......）这样的格式来存储的，元素和相应的分数一前一后紧靠在一起，分数按照从小到大的顺序排列，插入元素的时候，先通过分数顺序遍历 listpack，确定元素要插入哪个位置，如果分数相同，则按照元素的字典序进行排序

## 4.2 当 zset 底层使用 zskiplist 时，是怎么插入的？

这里只讨论元素不存在于 zskiplist 的情况。

redis 将元素、分数、层级关系封装在 zskiplistNode 这个数据结构里面。skiplist 维护了头zskiplistNode 和尾zskiplistNode ，每次插入元素的时候都从头部开始插入，首先生成一个随机层高度，平均每个节点的层数是 1.33。

用 update 数组记录了每层应更新的节点，去更新它们的层级引用。在跳表中，查找插入位置先从最上层开始查找，真正实现插入则从最底层开始修改。

[跳表是怎么实现插入的](https://softwareengineering.stackexchange.com/questions/287254/how-does-a-skip-list-work)

# 5 hash
