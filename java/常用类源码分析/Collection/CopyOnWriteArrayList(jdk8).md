读写分离

读的时候直接就是原来的数组

写的时候拷贝一份然后添加数据，然后赋值回array

```java
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
	
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
```

