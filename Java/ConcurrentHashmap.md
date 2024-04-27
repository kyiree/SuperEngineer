# 1 ConcurrentHashmap 是怎么解决并发问题的

以 JDK1.8 为例子，使用了 CAS 和 Synchronized。只有在写操作时才会使用synchronized关键字锁定少量必要的资源，比如只锁定链表的头部或者红黑树的根节点。读则是无锁读。
