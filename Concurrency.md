Concurrency

在CAS算法中，需要取出内存中某时刻的数据（由用户完成），在下一时刻比较并替换（由CPU完成，该操作是原子的）。这个时间差中，会导致数据的变化。

AtomicInteger实现：
内有一个volatile声明的域，执行incrementAndGet()方法时，会调用unsafe的getAndAddInt方法：
源码：
```java
/*
    var2-->offset 内存偏移地址
 */
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);	//一个native方法，getIntVolatile方法用于在对象指定偏移地址
												//（var2=valueOffset,通过反射拿到某个field的偏移量)处volatile读取一个int。
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));	//判断主内存中的值和
																		//当前期望的值（var1.value)
    																	//相等则更新成var5 + var4
    																	//不相等就继续循环（自旋）
                                                                        //从内存中拿到最新的var1的值去更新var5
    return var5;
}
```


CAS  的 ABA 问题： 其他线程将A的值改成B又改回来A
解决： 变量+版本号
实现类： AtomicStampedReferenc


while/if + CAS:
AtomicBoolean 让代码只执行一次：
因为一旦执行过后该变量便会变成true,那么之后的针对该变量的操作都不会的到执行
```java
if(atomicBoolean.compareAndSet(false, true)){
	//CODE
}
```


锁：
1. synchronized 依赖JVM
2. Lock 依赖特殊的CPU指令，代码实现，例如ReetrantLock


synchronized
1. 修饰代码块， 大括号括起来的代码，作用于括号中的对象
```java
synchronized(this/ClassA.class){
	//若是this则作用对象是当前对象
    //若是类对象则作用对象是这个类的所有对象
}
```
2. 修饰方法， 整个方法，作用于调用的对象
```java
public sychronized method(){
	
}
```
3. 修饰静态方法，作用于该类的所有对象
```java
public static synchronized method(){
	
}
```

synchronized/Lock/Atomic

synchronized: 不可中断，适合竞争不激烈的情况，可读性较好
Lock: 可以中断(unlock), **多样化**同步， 竞争激烈时能维持常态
Atomic: 竞争激烈时能维持常态, 比Lock性能好，*只能同步一个值*


可见性：

synchronized: 
	1. 加锁前，一定清空工作内存中共享变量的值，从而使用共享变量时需要从主内存中*重新*读取最新的值
	2. 解锁时， 必须将共享变量的最新值刷新回主内存
	3. 加锁和解锁是同一把锁

volatile：
通过内存屏障和限制处理器指令重排序实现可见性，实现volatile在JMM中的内存语义。
1. 对volatile变量写操作时，会在写操作之后加入一条store屏障指令，将本地内存中的共享变量的值刷新回主内存
2. 对volatile变量读操作时，会在写操作之后加入一条load屏障指令, 将主存中最新的值读取到工作内存


关于 volatile 重排序的问题：
对于 volatile 修饰的变量， 在读取该变量之前的指令，可以重排序；在读取该变量之后的指令可以重排序；
但重排序之后 volatile 变量的操作永远在中间。前后的重排序都不能逾越这个分界线。

但volatile不保证原子性，不能执行计数功能

volatile使用：
1. 写操作不依赖当前值(适合作为标记位)， 如果依赖当前值那么就是下面这种情况了，可能多个线程都判断满足条件A

```java
if (condition A){
    //do something
}
```
2. double check

有序性：

happens-before原则（可用来推导执行顺序）：
1. 程序次序规则：一个线程内，按照代码顺序执行
2. 锁定规则：一个unlock操作先行发生于对后面同一个锁的lock操作
3. volatile变量规则： 对一个变量的写操作先行发生于后面对这个变量的读操作
4. 传递规则


若不能由happens-before原则推到执行顺序，就不能保证有序性，那么就不是线程安全的


发布对象：使一个对象能被当前范围之外的代码所使用。
对象溢出：一种错误的发布，当一个对象还没有构造完成时，就使它被其他线程所见

对象溢出的例子：
```java
@NotThreadSafe
@NotRecommend
public class Escape {

    private int thisCanBeEscape = 0;

    public Escape () {
        new InnerClass();
    }

    private class InnerClass {

        public InnerClass() {
            System.out.println(Escape.this.thisCanBeEscape);
        }
    }

    public static void main(String[] args) {
        new Escape();
    }
}
```

在对象创建期间，this引用溢出了。


安全发布对象的方式：
1. 在静态初始化函数中初始化一个对象引用
2. 将对象的引用保存到volatile类型域或AtomicReference对象中
3. 将对象的引用保存到某个被正确构造的对象的final类型域中
4. 将对象的引用保存到一个被锁保护的域中 

