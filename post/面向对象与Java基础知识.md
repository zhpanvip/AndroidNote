
##  1.Java中“==” 和 equals 有什么
”==“ 是 Java 中一种操作符，它有两种比较方式

 - 1.对于基本数据类型来说，" ==" 判断的是两边的值是否相等
 - 2.对于引用类型来说，  "=="判断的是两边的引用是否相等，也就是判断两个对象是否指向了同一块内存区域。
 
而equals方法 是 Object 类定义的一个方法。源码如下：

```
    public boolean equals(Object obj) {
        return (this == obj);
    }
```
可以看到，在Object中其实equals方法与使用”==“是等价的。但是Java的开发人员赋予equals方法的意义是比较两个对象的值是否相等。因此就需要我们在有需要的时候去根据自身需要来重写equals方法。重写equals方法的同时，需要一并重写hashCode方法。

## 2.为什么重写 equals 方法必须重写 hashcode 方法
个人认为重写equals方法并不一定就要重写hashcode。需要根据情况来讨论。
如果这个对象是用到Hash表实现的数据结构中的话，那么一定要重写hashcode方法。比如这个对象可能会被存储到HashMap、HashSet这样的数据结构中。
但是如果重写equals方法仅仅是为了比较两个对象是的值否相等或者该数据仅仅是存储在List这样的数据结构中，则是否重写hashcode方法其实并无影响。
但通常情况下，一个类的使用场景较多，如果开始这个类没有存储到Hash表的数据结构中的需求，但是并不能确定以后是否会出现这样的情况，所以规定重写equals方法必须重写hashCode方法。

那equals方法与hashcode有什么关系呢？为什么规定重写equals方法就必须重写hashCode呢？这就需要从Hash表实现的数据结构说起。以HashMap为例，HashMap内部是通过数组和链表来对数据进行存储的（使用链地址法来处理Hash冲突）。在HashMap中会通过hashcode来计算Key的hash值，并根据计算的Hash值对该元素进行散列存储。如果碰到hash值重复的情况，则将该元素插入到链表的头部。有相同hash值的元素会在同一个链表中。如下图：

