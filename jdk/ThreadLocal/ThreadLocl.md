##本文结构
- `ThreadLocal`简介 (简要说明`ThreadLocal`的作用)
- `ThreadLocal实`现原理（说明`ThreadLocal`的常用方法和原理）
- `ThreadLocalMap`的实现 (说明核心数据结构`ThreadLocalMap`的实现)
 

## ThreadLocal简介
------------------------------
先贴一段官方的文档解释
```
/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 */
```
大意是ThreadLocal类提供了一个线程的本地变量，每一个线程持有一个这个变量的副本并且每个线程读取`get()`到的值是不一样的，可以通过`set()`方法设置这个值；
**在某些情况下使用ThreadLocal可以避免共享资源的竞争，同时与不影响线程的隔离性**。

通过threadLocal.set方法将对象实例保存在每个线程自己所拥有的threadLocalMap中，
这样每个线程使用自己保存的ThreadLocalMap对象，不会影响线程之间的隔离。

看到这里的第一眼我一直以为ThreadLocal是一个map，每一个线程都是一个key，对应一个value，但是是不正确的。正确的是每个线程持有一个ThreadLocalMap的副本，这个map的键是ThreadLocal对象，各个线程中同一key对应的值可以不一样。

## ThreadLocal实现原理
------------------------------
###`ThreadLocal`中的字段与构造方法
>
>详细说明参考注释
>```
>public class ThreadLocal<T> {
>   
>    //当前ThreadLocal对象的HashCode值，
>    //通过这个值可以定位Entry对象在ThreadLocalMap中的位置
>    //由nextHashCode计算得出
>    private final int threadLocalHashCode = nextHashCode();
>
>    //一个自动更新的AtomicInteger值，官方解释是会自动更新，怎么更新的不知道，
>    //看完AtomicInteger源码回来填坑
>    private static AtomicInteger nextHashCode =
>        new AtomicInteger();
> 
>    //ThreadLocal的魔数
>    //0x61c88647是斐波那契散列乘数,它的优点是通过它散列(hash)出来的结果分布会比较均匀，可以很大程度上避免hash冲突，
>    private static final int HASH_INCREMENT = 0x61c88647;
>
>    private static int nextHashCode() {
>        //原子操作：将给定的两个值相加
>        return nextHashCode.getAndAdd(HASH_INCREMENT);
>    }
>
>    /**
>     * 返回当前线程变量的初始值
>     * 这个方法仅在没有调用set方法的时候第一次调用get方法调用
>     */
>    protected T initialValue() {
>        return null;
>    }
>    //构造方法
>    public ThreadLocal() {
>    }
>   
>    //创建ThreadLocalMap，
>    //当前的ThreadLocal对象和value加入map当中
>    //赋值给当前线程的threadLocals字段
>    void createMap(Thread t, T firstValue) {
>        t.threadLocals = new ThreadLocalMap(this, firstValue);
>    }
>.......
>}
>```



### 常用方法`get()、set()、remove`


