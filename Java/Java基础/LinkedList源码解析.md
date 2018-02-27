#### LinkedList

```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{

    transient int size = 0;

    /**
     * Node用于存储具体的数据
     */
    transient Node<E> first;

    transient Node<E> last;
    
    /**
     * Node中的item用于存储具体的数据
     * Node中还保存了前一个节点和后一个节点
     * 所以LinkedList是通过双向链表来实现的
     */
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
    
    /**
     * LinkedList默认实现是空的
     */
    public LinkedList() {
    }
    
    /**
     * 添加数据
     */
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    
    /**
     * 添加的数据插入到末尾
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
    
    /**
     * 数据插入到头部
     */
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
    
    /**
     * 获取头节点，而first始终指向头节点
     */
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
    
    /**
     * 获取尾部节点，last始终指向尾部节点
     */
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

    
    /**
     * 删除数据，也就是删除节点
     * 可以看出LinkedList里面可以存储null
     */
    public boolean remove(Object o) {
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
    
    /**
     * 删除指定位置的节点
     */
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
    
    /**
     * 获取指定位置节点的方法
     */
    Node<E> node(int index) {
        // 这里先判断index的位置是在前半段还是后半段，从而减少循环遍历查找的次数，优化性能
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
    
    /**
     * 删除节点
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        // 每次删除节点都要更新头节点和尾部节点
        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
    
    /**
     * 检查下标是否越界，这里就是抛出IndexOutOfBoundsException的地方
     */
    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    
    /**
     * 检查是否包含某个元素
     */
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }
    
    /**
     * 获取LinkedList的元素数量
     */
    public int size() {
        return size;
    }
    
    /**
     * 默认添加到链表尾部
     */
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    
    /**
     * 插入节点到指定位置
     */
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    /**
     * 找出元素在链表中的位置
     */
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
    
    
    /**
     * 从1.5版本开始LinkedList增加了poll、peek等方法，因此LinkedList可以作为队列来使用
     */
    public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }
    
    public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }
    
    public E remove() {
        return removeFirst();
    }
    
    /**
     * 从1.6版本开始，增加了push和pop方法，因此LinkedList可以作为栈来使用
     */
    public void push(E e) {
        addFirst(e);
    }
    
    public E pop() {
        return removeFirst();
    }
    
    /**
     * LinkedList也提供了toArray方法
     */
    public Object[] toArray() {
        Object[] result = new Object[size];
        int i = 0;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        return result;
    }
    
}
```
- LinkedList底层是用双向链表实现的，插入数据比较快，复杂度为O(1)，但是对于查找和删除复杂度为O(n)（这跟直接删除节点不一样，需要先根据值遍历找到这个节点）
- 从1.5版本开始LinkedList增加了poll、peek等方法，因此LinkedList可以作为队列来使用；从1.6版本开始，增加了push和pop方法，因此LinkedList也可以作为栈来使用
- 当然，LinkedList可以保证插入元素的顺序，并且可以选择插入的顺序，默认add方法是插入到队尾
- 可以看出，LinkedList没有大小限制，默认的构造函数实现也是空的，因此不存在容量不够的情况，也没有扩容方法
- LinkedList不是线程安全的，只能用于单线程环境

##### 以上是基于Java1.8并且只介绍了常用的一些方法的原理，详细的LinkedList源码请查看：[LinkedList源码](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/LinkedList.java#LinkedList.Node)