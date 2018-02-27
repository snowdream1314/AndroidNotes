#### HashMap

```
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
    /**
     * 默认容量为16，必须为2的倍数
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

    /**
     * 最大的容量，必须为2的倍数且不超过2^30
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;
    
    /**
     * Java8中对HashMap进行了优化，当数组容量超过64，而链表长度超过8时，就会将链表转换为红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;
    
    /**
     * 在调用resize方法进行初始化或是扩容操作时，当数组下面的链表长度不超过6时，就会将链表由红黑树转为链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;
    
    /**
     * 当HashMap的容量大于64时，才会根据链表的长度来判断是否需要转换为红黑树，否则的话都是直接将HashMap扩容
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * 加载因子，HashMap的稀疏性，用于控制哈希冲突，比如说如果是1.1的话，意思就是10个口袋里放11个球，这样肯定会哈希冲突了
     * 但是也不能太低，否则也浪费空间
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    /**
     * Node节点，除了存储了值，还存储了下个节点的引用next，所以可以作为单链表的节点
     */
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

        /**
         * 这里分别计算键值对的hashCode抑或运算后作为哈希值
         */
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
    
    /**
     * 计算哈希值的方法
     */
    static final int hash(Object key) {
         int h;
         return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    /**
     * 可见HashMap通过数组来保存元素，每个数组的元素Node里有下个元素的引用，也就是用单链表结构来保存哈希冲突的元素；也就是数组+单链表的结构
     */
    transient Node<K,V>[] table;

    transient int size;

    /**
     * 这个是阈值，表示达到这个值以后就要进行扩容；其值等于capacity * load factor
     */
    int threshold;

    /**
     * 默认的加载因子
     */
    final float loadFactor = DEFAULT_LOAD_FACTOR;

    /**
     * HashMap被更改的次数，包括增加删除等操作都会被计数
     */
    transient int modCount;
    
    /**
     * HashMap的构造函数
     * 可以看到这里没有在构造函数中初始化数组
     * 初始化数组的地方放在了插入数据的时候，在resize方法中会对初始化的情况进行处理
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY) {
            initialCapacity = MAXIMUM_CAPACITY;
        } else if (initialCapacity < DEFAULT_INITIAL_CAPACITY) {
            initialCapacity = DEFAULT_INITIAL_CAPACITY;
        }

        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
    
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 默认是设置了加载因子为0.75
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; 
    }
    
    /**
     * 返回HashMap的size
     */
    public int size() {
        return size;
    }
    
    /**
     * size=0就表示空
     */
    public boolean isEmpty() {
        return size == 0;
    }
    
    /**
     * 根据key获取值，可以看到允许值为null
     */
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * 根据key获取对应节点
     * 每次都会检查是否第一个节点就是需要的节点，然后才去链表里找
     * 由于Java8加入了红黑树，所以在循环遍历链表的时候会判断是否是红黑树，以此优化查找性能
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
    
    /**
     * 看看key存不存在就是用getNode方法找找看，getNode返回空就是不存在
     */
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }
    
    /**
     * 插入数据
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * 插入数据
     *
     * onlyIfAbsent如果为true，就表示不改变原来的老数据；为false就覆盖原来的老数据
     * evict参数所在的方法afterNodeInsertion是用于给LinkedHashMap调用的，如果为true，则允许LinkedHashMap删除近期最少使用的数据
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //刚刚开始的时候，HashMap为空，就会在这里先进行初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //如果数组下标位置也就是数组的第一个节点为空，则说明还没有哈希冲突，直接插入数据即可    
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //进入这里说明哈希冲突了  
        else {
            Node<K,V> e; K k;
            //看看是不是插入的数据和第一个节点是同一个key，是的话就不用下面循环了
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果不是第一个节点，那就需要根据是红黑树还是链表来做相应的处理    
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //不是红黑树节点，那就是链表了，得循环遍历链表了
            else {
                for (int binCount = 0; ; ++binCount) {
                    //看看是不是到了链表的最后一个节点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //加入新节点后得判断是否到了阈值，到了阈值就需要扩容或是转换为红黑树了
                        if (binCount >= TREEIFY_THRESHOLD - 1) 
                            treeifyBin(tab, hash);
                        break;
                    }
                    //原来就有这个key就退出循环，直接看下面是替换老数据还是直接返回老数据了
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //e不为null，说明插入的是相同的key
            if (e != null) {
                V oldValue = e.value;
                //这里根据onlyIfAbsent来判断是否需要覆盖老数据，默认onlyIfAbsent是false，也就是覆盖老数据
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //这是用于给LinkedHashMap覆写的方法，用于LinkedHashMap调整节点的顺序  
                afterNodeAccess(e);
                return oldValue;
            }
        }
        
        //执行到这里，说明是插入了新的key
        ++modCount;
        //size增加，需要检查是否需要扩容
        if (++size > threshold)
            resize();
        //这是用于给LinkedHashMap覆写的方法，用于LinkedHashMap删除近期使用最少的数据     
        afterNodeInsertion(evict);
        return null;
    }
    
    /**
     * 扩容方法
     * 
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        //需要注意的是，这里取的是length而不是HashMap的size，所以这里的oldCap是HashMap数组的大小
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //如果老的HashMap容量已经到达最大阈值了，没法扩容了，直接将阈值设置为最大并返回，没办法只能让它冲突去了
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //否则的话，就扩容为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //oldCap为0，就说明是初始化的情况。
        //如果已经有阈值，则初始化的时候HashMap的容量就是阈值的大小
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //如果阈值也没有初始化，那就都用默认的值
        else {               
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //这里需要判断下上面的第二种只设置了容量的情况，需要再设置下新的阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //这里如果是初始化的情况，newThr=16*0.75=12，所以当size>12时就会触发扩容
        threshold = newThr;
        //生成新的数组，如果是初始化，会生成一个长度为16的数组
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //如果不是初始化的情况，就需要将老数据重新映射到新的数组中了
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    //释放老的节点的数据
                    oldTab[j] = null;
                    //如果只有一个节点的话，就重新映射到新的数组
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //不是只有一个节点，那就是原来有哈希冲突了，那可能是链表也可能是红黑树    
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //不是红黑树，那就是链表了
                    //下面也是Java8中的一个优化点    
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //这里把hash值与oldCap做与操作，而oldCap是2的倍数，所以低位肯定都是0
                            //比如说第一次扩容的时候oldCap是16，也就是 00010000，那这里就要看hash的高一位也就是000?xxxx中的？是1还是0了
                            //如果是0，那意思就是在新的数组里面的索引跟以前的一样；如果是1，那新的索引就是以前的下标+16
                            //后面再扩容时也是一样，再高一位是0，索引不变；如果是1，索引变为原索引+oldCap
                            //这么做的目的是将链表中的一些节点分散到新的数组中去，空间大了嘛，没必要还都挤在一起
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
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
    
    /**
     * 将链表转化为红黑树的方法
     * 可以看到，只有当HashMap的容量大于MIN_TREEIFY_CAPACITY时才会执行转化，否则都会执行resize方法，也就是进行扩容
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
    
    /**
     * 根据key删除节点
     */
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    /**
     * 删除节点
     *
     * matchValue如果为true，则只当value也相等的时候才删除节点
     * movable用于红黑树中，当为false时，只删除树中的节点而不移动其他节点
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            //要删除的就是第一个节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            //否则那就说明有多个哈希冲突的节点，那可能是链表也可能是红黑树了   
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                //是链表的话就得循环查找了
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //node不为null，说明找到要删除的节点了
            //这里会根据其他条件来进行处理
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                //下面是单链表删除节点
                //如果要删除的是第一个节点，那得把后面的节点补到数组里面
                else if (node == p)
                    tab[index] = node.next;
                //如果删除的是链表中的其它节点，把链表接上
                else
                    p.next = node.next;
                ++modCount;
                --size;
                //这里是供LinkedHashMap调用的方法
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
    
    /**
     * 删除所有数据
     */
    public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }

    /**
     * 判断是否包含特定值的节点
     */
    public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
    
    // LinkedHashMap需要覆写的方法
    void afterNodeAccess(Node<K,V> p) { }
    void afterNodeInsertion(boolean evict) { }
    void afterNodeRemoval(Node<K,V> p) { }

}

```
##### 下面来分析一下HashMap的哈希原理和散列值优化策略
- 主要涉及到的是2个方法：hash()方法和取模