单例模式的不同写法：
普通写法：
1. 懒汉模式,类在第一次使用时才初始化
```java
@NonThreadSafe
public class Singleton1 {
    private Singleton1(){

    }

    private static Singleton1 instance = null;
    public static Singleton1 getInstance(){
        if(instance == null){
            instance = new Singleton1();
        }
        return instance;
    }
}
```
线程不安全，假设两个线程同时拿到instance为null,那么new方法会使用两次。就不是真正的单例模式。
如果构造方法里还有一些其他的运算和操作，那么可能会出现逻辑错误。

例：假设同时有200个线程执行一下这个方法？那么set的size最终会变成多少呢？我们期望的单例应该返回1。
注：set添加时会根据(哈希码 && (同一对象的引用||equals))判读是否重复。
```java
//使用线程同步的Set,避免丢失添加
public static Set<Singleton1> set = Collections.synchronizedSet(new HashSet<Singleton1>());

private static void add() {
    Singleton1 instance = Singleton1.getInstance();
    set.add(instance);
}
```

结果是每次返回的值不同，但都大于1。

2. 饿汉模式,类在加载后就进行初始化,不管这个类是否以后会被使用
```java
@ThreadSafe
public class Singleton2 {
    private Singleton2(){

    }

    private static Singleton2 instance = new Singleton2();
    public static Singleton2 getInstance(){
        return instance;
    }

}
```
线程绝对安全，静态代码块必定只执行一次。

3. 双重检测 + volatile禁止指令重拍 的饿汉模式

```java
@ThreadSafe
public class Singleton {
    private Singleton(){

    }

    private static volatile Singleton instance = null;
    public static Singleton getInstance(){
        if(instance == null){   //B
            synchronized (Singleton.class){
                if(instance == null)
                    instance = new Singleton();    //A - 3
            }
        }
        return instance;
    }
}
```

若不加volatile修饰，则可能会出现指令重排
1、memory = allocate() 分配对象的内存空间
2、initInstance() 初始化对象
3、instance = memory 设置instance指向刚分配的内存

JVM和cpu优化，发生了指令重排

1、memory = allocate() 分配对象的内存空间
3、instance = memory 设置instance指向刚分配的内存
2、initInstance() 初始化对象

在A执行到3而还没执行2时，instance已经不为空，指向了刚刚分配到的地址。
这时候假设B正好在第一重判断中，判断instance并不为空。
那么直接对该对象的某些域访问会出问题，因为它没有被正确的初始化。
 

4. synchronized 修饰静态方法 getInstance()
```java
@ThreadSafe
@NotRecommend
public class SingletonExample3 {

    private SingletonExample3() {

    }

    private static SingletonExample3 instance = null;

    public static synchronized SingletonExample3 getInstance() {
        if (instance == null) {
            instance = new SingletonExample3();
        }
        return instance;
    }
}
```

synchronized修饰静态方法，那么能保证该方法在同一时间只能被该类的一个对象访问。因此是线程安全的，但这个锁比较重量级，作用的范围较大，降低并行度。

5. 用枚举类实现

```java
@ThreadSafe
public class Singleton6 {
    private Singleton6(){
        count.incrementAndGet();
    }


    public static AtomicInteger count = new AtomicInteger(0);
    public static Singleton6 getInstance(){
        return Singleton.INSTANCE.getInstance();
    }

    public enum Singleton{
        INSTANCE;
        private Singleton6 singleton;
        private Singleton(){
            singleton = new Singleton6();
        }
        public Singleton6 getInstance(){
            return singleton;
        }
    }
}
```
java的枚举类能保证构造方法仅被调用一次。因此肯定是线程安全的单例。

6. Holder模式（静态内部类或者枚举）
```java
public class StaticInnerClassSingleton {
    private static class InnerClass{
        private static StaticInnerClassSingleton staticInnerClassSingleton = new StaticInnerClassSingleton();
    }
    public static StaticInnerClassSingleton getInstance(){
        return InnerClass.staticInnerClassSingleton;
    }
    private StaticInnerClassSingleton(){
        if(InnerClass.staticInnerClassSingleton != null){
            throw new RuntimeException("单例构造器禁止反射调用");
        }
    }
}
```




不可变对象：

