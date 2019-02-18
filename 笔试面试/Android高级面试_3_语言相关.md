# Android 高级面试-3：语言相关

*Kotlin, Java, RxJava, 多线程/并发, 集合*

## 1、Java 相关

### 1.1 缓存相关

- **LruCache 的原理**
- **DiskLruCache 的原理**

LruCache 用来实现基于内存的缓存，LRU 就是**最近最少使用**的意思，LruCache 基于 **LinkedHashMap** 实现。LinkedHashMap 是在 HashMap 的基础之上进行了封装，除了具有哈希功能，还将数据插入到双向链表中维护。每次读取的数据会被移动到链表的尾部，当达到了缓存的最大的容量的时候就将链表的首部移出。使用 LruCache 的时候需要注意的是单位的问题，因为该 API 并不清楚要存储的数据是如何计算大小的，所以它提供了方法供我们实现大小的计算方式。（[《Android 内存缓存框架 LruCache 的源码分析》](https://juejin.im/post/5bea581be51d451402494af2)）

DiskLruCache 与 LruCache 类似，也是用来实现缓存的，并且也是基于 LinkedHashMap 实现的。不同的是，它是基于磁盘缓存的，LruCache 是基于内存缓存的。所以，LinkedHashMap 能够存储的空间更大，但是读写的速率也更慢。使用 DiskLruCache 的时候需要到 Github 上面去下载。OkHttp 和 Glide 的磁盘缓存都是基于 DiskLruCache 开发的。DiskLruCahce 内部维护了一个日志文件，记录了读写的记录的信息。其他的基本都是基础的磁盘 IO 操作。

- Glide 缓存的实现原理

------

### 1.2 List 相关

- **ArrayList 与 LinkedList 区别**

1. ArrayList 是**基于动态数组，底层使用 System.arrayCopy() 实现数组扩容；查找值的复杂度为 O(1)，增删的时候可能扩容，复杂度也比 LinkedList 高；如果能够大概估出列表的长度，可以通过在 new 出实例的时候指定一个大小来指定数组的初始大小，以减少扩容的次数；适合应用到查找多于增删的情形，比如作为 Adapter 的数据的容器**。
2. LinkedList 是**基于双向链表；增删的复杂度为 O(1)，查找的复杂度为 O(n)；适合应用到增删比较多的情形**。
3. 两种列表都不是线程安全的，Vector 是线程安全的，但是它的线程安全的实现方式是通过对每个方法进行加锁，所以性能比较低。

如果想线程安全地使用这列表类（可以参考下面的问题）

- **如何实现线程间安全地操作 List？**

我们有几种方式可以线程间安全地操作 List. 具体使用哪种方式，可以根据具体的业务逻辑进行选择。通常有以下几种方式：
1. 第一是在操作 List 的时候使用 `sychronized` 进行控制。我们可以在我们自己的业务方法上面进行加锁来保证线程安全。
2. 第二种方式是使用 `Collections.synchronizedList()` 进行包装。这个方法内部使用了**私有锁**来实现线程安全，就是通过对一个全局变量进行加锁。调用我们的 List 的方法之前需要先获取该私有锁。私有锁可以降低锁粒度。
3. 第三种是使用并发包中的类，比如在读多写少的情况下，为了提升效率可以使用 `CopyOnWriteArrayList` 代替 ArrayList，使用 `ConcurrentLinkedQueue` 代替 LinkedList. 并发容器中的 `CopyOnWriteArrayList` 在读的时候不加锁，写的时候使用 Lock 加锁。`ConcurrentLinkedQueue` 则是基于 CAS 的思想，在增删数据之前会先进行比较。

------

### 1.3 Map 相关

- **SparseArray 的原理**

SparseArray 主要用来替换 Java 中的 HashMap，因为 HashMap 将整数类型的键默认装箱成 Integer (效率比较低). 而 SparseArray **通过内部维护两个数组来进行映射**，并且使用**二分查找**寻找指定的键，所以**它的键对应的数组无需是包装类型**。SparseArray 用于当 HashMap 的键是 Integer 的情况，它会在内部维护一个 int 类型的数组来存储键。同理，还有 LongSparseArray, BooleanSparseArray 等，都是用来通过减少装箱操作来节省内存空间的。但是，因为它内部使用二分查找寻找键，所以其效率不如 HashMap 高，所以当要存储的键值对的数量比较大的时候，考虑使用 HashMap. 

- **HashMap、ConcurrentHashMap 以及 HashTable**
- **hashmap 如何 put 数据（从 hashmap 源码角度讲解）？（掌握 put 元素的逻辑）**

HashMap (下称 HM) 是哈希表，ConcurrentHashMap (下称 CHM) 也是哈希表，它们之间的区别是 HM 不是线程安全的，CHM 线程安全，并且对锁进行了优化。对应 HM 的还有 HashTable (下称 HT)，它通过对内部的每个方法加锁来实现线程安全，效率较低。

HashMap 的实现原理：HashMap 使用拉链法来解决哈希冲突，即当两个元素的哈希值相等的时候，它们会被方进一个桶当中。当一个桶中的数据量比较多的时候，此时 HashMap 会采取两个措施，要么扩容，要么将桶中元素的数据结构从链表转换成红黑树。因此存在几个常量会决定 HashMap 的表现。在默认的情况下，当 HashMap 中的已经被占用的桶的数量达到了 3/4 的时候，会对 HashMap 进行扩容。当一个桶中的元素的数量达到了 8 个的时候，如果桶的数量达到了 64 个，那么会将该桶中的元素的数据结构从链表转换成红黑树。如果桶的数量还没有达到 64 个，那么此时会对 HashMap 进行扩容，而不是转换数据结构。

从数据结构上，HashMap 中的桶中的元素的数据结构从链表转换成红黑树的时候，仍然可以保留其链表关系。因为 HashMap 中的 TreeNode 继承了 LinkedHashMap 中的 Entry，因此它存在两种数据结构。

HashMap 在实现的时候对性能进行了很多的优化，比如使用截取后面几位而不是取余的方式计算元素在数组中的索引。使用哈希值的高 16 位与低 16 进行异或运算来提升哈希值的随机性。

因为每个桶的元素的数据结构有两种可能，因此，当对 HashMap 进行增删该查的时候都会根据结点的类型分成两种情况来进行处理。当数据结构是链表的时候处理起来都非常容易，使用一个循环对链表进行遍历即可。当数据结构是红黑树的时候处理起来比较复杂。红黑树的查找可以沿用二叉树的查找的逻辑。

下面是 HashMap 的插入的逻辑，所有的插入操作最终都会调用到内部的 `putVal()` 方法来最终完成。

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    private V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;

        if ((tab = table) == null || (n = tab.length) == 0) { // 原来的数组不存在
            n = (tab = resize()).length;
        }

        i = (n - 1) & hash; // 取哈希码的后 n-1 位，以得到桶的索引
        p = tab[i]; // 找到桶
        if (p == null) { 
            // 如果指定的桶不存在就创建一个新的，直接new 出一个 Node 来完成
            tab[i] = newNode(hash, key, value, null);
        } else { 
            // 指定的桶已经存在
            Node<K,V> e; K k;

            if (p.hash == hash // 哈希码相同
                && ((k = p.key) == key || (key != null && key.equals(k))) // 键的值相同
            ) {
                // 第一个结点与我们要插入的键值对的键相等
                e = p;
            } else if (p instanceof TreeNode) {
                // 桶的数据结构是红黑树，调用红黑树的方法继续插入
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            } else {
                // 桶的数据结构是链表，使用链表的处理方式继续插入
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        // 已经遍历到了链表的结尾，还没有找到，需要新建一个结点
                        p.next = newNode(hash, key, value, null);
                        // 插入新结点之后，如果某个链表的长度 >= 8，则要把链表转成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) {
                            treeifyBin(tab, hash);
                        }
                        break;
                    }
                    if (e.hash == hash // 哈希码相同 
                        && ((k = e.key) == key || (key != null && key.equals(k))) // 键的值相同
                    ) {
                        // 说明要插入的键值对的键是存在的，需要更新之前的结点的数据
                        break;
                    }
                    p = e;
                }
            }

            if (e != null) { 
                // 说明指定的键是存在的，需要更新结点的值
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null) {
                    e.value = value;
                }
                return oldValue;
            }
        }

        ++modCount;

        // 如果插入了新的结点之后，哈希表的容量大于 threshold 就进行扩容
        if (++size > threshold) {
            resize(); // 扩容
        }
        return null;
    }