```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
- 上面首先调用Object的hashCode()方法来计算key的哈希值，返回的是int型；理论上哈希值的范围为int型的范围，也就是-2147483648到2147483648，前后加起来大概有40亿的空间，所以只要哈希函数映射得比较均匀松散，基本不会出现冲突
- 但是显然这个范围不能直接用来做HashMap的数组下标，需要进行取模运算得到相应的低位数据

```
//n是HashMap的容量，为2的倍数，比如初始容量为16，32位也就是0000000000010000，那n-1就是0000000000001111
//然后跟hash做与操作，那就相当于取了hash的低四位
p = tab[i = (n - 1) & hash]
```
- HashMap这样得到数组的下标，势必冲突会比较严重，所以我们看到在上面的hash()方法中还有一步操作是将哈希值h右移16位后做抑或操作，这步操作就是扰动，用于提高低位的随机性，降低哈希冲突

![image](http://upload-images.jianshu.io/upload_images/2196721-f22bed057328893f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 扩容时候对链表的优化

- 扩容的时候，在将原先的数据重新映射到新数组时，会对原来的链表进行优化。
- 优化的方式是通过与原数组的容量(为2的倍数)进行与操作，判断高一位是0还是1。如果是0则新的索引不变，比如原来的索引是5，那在新数组里面也放在索引为5的地方；而如果是1，则新的索引为原索引+原数组容量，比如原索引是5，如果高位为1，原容量是16，那新的索引就是5+16=21，也就是存放到索引为21的地方
- 通过这样的优化既省去了计算索引的步骤，又将原来存在的链表重新映射，相当于将原来冲突的节点数据重新分散到了新的数组里面了
- 同时还保持了原来链表中的顺序，而在以前的版本中会把原来的顺序倒过来，具体看下面的分析
```
//下面也是Java8中的一个优化点    
else { // preserve order
    Node<K,V> loHead = null, loTail = null;
    Node<K,V> hiHead = null, hiTail = null;
    Node<K,V> next;
    do {
        next = e.next;
        //这里把hash值与oldCap做与操作，而oldCap是2的倍数，所以低位肯定都是0
        //也就是说第一次扩容的时候oldCap是16，也就是 00010000，那这里就要看hash的高一位也就是000?xxxx中的？是1还是0了
        //如果是0，那意思就是在新的数组里面的索引跟以前的一样；如果是1，那新的索引就是以前的下标+16
        //后面再扩容时也是一样，再高一位是0，索引不变；如果是1，索引变为原索引+oldCap
        if ((e.hash & oldCap) == 0) {
            if (loTail == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
        }
        //高位是1，这样的话索引需要+oldCap；
        else {
            if (hiTail == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
        }
    } while ((e = next) != null);
    //loHead和hiHead中保持了原来的顺序
    if (loTail != null) {
        loTail.next = null;
        newTab[j] = loHead;
    }
    //高位为1的话，新的索引为老索引+oldCap
    if (hiTail != null) {
        hiTail.next = null;
        newTab[j + oldCap] = hiHead;
    }
}
```
- 下面简单看看原来的扩容操作

```
/**
 * 扩容方法
 * 先计算新的容量，然后通过transfer方法把数据添加到新的数组里
 * 数据迁移好了以后，重新计算容量阈值
 */
void resize(int newCapacity) {
    HashMapEntry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    HashMapEntry[] newTable = new HashMapEntry[newCapacity];
    transfer(newTable);
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

/**
 * 真正的扩容方法
 * 可见，这个扩容方法还是比较废事的，复杂度比较高
 * 扩容每个节点都要重新计算索引，而且可以看出是倒序的，也就是把原来链表中的顺序倒过来了
 */
void transfer(HashMapEntry[] newTable) {
    int newCapacity = newTable.length;
    for (HashMapEntry<K,V> e : table) {
        while(null != e) {
            HashMapEntry<K,V> next = e.next;
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}

/**
 * 根据哈希值取得数组的下标
 * 将HashMap的数组长度要取2的整次幂，这里用length-1正好相当于一个“低位掩码”，也就是通过&方法把高位都去掉了
 * 只保留了末尾4位，也就是0到15之间，刚刚好可以来做数组的下标
 * 可以看出取模的方法是一样的
 */
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

#### 总结
- HashMap不是线程安全的，只能用于单线程环境
- HashMap底层的结构是数组+链表的形式，哈希冲突的节点存入到对应数组元素的链表中
- HashMap在构造函数中没有初始化数组，而是在插入数据的时候进行处理
- HashMap的性能受散列的效果影响比较大，如果散列不够随机均匀，那哈希冲突就会严重，这样的话链表就会增加
- HashMap不能保证插入节点的顺序；
- Java8中对HashMap进行了一些优化，其中一个就是引入了红黑树。当HashMap的容量大于64，且链表的长度大于8的时候，就会将这个链表转换为红黑树以便提高性能
- Java8的另一个优化是在扩容的时候，会把原来的链表中的节点分散到新的数组中，并且不会像Java7那样有链表元素倒置的问题
- 关于根据哈希值来确定数组下标的问题可以参考[JDK 源码中 HashMap 的 hash 方法原理是什么？](https://www.zhihu.com/question/20733617)


##### 以上是基于Java1.8并且只介绍了常用的一些方法的原理，详细的HashMap源码请查看：[HashMap源码](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/HashMap.java?av=f)