#### `get()`方法
>
>`get()`方法的源码，具体的代码解释请看注释
>```$xslt
>//返回当前线程中保存的与当前ThreadLocal相关的线程变量的值（有点绕，可以看代码注释）
>public T get() {
>    Thread t = Thread.currentThread();
>    
>    //返回当前线程的threadLocals字段的值，类型是ThreadLocalMap
>    //暂时可以将ThreadLocalMap当作HashMap，下文解释
>    ThreadLocalMap map = getMap(t);
>    if (map != null) {
>
>        //Entry也可以按照HashMap的entry理解
>        //Entry保存了两个值，一个值是key，一个值是value
>        //返回当前ThreadLocalMap中当前ThreadLcoal对应的Entry
>        ThreadLocalMap.Entry e = map.getEntry(this);
>        if (e != null) {
>            @SuppressWarnings("unchecked")
>            T result = (T)e.value;
>
>            //返回Entry中对应的值
>            return result;
>        }
>    }
>    //如果当前线程的ThreadLocalMap不存在，则构造一个
>    return setInitialValue();
>}
>```
>> `getMap(Thread t)` 方法
>>```$xslt
>>//返回当前线程的threadLocals字段的值，类型为ThreadLocalMap
>>ThreadLocalMap getMap(Thread t) {
>>        return t.threadLocals;
>>    }
>>```
>
>>`getEntry(ThreadLocal<?> key)`方法
>>```
>>//table是一个数组，具体的可以看下文的ThreadLocalMap解释
>>//返回当前ThreadLocalMap中key对应的Entry
>>
>>private Entry getEntry(ThreadLocal<?> key) {
>>    //根据key值计算所属Entry所在的索引位置（同HashMap）
>>    
>>    int i = key.threadLocalHashCode & (table.length - 1);
>>    Entry e = table[i];
>>    //由于存在散列冲突，判断当前节点是否是对应的key的节点
>>    if (e != null && e.get() == key)
>>        //返回这个节点
>>        return e;
>>    else
>>        return getEntryAfterMiss(key, i, e);
>>}
>>```
>
>>  `setInitialValue()`方法
>>```
>>  private T setInitialValue() {
>>      
>>      //在构造方法部分有写，返回一个初始值，
>>      //默认情况（没有被子类重写）下是一个null值
>>      T value = initialValue();
>>      Thread t = Thread.currentThread();
>>      ThreadLocalMap map = getMap(t);
>>      //再次判断当前线程中threadLocals字段是否为空值
>>      if (map != null)
>>          //set可以看作HashMap中的put
>>          map.set(this, value);
>>      else
>>          //当map为null时
>>          //构造一个ThreadLocalMap，
>>          //并以自身为键，initialValue()的结果为值插入ThreadLocalMap
>>          //并赋值给当前线程的threadLocals字段
>>          createMap(t, value);
>>      return value;
>>  }
>>```
>
>>`createMap`方法
>>``` 
>>void createMap(Thread t, T firstValue) {
>>  //调用THreadLocalMap的构造方法
>>  //构造一个ThreadLocalMap，
>>  //使用给定的值构造一个存储的实例（Entry的对象）存储到map中
>>  //并保存到thread的threadLocals字段中
>>  t.threadLocals = new ThreadLocalMap(this, firstValue);
>>}
>>```

>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从`get()`方法中大概可以看出`ThreadLocalMap`是一个以`ThreadLocal`对象为键一个Map，并且这个`ThreadLocalMap`对象由`Thread`类维护，并保存在`threadLocals`字段中，不同的`ThreadLocal`对象可以以自身为键访问这个`Map`中对应位置的值。
>
>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当第一次调用`get()(之前没有调用过set()或者调用了remove())`时，会调`initialValue()添加当前ThreadLocal对象对应的值为null并返回。

#### `set()`方法
>
>`set()`方法的源码，具体的代码的解释请看注释
>```$xslt
>/**
> * 设定当前线程的线程变量的副本为指定值
> * 子类一般不用重写这个方法
> * override this method, relying solely on the {@link #initialValue}
> * method to set the values of thread-locals.
> *
> * @param value the value to be stored in the current thread's copy of
> *        this thread-local.
> */
>public void set(T value) {
>    Thread t = Thread.currentThread();
>    //返回当前线程的threadLocals字段的值，类型是ThreadLocalMap，同get()方法
>    ThreadLocalMap map = getMap(t);
>    //同get()一样 判断当前线程的threadLocals字段的是否为null
>    if (map != null)
>        //不为null，设置当前ThreadLocal（key）对应的值（value）为指定的value
>        map.set(this, value);
>    else
>        //null，创建ThreadLocalMap对象，将[t, value]加入map，并赋值给当前线程的localThreads字段
>        createMap(t, value);
>}
>```
#### `remove()`方法
>
> `remove()`方法的源码，具体代码的解释请看注释
>```
> public void remove() {
>     ThreadLocalMap m = getMap(Thread.currentThread());
>     if (m != null)
>           //从当前线程的threadLocals字段中移除当前ThreadLocal对象为键的键值对
>           //remove方法在 ThreadLocalMap中实现
>           m.remove(this);
>      }
>```

从上面的实现代码看，`get()、set()、remove`这三个方法都是很简单的从一个ThreadLocalMap中获取设置或者移除值，那么有一个核心就是`ThreadLocalMap`，那么下面就分析下`ThreadLocalMap`类

## ThreadLocalMap的实现
----------------------------
个人觉得`replaceStaleEntry()`、`expungeStaleEntry()`、`cleanSomeSlots()`这三个方法是ThreadLocal中非常重要难以理解的方法；
```
/**
 * ThreadLocalMap 是一个定制的哈希散列映射，仅仅用用来维护线程本地变量
 * 对其的所有操作都在ThreadLocal类里面。
 * 使用软引用作为这个哈希表的key值（软引用引用的对象在强引用解除引用后的下一次GC会被释放）
 * 由于不使用引用队列，表里的数据只有在表空间不足时才会被释放
 * （因为使用的时key-value，在key被释放·null·后这个表对应的位置不会变为null，需要手动释放）
 *  这个map和HashMap不同的地方是，
 *  在发生哈希冲突的时候HashMap会使用链表（jdk8之后也可能是红黑树）存储（拉链法）
 *  而这里使用的是向后索引为null的表项来存储（开放地址法）
 */
