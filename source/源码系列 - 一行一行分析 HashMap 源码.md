# 源码系列 - 一行一行分析 `HashMap` 源码

## 1 数据结构
- 开始之前，先简单介绍下 `HashMap` 数据结构，如下图（jdk1.8）

![HashMap数据结构](/Users/liushiwei/Downloads/HashMap数据结构.png)

如上图， `HashMap` 数据结构是一个Hash表，当表中一个节点元素个数小于8时是一个单向链表，大于等于8时，调整为红黑树（jdk1.8做的改进）

## 2 设计思想
- 数组+链表+红黑树
- 容量为2的n次方，提升数组索引位置的计算效率
- 采用高低链的方式来解决重复计算hash的问题，提升扩容效率
- 链表长度大于等于 `8` 转红黑树，提升查找效率

## 3 `put` 方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

#### 3.1 `hash` 方法
```java
static final int hash(Object key) {
    int h;
    // key为null直接返回0
    // (h = key.hashCode()) ^ (h >>> 16) --> 
    // hashCode右移16位 然后和自己取异或，其实为了让hashCode的高16位也能参与到接下来的计算（计算hash在数组中的位置），目的就是让数据更分散，降低hash冲突的概率
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### 3.2 `putVal` 方法

```java
/**
 *
 * @param key
 * @param value
 * @param onlyIfAbsent true（相同key不会覆盖）, false（相同key会覆盖）
 * @param evict 
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 若数组为空，则进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        // resize方法有两个逻辑分支（初始化和扩容），这里是初始化，具体后面分析
        n = (tab = resize()).length;
    // 走的这里，说明上面判断数组不为空，则判断数组索引i位置元素是否为空，为空，则新建一个Node放入该位置
    // 这里 (n - 1) & hash 等价于 hash % n ，使用 & 运算效率更高
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 进入 else 逻辑，说明上面判断都不成立
    else {
        // 梳理下，进入这里，说明数组索引i位置不为空，
        // 这里i位置可能是一个链表，可能是红黑树
        Node<K,V> e; K k;
        // 首先判断数组索引i位置第一个元素（不管是链表还是红黑树）
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 第一个元素就相等，就不必进行后面的逻辑了
            e = p;
        else if (p instanceof TreeNode)
            // 如果是红黑树，则走红黑树的逻辑，暂不分析红黑树
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 走到这里，说明是链表，且链表第一个元素不等于参数key
            // 这里binCount用来记录链表长度，用于判断是否要将链表转为红黑树
            for (int binCount = 0; ; ++binCount) {
                // 链表第一元素不等，则取下面一个元素进行判断，即 p.next
                if ((e = p.next) == null) {
                    // 进入这里，说明下个元素为空，则新建Node放入该位置
                    p.next = newNode(hash, key, value, null);
                    // 判断binCount是否达到需要将链表转为红黑树的阀值
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 将链表转为红黑树，暂不分析红黑树
                        treeifyBin(tab, hash);
                    // 新元素已经插入成功，跳出循环
                    break;
                }
                // 进入这里，说明下个元素不为空，进行判断
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 走到这里，说明上面两个if逻辑都没有成功跳出循环，则将当前e赋给p，继续遍历链表
                p = e;
            }
        }
        // e 不为空，说明HashMap中存在和key相等的元素
        if (e != null) { // existing mapping for key
            // 记录旧的值
            V oldValue = e.value;
            // 如果需要覆盖，或者旧值为空，则覆盖
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 这个方法HashMap没有实现，忽略
            afterNodeAccess(e);
            // 进到这里，直接返回旧值，不会再走下面逻辑
            return oldValue;
        }
    }
    // 走到这里，说明HashMap中不存在和key相等的元素，修改次数加1
    ++modCount;
    // 元素个数加1，并判断是否超过需要扩容的阀值
    if (++size > threshold)
        // 超过了阀值，则进行扩容
        resize();
    // 这个方法HashMap没有实现，忽略
    afterNodeInsertion(evict);
    // HashMap中不存在和key相等的元素，则最终返回null
    return null;
}
```
#### 3.3 `resize` 方法
```java
// 该方法包含两个逻辑分支，分别是初始化，和扩容，建议大家分开看（即先看初始化的逻辑，再看扩容的逻辑，不用混合着看），这样思路更清晰一些
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 走到这里，是扩容操作
        // 判断容量是否达到上限
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 走到这里，说明原始容量已经达到上限了，直接返回
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 确认下扩容一倍后是否超过最大容量
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 走到这里，是初始化操作
        // oldThr > 0说明新建HashMap时，指定了容量，这里直接赋值
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 走到这里，是初始化操作
        // 新建HashMap时，没有指定容量，这里进行默认值设置
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        // 走到这里，说明需要计算扩容阀值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    // 新建一个数组，容量为newCap
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 走到这里，说明原数组不为空，为扩容操作，需要做数据迁移
        for (int j = 0; j < oldCap; ++j) {
            // 循环遍历数组
            Node<K,V> e;
            // 判断数组j位置是否为空
            if ((e = oldTab[j]) != null) {
                // 数组j位置不为空，用e记录当前元素，将原数组j位置设为null
                oldTab[j] = null;
                // 判断下个为空是否为空
                if (e.next == null)
                    // 下一个元素为空，说明数组j位置只有一各元素
                    // 计算在新数组中的位置，并将e赋值给新的数组相应位置
                    newTab[e.hash & (newCap - 1)] = e;
                // 下一个元素不为空，判断该位置Node是否为空红黑树
                else if (e instanceof TreeNode)
                    // 为红黑色数，进行红黑树的迁移，暂不分析
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 走到这里，说明是链表
                    // 这里采用高低位两个链表，迁移原链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 如果该元素，e.hash & oldCap == 0 说明，该元素要放到低位链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 否则放到高位链表
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
                        // 低位链表，直接放到新数组相同位置
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 高位链表，放到新数组 [j+原始数组大小] 位置
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

-  这里需要解释下高低位链表，举个例子

```
假如原数组容量为16，扩容后数组容量为32，现有两个元素，hash值分别为1，17，这两个元素在原数组都落到索引为1的位置（因为 1%16=1, 17%16=1）。现在进行数组迁移，1、17在新数组的位置分别为1、17（因为1%31=1、17%31=17）。
17属于高位（因为17&16>0），所以17=1+16；1属于地位，（因为1&16=0），所以1=1
所以低位链表在新数组位置和原始位置相同，高位链表在新数组位置在原来位置基础上加原始容量
其实说白了，就是判断hash值二进制第五位是否为1，为1，则对新数组取模，必然落在新数组的后半部分，为0，则对新数组取模，必然落在新数组的前半部分
当n=2^k时，hash & (n-1) = hash % n
```

## 4 `get` 方法
```java
public V get(Object key) {
    Node<K,V> e;
    // hash函数和上面一样，不说了
    // getNode下面分析
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```
#### 4.1 `getNode` 方法
```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 首先判断数组不为空，且数组索引位置第一个元素不为空
    // 数组为空，或者数组索引位置元素为空，则该元素肯定不存在
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断头节点是否是该元素
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 判断是否还有下一个节点
        if ((e = first.next) != null) {
            // 红黑树逻辑，暂不分析
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 循环遍历链表，查找元素，直到下个元素为空为止
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
本文完。