```

上面是 HashMap 的插入的逻辑，可以看出，它也是根据头结点的类型，分成红黑树和链表两种方式来进行处理的。对于链表，上面已经给出了具体的插入逻辑。在链表的情形中，除了基础的插入，当链表的长度达到了 8 的时候还要将桶的数据结构从链表转型成为红黑树。对于红黑树类型的数据结构，它调用 TreeNode 的 `putTreeVal()` 方法来完成往红黑树中插入结点的逻辑。

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    // 查找根节点, 索引位置的头节点并不一定为红黑树的根结点
    TreeNode<K,V> root = (parent != null) ? root() : this;  
    for (TreeNode<K,V> p = root;;) {    // 将根节点赋值给 p, 开始遍历
        int dir, ph; K pk;
        
        if ((ph = p.hash) > h)  
        // 如果传入的 hash 值小于 p 节点的 hash 值，则将 dir 赋值为 -1, 代表向 p 的左边查找树
            dir = -1; 
        else if (ph < h)    
        // 如果传入的 hash 值大于 p 节点的 hash 值,则将 dir 赋值为 1, 代表向 p 的右边查找树
            dir = 1;

        // 如果传入的 hash 值和 key 值等于 p 节点的 hash 值和 key 值, 则 p 节点即为目标节点, 返回 p 节点
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))  
            return p;
        
        // 如果 k 所属的类没有实现 Comparable 接口 或者 k 和 p 节点的 key 相等
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) { 
            if (!searched) {    
                // 第一次符合条件, 该方法只有第一次才执行
                TreeNode<K,V> q, ch;
                searched = true;
                // 从 p 节点的左节点和右节点分别调用 find 方法进行查找, 如果查找到目标节点则返回
                if (((ch = p.left) != null && (q = ch.find(h, k, kc)) != null) 
                    || ((ch = p.right) != null && (q = ch.find(h, k, kc)) != null))  
                    return q;
            }
            // 使用定义的一套规则来比较 k 和 p 节点的 key 的大小, 用来决定向左还是向右查找
            dir = tieBreakOrder(k, pk); // dir<0 则代表 k<pk，则向 p 左边查找；反之亦然
        }
 
        TreeNode<K,V> xp = p;   // xp 赋值为 x 的父节点,中间变量,用于下面给x的父节点赋值
        // dir<=0 则向 p 左边查找,否则向 p 右边查找,如果为 null,则代表该位置即为 x 的目标位置
        if ((p = (dir <= 0) ? p.left : p.right) == null) {  
        	// 走进来代表已经找到 x 的位置，只需将 x 放到该位置即可
            Node<K,V> xpn = xp.next;    // xp 的 next 节点      
            // 创建新的节点, 其中 x 的 next 节点为 xpn, 即将 x 节点插入 xp 与 xpn 之间
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);   
            if (dir <= 0)   // 如果时 dir <= 0, 则代表 x 节点为 xp 的左节点
                xp.left = x;
            else        // 如果时 dir> 0, 则代表 x 节点为 xp 的右节点
                xp.right = x;
            xp.next = x;    // 将 xp 的n ext 节点设置为 x
            x.parent = x.prev = xp; // 将 x 的 parent 和 prev 节点设置为xp
            // 如果 xpn 不为空,则将 xpn 的 prev 节点设置为 x 节点,与上文的 x 节点的 next 节点对应
            if (xpn != null)    
                ((TreeNode<K,V>)xpn).prev = x;
            moveRootToFront(tab, balanceInsertion(root, x)); // 进行红黑树的插入平衡调整
            return null;
        }
    }
}
```