static class ThreadLocalMap {

    /**
     * 这个hash Map的条目继承了 WeakReference, 使用他的ref字段作为key（一个ThreadLocal对象）
     * ThreadLocal object).  注意当键值为null时代表整个键已经不再被引用（ThreadLocal
     * 对象已经被垃圾回收）因此可以删除对应的条目
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            //key值是当前TreadLocal对象的弱引用
            super(k);
            value = v;
        }
    }

    //类的属性字段，基本和HashMap作用一致
    
    //初始容量，必须是2的幂
    private static final int INITIAL_CAPACITY = 16;

    //哈希表 
    //长度必须是2的幂，必要时会调整大小
    private Entry[] table;

    //表中存储数据的个数
    private int size = 0;

    //当表中数据的个数达到这个值时需要扩容
    private int threshold; // Default to 0

    //设置threshold ，负载系数时2/3，也就是说当前表中数据的条目
    //达到表总容量的2/3就需要扩容
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }

    /**
     * Increment i modulo len.
     */
    //这个注释不明白，但是在代码实现中遍历表的的时候用来判断是否到了表的结尾
    //如果到了表节位就从表首接着这遍历
    private static int nextIndex(int i, int len) {
        return ((i + 1 < len) ? i + 1 : 0);
    }

    /**
     * Decrement i modulo len.
     */
    //这个注释不明白，但是在代码实现中遍历表的的时候用来判断是否到了表的头部
    //如果到了表节位就从表尾接着这遍历
    private static int prevIndex(int i, int len) {
        return ((i - 1 >= 0) ? i - 1 : len - 1);
    }

    //构造一个新map包含 (firstKey, firstValue).
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        //构造表
        table = new Entry[INITIAL_CAPACITY];
        //确定需要加入的数据的位置
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        构造一个新的Entry对象并放入到表的对应位置
        table[i] = new Entry(firstKey, firstValue);
        //设置当前表中的数据个数
        size = 1;
        // 设置需要扩容的临界点
        setThreshold(INITIAL_CAPACITY);
    }

    //这个方法在创建一个新线程调用到Thread.init()方法是会被调用
    //目的是将父线程的inheritableThreadLocals传递给子线程
    //创建的map会被存储在Thread.inheritableThreadLocals中
    //根据parentMap构造一个新的ThreadLocalMap，
    //这个map包含了所有parentMap的值
    //只有在createInheritedMap调用
    private ThreadLocalMap(ThreadLocalMap parentMap) {
        Entry[] parentTable = parentMap.table;
        int len = parentTable.length;
        setThreshold(len);
        //根据parentMap的长度（容量）构造table
        table = new Entry[len];

        //依次复制parentMap中的数据
        for (int j = 0; j < len; j++) {
            Entry e = parentTable[j];
            if (e != null) {
                @SuppressWarnings("unchecked")
                ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                if (key != null) {

                    //childValue在InheritableThreadLocal中实现
                    //也只有InheritableThreadLocal对象会调用这个方法
                    Object value = key.childValue(e.value);
                    Entry c = new Entry(key, value);
                    int h = key.threadLocalHashCode & (len - 1);

                    //解决哈希冲突的办法是向后索引
                    while (table[h] != null)
                        h = nextIndex(h, len);
                    table[h] = c;
                    size++;
                }
            }
        }
    }
