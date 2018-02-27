#### ArrayList

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    /**
     * 默认ArrayList的容量为10
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 可以看出ArrayList底层通过数组实现
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};
    
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};


    /**
     * 可以看出ArrayList基于数组实现
     */
    transient Object[] elementData;

    private int size;
    
    /**
     * 
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    
     /**
     *获取大小
     */
    public int size() {
        return size;
    }

    /**
     *判断是否为空
     */
    public boolean isEmpty() {
        return size == 0;
    }
    
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    /**
     * 找出目标的下标，不存在返回-1
     * 可以看出ArrayList中可以存储null
     */
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
    
    /**
     * indexOf方法从前往后找，lastIndexOf从后往前找
     * 
     */
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
    
    /**
     * 通过Arrays.copyOf方法返回包含所有数据的数组
     */
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }
    
    /**
     * 检查数组是否越界，这里就是抛出IndexOutOfBoundsException的地方
     */
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    
    /**
     * 获取元素很方便
     */
    public E get(int index) {
        //检查下标是否越界
        rangeCheck(index);
        return (E) elementData[index];
    }
    
    /**
     * 更新指定位置的元素
     */
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
    
    /**
     * 添加元素，每次添加之前都会检查下容量够不够，不够的话就会进行扩容
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    /**
     * 添加元素到指定位置，这个就需要移动元素了，而移动元素是通过System.arraycopy方法来拷贝数组，相对复杂度比较高
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    /**
     * grow方法进行扩容，最后调用的是Arrays.copyOf方法将老数据拷贝到新数组里面
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    /**
     * 删除元素也和添加元素一样分为删除指定位置元素和删除指定元素的情况
     * 不论哪种情况数组的删除复杂度都高
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        // 如果删除的是中间位置的元素就需要移动数组了，否则就不用移动数组
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        // 释放空出来的位置                     
        elementData[--size] = null; 

        return oldValue;
    }
    
    /**
     * 删除元素就没有获取元素那么方便了，得先遍历找到要删除的元素
     * 这里可以看到，ArrayList是允许值是null的
     */
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
    
    /*
     * 调用System.arraycopy方法实现删除元素
     * 并且这里过滤了删除的是末尾元素的情况
     */
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        //
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
    
    /*
     * 清空数组就是把数组的各个位置置null
     */
    public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }

}
```
#### 总结

- 底部基于数组实现，这样的话查找比较快，复杂度为O(1)，但是插入和删除数据就比较慢了，而且数据量越大插入和删除的速度越慢。复杂度为O(n)
- 默认容量大小为10，超过这个容量就会进行扩容，扩容的话最终调用的是System.arraycopy方法，这是一个native方法
- ArrayList里面允许存储null值
- ArrayList不是线程安全的，只能用于单线程环境下

##### 以上是基于Java1.8并且只介绍了常用的一些方法的原理，详细的ArrayList源码请查看：[ArrayList源码](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/ArrayList.java#ArrayList.Node)


