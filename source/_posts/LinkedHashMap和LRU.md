---
title: LinkedHashMap和LRU
date: 2018-08-20 14:28:05
categories: 
- Java
tags:
- Java
- LRU
---

刚好正在研究Android中LruCache缓存，它的实现其实也是使用了LinkedHashMap，所以今天就专门写博客记录一下相关知识。

## 存储结构

LinkedHashMap实际上是使用HashMap+双向链表，有关HashMap的详细知识就请看之前相关博客[HashMap源码分析](https://david1840.github.io/2018/08/12/HashMap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)。我们知道HashMap是以散列表的形式存储数据的，LinkedHashMap继承HashMap，所以它也是使用散列表存储数据，但是，会有额外的“Linked”双向链表把所有的数据连接起来。为什么要这样做？HashMap是无序的，而加上双向链表，就将所有数据有序管理起来。具体如下图：

![](LinkedHashMap和LRU/linkedhashmap.png)

在HashMap的基础上多了befor和after字段，用来形成双向链表。

## 两个例子

LinkedHashMap的核心就是存在存储顺序和可以实现LRU算法，所以下面我用两个例子证明这两种情况：

#### 存储顺序

```
public class LinkedHashMapTest {

    public static void main(String[] args) {
        LinkedHashMap<Integer, Integer> map = new LinkedHashMap<Integer, Integer>();
        for (int i = 0; i < 10; i++) {//按顺序放入1~9
            map.put(i, i);
        }
        System.out.println("原数据："+map.toString());
        map.get(3);
        System.out.println("查询存在的某一个："+map.toString());
        map.put(4, 4);
        System.out.println("插入已存在的某一个："+map.toString()); //直接调用已存在的toString方法，不然自己需要用迭代器实现
        map.put(10, 10);
        System.out.println("插入一个原本没存在的："+map.toString());
    }

    //输出结果
//  原数据：{0=0, 1=1, 2=2, 3=3, 4=4, 5=5, 6=6, 7=7, 8=8, 9=9}
//  查询存在的某一个：{0=0, 1=1, 2=2, 3=3, 4=4, 5=5, 6=6, 7=7, 8=8, 9=9}
//  插入已存在的某一个：{0=0, 1=1, 2=2, 3=3, 4=4, 5=5, 6=6, 7=7, 8=8, 9=9}
//  插入一个原本没存在的：{0=0, 1=1, 2=2, 3=3, 4=4, 5=5, 6=6, 7=7, 8=8, 9=9, 10=10}

}
```

观察以上代码，其实它是符合先进先出的规则的，不管你怎么查询插入已存在的数据，不会对排序造成影响，如果有新插入的数据将会放在最尾部。

#### LRU

启用LinkedHashMap的LRU规则是要使用它的三个参数的构造方法。

```
/**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;//是否开启LRU规则
    }
```

```
public class LinkedHashMapTest {

    public static void main(String[] args) {
        LinkedHashMap<Integer, Integer> map = new LinkedHashMap<Integer, Integer>(20, 0.75f, true);
        for (int i = 0; i < 10; i++) {//按顺序放入1~9
            map.put(i, i);
        }
        System.out.println("原数据："+map.toString());
        map.get(3);
        System.out.println("查询存在的某一个："+map.toString());
        map.put(4, 4);
        System.out.println("插入已存在的某一个："+map.toString()); //直接调用已存在的toString方法，不然自己需要用迭代器实现
        map.put(10, 10);
        System.out.println("插入一个原本没存在的："+map.toString());
    }

    //输出结果
//  原数据：{0=0, 1=1, 2=2, 3=3, 4=4, 5=5, 6=6, 7=7, 8=8, 9=9}
//  查询存在的某一个：{0=0, 1=1, 2=2, 4=4, 5=5, 6=6, 7=7, 8=8, 9=9, 3=3} //被访问（get）的3放到了最后面
//  插入已存在的某一个：{0=0, 1=1, 2=2, 5=5, 6=6, 7=7, 8=8, 9=9, 3=3, 4=4}//被访问（put）的4放到了最后面
//  插入一个原本没存在的：{0=0, 1=1, 2=2, 5=5, 6=6, 7=7, 8=8, 9=9, 3=3, 4=4, 10=10}//新增一个放到最后面

}
```

从上面可以看出，每当我get或者put一个已存在的数据，就会把这个数据放到双向链表的尾部，put一个新的数据也会放到双向链表的尾部。

## 实现原理

#### 构造函数
```
 public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }


    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }

    public LinkedHashMap() {
        super();
        accessOrder = false;
    }


    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super(m);
        accessOrder = false;
    }

    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```
5个构造函数，可以设置容量和加载因子，且默认情况下是不开启LRU规则。

#### 双向链表

```
 /**
     * HashMap.Node subclass for normal LinkedHashMap entries.
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after; //指向前后节点
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

    /**
     * The head (eldest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> head;//双向链表头节点（最老）

    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient LinkedHashMap.Entry<K,V> tail;//双向列表尾节点（最新
```

#### LRU实现

```
void afterNodeAccess(Node<K,V> e) { // 把当前节点e放到双向链表尾部
        LinkedHashMap.Entry<K,V> last;
        //accessOrder就是我们前面说的LRU控制，当它为true，同时e对象不是尾节点（如果访问尾节点就不需要设置，该方法就是把节点放置到尾节点）
        if (accessOrder && (last = tail) != e) {
        //用a和b分别记录该节点前面和后面的节点
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
             //释放当前节点与后节点的关系 
            p.after = null;
            //如果当前节点的前节点是空，
            if (b == null)
            //那么头节点就设置为a
                head = a;
            else
            //如果b不为null，那么b的后节点指向a
                b.after = a;
            //如果a节点不为空
            if (a != null)
                //a的后节点指向b
                a.before = b;
            else
                //如果a为空，那么b就是尾节点
                last = b;
                //如果尾节点为空
            if (last == null)
            //那么p为头节点
                head = p;
            else {
            //否则就把p放到双向链表最尾处
                p.before = last;
                last.after = p;
            }
            //设置尾节点为P
            tail = p;
            //LinkedHashMap对象操作次数+1
            ++modCount;
        }
    }
```

开启LRU后，put，get等方法都会调用这个函数来调整顺序。

```
 public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)//如果启用了LRU规则
            afterNodeAccess(e);//那么把该节点移到双向链表最后面
        return e.value;
    }
```

#### 移除Eldest

```
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```

LinkedHashMap有一个自带的移除最老数据的方法，默认返回false，我们可以在继承的时候重写这个方法，给定一个条件就可以控制存储在LinkedHashMap中的最老数据何时删除。触发这个删除机制，一般是在PUT一个数据进入的时候，但是LinkedHashMap并没有重写Put方法如何实现呢?在LinekdHashMap中，这个方法被包含在afterNodeInsertion()方法之中，而这个方法是重写了HashMap的，但是HashMap中并没有去实现它，所以在put的时候就会触发删除这个机制。
