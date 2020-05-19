# ArrayList
## 继承、实现关系

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
## 内部成员介绍
```
    //默认容量为10
    private static final int DEFAULT_CAPACITY = 10;

    //共享的空的数组
    private static final Object[] EMPTY_ELEMENTDATA = {};

    //共享的默认长度的数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    //实际操作的数组对象
    transient Object[] elementData; // non-private to simplify nested class access
```
## 构造方法
```
    //构造方法会初始化数组大小，但是由于List判断是返回size
    //在没有add前，size始终是0,此时不能使用set(),会出现outofbound
    //整个源码里，只能在add方法里看到size增大
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            //对应EMPTY_ELEMENTDATA
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    //还可通过集合Collection类来构造
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
## 方法介绍
### clone
```
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            //注意这里返回的是一个新的对象
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```
### get/set->elementData
```
@SuppressWarnings("unchecked")
    //直接返回数组下标位置的值
    E elementData(int index) {
        return (E) elementData[index];
    }

    /**
     * Returns the element at the specified position in this list.
     *
     * @param  index index of the element to return
     * @return the element at the specified position in this list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        //查看数组下标是否越界
        rangeCheck(index);

        return elementData(index);
    }

    /**
     * Replaces the element at the specified position in this list with
     * the specified element.
     *
     * @param index index of the element to replace
     * @param element element to be stored at the specified position
     * @return the element previously at the specified position
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E set(int index, E element) {
        //查看数组下标是否越界
        rangeCheck(index);
        //同HashMap，替换后会返回旧值
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```
### add -> ensureCapacityInternal

```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //从下标位置开始copy原数组，到一个新的数组，留出下标位置放置新值
        //如果此时数组长度过大，copy数据消耗资源增大，则导致性能下降
        //由于上述性质，如果需要插入/删除的数据离数组末端很近的话，性能就不一定慢
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```
### remove

```
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            //通过copy直接覆盖下标位置后，然后将最后一位置为null
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        //todo -- 这里为何可以让GC工作需要后续研究
        //20200519 -- ArrayList底层是通过数组实现，即ArrayList对象本身对内部数组数据也是引用关系，当数据被置为null，则不再引用，GC启动。
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    
    //同remove逻辑一样，只是没有返回值
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
### grow

```
    
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //1.5倍扩容，且返回值为int！
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        //实际扩容还是通过Arrays.copyOf复制数组
        //这里是返回了一个新的数组对象，并把新数组对象的引用给到elementData
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
### 线程安全
```
跟HashMap一样，线程不安全，有跟HashTable一样给所有方法加synchronized的Vector类。但是还是推荐juc里
```