- 集合 Set 实现 Hash 怎么防止碰撞

HashSet 内部通过 HashMap 实现，HashMap 解决哈希冲突使用的是拉链法，碰撞的元素会放进链表中，链表长度超过 8，并且已经使用的桶的数量大于 64 的时候，会将桶的数据结构从链表转换成红黑树。HashMap 在求得每个结点在数组中的索引的时候，会使用对象的哈希码的高八位和低八位求异或，来增加哈希码的随机性。

- HashSet 与 HashMap 怎么判断集合元素重复

HashSet 内部使用 HashMap 实现，当我们通过 `put()` 方法将一个键值对添加到哈希表当中的时候，会根据哈希值和键是否相等两个条件进行判断，只有当两者完全相等的时候才认为元素发生了重复。

- HashMap 的实现？与 HashSet 的区别？

HashSet 不允许列表中存在重复的元素，HashSet 内部使用的是 HashMap 实现的。在我们向 HashSet 中添加一个元素的时候，会将该元素作为键，一个默认的对象作为值，构成一个键值对插入到内部的 HashMap 中。(HashMap 的实现见上文。)

- TreeMap 具体实现

> !!!!!!!!!AAAAnswer

------

### 1.4 注解相关

- **对 Java 注解的理解**

Java 注解在 Android 中比较常见的使用方式有 3 种：    
1. 第一种方式是基于**反射**的。因为反射本身的性能问题，所以它通常用来做一些简单的工作，比如为类、类的字段和方法等添加额外的信息，然后通过反射来获取这些信息。    
2. 第二种方式是基于 **AnnotationProcessor** 的，也就是在编译期间动态生成样板代码，然后通过反射触发生成的方法。比如 ButterKnife 就使用注解处理，在编译的时候 find 使用了注解的控件，并为其绑定值。然后，当调用 `bind()` 的时候直接反射调用生成的方法。Room 也是在编译期间为使用注解的方法生成数据库方法的。在开发这种第三方库的时候还可能使用到 **Javapoet** 来帮助我们生成 Java 文件。    
3. 最后一种比较常用的方式是使用注解来取代枚举。因为枚举相比于常量有额外的内存开销，所以开发的时候通常使用常量来取代枚举。但是如果只使用常量我们无法对传入的常量的范围进行限制，因此我们可以使用注解来限制取值的范围。以整型为例，我们会在定义注解的时候使用注解 `@IntDef({/*各种枚举值*/})` 来指定整型的取值范围。然后使用注解修饰我们要方法的参数即可。这样 IDE 会给出一个提示信息，提示我们只能使用指定范围的值。（[《Java 注解及其在 Android 中的应用》](https://juejin.im/post/5b824b8751882542f105447d)）

关联：ButterKnife, ARouter

------

### 1.5 Object 相关

- **Object 类的 equal() 和 hashcode() 方法重写？**

这两个方法**都具有决定一个对象身份功能，所以两者的行为必须一致，覆写这两个方法需要遵循一定的原则**。可以从业务的角度考虑使用对象的唯一特征，比如 ID 等，或者使用它的全部字段来进行计算得到一个整数的哈希值。一般，我不会直接覆写该方法，除非业务特征非常明显。因为一旦修改之后，它的作用范围将是全局的。我们还可以通过 IDEA 的 generate 直接生成该方法。

- Object 都有哪些方法？

1. `wait() & notify()`, 用来对线程进行控制，以让当前线程等待，直到其他线程调用了 `notify()/notifyAll()` 方法。`wait()` 发生等待的前提是当前线程获取了对象的锁（监视器）。调用该方法之后当前线程会释放获取到的锁，然后让出 CPU，进入等待状态。`notify/notifyAll()` 的执行只是唤醒沉睡的线程，而不会立即释放锁，锁的释放要看代码块的具体执行情况。
2. `clone()` 与对象克隆相关的方法（深拷贝&浅拷贝？）
3. `finilize()`
4. `toString()`
5. `equal() & hashCode()`，见上

------

### 1.6 字符串相关

- **StringBuffer 与 StringBuilder 的区别？**

前者是线程安全的，每个方法上面都使用 synchronized 关键字进行了加锁，后者是非线程安全的。一般情况下使用 StringBuilder 即可，因为非多线程环境进行加锁是一种没有必要的开销。

- **对 Java 中 String 的了解**

1. String 不是基本数据类型。
2. String 是不可变的，JVM 使用字符串池来存储所有的字符串对象。
3. 使用 new 创建字符串，这种方式创建的字符串对象不存储于字符串池。我们可以调用`intern()` 方法将该字符串对象存储在字符串池，如果字符串池已经有了同样值的字符串，则返回引用。使用双引号直接创建字符串的时候，JVM 先去字符串池找有没有值相等字符串，如果有，则返回找到的字符串引用；否则创建一个新的字符串对象并存储在字符串池。

- **String 为什么要设计成不可变的？**

1. **线程安全**：由于 String 是不可变类，所以在多线程中使用是安全的，我们不需要做任何其他同步操作。
2. String 是不可变的，它的值也不能被改变，所以用来存储数据密码很**安全**。
3. **复用/节省堆空间**：实际在 Java 的开发当中 String 是使用最为频繁的类之一，通过 dump 的堆可以看出，它经常占用很大的堆内存。因为 java 字符串是不可变的，可以在 java 运行时节省大量 java **堆**空间。不同的字符串变量可以引用池中的相同的字符串。如果字符串是可变得话，任何一个变量的值改变，就会反射到其他变量，那字符串池也就没有任何意义了。

- 常见编码方式有哪些？
- Utf-8 编码中的中文占几个字节？int 型占几个字节？

------

### 1.7 线程控制

- **开启线程的三种方式，run() 和 start() 方法区别**
- **多线程：怎么用、有什么问题要注意；Android 线程有没有上限，然后提到线程池的上限**
- **Java 线程池、线程池的几个核心参数的意义**
- **线程如何关闭，以及如何防止线程的内存泄漏**

*如何开启线程，线程池参数；注意的问题：线程数量，内存泄漏*

```java
    //  方式 1：Thread 覆写 run() 方法；
    private class MyThread extends Thread {
        @Override
        public void run() {
            // 业务逻辑
        }
    }

    // 方式 2：Thread + Runnable
    new Thread(new Runnable() {
        public void run() {
            // 业务逻辑
        }
    }).start();

    // 方式 3：ExectorService + Callable
    ExecutorService executor = Executors.newFixedThreadPool(5);
    List<Future<Integer>> results = new ArrayList<Future<Integer>>();
    for (int i=0; i<5; i++) {
        results.add(executor.submit(new CallableTask(i, i)));
    }
```

**线程数量的问题：**

Android 中并没有明确规定可以创建的线程的数量，但是每个进程的资源是有限的，线程本身会占有一定的资源，所以受内存大小的限制，会有数量的上限。通常，我们在使用线程或者线程池的时候，不会创建太多的线程。线程池的大小经验值应该这样设置：（其中 N 为 CPU 的核数）

1. 如果是 CPU 密集型应用，则线程池大小设置为 `N + 1`；(大部分时间在计算)
2. 如果是 IO 密集型应用，则线程池大小设置为 `2N + 1`；(大部分时间在读写，Android)

下面是 Android 中的 AysncTask 中创建线程池的代码，

```java
    // CPU 的数量
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    // 核心线程的数量：只有提交任务的时候才会创建线程，当当前线程数量小于核心线程数量，新添加任务的时候，会创建新线程来执行任务
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    // 线程池允许创建的最大线程数量：当任务队列满了，并且当前线程数量小于最大线程数量，则会创建新线程来执行任务
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    // 非核心线程的闲置的超市时间：超过这个时间，线程将被回收，如果任务多且执行时间短，应设置一个较大的值
    private static final int KEEP_ALIVE_SECONDS = 30;

    // 线程工厂：自定义创建线程的策略，比如定义一个名字
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    // 任务队列：如果当前线程的数量大于核心线程数量，就将任务添加到这个队列中
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                /*corePoolSize=*/ CORE_POOL_SIZE,
                /*maximumPoolSize=*/ MAXIMUM_POOL_SIZE, 
                /*keepAliveTime=*/ KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                /*workQueue=*/ sPoolWorkQueue, 
                /*threadFactory=*/ sThreadFactory
                /*handler*/ defaultHandler); // 饱和策略：AysncTask 没有这个参数
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

饱和策略：任务队列和线程池都满了的时候执行的逻辑，Java 提供了 4 种实现；    
其他：
1. 当调用了线程池的 `prestartAllcoreThread()` 方法的时候，线程池会提前启动并创建所有核心线程来等待任务；
2. 当调用了线程池的 `allowCoreThreadTimeOut()` 方法的时候，超时时间到了之后，闲置的核心线程也会被移除。

**`run()` 和 `start()` 方法区别**：`start()` 会调用 native 的 `start()` 方法，然后 `run()` 方法会被回调，此时 `run()` 异步执行；如果直接调用 `run()`，它会使用默认的实现（除非覆写了），并且会在当前线程中执行，此时 Thread 如同一个普通的类。

```java
    private Runnable target;
    public void run() {
        if (target != null)  target.run();
    }
```

**线程关闭**，有两种方式可以选择，一种是使用中断标志位进行判断。当需要停止线程的时候，调用线程的 `interupt()` 方法即可。这种情况下需要注意的地方是，当线程处于阻塞状态的时候调用了中断方法，此时会抛出一个异常，并将中断标志位复位。此时，我们是无法退出线程的。所以，我们需要同时考虑一般情况和线程处于阻塞时中断两种情况。

另一个方案是使用一个 volatile 类型的布尔变量，使用该变量来判断是否应该结束线程。

```java
    // 方式 1：使用中断标志位
    @Override
    public void run() {
        try {
            while (!isInterrupted()) {
                // do something
            }
        } catch (InterruptedException ie) {  
            // 线程因为阻塞时被中断而结束了循环
        }
    }

    private static class MyRunnable2 implements Runnable {
        // 注意使用 volatile 修饰
        private volatile boolean canceled = false;

        @Override
        public void run() {
            while (!canceled) {
                // do something
            }
        }

        public void cancel() {
            canceled = true;
        }
    }
```

**防止线程内存泄漏**：

1. 在 Activity 等中使用线程的时候，将线程定义成静态的内部类，非静态内部类会持有外部类的匿名引用；
2. 当需要在线程中调用 Activity 的方法的时候，使用 WeakReference 引用 Activity；
3. 或者当 Activity 需要结束的时候，在 `onDestroy()` 方法中终止线程。

- wait()、notify() 与 sleep()

wait()/notify():

1. `wait()、notify() 和 notifyAll()` 方法是 Object 的本地 final 方法，无法被重写。

2. `wait()` 使当前线程阻塞，直到**接到通知或被中断**为止。前提是必须先获得锁，一般配合 synchronized 关键字使用，在 synchronized 同步代码块里使用 `wait()、notify() 和 notifyAll()` 方法。如果调用 `wait()` 或者 `notify()` 方法时，线程并未获取到锁的话，则会抛出 IllegalMonitorStateException 异常。再次获取到锁，当前线程才能从 `wait()` 方法处成功返回。

3. 由于 `wait()、notify() 和 notifyAll()` 在 synchronized 代码块执行，说明当前线程一定是获取了锁的。当线程执行 `wait()` 方法时候，**会释放当前的锁，然后让出 CPU，进入等待状态**。只有当 `notify()/notifyAll()` 被执行时候，才会唤醒一个或多个正处于等待状态的线程，然后继续往下执行，直到执行完 synchronized 代码块或是中途遇到 `wait()`，**再次释放锁**。    
也就是说，`notify()/notifyAll()` 的执行只是唤醒沉睡的线程，而不会立即释放锁，锁的释放要看代码块的具体执行情况。所以在编程中，尽量在使用了 `notify()/notifyAll()` 后立即退出临界区，以唤醒其他线程。

4. `wait()` 需要被 `try catch` 包围，中断也可以使 `wait` 等待的线程唤醒。

5. `notify()` 和 `wait()` 的顺序不能错，如果 A 线程先执行 `notify()` 方法，B 线程再执行 `wait()` 方法，那么 B 线程是无法被唤醒的。

6. `notify()` 和 `notifyAll()` 的区别：       
`notify()` 方法只唤醒一个等待（对象的）线程并使该线程开始执行。所以如果有多个线程等待一个对象，这个方法只会唤醒其中一个线程，选择哪个线程取决于操作系统对多线程管理的实现。    
`notifyAll()` 会唤醒所有等待 (对象的) 线程，尽管哪一个线程将会第一个处理取决于操作系统的实现。如果当前情况下有多个线程需要被唤醒，推荐使用 `notifyAll()` 方法。比如在生产者-消费者里面的使用，每次都需要唤醒所有的消费者或是生产者，以判断程序是否可以继续往下执行。

对于 `sleep()` 和 `wait()` 方法之间的区别，总结如下，

1. 所属类不同：`sleep()` 方法是 Thread 的静态方法，而 `wait()` 是 Object 实例方法。
2. 作用域不同：`wait()` 方法必须要在**同步方法或者同步块**中调用，也就是必须已经获得对象锁。而 `sleep()` 方法没有这个限制可以在任何地方种使用。
3. 锁占用不同：`wait()` 方法会释放占有的对象锁，使得该线程进入等待池中，等待下一次获取资源。而 `sleep()` 方法只是会让出 CPU 并不会释放掉对象锁；
4. 锁释放不同：`sleep()` 方法在休眠时间达到后如果再次获得 CPU 时间片就会继续执行，而 `wait()` 方法必须等待 `Object.notift()/Object.notifyAll()` 通知后，才会离开等待池，并且再次获得 CPU 时间片才会继续执行。

- 线程的状态

![](https://github.com/Shouheng88/Awesome-Java/blob/master/Java%E8%AF%AD%E8%A8%80%E5%92%8CJDK%E6%BA%90%E7%A0%81/res/Thread_states.jpg?raw=true)

1. 新建 (NEW)：新创建了一个线程对象。

2. 可运行 (RUNNABLE)：线程对象创建后，其他线程(比如 main 线程）调用了该对象的 `start()` 方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取 CPU 的使用权 。

3. 运行 (RUNNING)：RUNNABLE 状态的线程获得了 CPU 时间片（timeslice） ，执行程序代码。

4. 阻塞 (BLOCKED)：阻塞状态是指线程因为某种原因放弃了 CPU 使用权，也即让出了 CPU timeslice，暂时停止运行。直到线程进入 RUNNABLE 状态，才有机会再次获得 CPU timeslice 转到 RUNNING 状态。阻塞的情况分三种： 

    1. 等待阻塞：RUNNING 的线程执行 `o.wait()` 方法，JVM 会把该线程放入等待队列 (waitting queue) 中。

    2. 同步阻塞：RUNNING 的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则 JVM 会把该线程放入锁池 (lock pool) 中。

	3. 其他阻塞：RUNNING 的线程执行 `Thread.sleep(long)` 或 `t.join()` 方法，或者发出了 I/O 请求时，JVM 会把该线程置为阻塞状态。当 `sleep()` 状态超时、`join()` 等待线程终止或者超时、或者 I/O 处理完毕时，线程重新转入 RUNNABLE 状态。

5. 死亡 (DEAD)：线程 `run()`、`main()` 方法执行结束，或者因异常退出了 `run()` 方法，则该线程结束生命周期。死亡的线程不可再次复生。

- **死锁，线程死锁的 4 个条件？**
- **死锁的概念，怎么避免死锁？**

当两个线程彼此占有对方需要的资源，同时彼此又无法释放自己占有的资源的时候就发生了死锁。发生死锁需要满足下面四个条件，

1. **互斥**：某种资源一次只允许一个进程访问，即该资源一旦分配给某个进程，其他进程就不能再访问，直到该进程访问结束。（一个筷子只能被一个人拿）
2. **占有且等待**：一个进程本身占有资源（一种或多种），同时还有资源未得到满足，正在等待其他进程释放该资源。（每个人拿了一个筷子还要等其他人放弃筷子）
3. **不可抢占**：别人已经占有了某项资源，你不能因为自己也需要该资源，就去把别人的资源抢过来。（别人手里的筷子你不能去抢）
4. **循环等待**：存在一个进程链，使得每个进程都占有下一个进程所需的至少一种资源。（每个人都在等相邻的下一个人放弃自己的筷子）

产生死锁需要四个条件，那么，只要这四个条件中至少有一个条件得不到满足，就不可能发生死锁了。由于互斥条件是非共享资源所必须的，不仅不能改变，还应加以保证，所以，主要是破坏产生死锁的其他三个条件。

破坏占有且等待的问题：允许进程只获得运行初期需要的资源，便开始运行，在运行过程中逐步释放掉分配到的已经使用完毕的资源，然后再去请求新的资源。

破坏不可抢占条件：当一个已经持有了一些资源的进程在提出新的资源请求没有得到满足时，它必须释放已经保持的所有资源，待以后需要使用的时候再重新申请。释放已经保持的资源很有可能会导致进程之前的工作实效等，反复的申请和释放资源会导致进程的执行被无限的推迟，这不仅会延长进程的周转周期，还会影响系统的吞吐量。

破坏循环等待条件：可以通过定义资源类型的线性顺序来预防，可将每个资源编号，当一个进程占有编号为i的资源时，那么它下一次申请资源只能申请编号大于 i 的资源。

[《死锁的四个必要条件和解决办法》](https://blog.csdn.net/guaiguaihenguai/article/details/80303835#commentBox)

- **synchronized 与 Lock 的区别**
- Lock 的实现原理
- synchronized 的实现原理
- ReentrantLock 的内部实现
- CAS 介绍
- 如何实现线程同步？synchronized, lock, 无锁同步, voliate, 并发集合，同步集合

Java 虚拟机的同步是基于进入和退出管程 (Monitor) 对象实现，无论是显式同步 (有明确的 monitorenter 和 monitorexit 指令，即同步代码块) 还是隐式同步都是如此。同步方法并不是由 monitorenter 和 monitorexit 指令来实现同步的，而是由方法调用指令读取运行时常量池中方法的 `ACC_SYNCHRONIZED` 标志来隐式实现的。对于代码块则会在其同步的区域添加 monitorenter 和 monitorexit 指令，进入的时候需要先获取类的实例监视器。每进入一次锁的计数加 1，以此来统计占有的锁的次数。离开加锁区域的时候锁计数减 1，当计数减为 0 的时候，锁被释放。

1. **等待可中断**：当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待；（两种方式获取锁的时候都会使计数+1，但是方式不同，所以重入锁可以终端）
2. **公平锁**：当多个线程等待同一个锁时，公平锁会按照申请锁的时间顺序来依次获得锁；而非公平锁，当锁被释放时任何在等待的线程都可以获得锁（不论时间尝试获取的时间先后）。sychronized 只支持非公平锁，Lock 可以通过构造方法指定使用公平锁还是非公平锁。
3. **锁可以绑定多个条件**：ReentrantLock 可以绑定多个 Condition 对象，而 sychronized 要与多个条件关联就不得不加一个锁，ReentrantLock 只要多次调用newCondition 即可。

- volatile 原理和用法

voliate 关键字的两个作用

1. **保证变量的可见性**：当一个被 voliate 关键字修饰的变量被一个线程修改的时候，其他线程可以立刻得到修改之后的结果。当写一个 volatile 变量时，JMM 会把该线程对应的工作内存中的共享变量值刷新到主内存中，当读取一个 volatile 变量时，JMM 会把该线程对应的工作内存置为无效，那么该线程将只能从主内存中重新读取共享变量。
2. **屏蔽指令重排序**：指令重排序是编译器和处理器为了高效对程序进行优化的手段，它只能保证程序执行的结果时正确的，但是无法保证程序的操作顺序与代码顺序一致。这在单线程中不会构成问题，但是在多线程中就会出现问题。非常经典的例子是在单例方法中同时对字段加入 voliate，就是为了防止指令重排序。

volatile 是通过`内存屏障(Memory Barrier）` 来实现其在 JMM 中的语义的。内存屏障，又称内存栅栏，是一个 CPU 指令，它的作用有两个，一是保证特定操作的执行顺序，二是保证某些变量的内存可见性。如果在指令间插入一条内存屏障则会告诉编译器和 CPU，不管什么指令都不能和这条 Memory Barrier 指令重排序。Memory Barrier 的另外一个作用是强制刷出各种 CPU 的缓存数据，因此任何 CPU 上的线程都能读取到这些数据的最新版本。

- **手写生产者/消费者模式**

参考 [《并发编程专题-5：生产者和消费者模式》](https://blog.csdn.net/github_35186068/article/details/87537570) 中的三种写法。

------

### 1.8 并发包

- **ThreadLocal 的实现原理？**

ThreadLocal 通过将每个线程自己的局部变量存在自己的内部来实现线程安全。使用它的时候会定义它的静态变量，每个线程看似是从 TL 中获取数据，而实际上 TL 只起到了键值对的键的作用，实际的数据会以哈希表的形式存储在 Thread 实例的 Map 类型局部变量中。当调用 TL 的 `get()` 方法的时候会使用 `Thread.currentThread()` 获取当前 Thread 实例，然后从该实例的 Map 局部变量中，使用 TL 作为键来获取存储的值。Thread 内部的 Map 使用线性数组解决哈希冲突。([《ThreadLocal的使用及其源码实现》](https://juejin.im/post/5b44cd7c6fb9a04f980cb065))

- **并发类：并发集合了解哪些？**

1. ConcurrentHashMap：线程安全的 HashMap，对桶进行加锁，降低锁粒度提升性能。
2. ConcurrentSkipListMap：跳表，自行了解，给跪了……
3. ConCurrentSkipListSet：借助 ConcurrentSkipListMap 实现
4. CopyOnWriteArrayList：读多写少的 ArrayList，写的时候加锁
5. CopyOnWriteArraySet：借助 CopyOnWriteArrayList 实现的……
6. ConcurrentLinkedQueue：无界且线程安全的 Queue，其 `poll()` 和 `add()` 等方法借助 CAS 思想实现。锁比较轻量。

------

### 1.9 输入输出

- NIO

- 多线程断点续传原理

断点续传和断点下载都是用的用的都是 RandomAccessFile，它可以从指定的位置开始读取数据。断点续传是由服务器给客户端一个已经上传的位置标记position，然后客户端再将文件指针移动到相应的 position，通过输入流将文件剩余部分读出来传输给服务器。

如果要使用多线程来实现断点续传，那么可以给每个线程分配固定的字节的文件，分别去读，然后分别上传到服务器。

## 2、Kotlin 相关

## 3、设计模式

- 谈谈你对 Android 设计模式的理解
- 项目中常用的设计模式有哪些？
- 手写生产者-消费者模式？
- 手写观察者模式？
- 手写单例模式，懒汉和饱汉
- 适配器模式、装饰者模式、外观模式的异同？
