说明：concurrentHashMap 是和HashMap 相似的


sizeCtl：判断当前线程个数



put方法：

1. 计算key的hash
2. 如果hash与size-1 获得到数组的下标，如果下标上的node节点为空，cas赋值
3. 如果链表第一个的hash==moved标记，判断是否需要迁移，让当前线程帮助数组迁移
4. 如果下标的数组不为空，那么sync锁住链表第一个节点。如果node的hash>=0就是链表 小于0就是treebin ，这个是在treebin里面构造的
5. 还有一个里面还有一个addCount方法  counterCell 还存在一个缓存行的优化@Contended 注解，更快的计算，负责添加总值 concurentHashMap 存在counterCells+baseCount ，如果cas baseCount不成功，就随机从counterCells 里面取出数据cas 操作，成功后判断是否需要扩容