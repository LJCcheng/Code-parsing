## ArrayList源码

#### 重要属性

```java
    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
```

#### add(E e)方法

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // 先判定是否需要需要扩容
        elementData[size++] = e;
        return true;
    }

private void ensureCapacityInternal(int minCapacity) {
   		 //先计算容量，后判断是否需要扩容
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }


private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //如果第一次加元素，容量取10和minCapacity中的最大值
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
    //否则返回新的容量
        return minCapacity;
    }

private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
    	//如果新的容量大于当前数组长度，就扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length; //获取当前数组长度
        int newCapacity = oldCapacity + (oldCapacity >> 1); //扩容1.5倍
        if (newCapacity - minCapacity < 0) //如果扩容1.5倍之后还是小于新的容量
            newCapacity = minCapacity; //就把新的容量赋值给当前容量
        if (newCapacity - MAX_ARRAY_SIZE > 0)//如果当前容量大于当前数组最大长度
            newCapacity = hugeCapacity(minCapacity); //重新计算容量
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            //整型变量会发生溢出，容量变负数
            throw new OutOfMemoryError();
    	//如果当前新的容量大于整型最大值 - 8，就返回Integer.MAX_VALUE，否则就返回整型最大值-8 
    	//这里容量为什么要减去8，是因为数组还需要存储 对象头 和 对象填充 的信息
    	//java对象在某些虚拟机 堆内存中由 对象头 ，实例数据，对象填充 组成。一共32字节，所以这个减去8
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

#### add(int index, E element) 方法

```java
public void add(int index, E element) {
        rangeCheckForAdd(index); //判断是否越界

    	//重新计算容量，扩容等
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

#### remove(E e)方法

```java
public E remove(int index) {
        rangeCheck(index); //是否越界

        modCount++;
        E oldValue = elementData(index); //获取老数据

        int numMoved = size - index - 1; //获取需要移动的偏移量
        if (numMoved > 0)
            //将数组的index+1移到index
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
    	//gc
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

#### remove(Object o)方法

```java
public boolean remove(Object o) {
    	//遍历删除
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

private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```

#### clear()方法

```java
public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```

#### addAll(Collection<? extends E> c)方法

```java
public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
    	//计算容量，扩容等
        ensureCapacityInternal(size + numNew);  // Increments modCount
    	//将新数组移到原数组末尾
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
```

#### addAll(int index, Collection<? extends E> c)方法

```java
public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index); //判断是否越界

        Object[] a = c.toArray();
        int numNew = a.length;
    	//计算容量，扩容等
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```

#### removeRange(int fromIndex, int toIndex)方法

```java
protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex; //计算需要移动的偏移量
    	//将toIndex 之后的移到 fromIndex 位置
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
```

#### retainAll(Collection<?> c)方法

```java
public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }

private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    //如果c集合包含当前集合的元素，则把当前集合下标的w位置赋值为c集合元素
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                
                //因为w下标时候的元素都为无用元素，有用元素都在w之前
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

#### 迭代器

```java
private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        Itr() {}

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification(); //判断当前修改次数是否和期望修改次数一致，不一致报错
            int i = cursor;
            if (i >= size)
                //下标大于当前容量
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            //返回i位置元素
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                //
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                //这里将lastRet归为 -1，但是下次调用next()方法的时候又会重新赋值。
                //如果没有调用next()方法，这里重复调用remove()方法，就如第一行代码，抛出异常。
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                //函数式接口重复消费
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```