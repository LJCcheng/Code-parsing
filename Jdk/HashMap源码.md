## HashMap源码

#### 重要属性

```java
	static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16 默认容量

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30; //最大容量

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f; //默认负载因子

	int threshold; //阈值

	transient Node<K,V>[] table; //node数组

	final float loadFactor; //自定义负载因子
```

#### Node内部类

**HashMap内部元素**，包含`node`数组，`node`链表和 `TreeNode`

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

#### get(Object key)方法

```java
public V get(Object key) {
        Node<K,V> e;
    	//先计算key的hash值，然后通过hash值和key去获取节点
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

static final int hash(Object key) {
        int h;
    	//计算hash
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    	//如果map不为空，且hash值存在
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //这里直接先判断链表的第一个元素是否为要找的key
           	//hash相等且key相等直接返回
            //这里之所以先比较第一个节点，主要是因为这里的链表是为了解决hash冲突
            //如果第一个节点就是需要的key，就直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                //如果node节点属于树节点，就去树里面寻找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    //反之这里就是链表，遍历链表寻找
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

#### put(K key, V value)方法

```java
public V put(K key, V value) {
    	//添加元素
        return putVal(hash(key), key, value, false, true);
    }

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
    	//这里如果map为空，扩容
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            //如果node[]数组中不存在对应hash，证明之前不存在对应key，直接插入即可
            tab[i] = newNode(hash, key, value, null);
        else {
            //表示node[]节点之中已经存在对应key，即hash冲突
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //这里表示就是node[]中的对应链表第一个节点
                e = p;
            else if (p instanceof TreeNode)
                //节点属于树，则调用树的方法
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //如果遍历到链表尾部，还未找到key，就创建新结点插入链表的尾部
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //链表长度大于8，进行树化
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        //找到对应key，结束循环
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    //如果允许覆盖或者老值为null，就替换原值
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            //如果当前容量大于阈值，扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

#### resize()扩容

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
    	//获取老的容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
    	//获取老的阈值
        int oldThr = threshold;
    	//新的容量和阈值
        int newCap, newThr = 0;
    	//老的容量大于0
        if (oldCap > 0) {
            //老容量大于map最大容量
            if (oldCap >= MAXIMUM_CAPACITY) {
                //设定阈值为 Integer.MAX_VALUE
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //老的容量扩容两倍后小于map最大容量且老的容量大于默认容量16
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //老的阈值也扩大两倍
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            //如果老的容量小于等于0并且老的阈值大于0
            //就将老的阈值赋值给新的容量
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //hashMap默认容量16和默认阈值12
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            //如果新的阈值等于0，重新计算新的阈值
            float ft = (float)newCap * loadFactor;
            //和map最大容量进行比较
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
    	//重新赋值当前阈值
        threshold = newThr;
    	//创建新的node[]数组
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            //对老的node[]数组进行遍历
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        //表示不存在hash冲突，直接插入新的node[]数组
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //属于树结构，对树进行处理
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //对hash冲突形成的链表进行处理
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //这里对结点的hash和老的容量进行计算，将数据进行拆分为2段，低位lotail和高位hitail
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                //将e节点赋值给loTail，while循环重复操作
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                //将e节点赋值给hiTail，while循环重复操作
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //将链表插入到新的node[]数组
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

#### putAll(Map<? extends K, ? extends V> m)方法

```java
public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
                //如果map的node[]数组为null，重新计算当前容量
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    //重新计算阈值，将阈值设置为大于当前容量的第一个2的次方
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                //扩容
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                //循环单个插入
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```

#### remove(Object key)方法

```java
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            //在删除之前需要先找到对应node节点
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //直接就在node[]数组中找到节点
                node = p;
            else if ((e = p.next) != null) {
                //hash冲突
                if (p instanceof TreeNode)
                    //从树中获取
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            //遍历链表寻找对应node节点
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //找到node节点
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    //树中删除
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    //如果为链表的第一个元素，即头结点
                    tab[index] = node.next;
                else
                    //存在于链表中，就将node下一节点指向p的下一节点
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

#### clear()方法

```java
public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            //size设为0
            size = 0;
            //遍历清空node[]数组
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
```

#### containsValue(Object value)方法

```java
public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            //遍历node[]数组
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    //遍历链表
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```

#### 内部类`KeySet，Values，EntrySet`

```java
1. keySet只能拿到key，values只能拿到value，entrySet既可以拿到key，又可以拿到value
2. keySet这里继承了AbstractSet<K>，values继承了AbstractCollection<V>，entrySet继承了AbstractSet<Map.Entry<K,V>>
3. keySet对key进行操作，values对value进行操作，entrySet内部用的map.entry，所以都可以操作
```

#### computeIfAbsent()方法

```java
public V computeIfAbsent(K key,
                             Function<? super K, ? extends V> mappingFunction) {
        if (mappingFunction == null)
            throw new NullPointerException();
        int hash = hash(key);
        Node<K,V>[] tab; Node<K,V> first; int n, i;
        int binCount = 0;
        TreeNode<K,V> t = null;
        Node<K,V> old = null;
    	//先找到key对应的节点
        if (size > threshold || (tab = table) == null ||
            (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((first = tab[i = (n - 1) & hash]) != null) {
            if (first instanceof TreeNode)
                old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
            else {
                Node<K,V> e = first; K k;
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) {
                        old = e;
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
            V oldValue;
            //如果找到了就返回，不做任何操作
            if (old != null && (oldValue = old.value) != null) {
                afterNodeAccess(old);
                return oldValue;
            }
        }
    	//没找到，就执行函数式接口方法，
        V v = mappingFunction.apply(key);
        if (v == null) {
            return null;
        } else if (old != null) {
            //找到key，但是value为null，赋新值
            old.value = v;
            afterNodeAccess(old);
            return v;
        }
    	//不存在节点进行插入
        else if (t != null)
            //往树中插入
            t.putTreeVal(this, tab, hash, key, v);
        else {
            //新节点
            tab[i] = newNode(hash, key, v, first);
            if (binCount >= TREEIFY_THRESHOLD - 1)
                //树化
                treeifyBin(tab, hash);
        }
        ++modCount;
        ++size;
        afterNodeInsertion(true);
        return v;
    }
```

#### computeIfPresent方法

```java
public V computeIfPresent(K key,
                              BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (remappingFunction == null)
            throw new NullPointerException();
        Node<K,V> e; V oldValue;
        int hash = hash(key);
        if ((e = getNode(hash, key)) != null &&
            (oldValue = e.value) != null) {
            //找到对应key，就执行函数式方法
            V v = remappingFunction.apply(key, oldValue);
            if (v != null) {
                e.value = v;
                afterNodeAccess(e);
                return v;
            }
            else
                removeNode(hash, key, null, false, true);
        }
        return null;
    }
```

#### compute方法

```java
public V compute(K key,
                     BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (remappingFunction == null)
            throw new NullPointerException();
        int hash = hash(key);
        Node<K,V>[] tab; Node<K,V> first; int n, i;
        int binCount = 0;
        TreeNode<K,V> t = null;
        Node<K,V> old = null;
    	//先找到对应node节点
        if (size > threshold || (tab = table) == null ||
            (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((first = tab[i = (n - 1) & hash]) != null) {
            if (first instanceof TreeNode)
                old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
            else {
                Node<K,V> e = first; K k;
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) {
                        old = e;
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
        }
        V oldValue = (old == null) ? null : old.value;
    	//执行函数式方法
        V v = remappingFunction.apply(key, oldValue);
    	//存在节点进行替换
        if (old != null) {
            if (v != null) {
                //如果节点存在并且函数式方法执行后结果不为null
                old.value = v;
                afterNodeAccess(old);
            }
            else
                //否则删除节点
                removeNode(hash, key, null, false, true);
        }
    	//不存在节点新建节点
        else if (v != null) {
            //插入树
            if (t != null)
                t.putTreeVal(this, tab, hash, key, v);
            else {
                //创建新节点
                tab[i] = newNode(hash, key, v, first);
                if (binCount >= TREEIFY_THRESHOLD - 1)
                    //树化
                    treeifyBin(tab, hash);
            }
            ++modCount;
            ++size;
            afterNodeInsertion(true);
        }
        return v;
    }
```

