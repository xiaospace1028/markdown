SynchronousQueue 无缓冲阻塞队列

线程阻塞队列，生产线程生产的东西没有被消费掉，当前生产线程不能继续执行，直到生产的东西被消费



构造方法

```java
 public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
```

* 公平->TransferQueue 队列
* 非公平->TransferStack 堆栈

堆栈

```java
  E transfer(E e, boolean timed, long nanos) {
              SNode s = null; // constructed/reused as needed
            int mode = (e == null) ? REQUEST : DATA; //判断是消费者还是生产者
            for (;;) {
                SNode h = head;//拿到第一个
                if (h == null || h.mode == mode) {  // 如果为空或者第一个和传入的责任是一样的
                    if (timed && nanos <= 0) {      //如果不等待
                        if (h != null && h.isCancelled())
                            casHead(h, h.next);     //把现在头部next的对象变成头部对象
                        else
                            return null;//如果头为空直接返回
                    } else if (casHead(h, s = snode(s, e, h, mode))) {//头对象和新对象替换位置，因为下面会阻塞了所以头对象是空的
                        SNode m = awaitFulfill(s, timed, nanos);
                        if (m == s) {               // wait was cancelled
                            clean(s);
                            return null;
                        }
                        if ((h = head) != null && h.next == s)
                            casHead(h, s.next);     // help s's fulfiller
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    }
                } else if (!isFulfilling(h.mode)) { // try to fulfill
                    if (h.isCancelled())            // already cancelled
                        casHead(h, h.next);         // pop and retry
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                        for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       // m is s's match
                            if (m == null) {        // all waiters are gone
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
                            SNode mn = m.next;
                            if (m.tryMatch(s)) {
                                casHead(s, mn);     // pop both s and m
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                            } else                  // lost match
                                s.casNext(m, mn);   // help unlink
                        }
                    }
                } else {                            // help a fulfiller
                    SNode m = h.next;               // m is h's match
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else {
                        SNode mn = m.next;
                        if (m.tryMatch(h))          // help match
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }
```

阻塞方法

```java
SNode awaitFulfill(SNode s, boolean timed, long nanos) {
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Thread w = Thread.currentThread();
            int spins = (shouldSpin(s) ?//拿到自旋次数
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);
            for (;;) {
                if (w.isInterrupted())
                    s.tryCancel();//至为空不知道是做什么
                SNode m = s.match;//这里是什么意思？
                if (m != null)
                    return m;
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        s.tryCancel();
                        continue;
                    }
                }
                if (spins > 0)//自旋次数大于0
                    spins = shouldSpin(s) ? (spins-1) : 0;//减少一次自旋次数
                else if (s.waiter == null)//对象的线程不存在的时候
                    s.waiter = w; // 把当前线程设置进去并且等待下一次自旋
                else if (!timed)//没有超时判定的时候，直接阻塞
                    LockSupport.park(this);//阻塞
                else if (nanos > spinForTimeoutThreshold)//超过设定的最低的等待时间
                    LockSupport.parkNanos(this, nanos);//定时阻塞
            }
        }
```

