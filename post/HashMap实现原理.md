## 1.HashMap的工作原理
HashMap是基于哈希表实现的，用于存储key-value的键值对，并允许使用null值和null键。由于是基于Hash表实现的，因此HashMap具有较高的查询效率，理想情况下HashMap的查找时间复杂度可达到O(1)。

**(1)HashMap的存储结构**
HashMap实际上是一个“链表散列”的数据结构，即数组和链表的结合体（链地址法处理冲突）。HashMap内部封装了一个包含key和value的Entry类，用于存储键值对。在put操作中会根据key的hashcode计算在哈希表中的存储位置，并将Entry存入该位置。由于存在Hash冲突的情况，HashMap采用了链地址法来处理Hash冲突。即使用链表的形式将相同哈希值的元素连起来。如下图所示：
![在这里插入图片描述](https://img-blog.csdn.net/20180422235248719?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Zpc2FudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
取元素时在HashMap的get方法中计算key的哈希值，并定位到元素所在桶的位置，接着使用equals方法查找到目标元素。

**（2) HashMap的扩容与ReHash**
由于HashMap存的长度是确定的，可以初始化时候指定长度或者默认长度16。随着HashMap中插入的元素越来越多，发生哈希冲突的概率会越来越大，相应的查找的效率就会越来越低。这意味着影响哈希表性能的因素除了哈希函数与处理冲突的方法之外，还与哈希表的装填因子大小有关。

> 我们将哈希表中元素数与哈希表长度的比值称为**装填因子**。装填因子 **α= $\frac{哈希表中元素数}{哈希表长度}$**

在HashMap中，装填因子的阈值为0.75，当装填因子大于0.75时则会出发HashMap的扩容机制。这里我们应该知道，扩容并不是在原数组基础上扩大容量，而是需要申请一个长度为原来2倍的新数组。因此，扩容之后就需要将原来的数据从旧数组中重新散列存放到扩容后的新数组。这个过程我们称之为Rehash。

Rehash的操作将会重新散列扩容前已经存储的数据，这一操作涉及大量的元素移动，是一个非常消耗性能的操作。因此，在开发中我们应该尽量避免Rehash的出现。比如，可以预估元素的个数，事先指定哈希表的长度，这样可以有效减少Rehash。

**（3）JDK1.8中对HashMap的优化**
JDK1.7 中，HashMap 采用位桶 + 链表的实现，即使用链表来处理冲突，同一 hash 值的链表都存储在一个数组中。但是当位于一个桶中的元素较多，即 hash 值相等的元素较多时，通过 key 值依次查找的效率较低。如下图所示的元素查找效率直接降低为了链表的时间复杂度o(n)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200923232503838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)
为了优化这一问题，JDK 1.8 在底层结构方面做了一些改变，当每个桶中元素大于 8 的时候，会转变为红黑树，而红黑树的查找效率为o(logn)，高于链表的查找效率。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200927225706751.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)
详情参考：
[面试官：哈希表都不知道，你是怎么看懂HashMap的？](https://blog.csdn.net/qq_20521573/article/details/108701471)
[HashMap原理深入理解](https://blog.csdn.net/visant/article/details/80045154)
[HashMap的工作原理](https://blog.csdn.net/ty564457881/article/details/78206049)

## 2.HashMap扩容为什么是2的幂次方？

- 为了保证扩容后的数组索引与扩容前的数组索引一致

- Rehash时的取余操作hash % length == hash & (length - 1)这个关系只有在length等于二的幂次方时成立

### 1). 保证新数组与老数组的索引一致

HashMap的初始容量是2的4次幂，扩容时候也是2的倍数。这样可以使添加的元素均匀的分布在HashMap中的数组上，减少Hash冲突。避免形成链表结构，进而提升查询效率。

16的二进制表示为10000，length-1的二进制位表示为01111，扩容之后变为32，二进制表示为100000，减1之后的二进制位为011111。

扩容前后只有一位的差距，即相当于(32-1)>>1==(16-1)。如下图所示：



![1b9f4958eac5224f0e625dc679b3282d](https://img-blog.csdnimg.cn/img_convert/1b9f4958eac5224f0e625dc679b3282d.png)

这样在通过has&(length-1)时，只要hash对应的最左边的哪一个差异位为0，就能保证到扩容后的数组和老数组的索引一致。

当数组长度保持2的次幂，length-1的低位都为1，这样会使得数组索引index更加均匀。如下图:

![a488d3b9cf221fac3f00681604fba489](https://img-blog.csdnimg.cn/img_convert/a488d3b9cf221fac3f00681604fba489.png)

上面的&运算，高位是不会对结果产生影响的（hash函数采用各种位运算也是为了使得低位更加散列）。只关注低位的bit,如果低位全部为1，那么对于h低位部分来说任何一位的变化都会对结果产生影响。也就是说，只要得到index=21这个存储位置，h的低位只有这一种组合。这也是数组长度设计为2次幂的原因

![c28b9183a595598f0a4f2561797d51fe](https://img-blog.csdnimg.cn/img_convert/c28b9183a595598f0a4f2561797d51fe.png)

如果不是2的次幂，也就是低位不是全为1此时，要使得index=21，h的低位部分不再具有唯一性了，哈希冲突的几率会变的更大，同时，index对应的这个bit位无论如何不会等于1了，而对应的那些数组位置也就被白白浪费了。


https://www.cxyzjd.com/article/weixin_44273302/113733422

https://blog.csdn.net/samniwu/article/details/90550196

