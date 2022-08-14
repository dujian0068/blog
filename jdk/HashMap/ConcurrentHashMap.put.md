测试代码：
```java
public class ConcurrentHashMapTest {
    public static void main(String[] args) {
        ConcurrentHashMap<String ,String> map=new ConcurrentHashMap<>();
        map.put("test","test");
    }
}
```
`put()`方法
```
public V put(K key, V value) {
    return putVal(key, value, false);
}
```
`putVal()`方法
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 初始化哈希表
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();

        // i为计算出来的当前key对应的tab的位置
        // f为tab中对应位置的node节点
        // f为null，则这个槽没有被占用，可以使cas操纵尝试修改
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }

        // 好像说这块是扩容的状态，ok那就看扩容再回头看
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        //插入新的键值对的过程
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //判断这个key在不在这个map中，替换旧值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
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

`initTable()`方法
```

private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {

        // sizeCtl 哈希表初始化或者扩容的控制器，当sizeCtl<0时，代表正在扩容或者初始化
        // -1代表初始化 
        // -（1+活动的扩容的线程的数目）代表扩容
        if ((sc = sizeCtl) < 0)
            //有线程正再执行初始化则没有执行初始化的线程让出cpu占用
            Thread.yield(); // lost initialization race; just spin

        // 如果没有线程正在执行扩容操作，则当前线程尝试使用cas操作将sizeCtl修改为1，成功则执行扩容，否则则循环
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    // 计算初始容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //计算下次扩容的阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                //设置sizeCtl为扩容阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```