





# JDK1.7

**基于分段锁，减少锁粒度。**只有在同一个分段内才存在竞态关系。ConcurrentHashMap中的分段锁称为Segment，它继承了ReentrantLock，有着类似于HashMap的结构，即内部拥有一个HashEntry数组，数组中的每个元素又是一个链表：HashEntry中的value以及next都被volatile修饰。

并发度可以理解为程序运行时能够同时更新ConccurentHashMap且不产生锁竞争的最大线程数，实际上就是ConcurrentHashMap中的**分段锁**个数。ConcurrentHashMap默认的并发度为16，但用户也可以在构造函数中设置并发度。

## 延迟加载分段锁

JDK7中除了第一个Segment之外，剩余的Segments采用的是延迟初始化的机制：每次put之前都需要检查key对应的Segment是否为null，如果是则调用ensureSegment()以确保对应的Segment被创建。

ensureSegment可能在并发环境下被调用，但与想象中不同，ensureSegment并未使用锁来控制竞争，而是使用了Unsafe对象的getObjectVolatile()提供的原子读语义结合CAS来确保Segment创建的原子性。

```
if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // recheck
                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                       == null) {
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
}
```



## put（）

JDK7版本的ConcurrentHashMap在获得Segment锁的过程中，做了一定的优化 - 在真正申请锁之前，put方法会通过tryLock()方法尝试获得锁，在尝试获得锁的过程中会对对应hashcode的链表进行遍历，如果遍历完毕仍然找不到与key相同的HashEntry节点，则为后续的put操作提前创建一个HashEntry。当tryLock一定次数后仍无法获得锁，则通过lock申请锁。

在获得锁之后，Segment对链表进行遍历，如果某个HashEntry节点具有相同的key，则更新该HashEntry的value值，否则新建一个HashEntry节点，将它设置为链表的新head节点并将原头节点设为新head的下一个节点。新建过程中如果节点总数（含新建的HashEntry）超过threshold，则调用rehash()方法对Segment进行扩容，最后将新建HashEntry写入到数组中。

put方法中，链接新节点的下一个节点（HashEntry.setNext()）以及将链表写入到数组中（setEntryAt()）都是通过Unsafe的putOrderedObject()方法来实现，这里并未使用具有原子写语义的putObjectVolatile()的原因是：JMM会保证获得锁到释放锁之间所有对象的状态更新都会在锁被释放之后更新到主存，从而保证这些变更对其他线程是可见的。

```

```



### get与containsKey

get与containsKey两个方法几乎完全一致：他们都没有使用锁，而是通过Unsafe对象的getObjectVolatile()方法提供的原子读语义，来获得Segment以及对应的链表，然后对链表遍历判断是否存在key相同的节点以及获得该节点的value。但由于遍历过程中其他线程可能对链表结构做了调整，因此get和containsKey返回的可能是过时的数据，这一点是ConcurrentHashMap在弱一致性上的体现。如果要求强一致性，那么必须使用Collections.synchronizedMap()方法。

### size、containsValue



这些方法都是基于整个ConcurrentHashMap来进行操作的，他们的原理也基本类似：首先不加锁循环执行以下操作：循环所有的Segment（通过Unsafe的getObjectVolatile()以保证原子读语义），获得对应的值以及所有Segment的modcount之和。如果连续两次所有Segment的modcount和相等，则过程中没有发生其他线程修改ConcurrentHashMap的情况，返回获得的值。

当循环次数超过预定义的值时，这时需要对所有的Segment依次进行加锁，获取返回值后再依次解锁。值得注意的是，加锁过程中要强制创建所有的Segment，否则容易出现其他线程创建Segment并进行put，remove等操作。代码如下：

```
for(int j =0; j < segments.length; ++j)

ensureSegment(j).lock();// force creation
```

一般来说，应该避免在多线程环境下使用size和containsValue方法。

注1：modcount在put, replace, remove以及clear等方法中都会被修改。

注2：对于containsValue方法来说，如果在循环过程中发现匹配value的HashEntry，则直接返回true。

最后，与HashMap不同的是，ConcurrentHashMap并不允许key或者value为null，按照Doug Lea的说法，这么设计的原因是在ConcurrentHashMap中，一旦value出现null，则代表HashEntry的key/value没有映射完成就被其他线程所见，需要特殊处理。在JDK6中，get方法的实现中就有一段对HashEntry.value == null的防御性判断。但Doug Lea也承认实际运行过程中，这种情况似乎不可能发生（参考：http://cs.oswego.edu/pipermail/concurrency-interest/2011-March/007799.html）。

