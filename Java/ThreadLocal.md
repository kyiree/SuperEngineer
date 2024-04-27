# 1 用途

ThreadLocal 是 Java 中的一个工具类，用于创建线程变量。实现线程隔离。在项目中，例如事务上下文、用户会话上下文都可以设置到 ThreadLocal 里面，这些数据只有当前线程才能访问。

# 2 底层原理与实现

每个线程持有一个自己的 ThreadLocalMap 实例，每个 ThreadLocal 对象作为键，线程局部变量的值作为映射值。当通过 ThreadLocal.set(value) 方法设置线程局部变量的值时，ThreadLocal 会检查当前线程的 ThreadLocalMap，如果不存在，则创建一个新的 ThreadLocalMap 并将其与当前线程关联。然后，它使用自身 (ThreadLocal 实例) 作为键，将值存储到这个线程的 ThreadLocalMap 中。ThreadLocal 使用弱引用（WeakReference）作为键的引用类型，这意味着一旦 ThreadLocal 对象没有其他强引用指向它，它就可以被垃圾收集器回收。这种设计可以防止因 ThreadLocal 引起的内存泄漏问题。但是，如果不手动调用 ThreadLocal.remove() 清理，ThreadLocalMap 中存储的值仍然可能导致内存泄漏。ThreadLocalMap 使用线性探测法来解决哈希冲突，这意味着如果一个键的计算哈希码已经被占用，它会顺序查找下一个空闲的槽位来存储这个键值对。当调用 ThreadLocal.get() 方法时，ThreadLocal 会访问当前线程的 ThreadLocalMap，并使用自己作为键来获取对应的值。如果找到了值，则返回；如果没有找到，则调用 ThreadLocal 的 initialValue() 方法来提供一个初始值null并存储在 ThreadLocalMap 中。
