# fail

> LinkedBlockingQueue底层实现原理

> ArrayBlockingQueue、LinkedBlockingQueue 、ConcurrentLinkedQueue的实现区别

> 线程池任务拒绝策略忘了

> Timer缺陷

> WorkStealingPool?

> ForkJoinPool? parallel stream

> 单例模式-枚举（模板方法模式）

> sql执行流程（产生redolog，binlog）

> select 语句各部分执行顺序

> redo log和顺序读写/写磁盘有关系？

> 为什么隔离级别要用RR

> RR如何解决脏读和不可重复读（主要是当前读 ）

> undolog清理时机

> 索引分类（聚簇索引、非聚簇索引）

> like "%abc" 怎么走索引（倒序存cba）

# fail2

put流程 忽略了讲述hash值比较和equal比较、数组阈值 核心原理：hash值计算数组下标、equals解决哈希冲突 

为什么要用红黑树（应该可以），为什么不用二叉查找树（红黑树平衡 ）， 为什么不直接用红黑树：红黑树缺点

ConcurrentHashMap线程 安全原理、synchronized优化

内存碎片导致我们没办法分配大量连续的内存，从而导致多次触发GC

三种GC算法的场景。标记复制适合存活少，回收多的场景。分代：年轻代标记复制 

GC Root包含哪些  

Mysql 响应慢：1. 有慢查询，看慢查询日志，explain，改造sql或者索引。2. 表比较大，水平切分/垂直切分，分库

3.读写分离 4.缓存   

> 查询 、 索引、单表数据量、流量、  热点数据



为什么索引比不走索引慢  先说清楚B+树结构 能不能从数据页的角度说一些

B+树索引怎么处理范围查询