## 扩容



# JDK1.8

## sizeCtl

控制标识符

- 负数代表正在进行初始化或扩容操作

- -1代表正在初始化

- -N 表示有N-1个线程正在进行扩容操作

- 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，这一点类似于扩容阈值的概念。还后面可以看到，它的值始终是当前ConcurrentHashMap容量的0.75倍，这与loadfactor是对应的。

```java
   /**
     * 盛装Node元素的数组 它的大小是2的整数次幂
     * Size is always a power of two. Accessed directly by iterators.
     */
    transient volatile Node<K,V>[] table;
    // hash表初始化或扩容时的一个控制位标识量。
     private transient volatile int sizeCtl; 
     
       private static int RESIZE_STAMP_BITS = 16;
	
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
    
     static final int MOVED     = -1; // hash值是-1，表示这是一个forwardNode节点
    static final int TREEBIN   = -2; // hash值是-2  表示这时一个TreeBin节点
```



## put（）

```java
 final V putVal(K key, V value, boolean onlyIfAbsent) {
 //chm健和值都不可为空
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        //关键点！！！因为CAS需要不断失败重试，所有这是循环添加
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
                //情况一：hash出的对应下标没有结点,没有碰撞
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //关键点！！！CAS添加结点
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                             //添加失败则break进入下一循环
                    break;                   // no lock when adding to empty bin
            }
            //情况二：hash碰撞，且正在扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);//协助扩容
                //情况三：hash碰撞，为链表/树
            else {
                V oldVal = null;
                //关键点！！！锁住头结点，对树/链表进行操作时，先锁住头结点
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;//链表计数器
                            for (Node<K,V> e = f;; ++binCount) {
                             ...链表的插入与hashmap无异，这里就不放代码了...
                            }
                        }
                        //是不是树节点
                        else if (f instanceof TreeBin) {
                         ...红黑树的插入与hashmap无异，这里就不放代码了...
                        }
                    }
                }
                if (binCount != 0) {
                //还是原来的配方，链表长度是否大于8，大于就化为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }

```



总结

1.判断Node数组是否初始化,

2.通过hash定位数组的索引坐标,是否有Node节点,**如果没有则在循环中使用CAS进行添加(链表的头节点),添加失败则进入下次循环。**

3.如果检査到内部正在扩容,就帮助它一块扩容。

4.如果f=nu,则使用 synchronized锁住 头元素(链表/红黑二叉树的头元素)

- 41如果是Node(链表结构)则执行链表的添加操作。
- 4.2如果是 Treenode(树型结构)则执行树添加操作。

5.判断链表长度是否已经达到树化的阀值



## 扩容

并发扩容，实现方式是，将表拆分，让每个线程处理自己的区间。通过给每个线程分配桶区间，避免线程间的争用。而如果有新的线程想 put 数据时，也会帮助其扩容。

会将链表拆成两份，规则是根据节点的 hash 值取于 length，如果结果是 0，放在低位，否则放在高位。

## 总结

JDK6,7中的ConcurrentHashmap主要使用Segment来实现减小锁粒度，把HashMap分割成若干个Segment，在put的时候需要锁住Segment，get时候不加锁，使用volatile来保证可见性，当要统计全局时（比如size），首先会尝试多次计算modcount来确定，这几次尝试中，是否有其他线程进行了修改操作，如果没有，则直接返回size。如果有，则需要依次锁住所有的Segment来计算。

jdk7中ConcurrentHashmap中，当长度过长碰撞会很频繁，链表的增改删查操作都会消耗很长的时间，影响性能,所以jdk8 中完全重写了concurrentHashmap,代码量从原来的1000多行变成了 6000多 行，实现上也和原来的分段式存储有很大的区别。

主要设计上的变化有以下几点: 

1. 不采用segment而采用node，锁住node来实现减小锁粒度。
2. 设计了MOVED状态 当resize的中过程中 线程2还在put数据，线程2会帮助resize。
3. 使用3个CAS操作来确保node的一些操作的原子性，这种方式代替了锁。
4. sizeCtl的不同值来代表不同含义，起到了控制的作用。

至于为什么JDK8中使用synchronized而不是ReentrantLock，我猜是因为JDK8中对synchronized有了足够的优化吧。

> [总结](https://www.cnblogs.com/yuluoxingkong/p/9265730.html)
>
> [扩容这篇讲的很好](https://juejin.im/post/5b00160151882565bd2582e0)
>
> [占小狼扩容](https://www.jianshu.com/p/f6730d5784ad)
>
> [源码分析](https://my.oschina.net/hosee/blog/675884)



