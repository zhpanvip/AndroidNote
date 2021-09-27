
----
![这里写图片描述](https://img-blog.csdn.net/20180907225150827?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


![这里写图片描述](https://img-blog.csdn.net/20180907225012225?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


## 2.为什么HashMap在多线程并发存在死循环的问题，JDK1.8中做了哪些优化？
详情参考
[《我们一起进大厂》系列-HashMap](https://juejin.cn/post/6844904017269637128)
[老生常谈，HashMap的死循环](https://juejin.cn/post/6844903554264596487)
[HashMap为何从头插入改为尾插入](https://juejin.cn/post/6844903682664824845)

## 3.Hashtable与HashMap有什么区别？
**1.安全性**
Hashtable是线程安全，HashMap是非线程安全。HashMap的性能会高于Hashtable，我们平时使用时若无特殊需求建议使用HashMap，在多线程环境下若使用HashMap需要使用Collections.synchronizedMap()方法来获取一个线程安全的集合（Collections.synchronizedMap()实现原理是Collections定义了一个SynchronizedMap的内部类，这个类实现了Map接口，在调用方法时使用synchronized来保证线程同步

**2.是否可以使用null作为key**

HashMap可以使用null作为key，不过建议还是尽量避免这样使用。HashMap以null作为key时，总是存储在table数组的第一个节点上。而Hashtable则不允许null作为key

**3.继承了什么，实现了什么**

HashMap继承了AbstractMap，HashTable继承Dictionary抽象类，两者均实现Map接口

**4.默认容量及如何扩容**
HashMap的初始容量为16，Hashtable初始容量为11，两者的填充因子默认都是0.75。HashMap扩容时是当前容量翻倍即:capacity  2，Hashtable扩容时是容量翻倍+1即:capacity  (2+1)

**5.底层实现**

HashMap和Hashtable的底层实现都是数组+链表结构实现

**6.计算hash的方法不同**

Hashtable计算hash是直接使用key的hashcode对table数组的长度直接进行取模，HashMap计算hash对key的hashcode进行了二次hash，以获得更好的散列值，然后对table数组长度取模

## 4.了解ConcurrentHashMap吗？它是怎么实现的?
在多线程环境下，使用HashMap进行put操作时存在丢失数据的情况，为了避免这种bug的隐患，强烈建议使用ConcurrentHashMap代替HashMap。

HashTable是一个线程安全的类，它使用synchronized来锁住整张Hash表来实现线程安全，即每次锁住整张表让线程独占，相当于所有线程进行读写时都去竞争一把锁，导致效率非常低下。ConcurrentHashMap可以做到读取数据不加锁，并且其内部的结构可以让其在进行写操作的时候能够将锁的粒度保持地尽量地小，允许多个修改操作并发进行，其关键在于使用了锁分段技术。它使用了多个锁来控制对hash表的不同部分进行的修改。对于JDK1.7版本的实现, ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的Hashtable，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行。JDK1.8的实现降低锁的粒度，JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry（首节点）。

**实现原理**

对于JDK1.7版本的实现，ConcurrentHashMap 为了提高本身的并发能力，在内部采用了一个叫做 Segment 的结构，一个 Segment 其实就是一个类 Hash Table 的结构，Segment 内部维护了一个链表数组，我们用下面这一幅图来看下 ConcurrentHashMap 的内部结构,从下面的结构我们可以了解到，ConcurrentHashMap 定位一个元素的过程需要进行两次Hash操作，第一次 Hash 定位到 Segment，第二次 Hash 定位到元素所在的链表的头部，因此，这一种结构的带来的副作用是 Hash 的过程要比普通的 HashMap 要长，但是带来的好处是写操作的时候可以只对元素所在的 Segment 进行操作即可，不会影响到其他的 Segment，这样，在最理想的情况下，ConcurrentHashMap 可以最高同时支持 Segment 数量大小的写操作（刚好这些写操作都非常平均地分布在所有的 Segment上），所以，通过这一种结构，ConcurrentHashMap 的并发能力可以大大的提高。我们用下面这一幅图来看下ConcurrentHashMap的内部结构详情图，如下:

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/571961bfa9ba57846aaeb457326c138d.png#pic_center)

为什么要用二次hash，主要原因是为了构造分离锁，使得对于map的修改不会锁住整个容器，提高并发能力。当然，没有一种东西是绝对完美的，二次hash带来的问题是整个hash的过程比hashmap单次hash要长，所以，如果不是并发情形，不要使concurrentHashmap。

JAVA7之前ConcurrentHashMap主要采用锁机制，在对某个Segment进行操作时，将该Segment锁定，不允许对其进行非查询操作，而在JAVA8之后采用CAS无锁算法，这种乐观操作在完成前进行判断，如果符合预期结果才给予执行，对并发操作提供良好的优化.

参考：[一文读懂Java ConcurrentHashMap原理与实现](https://zhuanlan.zhihu.com/p/104515829)

## 5.可以使用CocurrentHashMap来代替Hashtable吗？
我们知道Hashtable是synchronized的，但是ConcurrentHashMap同步性能更好，因为它仅仅根据同步级别对map的一部分进行上锁。ConcurrentHashMap当然可以代替HashTable，但是HashTable提供更强的线程安全性。它们都可以用于多线程的环境，但是当Hashtable的大小增加到一定的时候，性能会急剧下降，因为迭代时需要被锁定很长的时间。因为ConcurrentHashMap引入了分割(segmentation)，不论它变得多么大，仅仅需要锁定map的某个部分，而其它的线程不需要等到迭代完成才能访问map。简而言之，在迭代的过程中，ConcurrentHashMap仅仅锁定map的某个部分，而Hashtable则会锁定整个map。

## 6.ConcurrentHashMap有什么缺陷吗？
ConcurrentHashMap 是设计为非阻塞的。在更新时会局部锁住某部分数据，但不会把整个表都锁住。同步读取操作则是完全非阻塞的。好处是在保证合理的同步前提下，效率很高。坏处是严格来说读取操作不能保证反映最近的更新。例如线程A调用putAll写入大量数据，期间线程B调用get，则只能get到目前为止已经顺利插入的部分数据。

## 7.ConcurrentHashMap在JDK 7和8之间的区别

JDK1.8的实现降低锁的粒度，JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry（首节点）

JDK1.8版本的数据结构变得更加简单，使得操作也更加清晰流畅，因为已经使用synchronized来进行同步，所以不需要分段锁的概念，也就不需要Segment这种数据结构了，由于粒度的降低，实现的复杂度也增加了

JDK1.8使用红黑树来优化链表，基于长度很长的链表的遍历是一个很漫长的过程，而红黑树的遍历效率是很快的，代替一定阈值的链表，这样形成一个最佳拍档
## 9.Java中HashMap和HashTable的区别？

**相同点**

HashMap 和 HashTable 都是基于哈希表实现的，其内部每个元素都是 key-value 键值对，HashMap 和 HashTable 都实现了 Map、Cloneable、Serializable 接口。

**不同点**

- **（1）继承的父类不同**

HashMap是继承自AbstractMap类，而HashTable是继承自Dictionary类。不过它们都同时实现了map、Cloneable（可复制）、Serializable（可序列化）这三个接口
- **（2） 线程安全性不同**

Hashtable是线程安全的，它的每个方法中都加入了Synchronize方法。在多线程并发的环境下，可以直接使用Hashtable，不需要自己为它的方法实现同步
HashMap不是线程安全的，在多线程并发的环境下，可能会产生死锁等问题。具体的原因在下一篇文章中会详细进行分析。使用HashMap时就必须要自己增加同步处理，虽然HashMap不是线程安全的，但是它的效率会比Hashtable要好很多。

- **（3）空值不同**

- HashMap 允许空的 key 和 value 值，HashTable 不允许空的 key 和 value 值。HashMap 会把 Null key 当做普通的 key 对待。不允许 null key 重复。

- **（4） 初始容量大小和每次扩充容量大小的不同**

Hashtable默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。

## 10.HashMap 和 HashSet 的区别

HashSet 继承于 AbstractSet 接口，实现了 Set、Cloneable,、java.io.Serializable 接口。HashSet 不允许集合中出现重复的值。HashSet 其实就是用HashMap来实现的，所有对 HashSet 的操作其实就是对 HashMap 的操作。所以 HashSet 也不保证集合的顺序，也不是线程安全的容器。

## 11.请说出 ArrayList和LinkedList的区别？
1)ArrayList和LinkedList可想从名字分析，它们一个是Array(动态数组)的数据结构，一个是Link(链表)的数据结构，此外，它们两个都是对List接口的实现。
前者是数组队列，相当于动态数组；后者为双向链表结构，也可当作堆栈、队列、双端队列

2)当随机访问List时（get和set操作），ArrayList比LinkedList的效率更高，因为LinkedList是线性的数据存储方式，所以需要移动指针从前往后依次查找。

3)当对数据进行增加和删除的操作时(add和remove操作)，LinkedList比ArrayList的效率更高，因为ArrayList是数组，所以在其中进行增删操作时，会对操作点之后所有数据的下标索引造成影响，需要进行数据的移动。

4)从利用效率来看，ArrayList自由性较低，因为它需要手动的设置固定大小的容量，但是它的使用比较方便，只需要创建，然后添加数据，通过调用下标进行使用；而LinkedList自由性较高，能够动态的随数据量的变化而变化，但是它不便于使用。

5)ArrayList主要控件开销在于需要在lList列表预留一定空间；而LinkList主要控件开销在于需要存储结点信息以及结点指针信息。

## 12.Java 中 Set 与 List 有什么不同?
List,Set都是继承自Collection接口

List特点：元素有放入顺序，元素可重复 ，

Set特点：元素无放入顺序，元素不可重复（注意：元素虽然无放入顺序，但是元素在set中的位置是有该元素的HashCode决定的，其位置其实是固定的）

List接口有三个实现类：LinkedList，ArrayList，Vector ，

Set接口有两个实现类：HashSet(底层由HashMap实现)，LinkedHashSet

## 13.请说出 ArrayList和Vector的区别？

1）不同的在于序列化方面：ArrayList比Vector安全

在ArrayList集合中：

//采用elementData数组来存储集合元素
private transient Object[] elementData;

在Vector集中中：

//采用elementData数组来保存集合元素
private Object[] elementData;

从源码可以看出，ArrayList提供的writeObject和readObject方法来实现定制序列化，而Vector只是提供了writeObject方法，并没有完全实现定制序列化。

2）不同点在于Vector是线性安全的，ArrayList是非线性安全的

ArrayList和Vector的绝大部分方法都是一样的，甚至连方法名都一样,只是Vector的方法大都添加关键之synchronized修饰。

在add方法中，Vector电泳的是insertElementAt(element，index)；

public synchronized void isnertElementAt(E obj,int index);

将 ArrayList中的add（int index，E element）方法和Vector的isnertElementAt（E Obj，int index）方法进行对比，可以发现vectorde insertElementAt(E obj,int index)方法只是多了synchronized修饰。

3）扩容上区别

ArrayList集合和Vector集合底层都是数组实现的，在数组容量不足的时候采取的扩容机制不同。

ArrayList集合容量不足，采取在原有容量基础上扩充为原来的1.5倍。

而Vector则多了一个选择：当capacityIncrement实例变量大于0时，扩充为原有容量加上capacityIncrement的容量值。否则采取在原有容量基础上扩充为原来的1.5倍。

## 14.说出ArrayList,Vector, LinkedList的存储性能和特性

ArrayList 和Vector都是使用数组方式存储数据，此数组元素数大于实际存储的数据以便增加和插入元素，它们都允许直接按序号索 引元素，但是插入元素要涉及数组元素移动等内存操作，所以索引数据快而插入数据慢，Vector由于使用了synchronized方法（线程 安全），通常性能上较ArrayList差，而LinkedList使用双向链表实现存储，按序号索引数据需要进行前向或后向遍历，但是插入数 据时只需要记录本项的前后项即可，所以插入速度较快。

## 15.Collection 和 Collections的区别？

Collection是集合类的上级接口，继承与他的接口主要有Set 和List。

Collections是针对集合类的一个帮助类，他提供一系列静态方法实现对各种集合的搜索、排序、线程安全化等操作。