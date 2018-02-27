#### HashTable

```
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable {
    
    /**
     * 结构和HashMap一样，也是数组+链表
     * 下面的几个参数也是一样的作用
     */
    private transient Entry<?,?>[] table;
    
    private transient int count;

    private int threshold;

    private float loadFactor;

    private transient int modCount = 0;

    private static final long serialVersionUID = 1421746759512286392L;

    private static int hash(Object k) {
        return k.hashCode();
    }

    /**
     * 构造方法
     */
    public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (initialCapacity <= MAX_ARRAY_SIZE + 1) ? initialCapacity : MAX_ARRAY_SIZE + 1;
    }
    
    /**
     * 与HashMap不一样的是，默认容量是11
     */
    public Hashtable() {
        this(11, 0.75f);
    }
    
    /**
     * 与HashMap不一样的是，很多方法都加了synchronized，所以是线程安全的
     */
    public synchronized int size() {
        return count;
    }


    public synchronized boolean isEmpty() {
        return count == 0;
    }
    
    /**
     * 简单粗暴的查找
     * 与HashMap不一样的是，很多方法都加了synchronized，所以是线程安全的
     */
    public synchronized boolean contains(Object value) {
        if (value == null) {
            throw new NullPointerException();
        }

        Entry<?,?> tab[] = table;
        for (int i = tab.length ; i-- > 0 ;) {
            for (Entry<?,?> e = tab[i] ; e != null ; e = e.next) {
                if (e.value.equals(value)) {
                    return true;
                }
            }
        }
        return false;
    }
    
    /**
     * Hashtable在求hash值对应的位置索引时，用取模运算，而HashMap在求位置索引时，则用与运算，且这里一般先用hash&0x7FFFFFFF后，再对length取模，&0* x7FFFFFFF的目的是为了将负的hash值转化为正值，因为hash值有可能为负数，而&0x7FFFFFFF后，只有符号外改变，而后面的位都不变。
     */
    public synchronized boolean containsKey(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return true;
            }
        }
        return false;
    }
    
    /**
     * 查找的思路和HashMap一样
     */
    public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
    
    /**
     * 和HashMap不一样的是，HashTable中不允许value为null
     */   
    public synchronized V put(K key, V value) {
        // 不允许value为null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            // 如果已经存在对应的key，则替换旧的值
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
        // 添加的是新的key
        addEntry(hash, key, value, index);
        return null;
    }
    
    /**
     * 添加新的key
     */  
    private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry<?,?> tab[] = table;
        if (count >= threshold) {
            // 如果超出阈值，就进行扩容操作
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
    
    /**
     * 扩容方法
     * 和HashMap不一样的是，HashTable的容量为原来容量的2倍+1
     * 扩容的复杂度也很高
     */  
    @SuppressWarnings("unchecked")
    protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // 已经超出MAX_ARRAY_SIZE了就没办法扩容了
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
    
    /**
     * 删除方法
     * 和HashMap一样的套路
     * 
     */  
    public synchronized V remove(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
            // 找到节点的话删除并返回删除的节点的值
            if ((e.hash == hash) && e.key.equals(key)) {
                modCount++;
                if (prev != null) {
                    prev.next = e.next;
                } else {
                    tab[index] = e.next;
                }
                count--;
                V oldValue = e.value;
                e.value = null;
                return oldValue;
            }
        }
        return null;
    }
}
```
#### 总结
- HashTable的底层结构和HashMap是一样的，都是数组+链表
- HashTable是线程安全的
- HashTable的默认大小是11，容量大小也没有限制一定要是2的倍数
- HashTable不允许存value为null的值
- Hashtable在求hash值对应的位置索引时，用取模运算，先用hash&0x7FFFFFFF后，再对length取模，&0x7FFFFFFF的目的是为了将负的hash值转化为正值，因为hash值有可能为负数，而&0x7FFFFFFF后，只有符号位改变，而后面的位都不变。

##### 以上是基于Java1.8并且只介绍了常用的一些方法的原理，详细的HashMap源码请查看：[HashTable源码](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/Hashtable.java?av=f)