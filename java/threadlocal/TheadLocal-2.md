## InheritableThreadLocal

 [上一章](TheadLocal-1.md) 讲了ThreadLocal源码，这一章讲一下关于**InheritableThreadLocal**就是子线程中需要使用父线程的相关字段用到的类，我们先看一下下面这段代码

```java
    public static void main(String[] args) throws InterruptedException {
        InheritableThreadLocal local = new InheritableThreadLocal();
        ExecutorService fixedThreadPool = Executors.newFixedThreadPool(1);
        local.set("local--初始化");
        Runnable task = () -> {
            String name1 = Thread.currentThread().getName();
            Object o = local.get();
            System.out.println(name1 + ":local" + o);
        };
        fixedThreadPool.submit(task);//第一次
        local.set("local--修改");
        fixedThreadPool.submit(task);//第二次
    }
```

结果

```console
pool-1-thread-1:locallocal--初始化
pool-1-thread-1:locallocal--初始化
```

也就是说在第一次确实能拿到父类线程中的，但是第二次当父类线程把值给变换的时候子线程缺没有相关联的变，那么这个原因是什么？有什么办法解决呢？

原因：

Thread#init 方法里面

```java
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }
        this.name = name;
      //主线程创建子线程，当前还在主线程
        Thread parent = currentThread();
       .....省略代码....校验权限
       
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
          //拷贝到子线程
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
```

所以他只是在创建线程的时候拷贝父线程的`inheritableThreadLocals`并没有一个跟随的检测的机制，可以适合TraceId，但是如果有需求想要创建有主线程变更后会更新的功能，需要使用阿里巴巴开源的jar包[TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local) 里面有三种使用方式，agent，修饰线程池，修饰Thread*的方式使用方法以及对比效果

```java
    public static void main(String[] args) throws InterruptedException {
        InheritableThreadLocal local = new InheritableThreadLocal();
        TransmittableThreadLocal transLocal = new TransmittableThreadLocal();
        ExecutorService fixedThreadPool = Executors.newFixedThreadPool(1);
        local.set("local--初始化");
        transLocal.set("transLocal--初始化");
        Runnable task = () -> {
            String name1 = Thread.currentThread().getName();
            Object o = local.get();
            Object transO = transLocal.get();
            System.out.println(name1 + ":local" + o);
            System.out.println(name1 + ":transLocal" + transO);
        };
        fixedThreadPool.submit(TtlRunnable.get(task));
        local.set("local--修改");
        transLocal.set("transLocal--修改");
        fixedThreadPool.submit(TtlRunnable.get(task));
    }
```

结果

```console
pool-1-thread-1:locallocal--初始化
pool-1-thread-1:transLocaltransLocal--初始化
pool-1-thread-1:locallocal--初始化
pool-1-thread-1:transLocaltransLocal--修改
```

[下一章](TheadLocal-3.md)讨论一下`TransmittableThreadLocal`这个包的源码

