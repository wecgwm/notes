1. happer-before、内存屏障 
	1.1 有 mesi 缓存一致性协议为什么还需要 volatile 关键字来实现可见性  

2. UnSafe、Atomic、CAS、Lock

    2.1 如果 CopyOnWriteArrayList 使用拷贝原数组的方式实现线程安全是为了遍历/读取可以不加锁；那为什么 ConcurrentHashMap不需要拷贝原数组也能实现遍历/读取不加锁
3. Leetcode多线程题
   
    