为什么不可变对象是安全的呢？
因为你可以不用考虑这个对象可能被发布到哪些线程使用，他们使用的都会是这个对象初始的值，因为不可变对象的各个域都是final的，他们都是不可变的，对象一旦被创建，被初始化之后，它内部的各个值都不能被第二次改变。因此所有线程拿到的都是同一个对象的统一状态，不同线程之间不对这个共享资源作任何修改。相当于它是只读的，只读的话各个线程之间是完全隔离的，因此不会出现由于多线程引发的数据一致性的问题。

满足条件：
1. 创建后状态就不能修改，不能改变指向
2. 对象的所有域都是final域
3. 对象是正确创建的（在对象创建期间，this引用没有溢出）


final：
1. 修饰类， 不能被继承
2. 修饰方法， 锁定方法不被继承类修改，也就是不能被覆盖
3. 修饰基本数据类型变量， 数值不能被修改
4. 修饰引用类型变量， 不能改变引用指向， 但可以改变这个对象内部属性的值（因此不可变对象，注意是不是所有的域都是不可变的）


Java库里的不可变对象：也就是只读的集合
1. Collections.unmodifiableList(List<T> list), 类似的还有map/set
实现原理：
    通过创建一个新的内部类（Unmodifiable）对象，内部有一个list域，将list域指向传入的list参数
改写所有引起修改的set/add/remove等方法，方法体直接抛异常即可。

```java
public boolean add(E e) {
    throw new UnsupportedOperationException();
}
```

2. Guava的ImmutableXXX， set/list/mapj


线程封闭： 把对象封装到一个线程里， 让对象只能被这个线程看到
1. 堆栈封闭： 局部变量， 无并发问题
2. ThreadLocal线程封闭： 通过一个Map<ThreadID, Object> 

线程不安全的类：
1. StringBuilder/StringBuffer， 后者为线程的同步类
这两个类都继承自AbstractStringBuilder，查看源码分析append方法：

```java
//抽象类中的append
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}

//StringBuilder
@Override
public StringBuilder append(String str) {
    super.append(str);
    return this;
}

//StringBuffer
@Override
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}
```

可以看到StringBuffer在append方法上添加了synchronized关键字修饰,限定这个方法在同一时间只允许持有这个StringBuffer对象的一个线程所调用。

以下的同步容器都是在方法上额外修饰synchronized实现的同步方法。

2. ArrayList/Vector
3. HashMap/HashTable
4. Collections.synchronizedXXX(List/Set/Map)




**同步容器**并不是所有场合下都能做到线程安全，例如下面的例子。而且对整个读写方法都加上了synchronized,性能也不高。因此我们接下来会介绍**并发容器**。

```java
private static Vector<Integer> vector = new Vector<>();

    public static void main(String[] args) {

        while (true) {

            for (int i = 0; i < 10; i++) {
                vector.add(i);
            }

            Thread thread1 = new Thread() {
                public void run() {
                    for (int i = 0; i < vector.size(); i++) {
                        vector.remove(i);
                    }
                }
            };

            Thread thread2 = new Thread() {
                public void run() {
                    for (int i = 0; i < vector.size(); i++) {
                        vector.get(i);
                    }
                }
            };
            thread1.start();
            thread2.start();
        }
    }
```

线程1执行删除vector中所有元素的操作， 线程2执行获取所有元素的操作。
这个程序的结果是抛出java.lang.ArrayIndexOutOfBoundsException。

原理类似于以下这个模型：
先检查再执行模型： `if(condition(a)) { handle(a);}`

condition(a) 是一个同步方法， handle(a)也是一个同步方法。
但在condition和handle之间的间隙有可能其他线程会做一些其他的操作，也就是这两个同步方法合在一起不是原子性的。
因此在handle(a)时，会出现一些线程不安全带来的异常或逻辑错误。

关于迭代器：
在使用迭代器或者增强型for循环时，本质都是迭代器模式。
在这种遍历模式下，直接修改集合的结构是不允许的，会触发fail-fast机制，抛出java.util.ConcurrentModificationException
正确的遍历中修改应调用迭代器自身的remove方法。

普通的for循环是没有问题的。


并发容器：
1. ArrayList -- CopyOnWriteArrayList
2. HashSet -- CopyOnWriteArraySet


CopyOnWrite：
```java
/** The lock protecting all mutators */
final transient ReentrantLock lock = new ReentrantLock();

/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;
```
所有读方法包括size()都没加锁，因此读的性能很好

写方法：
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

通过可重入锁，实现互斥。保证修改集合结构的操作在同一时间只在一个线程内执行。

既然已经在写方法中加锁，为什么还要进行Copy？