```
`getEntry()`获取指定`key`对应的`Entry`对象
```
    /**
     * 通过key获取key-value对.  这个方法本身只处理快速路径（直接命中）
     * 如果没有命中继续前进（getEntryAfterMiss）
     * 这是为了使命中性能最大化设计
     */
    private Entry getEntry(ThreadLocal<?> key) {
        //计算key应该出现在表中的位置
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        //判断获取到的entry的key是否是给出的key
        if (e != null && e.get() == key)
            //是就返回entry
            return e;
        else
            //否则向后查找，
            return getEntryAfterMiss(key, i, e);
    }

    /**
     * getEntry的派生，当直接哈希的槽里找不到键的时候使用
     *
     * @param  key the thread local object
     * @param  i key在table中哈希的结果
     * @param  e the entry at table[i]
     * @return the entry associated with key, or null if no such
     */
    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
        
        //根绝ThreadLocalMapc插入的方法，插入时通过哈希计算出来的槽位不为null
        //则向后索引，找到一个空位放置需要插入的值
        //所以从哈希计算的槽位到插入值的位置中间一定是不为null的
        //因为e!=null可以作为循环终止条件
        while (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == key)
                //如果命中则返回
                return e;
            if (k == null)
                //e!=null && k==null,证明对应的ThreadLcoal对象已经被释放
                //那么这个位置的entry就可以被释放
                //释放位置i上的空间
                //释放空间也是ThreadLocalMap与HashMap不相同的地方
                expungeStaleEntry(i);
            else
                //获取下一个查找的表的索引下标
                //当i>=len时会从0号位重新开始查找
                i = nextIndex(i, len);
            e = tab[i];
        }
        //没找到返回null
        return null;
    }
```
`set()`修改或者创建指定的`key`对应的`Entry`对象
```

    //添加一个key-value对
    private void set(ThreadLocal<?> key, Object value) {

        // 不像get一样使用快速路径，
        // set创建新条目和修改现有条目一样常见
        // 这种情况下快速路径通常会失败

        Entry[] tab = table;
        int len = tab.length;
        //计算应该插入的槽的位置
        int i = key.threadLocalHashCode & (len-1);

        //哈希计算的槽位到插入值的位置中间一定是不为null的
        //该位置是否位null可以作为循环终止条件
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            
            ThreadLocal<?> k = e.get();

            //修改现有的键值对的值
            if (k == key) {
                e.value = value;
                return;
            }
            
            //e!=null && k==null,证明对应的ThreadLcoal对象已经被释放
            //那么这个位置的entry就可以被释放
            //释放位置i上的空间
            //释放空间也是ThreadLocalMap与HashMap不相同的地方
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        
        //在可以插入值的地方插入
        tab[i] = new Entry(key, value);
        int sz = ++size;
        //清除部分k是null的槽然后判断是否需要扩容
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            //扩容
            rehash();
    }

```
`remove()`移除指定`key`对应的`Entry`对象
```
    /**
     * Remove the entry for key.
     */
    private void remove(ThreadLocal<?> key) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);
        //同set()
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            if (e.get() == key) {
                //弱引用引用置空，标记这个槽已经是旧槽
                e.clear();
                //清理旧槽
                expungeStaleEntry(i);
                return;
            }
        }
    }
```
`set()`过程中处理旧槽的核心方法——`replaceStaleEntry()`
```

    /**
     * Replace a stale entry encountered during a set operation
     * with an entry for the specified key.  The value passed in
     * the value parameter is stored in the entry, whether or not
     * an entry already exists for the specified key.
     *
     * As a side effect, this method expunges all stale entries in the
     * "run" containing the stale entry.  (A run is a sequence of entries
     * between two null slots.)
     *
     * @param  key the key
     * @param  value the value to be associated with key
     * @param  staleSlot index of the first stale entry encountered while
     *         searching for key.
     */
    //run：两个空槽中间所有的非空槽
    //姑且翻译成运行区间
    private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                   int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;

        // 备份检查当前运行中的以前的陈旧条目.
        // 我们每次清理整个运行区间，避免垃圾收集器一次释放过多的引用
        // 而导致增量的哈希


        // slotToExpunge删除的槽的起始位置，因为在后面清除(expungeStaleEntry)
        // 的过程中会扫描从当前位置到第一个空槽之间的位置，所以这里只需要判断出
        // 扫描的开始位置就可以
        int slotToExpunge = staleSlot;
        
        //向前扫描找到最前面的旧槽
        for (int i = prevIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = prevIndex(i, len))
            //在向前扫描的过程中找到了旧槽旧覆盖旧槽的位置
            if (e.get() == null)
                slotToExpunge = i;

        // Find either the key or trailing null slot of run, whichever
        // occurs first
        //从传进来的旧槽的位置往后查找key值或者第一个空槽结束
        for (int i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();

            // 如果在staleSlot位置的槽后面找到了指定的key，那么将他和staleSlot位置的槽进行交换
            // 以保持哈希表的顺序
            // 然后将新的旧槽护着上面遇到的任何过期的槽通过expungeStaleEntry删除
            // 或者重新哈希所有运行区间的其他条目

            //找到了对应的key，将他和staleSlot位置的槽进行交换
            if (k == key) {
                e.value = value;

                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;

                // 判断清理旧槽的开始位置
                // 如果在将staleSlot之前没有旧槽，那么就从当前位置为起点清理
                if (slotToExpunge == staleSlot)
                    slotToExpunge = i;
                
                //expungeStaleEntry清理从给定位置开始到第一个null的区间的空槽
                //并返回第一个null槽的位置p
                //cleanSomeSlots从expungeStaleEntry返回的位置p开始清理log(len)次表

                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);

                //将给定的key-value已经存放，也清理了相应运行区间 ，返回
                return;
            }

            // 如果在向后查找后没有找到对应的key值，而当前的槽是旧槽
            // 同时如果在向前查找中也没查找到旧槽
            // 那么进行槽清理的开始位置就是当前位置
            //为什么不是staleSlot呢？因为在这个位置创建指定的key-value存放
            if (k == null && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }

        // 如果在向后查找后没有找到对应的key值，在staleSlot位置创建该key-value
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);

        // 确定扫描过程中发现过空槽
        if (slotToExpunge != staleSlot)
            //清理
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }

