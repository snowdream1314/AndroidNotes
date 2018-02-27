#### LinkedHashMap

```
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{

    /**
     * HashMap.Node的子类
     * 增加了前后节点的引用，因此不再是单链表而是双向链表
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    
    /**
     * 双向链表的头节点和尾节点
     * 
     */
    transient LinkedHashMap.Entry<K,V> head;
    transient LinkedHashMap.Entry<K,V> tail;

    /**
     * 排序规则的标志
     * false表示按节点插入顺序排序
     * true表示按节点访问顺序排序
     * 默认是false
     */
    final boolean accessOrder;
    
    /**
     *将节点插入链表尾部
     */
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        //如果是插入第一个节点，则头节点和尾节点都指向这个节点
        if (last == null)
            head = p;
        //已经插入过节点了，就直接插入链表尾部    
        else {
            p.before = last;
            last.after = p;
        }
    }
    
    /**
     *将dst节点替换src节点
     */
    private void transferLinks(LinkedHashMap.Entry<K,V> src,
                               LinkedHashMap.Entry<K,V> dst) {
        LinkedHashMap.Entry<K,V> b = dst.before = src.before;
        LinkedHashMap.Entry<K,V> a = dst.after = src.after;
        //src是第一个节点
        if (b == null)
            head = dst;
        else
            b.after = dst;
        //src是最后一个节点
        if (a == null)
            tail = dst;
        else
            a.before = dst;
    }

    /**
     *初始化LinkedHashMap，这里在调用HashMap的初始化方法之外，还需要将头节点和尾节点初始化
     */
    void reinitialize() {
        super.reinitialize();
        head = tail = null;
    }
    
    /**
     * 这里覆写了HashMap的newNode方法
     * 生成新的节点并插入到尾部
     */
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
    
    /**
     * 用next节点替换p节点
     */
    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
        LinkedHashMap.Entry<K,V> t =
            new LinkedHashMap.Entry<K,V>(q.hash, q.key, q.value, next);
        transferLinks(q, t);
        return t;
    }
    
    /**
     * 红黑树节点的相关方法
     */
    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p);
        return p;
    }

    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
        TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
        transferLinks(q, t);
        return t;
    }
    
    /**
     * 覆写HashMap中的afterNodeRemoval方法
     * 将节点从链表中删除
     */
    void afterNodeRemoval(Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        
        //删除的是第一个节点
        if (b == null)
            head = a;
        else
            b.after = a;
        //删除的是最后一个节点
        if (a == null)
            tail = b;
        else
            a.before = b;
    }

    /**
     * 覆写HashMap中的afterNodeInsertion方法
     * evict为true并且removeEldestEntry方法也为true的话就会删除近期最少使用的节点
     * 所以这里可以看出LinkedHashMap能实现LRU算法
     * 由HashMap的源码可知evict默认是true
     */
    void afterNodeInsertion(boolean evict) {
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            //满足条件则删除近期使用最少的节点
            removeNode(hash(key), key, null, false, true);
        }
    }

    /**
     * 覆写HashMap中的afterNodeAccess方法，每当节点被访问时被调用
     * 如果accessOrder为true，也就是LinkedHashMap的节点是按照访问顺序排序的，则需要将节点移到链表的末尾
     * 这也是LinkedHashMap实现LRU算法的条件之一，即accessOrder要为true
     */
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
    
    /**
     * 可以看到默认accessOrder为false，也就是按节点插入顺序排序
     */
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
    
    /**
     * 这个构造函数可以设置排序规则
     * 所以如果要实现LRU算法，也就是要设置accessOrder为true，就要用这个构造函数
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    
    /**
     * 判断值是否存在，只需要循环双向链表
     * 这里相比HashMap需要2层循环，提升了效率
     */
    public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
    
    /**
     * LinkedHashMap的get方法调用的是HashMap的getNode方法
     * 注意这里会调用recordAccess方法
     */
    public V get(Object key) {
        Node<K,V> e;
        //如果没找到返回null
        if ((e = getNode(hash(key), key)) == null)
            return null;
        //如果节点是按照访问顺序排序的，那就得去更新下顺序，把刚刚获取的节点放到链表末尾去    
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
    
    /**
     * 跟上面的方法一样，只是如果对应的key的节点不存在，会返回一个默认的数据
     */
    public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;
       if (accessOrder)
           afterNodeAccess(e);
       return e.value;
   }
   
    /**
     * 这个好理解，清空数据
     */
   public void clear() {
        super.clear();
        head = tail = null;
    }
    
    /**
     * 这个方法用于控制是否要删除近期使用最少的节点
     * 默认返回是false，所以如果要实现LRU算法，需要覆写该方法，比如节点满了返回true以便删掉使用较少的节点
     */
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
    
    <!--省略其他方法-->
    
}
```
##### LinkedHashMap没有put方法，是怎么插入数据的呢
- 答案就在newNode方法

```
/**
 * 这里覆写了HashMap的newNode方法
 * 生成新的节点并插入到尾部
 */
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

/**
 * 红黑树节点的相关方法
 */
TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
    TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
    linkNodeLast(p);
    return p;
}
```
- LinkedHashMap实际只是在HashMap的基础上加上了一个双向链表来保存节点插入的顺序，因此很多的逻辑都和HashMap是一样的。比如插入节点时，LinkedHashMap相比HashMap只需要在HashMap的基础上将节点插入双向链表，以及根据排序要求更新节点的顺序就行了
- 对于插入双向链表的功能，LinkedHashMap覆写了HashMap的newNode方法；而对于更新节点的顺序问题，LinkedHashMap覆写了afterNodeAccess方法和afterNodeInsertion方法

##### 总结
- LinkedHashMap继承自HashMap，是HashMap的子类，因此也不是线程安全的
- LinkedHashMap底层存储结构与HashMap一样，不同的是LinkedHashMap增加了一个双向链表的头节点，插入的数据除了插入HashMap，还会插入链表中，因而可以保存插入节点的顺序
- LinkedHashMap的节点在HashMap节点的基础上增加了前后节点的引用
- LinkedHashMap可以插入null
- LinkedHashMap相比HashMap在查找值和删除值时效率要高
- LinkedHashMap还可以设置按插入顺序排序或是按访问顺序排序，默认是按插入顺序排序
- LinkedHashMap没有put方法，而是覆写了afterNodeAccess方法和afterNodeInsertion方法。当插入的数据已经存在时，会调用afterNodeAccess方法看是否需要将数据插入到链表末尾；当插入的数据是新数据时，会通过afterNodeInsertion方法来根据设置删除近期使用最少的节点
- LinkedHashMap可以用来实现LRU算法。首先需要用可以设置accessOrder的构造函数设置accessOrder为true，也就是按照节点访问顺序排序；然后removeEldestEntry方法设置当超过节点数时返回true，也就是删除近期最少使用的数据

##### 以上是基于Java1.8并且只介绍了常用的一些方法的原理，详细的LinkedHashMap源码请查看：[LinkedHashMap源码](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/LinkedHashMap.java#LinkedHashMap.afterNodeInsertion%28boolean%29)