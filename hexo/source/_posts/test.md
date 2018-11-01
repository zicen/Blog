---
title: test
date: 2018-11-01 23:06:43
tags:

 上一篇文章[HashMap的底层原理探索](https://www.jianshu.com/p/2db05dbcba2d)我们分析了JDK1.7中Hashmap的源码实现，但是在JDK1.8的时候HashMap的实现做了很大的变动和优化。1.7和1.7之前HashMap都是“数组+链表”实现的，1.8之后就是“数组+（链表或红黑树）”来实现的了。这里我特地将“链表或红黑树”用一对括号括在一起，因为HashMap底层依旧是一个数组，然后数组中的元素是链表或者红黑树来实现的。

HashMap的实现就先介绍到这，下面就先讲一下红黑树，然后才能理解为什么要用红黑树来优化HashMap，用红黑树有什么好处。

##一.红黑树介绍

####1.二叉查找树介绍

在介绍红黑树之前我们要先理解二叉查找树的数据结构。下面简单介绍一下。

![image](http://upload-images.jianshu.io/upload_images/3804905-337fecca8fc5b285.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面这张图就是一个二叉查找树。二叉查找树有如下几条特性

（1）.左子树上所有结点的值均小于或等于它的根结点的值。

（2）.右子树上所有结点的值均大于或等于它的根结点的值。

（3）.左、右子树也分别为二叉排序树

那既然他名字中是有“查找”的，那么他是怎么查找的呢？比如我们要查找10这个元素，查找过程为：首先找到根节点，然后根据二叉查找树的第一二条特性，我们知道要查找的10>9所以是在根节点的右边去查找，找到13，10<13,所以接着在13的左边找，找到11,10<11,继续在11的左边查找，这样就找到了10.这其实就是二分查找的思想。最后我们要查出结果所需的最大次数就是二叉树的高度！（二分查找的思想:找到数组的中间位置的元素v，将数组分成>v和<v两部分，然后将v和要查找的数据进行一个比较，如果大于v那么就在>v的部分再次进行二分查找，否则就在<v的部分进行二分查找，直到找到对应的元素。）

那既然要查出结果所需的最大次数就是二叉树的高度，那这个高度会不会有时候很长呢？

比如我们依次插入 根节点为9如下五个节点：7,6,5,4,3。依照二叉查找树的特性，结果会变成什么样呢？7,6,5,4,3一个比一个小，那么就会成一条直线，也就是成为了线性的查询，时间复杂度变成了O（N）级别。为了解决这种情况，该红黑树出场了。

####2.红黑树介绍

红黑树其实就是一种**自平衡**的二叉查找树。他这个自平衡的特性就是对HashMap中链表可能会很长做出的优化。

红黑树是每个节点都带有颜色属性的二叉查找树，颜色或红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求:

性质1\. 节点是红色或黑色。

性质2\. 根节点是黑色。

性质3 每个叶节点（NIL节点，空节点）是黑色的。

性质4 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)

性质5\. 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。

下面这棵树就是一个典型的红黑树

![image](http://upload-images.jianshu.io/upload_images/3804905-2d19db15881c3905.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

红黑树那么多规则，那么在插入和删除元素会不会破坏红黑树的规则呢？什么情况下会破坏？什么情况下不会？

比如我们向原红黑树插入为14的新节点就不会破坏规则

![image](http://upload-images.jianshu.io/upload_images/3804905-6a0c11cecd3d6216.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

向原红黑树插入值为21的新节点就破坏了规则

![image](http://upload-images.jianshu.io/upload_images/3804905-6b3ac6eb56a33d90.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么红黑树是怎么维护这个二叉查找树的自平衡性的呢？

红黑树通过**“变色”和“旋转”**来维护红黑树的规则，变色就是让黑的变成红的，红的变成黑的，旋转又分为“左旋转”和“右旋转”。这个比较复杂，容易晕，我们就只要知道红黑树就是通过这种方式来实现它的自平衡性就行了。

## 二.HashMap中是怎么使用红黑树的？

JDK1.7的时候是使用一个Entry数组来存储数据，用key的hashcode取模来决定key会被放到数组里的位置，如果hashcode相同，或者hashcode取模后的结果相同（hash collision），那么这些key会被定位到Entry数组的同一个格子里，这些key会形成一个链表。在hashcode特别差的情况下，比方说所有key的hashcode都相同，这个链表可能会很长，那么put/get操作都可能需要遍历这个链表。也就是说时间复杂度在最差情况下会退化到O(n)

红黑树我们大致了解了，他的好处就是他的自平衡性，让他这棵树的最大高度为2log(n+1)，n是树的节点数。那么红黑树的复杂度就只有 O(log n)。

下面我们通过追踪JDK1.8 HashMap的put方法的源码来理解
```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```
put方法调用了putVal方法
```
static final int TREEIFY_THRESHOLD = 8;

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
//以看到这里的数组和1.7不同，是使用了一个Node数组来存储数据。
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
//如果超过了8个，那么会调用treeifyBin函数，将链表转换为红黑树。 
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
```
通过putVal方法可以看到这里的数组和1.7不同，是使用了一个Node数组来存储数据。那这个Node和1.7里面的Entry的区别是什么呢？

![image.png](https://upload-images.jianshu.io/upload_images/3804905-f62c7eb7779224e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


HashMap中的红黑树节点 采用的是TreeNode 类 

![image.png](https://upload-images.jianshu.io/upload_images/3804905-49315e1bb0e45e8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![image.png](https://upload-images.jianshu.io/upload_images/3804905-0fb959328d69f0a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


TreeNode和Entry都是Node的子类，也就说Node可能是链表结构，也可能是红黑树结构。

如果插入的key的hashcode相同，那么这些key也会被定位到Node数组的同一个格子里。如果同一个格子里的key不超过8个，使用链表结构存储。如果超过了8个，那么会调用treeifyBin函数，将链表转换为红黑树。

先分析到这。。。。。

---
