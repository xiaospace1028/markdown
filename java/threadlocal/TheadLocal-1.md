## 



问题：

* 当一个线程不想用责任链模式传参数，但是下面方法要用上面的参数的时候类似于sessionId、userId、traceId等等，都可以用threadLocal实现
* 当子线程需要父类线程的参数时候怎么传递参数-->InheritableThreadLocal
* InheritableThreadLocal是否适用于多线程-->TransmittableThreadLocal

---

### ThreadLocal、Thread、ThreadLocalMap

-->一开始读的时候很懵逼 **Thread** 包含 **ThreadLocalMap** ，**ThreadLocal** 里面包括 **ThreadLocalMap** 类 ，**ThreadLocalMap** 里面的key 又是 **ThreadLocal**  父与子的关系，真的让人头大，下面我梳理一下里面的关系吧。

从常用的使用方式入手

```java
ThreadLocal<String> local = new ThreadLocal();
local.set("");
String s = local.get();
```

点到get，set方法中以后存在

`ThreadLocalMap map = getMap(t);`

获取map的方法，这个是为了拿到map类，这个map类是在 `createMap(t, value); ` 方法中创建而 **ThreadLocalMap** 是放在 **Thread** 中，所以对于一个**Thread**来说只存在于一个**ThreadLocalMap**<threadLocals>（这里抛开inheritableThreadLocals，后面说到这个），set方法里面存在`map.set(this, value);`，get方法里面`ThreadLocalMap.Entry e = map.getEntry(this);`

那么进一步分析一下ThreadLocalMap

基础知识

```java
int h = k.threadLocalHashCode & (len - 1); //位运算 快速取下标 类似于 5%2 =2 
private static final int INITIAL_CAPACITY = 16;
```

和hashmap一致  初始化大小为16，

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
}
```

ThreadLocalMap 里面存在的 Entry 继承 WeakReference 弱引用 方便gc回收，但是会造成内存泄漏（只要线程没有被复用不会存在这种情况，但是还是手动去除里面存放的value，而且当调用set方法的时候）

```java
 private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
						//向后寻找一个为null的index存下
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
              //找到key的时候将value进行替换
                if (k == key) {
                    e.value = value;
                    return;
                }
              //如果原始的key 被gc不存在就进搜索
                if (k == null) {
                  //搜索数组里面是否存在相同key的ThreadLocal的Entry，但是被gc回收的
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
}
```

replaceStaleEntry 解析

```java
 private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
						
   					//向前搜索Entry key为空的 记下 下标 slotToExpunge ,并且一直推到最前面记下最前面的key为空的i，方便后面清除
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
						//向后搜索Entry 为空的 记下 下标 slotToExpunge 
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
							
              //如果找到发生hash冲突的key老的hash 推到后面的位置，新的i存放旧值
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
              //如果没有找到相同值，就找新的为空的下标记录
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
   				//清除记录下标到结尾
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```



replaceStaleEntry 这个方法也就是帮助找到存在原来的entry 并且清理null key的这么一个东西，其实`expungeStaleEntry(slotToExpunge)`空出当前位置方法 ( 清理为null的key和下推index找到null存放)存在好多地方（是真的很怕内存溢出，几乎所有方法里面都有这个）

```java
private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
						//废话不多说直接置为空
            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;//设置map里面真实的size
						//搜索为空的数据其实也是清理的的一部分
            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                      	// 发生hash冲突的时候怎么办？直接++的方式一直找到一个为空的位置放下，所以其实不建议使用非常多的ThreadLocal在一个Thread里面，
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

还有一个小知识点。魔数

```java
private static final int HASH_INCREMENT = 0x61c88647;
```

是为了离散队列使得分布更加均匀具体看[参考1](https://zhuanlan.zhihu.com/p/40515974)

解决hash冲突的方式

1. 开放定址法
   *  就是ThreadLocalMap 采用的方案，适合总条数不多，hash 冲突不大的时候使用，而且上面采用的是定量增长的方式，找到一个null的位置，存入当然里面还包括其他情况替换下推等处理方式具体看上面源码
2. 链地址法
   *  这个就是hashmap使用的方式，hash冲突就放入链表当中
3. 再哈希法
   *  找到下一个hash方法再计算，直到不冲突
4. 建立公共溢出区
   * 就是把冲突的都放在另一个地方，不在表里面

现在看get方法

```java
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
          //不为空且key一致就返回
            if (e != null && e.get() == key)
                return e;
            else
              //找到下标一致但是key缺不一致的数据
                return getEntryAfterMiss(key, i, e);
        }
```

getEntryAfterMiss方法

```java
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;
						//比较拿到下一个位置++
            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                  //剔除空的key
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

这章讨论了线程中参数可见，[下一章](TheadLocal-2.md) 讨论一下父线程变量子线程的可见性 

---

参考：

1、[从 ThreadLocal 的实现看散列算法](https://zhuanlan.zhihu.com/p/40515974)

2、[解决Hash冲突四种方法](https://blog.csdn.net/yeiweilan/article/details/73412438)