1. 首先， 做一个copy, 在这个copy上进行增加、删除或者修改，这些操作不会影响原来的Objects[] array。
2. 其次，在一个线程使用迭代器进行遍历时，另外的线程对这个集合修改，不会抛出ConcurrentModificationException，也就是可以实现一个安全的遍历。虽然在遍历过程中只是一个快照，不能实时的拿到最新修改的值，但它可以保证最终的一致性。实时的一致性还需要结合一定的同步措施。

COWIterator源码：

```java
static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }

    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    public boolean hasPrevious() {
        return cursor > 0;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }

    public int nextIndex() {return cursor; }
    public int previousIndex() {return cursor-1; }

    public void remove() {throw new UnsupportedOperationException(); }
    public void set(E e) {throw new UnsupportedOperationException(); }
    public void add(E e) {throw new UnsupportedOperationException(); }
}
```

可以看到这个*Iterator*的实现禁止了一切修改操作，只能进行只读的遍历。并且很关键的一点，这个迭代器持有的是一个snapshot,在遍历期间如果其他线程对这个集合修改，不会影响snapshot最初指向的对象数组，因为每一个修改的方法都没有对最初的`Object[] element`作出任何修改。而当所有在修改发生后的遍历结束后，这个snapshot也就相应的失效，因为下一次他们将指向新的`Object[] element`
,所以当初保存的那个*快照snapshot*所指向的对象将会变成不可达从而被*gc*回收，可以看到当修改频繁发生时，*gc*也会频繁的发生，也就是说*CopyOnWrite*是一个空间时间消耗比较大的算法。

### ConcurrentModificationException与fail-fast

迭代器的遍历方式才支持fail-fast,能针对遍历时修改集合的情况快速抛出异常（单线程/多线程都适用，并不是只有多线程才会fail-fast），但如果出现遍历时修改也不一定抛出，因为modcount并没有加volatile修饰，也没有加锁，那么迭代器读到的modcout还未变化，而实际上已经发生了修改。

有了fail-fast我们在使用线程不安全的集合，例如ArrayList, HashMap时可以多了一种额外的检错机制，能快速的发现类似遍历时修改这种线程不安全的问题。

3. TreeSet -- ConcurrentSkipListSet
4. HashMap -- ConcurrentHashMap
5. TreeMap -- ConcurrentSkipListMap


ConcurretHashMap:
JDK1.7采用分段锁保证高可用，需要改变hash寻址的方式，在段内还需要再散列。Segment之间同步扩容也是个问题。
锁的粒度从整个Map降低到了一个Segment,但Doug Lea还不满足，在JDK1.8中废弃了Segment，源码之所以保留是为了序列化的兼容性。他将拉链改成了超过8个节点就用红黑树。进一步降低了碰撞发生时的查找效率。

在锁方面，get无锁正常寻址。put时，当节点对应hash的位置没有节点时，通过CAS操作进行put。当对应hash位置已经有冲突时，用synchronized锁住这个链表的头或者是树的根，来实现线程安全。进一步降低了锁的粒度。

同时通过节点的特殊的hash值还可以判断现在是否在扩容，如果是，反正拿不到数据，调用该方法的线程不如帮助扩容。每个线程每次负责迁移其中的一部分。

ConcurrentLinkedQueue
通过合并两次插入，减少CAS次数的思想。不保证tail一定指向队列的尾端，而是在有必要时才CAS设置尾巴。
peek也是一样，当head里有元素时直接取出，并将head的元素cas设为null,而不cas更新head。只在有必要时更新头部。
尽量通过合并减少CAS的操作。类似于MySQL的change buffer，合并完一次更新，降低IO次数。

LinkedBlockingQueue
有界的需要两个condition(notFull,notEmpty) ，无界的需要一个notEmpty。
putLock takeLock,两个condition，notFull, notEmpty.
当队列插入一个元素后不为空那么notEmpty.signal()
删除一个元素后大小为容量-1，那么notFull.signal()

当take时，如果队列为空那么，notEmpty.await();
当put时，如果队列满，那么notFull.await();

同时由于两把锁分别控制头部、尾部。那么还需要一个原子类来记录队列的当前大小，避免数据错乱。

Fixed-> LinkedBlockingQueue有界
Single-> LinkedBlockingQueue有界  可以保证任务的按序执行
Scheduled-> DelayQueue<E extends Delay> 无界 隐含了PriorityQueue，时间短的优先执行。
Cached-> SynchronousQueue 只负责传递

Cached默认core pool size为0， maxPoolSize为(1 << 32)，因此当提交任务的速率超过线程池中poll/take速率时，CachedPool会不断的创建新的线程，内存消耗不可控制。