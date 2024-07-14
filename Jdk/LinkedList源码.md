## LinkedList源码

#### 重要属性

```java
	transient int size = 0; 

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
```

#### addAll(int index, Collection<? extends E> c)方法

```java
public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index); //检查是否越界

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false; //容量为0直接返回

    	//初始化数据
        Node<E> pred, succ;
        if (index == size) {
            //队尾
            //如果当前下标index 为 size ，则当前节点为null，前一个节点为 last
            succ = null;
            pred = last;
        } else {
            //否则获取当前节点和前一节点
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            //创建新结点
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                //如果前一节点为null，表示为第一个节点
                first = newNode;
            else
                //否则将新结点插入前一节点的后面
                pred.next = newNode;
            
            //将当前节点赋值为前一节点，循环调用，为每个node节点添加它的下一节点
            pred = newNode;
        }

    	//这里是已经将所有node节点全部插入完成之后执行逻辑
        if (succ == null) {
            //队尾
            //将循环之后的pred赋值为last节点，即最后一个节点
            last = pred;
        } else {
            //将succ的前一节点指向pred，pred后一节点指向succ
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
```

#### add(E e)方法

```java
public boolean add(E e) {
        linkLast(e);//加入队尾
        return true;
    }

void linkLast(E e) {
    	//队尾结点
        final Node<E> l = last;
    	//创建新结点
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            //链表为空
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

#### addFirst(E e)方法

```java
public void addFirst(E e) {
        linkFirst(e);
    }

private void linkFirst(E e) {
    	//加入链表头部
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            //链表为空
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```

#### linkBefore(E e, Node<E> succ)方法

```java
void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
    	//获取当前节点前一节点
        final Node<E> pred = succ.prev;
    	//创建含有前节点和后节点的新结点
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            //将前一节点的下一节点指向新结点
            pred.next = newNode;
        size++;
        modCount++;
    }
```

#### removeFirst方法

```java
public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
    	//头结点设为next
        first = next;
        if (next == null)
            last = null;
        else
            //前一节点设为null
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```

#### remove(int index)方法

```java
public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index)); //删除index节点
    }

E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item; //获取当前节点元素
        final Node<E> next = x.next; //获取后一节点
        final Node<E> prev = x.prev; //获取前一节点

        if (prev == null) {
            //赋值头结点
            first = next;
        } else {
            //修改前一节点的下一节点属性
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            //尾结点赋值
            last = prev;
        } else {
            //修改后一节点的前一节点属性
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

#### remove(Object o)方法

```java
public boolean remove(Object o) {
    	//遍历删除
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
```

#### node(int index)方法

```java
Node<E> node(int index) {
        // assert isElementIndex(index);
    	//这里判断是从头开始遍历还是从尾开始遍历
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

#### peek()方法

```java
public E peek() {
    final Node<E> f = first;
    //直接返回头结点
    return (f == null) ? null : f.item;
}
```

#### poll()方法

```java
public E poll() {
    	//删除头结点
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }
```

#### add(int index, E element)方法

```java
public void add(int index, E element) {
        checkPositionIndex(index); //判断是否越界

    	//判断是插入尾部还是插入index下标之前
        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
```



