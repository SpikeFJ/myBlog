# 添加 
```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;//初始化大小16
static final int MAXIMUM_CAPACITY = 1 << 30; //最大容量2的30次方
static final float DEFAULT_LOAD_FACTOR = 0.75f;//默认加载因子
static final int TREEIFY_THRESHOLD = 8;//当链表个数大于此值时，转换成红黑树
static final int UNTREEIFY_THRESHOLD = 6;//当链表个数小于此值时，分解红黑树
static final int MIN_TREEIFY_CAPACITY = 64;

//内部节点结构
static class Node<K,V> implements Map.Entry<K,V> 
{
final int hash;
final K key;
V value;
Node<K,V> next;
}

public HashMap() {
this.loadFactor = DEFAULT_LOAD_FACTOR; 
}
public HashMap(int initialCapacity) {
this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
public HashMap(int initialCapacity, float loadFactor) {
//初始化容量小于0，报错
if (initialCapacity < 0)
throw new IllegalArgumentException("Illegal initial capacity: " +
initialCapacity);
//初始化容量大于最大值，则采用最大值 
if (initialCapacity > MAXIMUM_CAPACITY)
initialCapacity = MAXIMUM_CAPACITY;
//加载因子不能小于等于0或非数字
if (loadFactor <= 0 || Float.isNaN(loadFactor))
throw new IllegalArgumentException("Illegal load factor: " +
loadFactor);
this.loadFactor = loadFactor;
this.threshold = tableSizeFor(initialCapacity);
}

//调整用户输入的容量大小，**返回大于输入参数且最近的2的整数次幂的数**
static final int tableSizeFor(int cap) {
int n = cap - 1;
n |= n >>> 1;
n |= n >>> 2;
n |= n >>> 4;
n |= n >>> 8;
n |= n >>> 16;
return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

关于tableSizeFor方法最直观的解析是如果n=0x01000000,则
n >>> 1--->00100000; n |= n >>> 1---->01100000,所以移位的作用是将1后面所有的位数都置为1
0x01000000--->0x01111111
n+1---------->0x10000000
```

## 思考
> 为什么采用 (n - 1) & hash 这个算法来定位元素的位置

添加：
```java
public V put(K key, V value) {
return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
int h;
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
boolean evict) {
Node<K,V>[] tab; 
Node<K,V> p; //p表示根据hash计算出来应该放置的位置，p有可能是链表或红黑树
int n, i;
if ((tab = table) == null || (n = tab.length) == 0)
n = (tab = resize()).length;
//p位置没有值，则直接new一个节点占位，HashMap根据 (n - 1) & hash 求
    public V put(K key, V value) {
            return putVal(hash(key), key, value, false, true);
        }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //如果数组为null或长度为空，则扩容，n=扩展后的数组长度
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //i = (n-1)&hash 即key元素将要插入的数组位置
        //p =table[i],入股将要插入的位置是空，则在该位置新建元素
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;

            //p和key值相同，则复制给e,留待最后做替换操作
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                //如果p和key不相同，且p是树节点，则调用树插入
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //如果p和key不相同且不是树节点，则遍历next元素(超过8，需要切换成树)
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }