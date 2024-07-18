## ConcurrentHashMap源码

#### 重要属性

属性和hashMap的属性差不多

```java
private static final float LOAD_FACTOR = 0.75f; //负载因子

private static final int DEFAULT_CAPACITY = 16; //默认容量

private static final int MAXIMUM_CAPACITY = 1 << 30; //最大容量

static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8; //最大容量 - 8

区别：部分属性通过 volatile 进行修饰
    transient volatile Node<K,V>[] table;

	private transient volatile Node<K,V>[] nextTable;

	private transient volatile int sizeCtl;
```

#### 内部类Node

比HashMap中的Node内部类多了`find()`方法，Node的子类`TreeBin`，`ForwardingNode `和 `TreeNode`重写了find()方法，自己实现了find()寻找节点的方法。

```java
Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    //链表循环查找
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
```

#### 构造函数

```java
public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
    	//如果初始容量大于最大容量的一半，容量赋值为最大容量，否则设为最接近初始容量的2的次方
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
```

#### size()方法

```java
public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }

final long sumCount() {
    	//把CounterCell[]数组的元素的值累加返回
    	//这个CounterCell[]数组在每次添加，删除节点的时候都会修改值
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

#### isEmpty()方法

```java
public boolean isEmpty() {
    	//将CounterCell[]中元素累加比较
        return sumCount() <= 0L; // ignore transient negative values
    }
```

#### get(Object key)方法

```java
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    	//重新计算hashCode
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            //如果tab[]中存在
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                //1. eh为-1时，表示正在迁移，需要借助 Node子类ForwardingNode的find()方法来寻找
                //2. eh为-2时，表示节点为TreeBin红黑树节点，调用Node子类TreeBin的find()方法来寻找
                return (p = e.find(h, key)) != null ? p.val : null;
            //链表节点遍历链表
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }

static final int spread(int h) {
    	//将高位辐射到低位，更加均匀分布，减少hash冲突
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

#### Node子类ForwardingNode

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
    
    Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    //nextTable为空直接返回null
                    return null;
                for (;;) {
                    int eh; K ek;
                    //如果就是当前节点直接返回
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
                        //如果是特殊节点
                        //1. 转发节点，继续转发
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            //2. 红黑树查找
                            return e.find(h, k);
                    }
                    //所有节点全部遍历完没找到返回null
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
}
```

#### Node子类TreeBin

- `find()`方法

  ```java
  final Node<K,V> find(int h, Object k) {
              if (k != null) {
                  //循环遍历当前节点
                  for (Node<K,V> e = first; e != null; ) {
                      int s; K ek;
                      //如果已经加锁
                      if (((s = lockState) & (WAITER|WRITER)) != 0) {
                          if (e.hash == h &&
                              ((ek = e.key) == k || (ek != null && k.equals(ek))))
                              return e;
                          //遍历
                          e = e.next;
                      }
                      //加读锁成功
                      else if (U.compareAndSwapInt(this, LOCKSTATE, s,
                                                   s + READER)) {
                          TreeNode<K,V> r, p;
                          try {
                              //调用TreeNode 的 find()方法
                              p = ((r = root) == null ? null :
                                   r.findTreeNode(h, k, null));
                          } finally {
                              Thread w;
                              //执行结束，如果有写操作在等待读操作完成并且读操作已经全部结束
                              if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                                  (READER|WAITER) && (w = waiter) != null)
                                  //唤醒写线程
                                  LockSupport.unpark(w);
                          }
                          return p;
                      }
                  }
              }
              return null;
          }
  ```

#### Node子类TreeNode

```java
Node<K,V> find(int h, Object k) {
            return findTreeNode(h, k, null);
        }

        /**
         * Returns the TreeNode (or null if not found) for the given key
         * starting at given root.
         */
        final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
            if (k != null) {
                TreeNode<K,V> p = this;
                //循环当前树节点
                do  {
                    int ph, dir; K pk; TreeNode<K,V> q;
                    TreeNode<K,V> pl = p.left, pr = p.right;
                    if ((ph = p.hash) > h)
                        //左子树
                        p = pl;
                    else if (ph < h)
                        //右子树
                        p = pr;
                    else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                        //找到结束
                        return p;
                    else if (pl == null)
                        //左子树为null，赋值为 右子树
                        p = pr;
                    else if (pr == null)
                        //右子树为null，赋值为 左子树
                        p = pl;
                    else if ((kc != null ||
                              (kc = comparableClassFor(k)) != null) &&
                             (dir = compareComparables(kc, k, pk)) != 0)
                        p = (dir < 0) ? pl : pr;
                    else if ((q = pr.findTreeNode(h, k, kc)) != null)
                        //递归查询
                        return q;
                    else
                        p = pl;
                } while (p != null);
            }
            return null;
        }