![在这里插入图片描述](https://img-blog.csdn.net/20180422235248719?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Zpc2FudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
而从HashMap中取数据的时候，首会先根据Key的hash值确定元素桶的位置，接着通过Key的equals方法对比获取到查找的元素。那么，如果一个作为HashMap Key的对象只重写了equals方法，但没有重写hashcode，就会造成两个equals相等的Key，有不同的hashcode，进而造成了HashMap的同一个Key出现映射了两个Value的情况，显然这违背了HashMap的初衷，导致数据的错误。因此，规定重写equals方法需要一并重写hashcode方法。


##  3.下面的代码在JVM中生成了几个String对象？JVM是如何对其进行内存分配的？
```
String s1 = new String("abc")
```
对于这一问题应该分情况讨论，因为虚拟机在JDK7之前与之后对于字符串常量池的实现是不一样的。

**（1）JDK7之前**
对于String类型的数据，JVM维护了一个字符串常量池来存放字符串对象。JDK7之前，字符串常量池位于方法区（永久代）。上述代码中如果常量池中还没有”abc"对象，则会首先在字符常量池中生成一个”abc"对象，如果“abc"已经存在于字符串常量池，则不会再次生成。注意这个字符串对象是位于永久代而非堆内存。接着，在执行new的时候在堆内存中再生成一个“abc"对象，并将这个对象的引用赋值给了s1。因此上述代码可能生成两个String对象，也可能生成一个。如下图：
![在这里插入图片描述](https://img-service.csdnimg.cn/img_convert/b884fab5f0f3529dc7505374a8ff0b87.png#pic_center)


而在JDK7之后，由于常量池被移到了堆内存，并且常量池中不再存放字符对象，而是存放字符串的引用。
那么上述代码中则首先会在堆内存中生成一个“abc"对象，并将这个对象的引用会放入常量池。接着，在执行new时，会再次在堆中生成一个”abc"对象，并将这个对象引用赋值给s1。如下图：

![在这里插入图片描述](https://img-service.csdnimg.cn/img_convert/bfc6123171cfba3505c4b1c367fb1902.png#pic_center)

因此，无论是在JDK7之前还是之后，这一代码都会生成一个或者两个字符串对象，只是内存的分配会有所区别。

可参考:[Java进阶--深入理解Java中的字符串（一）](https://blog.csdn.net/qq_20521573/article/details/108425352)

## 4.了解String的intern()方法吗？它有什么作用？
当调用intern方法的时候，如果字符串常量池中已经包含了一个等于该String对象的字符串，则直接返回字符串常量池中该字符串的引用。否则，会将该字符串对象包含的字符串添加到常量池，并返回此对象的引用。

详情可参考:[Java进阶--深入理解Java中的字符串（一）](https://blog.csdn.net/qq_20521573/article/details/108425352)


##  5.String、StringBuffer与StringBuilder有区别？
（1）String 类被final修饰，同时内部维护的char[] 数组也是final修饰的，意味着String类是无法被继承和改变的。它内部不存在扩容机制。字符串拼接，截取等操作都会生成一个新的字符串对象。因此，频繁操作String对象效率会比较低。

（2）StringBuilder继承自AbstractStringBuilder， 类内部维护了一个可变长度char[] (即没有final修饰)， 初始化数组容量为16，当char[]的空间不足时会启用存在扩容机制。在操作StringBuilder的字符串时会调用System的native方法进行数组的拷贝。因此，不会像String一样重新生成新的字符串对象。 
但在调用 toString方法时不会共享StringBuilder对象内部的char[]，而是进行一次char[]的copy操作，使用新生成的char[]数组重新生成的String对象。toString代码如下：

```java
public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
```

另外，StringBuilder是非线程安全的。

（3）StringBuffer同样也继承自AbstractStringBuilder，由于大部分逻辑都是在AbstractStringBuilder中实现的，所以StringBuilder与StringBuilder差别不大。只是StringBuffer重写了相关方法，并加了synchronize关键字，保证了线程安全。另外一点是StringBuffer有一个String类型的toStringCach的缓存，在进行字符串操作的时候都会更新toStringCache的值。在调用toString方法时，返回的String共享了toStringCach。如下代码：

```java
@Override
    public synchronized String toString() {
        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);
        }
        return new String(toStringCache, true);
    }
```
    
详情参考：[Java进阶--深入理解Java中的字符串（二）](https://blog.csdn.net/qq_20521573/article/details/108521881)

##  6.访问修饰符public,private,protected,以及不写（默认）时的区别？

![在这里插入图片描述](https://img-blog.csdn.net/20180924164223893?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

类的成员不写访问修饰时默认为default。默认对于同一个包中的其他类相当于公开（public），对于不是同一个包中的其他类相当于私有（private）。受保护（protected）对子类相当于公开，对不是同一包中的没有父子关系的类相当于私有。Java中，外部类的修饰符只能是public或默认，类的成员（包括内部类）的修饰符可以是以上四种。

## 7.final有哪几种用法？每种用法是什么含义？
- 1）成员变量被final关键字修饰，意味着这个变量一旦被初始化便不可改变，相当于一个常量。final修饰的成员变量只能在初始化时或者构造方法中赋值。
- 2）方法被final关键字修饰时，意味着这个方法不能被其子类重写。但子类仍然可以继承这个方法，也就是说可以在子类中直接使用这个final方法。
- 3）当一个类被final修饰时，意味着这个类不能再被其他类继承。也就是说这个类在一个继承树中是一个叶子类，并且此类的设计已被认为很完美而不需要进行修改或扩展。

## 8.static 关键的作用

static 是 Java 中非常重要的关键字，static 表示的概念是 静态的，在 Java 中，static 主要用来：

- 修饰变量，static 修饰的变量称为静态变量、也称为类变量，类变量属于类所有，对于不同的类来说，static 变量只有一份，static 修饰的变量位于方法区中；static 修饰的变量能够直接通过 类名.变量名 来进行访问，不用通过实例化类再进行使用。
- 修饰方法，static 修饰的方法被称为静态方法，静态方法能够直接通过 类名.方法名 来使用，在静态方法内部不能使用非静态属性和方法
- static 可以修饰代码块，主要分为两种，一种直接定义在类中，使用 static{}，这种被称为静态代码块，一种是在类中定义静态内部类，使用 static class xxx 来进行定义。
- static 可以用于静态导包，通过使用 import static xxx  来实现，这种方式一般不推荐使用
- static 可以和单例模式一起使用，通过双重检查锁来实现线程安全的单例模式。

详情请参考 [static 还能难得住我？](https://mp.weixin.qq.com/s?__biz=MzI0ODk2NDIyMQ==&mid=2247484455&idx=1&sn=582d5d2722dab28a36b6c7bc3f39d3fb&chksm=e999f135deee7823226d4da1e8367168a3d0ec6e66c9a589843233b7e801c416d2e535b383be&token=1154740235&lang=zh_CN#rd)


## 9.内部类可以引用外部类的成员吗？有没有什么限制？

内部类如果不是static修饰的，那么它可以通过"this."访问创建它的外部类对象的所有属性。这是因为非静态内部类默认持有外部类的引用。怎么理解呢？其实是因为编译器在编译非静态内部类的时候给非静态内部类的构造方法加了一个外部类的参数，所以非静态内部类可以通过这个参数访问到外部类程成员和方法。可以通过javap -p来反汇编字节或者通过反射调用非静态内部类的构造方法来验证。
另外，非静态内部类无法直接实例化，而是需要先实例化外部类后再实例化非静态内部类，如果是通过反射调用非静态内部类的构造方法，则必须传入外部类的对象才可成功实例化非静态内部类。


内部类如果是sattic修饰的，则这个内部类和普通的类其实并没有差别，它只可以访问外部类对象的所有static属性。
一般普通类只有public或package的访问修饰，而内部类可以实现static，protected，private等访问修饰。
当从外部类继承的时候，内部类是不会被覆盖的，它们是完全独立的实体，每个都在自己的命名空间内，如果从内部类中明确地继承，就可以覆盖原来内部类的方法。

## 10.int和Integer有什么区别？
Java是一个近乎纯洁的面向对象编程语言，但是为了编程的方便还是引入了基本数据类型，但是为了能够将这些基本数据类型当成对象操作，Java为每一个基本数据类型都引入了对应的包装类型（wrapper class），int的包装类就是Integer，从Java 5开始引入了自动装箱/拆箱机制，使得二者可以相互转换。
Java 为每个原始类型提供了包装类型：
- 原始类型: boolean，char，byte，short，int，long，float，double
- 包装类型：Boolean，Character，Byte，Short，Integer，Long，Float，Double

```
class AutoUnboxingTest {
 
    public static void main(String[] args) {
        Integer a = new Integer(3);
        Integer b = 3;                  // 将3自动装箱成Integer类型
        int c = 3;
        System.out.println(a == b);     // false 两个引用没有引用同一对象
        System.out.println(a == c);     // true a自动拆箱成int类型再和c比较
    }
}
```
最近还遇到一个面试题，也是和自动装箱和拆箱有点关系的，代码如下所示：

```
public class Test03 {
 
    public static void main(String[] args) {
        Integer f1 = 100, f2 = 100, f3 = 150, f4 = 150;
 
        System.out.println(f1 == f2);
        System.out.println(f3 == f4);
    }
}
```
如果不明就里很容易认为两个输出要么都是true要么都是false。首先需要注意的是f1、f2、f3、f4四个变量都是Integer对象引用，所以下面的==运算比较的不是值而是引用。装箱的本质是什么呢？当我们给一个Integer对象赋一个int值的时候，会调用Integer类的静态方法valueOf，如果看看valueOf的源代码就知道发生了什么。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
IntegerCache是Integer的内部类，其代码如下所示：

```java
/**
     * Cache to support the object identity semantics of autoboxing for values between
     * -128 and 127 (inclusive) as required by JLS.
     *
     * The cache is initialized on first usage.  The size of the cache
     * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
     * During VM initialization, java.lang.Integer.IntegerCache.high property
     * may be set and saved in the private system properties in the
     * sun.misc.VM class.
     */
 
    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];
 
        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;
 
            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);
 
            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }
 
        private IntegerCache() {}
    }
```
简单的说，如果整型字面量的值在-128到127之间，那么不会new新的Integer对象，而是直接引用常量池中的Integer对象，所以上面的面试题中f1= =f2的结果是true，而f3==f4的结果是false。

## 11.Java 面向对象的特征有哪些方面？
- 抽象：抽象是将一类对象的共同特征总结出来构造类的过程，包括数据抽象和行为抽象两方面。抽象只关注对象有哪些属性和行为，并不关注这些行为的细节是什么。
- 继承：继承是从已有类得到继承信息创建新类的过程。提供继承信息的类被称为父类（超类、基类）；得到继承信息的类被称为子类（派生类）。继承让变化中的软件系统有了一定的延续性，同时继承也是封装程序中可变因素的重要手段（如果不能理解请阅读阎宏博士的《Java与模式》或《设计模式精解》中关于桥梁模式的部分）。
- 封装：通常认为封装是把数据和操作数据的方法绑定起来，对数据的访问只能通过已定义的接口。面向对象的本质就是将现实世界描绘成一系列完全自治、封闭的对象。我们在类中编写的方法就是对实现细节的一种封装；我们编写一个类就是对数据和数据操作的封装。可以说，封装就是隐藏一切可隐藏的东西，只向外界提供最简单的编程接口（可以想想普通洗衣机和全自动洗衣机的差别，明显全自动洗衣机封装更好因此操作起来更简单；我们现在使用的智能手机也是封装得足够好的，因为几个按键就搞定了所有的事情）。
- 多态性：多态性是指允许不同子类型的对象对同一消息作出不同的响应。简单的说就是用同样的对象引用调用同样的方法但是做了不同的事情。多态性分为编译时的多态性和运行时的多态性。如果将对象的方法视为对象向外界提供的服务，那么运行时的多态性可以解释为：当A系统访问B系统提供的服务时，B系统有多种提供服务的方式，但一切对A系统来说都是透明的（就像电动剃须刀是A系统，它的供电系统是B系统，B系统可以使用电池供电或者用交流电，甚至还有可能是太阳能，A系统只会通过B类对象调用供电的方法，但并不知道供电系统的底层实现是什么，究竟通过何种方式获得了动力）。方法重载（overload）实现的是编译时的多态性（也称为前绑定），而方法重写（override）实现的是运行时的多态性（也称为后绑定）。运行时的多态是面向对象最精髓的东西，要实现多态需要做两件事：1). 方法重写（子类继承父类并重写父类中已有的或抽象的方法）；2). 对象造型（用父类型引用引用子类型对象，这样同样的引用调用同样的方法就会根据子类对象的不同而表现出不同的行为）。



## 12.简述Java反射机制，反射的作用和应用？
在Java中，所有已经被虚拟机的类加载器加载过的类（称为T）都会在虚拟机中生成一个唯一的与T类所对应的Class<T>对象。在程序运行时，通过这个Class<T>对象，我们可以实例化出来一个T对象；可以通过Class<T>对象访问T对象中的任意成员变量，调用T对象中的任意方法，甚至可以对T对象中的成员变量进行修改。我们将这一系列操作称为Java的反射机制。

到这里我们发现，其实Java的反射也没有那么神秘了。说白了就是通过Class对象来操控我们的对象罢了。因此，接下来我们想要弄懂反射只需要来详细的认识一下Class这个类给我们提供的API即可。

详情请参考：[Java进阶--深入理解Java的反射机制
](https://blog.csdn.net/qq_20521573/article/details/111751156)

## 13.Java泛型是什么？泛型的类型擦除是怎么回事？
详情参考:[Java进阶--Java中的泛型详解](https://blog.csdn.net/qq_20521573/article/details/112712619)

## 14.什么是序列化以及用途，什么时候使用序列化？
序列化是指把对象转换为字节序列的过程称为对象的序列化；而反序列化是指把字节序列恢复为对象的过程称为对象的反序列化。
对象的序列化主要有两种用途：
1）把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中；
2）在网络上传送对象的字节序列。
什么时候使用序列化：
1）对象序列化可以实现分布式对象。主要应用例如：RMI要利用对象序列化运行远程主机上的服务，就像在本地机上运行对象时一样。
2）java对象序列化不仅保留一个对象的数据，而且递归保存对象引用的每个对象的数据。可以将整个对象层次写入字节流中，可以保存在文件中或在网络连接上传递。利用对象序列化可以进行对象的"深复制"，即复制对象本身及引用的对象本身。序列化一个对象可能得到整个对象序列。

## 15.如何实现对象克隆？
有两种方式：
1). 实现Cloneable接口并重写Object类中的clone()方法；
2). 实现Serializable接口，通过对象的序列化和反序列化实现克隆，可以实现真正的深度克隆，代码如下。

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
 
public class MyUtil {
 
    private MyUtil() {
        throw new AssertionError();
    }
 
    public static <T> T clone(T obj) throws Exception {
        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bout);
        oos.writeObject(obj);
 
        ByteArrayInputStream bin = new ByteArrayInputStream(bout.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bin);
        return (T) ois.readObject();
 
        // 说明：调用ByteArrayInputStream或ByteArrayOutputStream对象的close方法没有任何意义
        // 这两个基于内存的流只要垃圾回收器清理对象就能够释放资源，这一点不同于对外部资源（如文件流）的释放
    }
}
```
下面是测试代码：

```java
import java.io.Serializable;
 
/**
 * 人类
 * @author 骆昊
 *
 */
class Person implements Serializable {
    private static final long serialVersionUID = -9102017020286042305L;
 
    private String name;    // 姓名
    private int age;        // 年龄
    private Car car;        // 座驾
 
    public Person(String name, int age, Car car) {
        this.name = name;
        this.age = age;
        this.car = car;
    }
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public int getAge() {
        return age;
    }
 
    public void setAge(int age) {
        this.age = age;
    }
 
    public Car getCar() {
        return car;
    }
 
    public void setCar(Car car) {
        this.car = car;
    }
 
    @Override
    public String toString() {
        return "Person [name=" + name + ", age=" + age + ", car=" + car + "]";
    }
 
}
```

```java
/**
 * 小汽车类
 * @author 骆昊
 *
 */
class Car implements Serializable {
    private static final long serialVersionUID = -5713945027627603702L;
 
    private String brand;       // 品牌
    private int maxSpeed;       // 最高时速
 
    public Car(String brand, int maxSpeed) {
        this.brand = brand;
        this.maxSpeed = maxSpeed;
    }
 
    public String getBrand() {
        return brand;
    }
 
    public void setBrand(String brand) {
        this.brand = brand;
    }
 
    public int getMaxSpeed() {
        return maxSpeed;
    }
 
    public void setMaxSpeed(int maxSpeed) {
        this.maxSpeed = maxSpeed;
    }
 
    @Override
    public String toString() {
        return "Car [brand=" + brand + ", maxSpeed=" + maxSpeed + "]";
    }
 
}
```

```java
class CloneTest {
 
    public static void main(String[] args) {
        try {
            Person p1 = new Person("Hao LUO", 33, new Car("Benz", 300));
            Person p2 = MyUtil.clone(p1);   // 深度克隆
            p2.getCar().setBrand("BYD");
            // 修改克隆的Person对象p2关联的汽车对象的品牌属性
            // 原来的Person对象p1关联的汽车不会受到任何影响
            // 因为在克隆Person对象时其关联的汽车对象也被克隆了
            System.out.println(p1);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
注意：基于序列化和反序列化实现的克隆不仅仅是深度克隆，更重要的是通过泛型限定，可以检查出要克隆的对象是否支持序列化，这项检查是编译器完成的，不是在运行时抛出异常，这种是方案明显优于使用Object类的clone方法克隆对象。让问题在编译的时候暴露出来总是优于把问题留到运行时。

**7.Comparator接口简单描述一下？**

java.util.Comparator是比较器接口，如果我们需要控制某个类的次序并且该类本身不支持排序，那么就可以建立一个类比较器来进行排序，实现方式很简单只需要实现java.util.Comparator接口。
java.util.Comparator接口只包括两个函数，它的源码如下：

```
package java.util;    

public interface Comparator<T> {    

    int compare(T o1, T o2);    

    boolean equals(Object obj);    

}   
```

1）若一个类要实现java.util.Comparator接口：它一定要实现int compare(T o1, T o2) 函数，而另一个可以不实现boolean equals(Object obj) 函数
2）int compare(T o1, T o2) 是比较o1和o2的大小，如果返回值为负数意味着o1比o2小，否则返回为零意味着o1等于o2，返回为正数意味着o1大于o2
## 16.Comparable 接口简单描述一下？
Comparable 是排序接口。若一个类实现了Comparable接口，就意味着“该类支持排序”。  即然实现Comparable接口的类支持排序，假设现在存在“实现Comparable接口的类的对象的List列表(或数组)”，则该List列表(或数组)可以通过 Collections.sort（或 Arrays.sort）进行排序。

此外，“实现Comparable接口的类的对象”可以用作“有序映射(如TreeMap)”中的键或“有序集合(TreeSet)”中的元素，而不需要指定比较器。
Comparable 接口仅仅只包括一个函数，它的定义如下：

```
package java.lang;
import java.util.*;

public interface Comparable<T> {
    public int compareTo(T o);
}
```
假设我们通过 x.compareTo(y) 来“比较x和y的大小”。若返回“负数”，意味着“x比y小”；返回“零”，意味着“x等于y”；返回“正数”，意味着“x大于y”。
## 17.comparable 和 comparator的区别
　Comparable是排序接口，若一个类实现了Comparable接口，就意味着“该类支持排序”。而Comparator是比较器，我们若需要控制某个类的次序，可以建立一个“该类的比较器”来进行排序。

　　Comparable相当于“内部比较器”，而Comparator相当于“外部比较器”。

　　两种方法各有优劣， 用Comparable 简单， 只要实现Comparable 接口的对象直接就成为一个可以比较的对象，但是需要修改源代码。 用Comparator 的好处是不需要修改源代码， 而是另外实现一个比较器， 当某个自定义的对象需要作比较的时候，把比较器和对象一起传递过去就可以比大小了， 并且在Comparator 里面用户可以自己实现复杂的可以通用的逻辑，使其可以匹配一些比较简单的对象，那样就可以节省很多重复劳动了。