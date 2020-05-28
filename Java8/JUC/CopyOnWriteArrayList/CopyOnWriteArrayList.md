# CopyOnWriteArrayList
## 继承、实现关系

```
//基本与ArrayList一样
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## 内部成员介绍
```
    //首当其冲的就是可重入锁
    final transient ReentrantLock lock = new ReentrantLock();
    //私有的实际操作对象，仅仅可通过getArray/setArray方法操作
    private transient volatile Object[] array;
    final Object[] getArray() {
        return array;
    }

    final void setArray(Object[] a) {
        array = a;
    }
```
## 构造方法

```
    //默认是创建一个空的Object数组
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }
    
    //下列核心还是通过Arrays.copyOf来复制一个对象
    //也可以传集合
    public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            elements = c.toArray();
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elements.getClass() != Object[].class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }
    
    //传一个复制的数组
    public CopyOnWriteArrayList(E[] toCopyIn) {
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }
```

## get/set 方法

```
    //get不涉及线程安全，通过volatile修饰的array，可实时从主存中获取最新数据
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
    
    public E get(int index) {
        return get(getArray(), index);
    }
    
    //set开始就可以看到可重入锁
    public E set(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            E oldValue = get(elements, index);

            if (oldValue != element) {
                int len = elements.length;
                //如果设置的新值与旧值不同，则会copy原有数组
                Object[] newElements = Arrays.copyOf(elements, len);
                //然后将copy的新数组的index设置为新值
                newElements[index] = element;
                然后更新到对象存储数组中
                setArray(newElements);
            } else {
                // Not quite a no-op; ensures volatile write semantics
                setArray(elements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```

## add/remove 方法
```
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            //直接向末位添加时，也是会复制一个新的len+1大小的数组
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            //然后把最后一位赋值
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    public void add(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //向索引位置添加时
            Object[] elements = getArray();
            int len = elements.length;
            //先判断索引位置有效性
            if (index > len || index < 0)
                throw new IndexOutOfBoundsException("Index: "+index+
                                                    ", Size: "+len);
            Object[] newElements;
            int numMoved = len - index;
            //如果索引位位末位就跟add(E e)方法一样
            if (numMoved == 0)
                newElements = Arrays.copyOf(elements, len + 1);
            //否则会跟ArrayList一样，按照索引位复制两次数组(效率较低)
            else {
                newElements = new Object[len + 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index, newElements, index + 1,
                                 numMoved);
            }
            newElements[index] = element;
            setArray(newElements);
        } finally {
            lock.unlock();
        }
    }
    
    public boolean remove(Object o) {
        Object[] snapshot = getArray();
        int index = indexOf(o, snapshot, 0, snapshot.length);
        return (index < 0) ? false : remove(o, snapshot, index);
    }

    
    //remove方法类似add，只是通过两次数组复制将索引位覆盖
    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                Object[] newElements = new Object[len - 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                setArray(newElements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
    
    private boolean remove(Object o, Object[] snapshot, int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] current = getArray();
            int len = current.length;
            if (snapshot != current) findIndex: {
                int prefix = Math.min(index, len);
                for (int i = 0; i < prefix; i++) {
                    if (current[i] != snapshot[i] && eq(o, current[i])) {
                        index = i;
                        break findIndex;
                    }
                }
                if (index >= len)
                    return false;
                if (current[index] == o)
                    break findIndex;
                index = indexOf(o, current, index, len);
                if (index < 0)
                    return false;
            }
            Object[] newElements = new Object[len - 1];
            System.arraycopy(current, 0, newElements, 0, index);
            System.arraycopy(current, index + 1,
                             newElements, index,
                             len - index - 1);
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

## 结合理解

```
线程安全：直接明了 -- ReentrantLock lock = new ReentrantLock()
写时复制：对ArrayList修改时，会System.arraycopy()，从而不影响读操作
```


