1. happer-before、内存屏障 
	1.1 有mesi缓存一致性协议为什么还需要volatile关键字实现可见性  

2. UnSafe、Atomic、CAS、Lock

    2.1 如果CopyOnWriteArrayList使用拷贝原数组的方式实现线程安全是为了遍历/读取可以不加锁

为什么ConcurrentHashMap不需要拷贝原数组也能实现遍历/读取不加锁