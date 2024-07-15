## HashSet源码

#### 重要属性

hashset本质就是hashmap，利用了hashMap 的key

```java
	private transient HashMap<E,Object> map; //核心hashmap

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object(); //所有key都存的value
```

#### 构造函数

```java
public HashSet(Collection<? extends E> c) {
    	//新建hashMap，c集合容量除以负载因子0.75并且加一计算出新的容量，然后和默认的hashMap容量16做比较
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    	//这里调用的父类AbstractCollection的addAll()方法
        addAll(c);
    }

public boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c)
            //这里调用父类的add()方法，由于hashSet自己重写了add()方法，所以执行hashSet的add()方法
            if (add(e))
                modified = true;
        return modified;
    }

public boolean add(E e) {
    	//本质就是往hashMap中加入key
        return map.put(e, PRESENT)==null;
    }
```

还有其他构造函数，都是重载方法，为了给内部hashMap设置不同的负载因子或者初始容量

#### 迭代器

```java
public Iterator<E> iterator() {
    	//调用hashMap 的内部类 keySet
        return map.keySet().iterator();
    }
```

#### add(E e)方法

```java
public boolean add(E e) {
    	//利用hashMap的key唯一的特性
        return map.put(e, PRESENT)==null;
    }
```

#### remove(Object o)方法

```java
public boolean remove(Object o) {
    	//调用hashMap的remove()方法
        return map.remove(o)==PRESENT;
    }
```