```

#### put(K key, V value)方法

```java
public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        //计算hash
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //不存在则cas比较替换
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                //hash为-1表示正在迁移，这里帮助迁移
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //加锁，锁住f，分段锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            //hash>= 0表示链表，遍历链表
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    //链表中找到，判断是否覆盖
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    //插入链表尾部
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            //调用红黑树putTreeVal方法
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        //容量大于8树化
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        //往CounterCell[]设置值，就是size()方法中的数据源
        addCount(1L, binCount);
        return null;
    }

private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                //sizeCtl为-1表示在初始化中等待
                Thread.yield(); // lost initialization race; just spin
            //将sizeCtl cas 替换为 -1
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        //初始化容量，创建新Node[]数组
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //最后将sizeCtl 替换为 sc，表示初始化完成
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

#### remove(Object key)方法

```java
public V remove(Object key) {
        return replaceNode(key, null, null);
    }

    /**
     * Implementation for the four public remove/replace methods:
     * Replaces node value with v, conditional upon match of cv if
     * non-null.  If resulting value is null, delete.
     */
    final V replaceNode(Object key, V value, Object cv) {
        //计算hash
        int hash = spread(key.hashCode());
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                //不存在，直接结束
                break;
            else if ((fh = f.hash) == MOVED)
                //在迁移中，帮助迁移
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                boolean validated = false;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            //hash>=0表示链表
                            validated = true;
                            for (Node<K,V> e = f, pred = null;;) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    V ev = e.val;
                                    if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                        oldVal = ev;
                                        if (value != null)
                                            //新值将老值替换
                                            e.val = value;
                                        else if (pred != null)
                                            //将节点下一节点指向前一节点的next属性，即跳过当前节点，指向下一节点
                                            pred.next = e.next;
                                        else
                                            //将 i 节点用 e.next替换
                                            setTabAt(tab, i, e.next);
                                    }
                                    break;
                                }
                                pred = e;
                                if ((e = e.next) == null)
                                    //遍历结束
                                    break;
                            }
                        }
                        else if (f instanceof TreeBin) {
                            //红黑树
                            validated = true;
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                //红黑树中找到节点
                                V pv = p.val;
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    //新值替换老值
                                    if (value != null)
                                        p.val = value;
                                    else if (t.removeTreeNode(p))
                                        //红黑树中删除成功，更新tab
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                    }
                }
                if (validated) {
                    if (oldVal != null) {
                        if (value == null)
                            //修改size
                            addCount(-1L, -1);
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }
```

#### clear()方法

```java
public void clear() {
        long delta = 0L; // negative number of deletions
        int i = 0;
        Node<K,V>[] tab = table;
        while (tab != null && i < tab.length) {
            int fh;
            Node<K,V> f = tabAt(tab, i);
            if (f == null)
                //当前节点为null跳过
                ++i;
            else if ((fh = f.hash) == MOVED) {
                //帮助迁移
                tab = helpTransfer(tab, f);
                i = 0; // restart
            }
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //获取节点
                        Node<K,V> p = (fh >= 0 ? f :
                                       (f instanceof TreeBin) ?
                                       ((TreeBin<K,V>)f).first : null);
                        while (p != null) {
                            --delta;
                            p = p.next;
                        }
                        //设置tab[i]为null，清除引用
                        setTabAt(tab, i++, null);
                    }
                }
            }
        }
        if (delta != 0L)
            //改变size
            addCount(delta, -1);
    }
```

#### transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)方法

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                //如果nextTab为null，新建nextTab
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
    	//创建ForwardingNode节点，这个节点的hash就是-1，保存的nextTab的引用
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    //迁移完成
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            
            //i越界情况
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    //迁移完成
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                //桶为空，设置为正在迁移
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                //表示正在迁移
                advance = true; // already processed
            else {
                //这里锁f，分段锁住tab[]数组i元素，对i元素进行迁移
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            //链表
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            //新建链表
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            //cas替换nextTab
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            //红黑树
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            
                            //新建红黑树
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            
                            //cas比较替换
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

#### 树化

```java
/**
 * putVal()方法容量大于8就会调用这个方法，实际上容量大于64才会树化
 */
private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                //容量小于64，扩容
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                //分段锁住，树化
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            //循环创建树节点
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        //替换tab[]
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```