```

`replaceStaleEntry()`计算了很久的扫描清理的起点位置，总结下应该分为四种情况：
 - `prev`方向上没有旧槽，在`next`方向上找到`key`值之前没有找到旧槽，那么就交换`key`和`staleSlot`然后从当前位置向后清理空槽
 - `prev`方向上没有旧槽，在`next`方向上没有找到`key`没有找到旧槽，那么在`staleSlot`位置创建指定的`key-value`，
 - `prev`方向上没有旧槽，在`next`方向上没有找到`key`但是找到旧槽，那么在`staleSlot`位置创建指定的`key-value`，并从找到的第一个旧槽的位置开始清理旧槽
 - `prev`方向上找到旧槽，在`next`方向上没有找到`key`，那么在`staleSlot`位置创建指定的`key-value`，从`prev`方向最后一个找到的旧槽开始清理旧槽
 - `prev`方向上找到旧槽，在`next`方向上找到`key`，那么就交换`key`和`staleSlot`，从`prev`方向最后一个找到的旧槽开始清理旧槽
 
 *求推荐个好看的画图工具*

清理旧槽的核心方法——`expungeStaleEntry()`和`cleanSomeSlots()`

```
    /**
     * 删除指定位置上的旧条目，并扫描从当前位置开始到第一个发现的空位之间的陈旧条目
     * 还会删除尾随的第一个null之前的所有旧条目
     */
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;

        // 释放槽
        tab[staleSlot].value = null;
        tab[staleSlot] = null;
        size--;

        // 重新哈希直到遇到null
        Entry e;
        int i;
        //第staleSlot槽已经空出来，
        //从下一个槽开始扫描旧条目直到遇到空槽
        //因为整个表只适用2/3的空间，所以必然会遇到空槽
        //删除扫描期间遇到的空槽
        for (i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            //遇到的空槽直接删除
            if (k == null) {
                e.value = null;
                tab[i] = null;
                size--;
                //非空槽通过重新哈希找到清理后适当的存储位置
            } else {
                int h = k.threadLocalHashCode & (len - 1);
                //重新哈希的结果不在原位置，那么将原位置的槽空出来
                if (h != i) {
                    tab[i] = null;

                    // Unlike Knuth 6.4 Algorithm R, we must scan until
                    // null because multiple entries could have been stale.
                    //寻找到从第h位开始的第一个空槽放置
                    while (tab[h] != null)
                        h = nextIndex(h, len);
                    tab[h] = e;
                }
            }
        }
        //返回扫描过程中遇到的第一个空槽
        return i;
    }

    /**
     * Heuristically scan some cells looking for stale entries.
     * This is invoked when either a new element is added, or
     * another stale one has been expunged. It performs a
     * logarithmic number of scans, as a balance between no
     * scanning (fast but retains garbage) and a number of scans
     * proportional to number of elements, that would find all
     * garbage but would cause some insertions to take O(n) time.
     *
     * @param i a position known NOT to hold a stale entry. The
     * scan starts at the element after i.
     *
     * @param n scan control: {@code log2(n)} cells are scanned,
     * unless a stale entry is found, in which case
     * {@code log2(table.length)-1} additional cells are scanned.
     * When called from insertions, this parameter is the number
     * of elements, but when from replaceStaleEntry, it is the
     * table length. (Note: all this could be changed to be either
     * more or less aggressive by weighting n instead of just
     * using straight log n. But this version is simple, fast, and
     * seems to work well.)
     *
     * @return true if any stale entries have been removed.
     */
    //扫描一部分表清理其中的就条目，
    //扫描log(n)个
    private boolean cleanSomeSlots(int i, int n) {
        //在这次清理过程中是否清理了部分槽
        boolean removed = false;
        Entry[] tab = table;
        int len = tab.length;
        do {
            //从i的下一个开始查找（第i个在调用这个方法前已经查找）
            i = nextIndex(i, len);
            Entry e = tab[i];
            //key=e.get(),e不为null，但key为null
            if (e != null && e.get() == null) {
                n = len;
                //标记清理过某个槽
                removed = true;
                //清理过程
                i = expungeStaleEntry(i);
            }
            //只扫描log(n)个槽，一方面可以保证避免过多的空槽的堆积
            //一方面可以保证插入或者删除的效率
            //因为删除的时候会扫描两个空槽之前的槽位
            //每个槽位全部扫描的话时间复杂度会高，
            //因为在expungeStaleEntry扫描当前位置到第一个空槽之间所有的旧槽
            //所以在这里进行每个槽位的扫描会做很多重复的事情，效率低下
            //虽然扫描从i开始的log（n）也会有很多重复扫描，但是通过优良的哈希算法
            //可以减少哈希冲突也就可以减少重复扫描的数量
        } while ( (n >>>= 1) != 0);
        return removed;
    }

    /**
     * Re-pack and/or re-size the table. First scan the entire
     * table removing stale entries. If this doesn't sufficiently
     * shrink the size of the table, double the table size.
     */
    private void rehash() {
        expungeStaleEntries();

        // Use lower threshold for doubling to avoid hysteresis
        if (size >= threshold - threshold / 4)
            resize();
    }

    //扩容至当前表的两倍容量
    private void resize() {
        Entry[] oldTab = table;
        int oldLen = oldTab.length;
        int newLen = oldLen * 2;
        Entry[] newTab = new Entry[newLen];
        int count = 0;

        for (int j = 0; j < oldLen; ++j) {
            Entry e = oldTab[j];
            if (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null; // Help the GC
                } else {
                    //在新的哈希表中的哈希位置
                    int h = k.threadLocalHashCode & (newLen - 1);
                    while (newTab[h] != null)
                        h = nextIndex(h, newLen);
                    newTab[h] = e;
                    count++;
                }
            }
        }
        //设置下一个需要扩容的临界量
        setThreshold(newLen);
        size = count;
        //替换旧表
        table = newTab;
    }

    /**
     * Expunge all stale entries in the table.
     */
    private void expungeStaleEntries() {
        Entry[] tab = table;
        int len = tab.length;
        for (int j = 0; j < len; j++) {
            Entry e = tab[j];
            if (e != null && e.get() == null)
                expungeStaleEntry(j);
        }
    }
}
```

##一些问题
--------------
- 为什么使用弱引用：
    因为使用弱引用不会影响ThreadLocal对象被释放后的垃圾回收，由于使用了弱引用，
    被释放的对象只能存活到下一次`gc`，对象被回收后弱引用就变为null，这时候就可以进行判断这个位置的条目是否已经是旧条目，
    从而进行清理。防止内存泄漏，
    尽管软引用的对象也可以被垃圾回收掉，但是对象会存活到内存不足这样会造成内存泄漏
- 如何解决旧的槽中`Entry`对象不被回收造成内存泄漏问题：在`get()`中碰到旧槽会通过`expungeStaleEntry()`方法来清理旧槽，
    清理完成后会继续向后清理直到遇到第一个空槽，在此期间会进行哈希重定位操作，
    将每个槽中的都西昂放在合适的位置以维持哈希表的顺序；
    在`set()`方法中调用`replaceStaleEntry`-->`expungeStaleEntry()`清理整个运行区间内的旧槽，
    并调用`cleanSomeSlots()`循环扫描log(n)个槽位进行清理，使用优秀的哈希算法减少每次调用到`expungeStaleEntry()`重复清理同一区间的工作；

-------------

 $\color{#FF3030}{转载请标明出处}$