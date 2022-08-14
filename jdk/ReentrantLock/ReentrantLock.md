##本文结构
- Tips：说明一部分概念及阅读源码需要的基础内容
- ReentrantLock简介
- 公平机制：对于公平机制和非公平机制进行介绍，包含对比
- 实现：`Sync`源码解析额，公平和非公平模式的加锁、解锁过程及源码分析
- 公平锁和非公平锁的加锁流程图
- `ReentrantLock`提供的一些其他方法
- `Condition`：这里只是一提，不会有什么有意义的内容
- `ReentrantLock`全部源码理解：个人阅读`ReentrantLock`源码的时候做的注释，需要结合`AQS`一块理解

##Tips：
-------
- 同步等待队列：即普遍的资料中提到的同步队列，由`AQS`维护。在代码的英文注释中写道wait queue，因此我在这里翻译成同步等待队列
- 条件等待队列：即普遍的资料中提到的等待队列，由`AQS.Condition`维护，在代码的英文注释中也是写道`wait queue`，因此我在这里翻译成条件等待队列
- 本篇文章会大量提到`AQS`这个类，并且大量的方法都在`AQS`中实现，本文对会对使用到的方法进行解释，但是对于`AQS`内部的属性没有过多解释，后续篇章写`AQS`会专门写，建议可以了解一下，有助于理解
- 建议阅读前先了解`AQS`中的`Node`类,有助于阅读，本文不会说明

## ReentrantLock简介
----------
在多线程编程中，同步和互斥是一个非常重要的问题。
在java中可以通过使用`synchronized`来实现共享资源的独占，
除此之外还可以使用`Lock`提供的方法来实现对共享资源的独占。
而且`Lock`对比`synchronized`具有更高的灵活性。

`ReentrantLock`是`Lock`接口的一种实现，它提供了公平和非公平两种锁的公平机制供开发者选择，
并且实现了锁的可重入性（指的是对同一临界资源重复加锁，注意：加锁多少次就一定要解锁多少次）。


## 公平机制
-------
###概念
`ReentrantLock`提供了公平锁和非公平锁两个版本供开发者选择：
 - 公平锁：所有请求获取锁的线程按照先后顺序排队获取锁，下一个获取锁的线程一定是等候获取锁时间最长的线程，所得获取满足`FIFO`特点。
    
 - 非公平锁：请求获取锁的线程不一定需要排队，只要锁没有被获取，不论是否有线程在等待获取所，他就可以进行加锁操作。eg：所有在队列中等待锁的线程都没有被操作系统调度到并且锁没有被获取，此时正在指定的线程需要获取锁就可以直接尝试获取锁
 
###对比
||公平锁|非公平锁|
|:----:   |:---:|:-----: |
|优点 |所有的线程都可以获取到资源不会饿死在队列当中|可以减少CPU唤醒线程的开销，整体的吞吐效率会高点，CPU也不必取唤醒所有线程，会减少唤起线程的数量 |
|缺点 |吞吐量会下降很多，队列里面除了第一个线程，其他的线程都会阻塞，cpu唤醒阻塞线程的开销会很大|可能导致队列中间的线程一直获取不到锁或者长时间获取不到锁，导致饿死|


##实现
---------

###抽象类`Sync`


公平与非公平的实现主要靠锁的同步器来实现，他们都是内部抽象类`Sync`的子类(姑且称之为抽象同步器)。

$\color{#FF3030}{Sync及父类AQS提供了整个加锁和解锁的过程及排队等待的过程，}$并暴露出抽象方法`lock()`供实现不同的锁公平机制。

`ReentrantLock`中`Sync`的子类`NonfairSync`提供非公平锁同步机制,`FairSync`提供公平的锁同步机制。

代码的解释请看注释
```
    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    //提供了这个锁的同步器的基础方法，子类NonfairSync和FairSync提供了公平和非公平两种同步器的实现
    //使用AQS的state来标识锁的状态
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        //抽象方法加锁，加锁过程交给子类实现以提供不同的公平机制
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        //默认提供了非公平机制的加锁过程
        //acquires 申请加锁的次数，一般情况下是一次，但是有多次的情况，在Condition中会看到
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //获取锁的状态，getState在AQS中实现
            int c = getState();
            //锁空闲
            if (c == 0) {
                //加锁，加锁成功设置锁的属于哪个线程信息
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //当前线程已经成功获取了锁，这块也是锁的可重入性的体现
            else if (current == getExclusiveOwnerThread()) {
                //将锁的持有次数加给定的次数即可
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                //设置加锁次数
                setState(nextc);
                return true;
            }
            return false;
        }
        
        //释放锁的过程
        //releases释放锁的次数，一般情况下是一次，但是有多次的情况，在Condition中会看到
        protected final boolean tryRelease(int releases) {
            //getState在AQS中实现
            int c = getState() - releases;
            //独占锁释放锁的时候谁获取的锁谁用完释放，期间不许其他线程使用
            /如果锁的拥有者不是当前线程代码结构则出了问题
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            //由于可重入性，只有在释放releases次后锁状态为0才完全释放锁，锁才不被占有
            boolean free = false;
            //如果释放锁后锁状态为0，则表示当前线程不再持有这个锁
            //则将持有锁的线程exclusiveOwnerThread置null
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            //设置锁状态，，在AQS中实现
            setState(c);
            return free;
        }

        //检查当前线程是否持当前锁
        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        //条件等待队列
        //具体实现在AQS中实现
        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class
        //获取当前锁的持有者
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        //获取当前线程持有锁的次数
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        //检查锁是否被持有
        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```
###公平锁

公平锁提供了一种绝对等待时间公平的机制，锁永远会被同步等待队列中等待时间最长的获取到，
这样可以保证每个在等待的线程在程序不退出的情况下都可以获取到锁。
但是每一个请求的锁的线程都将进入到同步等待队列中阻塞休眠，线程的休眠和唤醒需要耗费额外的时间，会降低效率，降低吞吐量
（整个在同步等待队列中阻塞休眠的操作不是绝对的，只有所没有被占有，并且同步等待队列为空时可以直接获取锁或者递归调用同一个线程获取锁）

####公平锁的同步器
```
    /**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            //调用AQS提供的请求加锁的方法
            //紧接着下一个代码片段解释
            acquire(1);
        }



        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        //公平版本的tryAcquire，
        //第一个获取锁的线程和已经获取锁的线程递归调用获取锁不需要再排队等待
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //判断同步等待队列中是否有其他等待获取锁的线程
                //没有并且锁没有被其他线程持有则可以直接获取锁
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //如果当前线程持有锁，则可以锁计数器加一让该线程再一次获取锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //否则当前线程在这次尝试获取锁之前就没有获取到锁
            //同步等待队列中有其他线程在尝试获取锁
            //则获取锁失败，下一步应该会被加入到同步等待队列
            return false;
        }
    }
```
简单看下`hasQueuedPredecessors()`方法,这里只解释满足公平锁加锁条件的情况
```
/**
 * Queries whether any threads have been waiting to acquire longer
 * than the current thread.
 */
//这个方法是再AQS中提供的用来判断线同步等待队列中是否还有等待时间比当前线程等待时间更长的线程
//tail是队列的尾节点，head是头节点，每个节点会代表一个线程首尾节点除外
//再tryAcquire中我们希望它返回的时false那么看下返回false代表那种情况
//返回false要求 h != t && ((s = h.next) == null|| s.thread != Thread.currentThread())为false，那么要么h==t就是头尾节点是同一个，队列为空
//要么(s = h.next) == null|| s.thread != Thread.currentThread()为false，
//这就要求h.next!=null 并且h.next就是当前线程，也就是说队列中第一个等待获取锁的线程就是当前线程
//那么就可以直接加锁；
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}

```
####公平锁的加锁

在公平锁的同步器中提供了`lock()`方法用来进行加锁操作，
可重入锁进行加锁的方法也确实是调用同步器的`lock()`来实现加锁

加锁入口：`lock()`方法；`lock()`方法定义在`ReentrantLock`类里面，通过调用同步器的加锁操作完成加锁过程，公平机制不影响对外暴露的接口
```
    /**
     * Acquires the lock.
     *
     * <p>Acquires the lock if it is not held by another thread and returns
     * immediately, setting the lock hold count to one.
     *
     * <p>If the current thread already holds the lock then the hold
     * count is incremented by one and the method returns immediately.
     *
     * <p>If the lock is held by another thread then the
     * current thread becomes disabled for thread scheduling
     * purposes and lies dormant until the lock has been acquired,
     * at which time the lock hold count is set to one.
     */
    public void lock() {
        sync.lock();
    }

```
官方的注释说明了`lock`操作会做那些事情：
1. 获取锁，如果所没有被其他线程获取，那么将获取这个锁并返回，并设置这个锁的状态为1标识被加锁一次（锁被获取一次）
2. 如果当前线程已经持有锁，那么锁计数器自增1
3. 如果锁被其他线程持有，那么当前线程会被阻塞进入睡眠状态并进入同步的等待队列，直到锁被获取，然后将锁的计数器设置为1

$\color{#00FF00}{这里面好像没说明公平的方式哈，只要锁没被获取就立刻获取，感觉公平锁是绿的}$

加锁的过程直接调用了同步器的`lock()`方法在上述的同步器中`lock()`调用`acquire()`方法

`acquire()`方法在`AQS`中实现

具体解释见注释
```
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    //以独占的方法加锁，并且忽略中断
    //那是不是还有响应中断的加锁呢？？
    public final void acquire(int arg) {
        //先尝试调用同步器的tryAcquire()方法加锁
        if (!tryAcquire(arg) &&
            //加锁失败的情况下将当前线程放入同步等待队列中
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            
            //acquireQueued返回值是线程在等待期间是否被中断，如果有则还原中断现场
            selfInterrupt();
    }
```
`addWaiter()`方法，使用当前线程构造一个同步等待队列的节点，并且放在队尾,在`AQS`中实现
```
    /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        //为当前的线程构造一个同步等待队列中的节点，在可重入锁中式排他的模式(mode==Node.EXCLUSIVE)
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //如果队尾不为空
        if (pred != null) {
            node.prev = pred;//将当前节点放在队尾
            //使用CAS操作原子的设置队尾节点
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //如果对尾节点为空或者CAS设置队尾节点失败调用enq方法将当前线程节点添加进队列
        enq(node);
        return node;
    }
```
`enq()`方法：如果队列不存在或者使用CAS操作使节点入队失败，则进入此方法构造队列并将节点入队,在`AQS`中实现
```
    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {

        //用一个死循环将当前节点添加进队列，如果队列不存在则创建队列
        for (;;) {
            Node t = tail;

            //尾节点为空则堆类不存在进行初始化队列，创建头节点
            if (t == null) { // Must initialize

                //使用CAS操作创建头节点，
                //为什么式CAS操作，试想刚判断出队列不存在需要创建头节点，
                //此时线程发生线程的调度当前线程阻塞，另一个线程做同样的操作并创建了队列
                //当前线程再次被唤醒后继续创建队列，会有线程安全问题
                if (compareAndSetHead(new Node()))
                    //尾节点指向头节点
                    tail = head;

            //队列存在死循环的将当前线程对应的节点放入到队列当中
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
`acquireQueued()`方法，对于排队等待的线程应当使其进行阻塞，减少调度以及CPU空转的时间，除非下一个就到了这个线程获取锁,在`AQS`中实现

这个方法的设计上如果自身是头节点的后继节点，那么有可能头节点会很快处理完成任务释放锁，自己就可以获取到锁，避免进行线程阻塞、唤醒操作，减少资源消耗
```
    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            
            //当前线程中断的标志位
            boolean interrupted = false;
            for (;;) {
                //获取当前节点的前驱节点
                final Node p = node.predecessor();
                
                //前驱节点是头节点则尝试调用同步器的tryAcquire获取锁
                //第一次进入者方法时尝试一次获取锁，获取失败则会进行判断是否需要进行阻塞
                //如果需要阻塞则阻塞，如果不需要阻塞则会将前驱节点的waitStatus设置为SIGNAL，
                //再次循环获取锁失败，再次进入判断是否需要阻塞时一定会被阻塞
                //获取失败即便先驱节点是头节点也会被阻塞
                if (p == head && tryAcquire(arg)) {

                    //成功获取锁则将当前节点设置为头节点
                    setHead(node);
            
                    //原先区节点的下一个节点置空，
                    p.next = null; // help GC  //怎么help个人理解当当前节点称为头节点再被释放的时候，那么当前节点可以做到不可达，从而gc
                    
                    failed = false;
                    //返回线程在队列中等待期间是否被中断
                    return interrupted;
                }
                
                //在先驱节点不是头结点的情况下阻塞当前线程并使其睡眠
                //直到被其他线程唤醒，这样可以减少CPU的空转，提高效率
                //从阻塞唤醒后继续for循环直到获取到锁，
                if (shouldParkAfterFailedAcquire(p, node) &&
                    //阻塞当前线程直到有其他线程唤醒，并返回中断信息
                    parkAndCheckInterrupt())
                    //如果在线程阻塞休眠期间线程被中断则设置终端标记位，
                    interrupted = true;
            }
        } finally {
            //如果没有获取到锁（获取锁的过程出了意外），或者取消了获取锁，则取消当前线程获取锁的操作
            if (failed)
                cancelAcquire(node);
        }
    }
```
`shouldParkAfterFailedAcquire()`方法,在`AQS`中实现
```
    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //获取先驱节点的等待状态
        int ws = pred.waitStatus;

        //SIGNAL前驱节点准备好唤醒后继节点，后继节点可以安全的阻塞
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        
        //大于0表示先驱节点已经被取消获取锁及排队
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            //循环向前找到一个没有取消的节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            //更新先驱节点
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            //// 更新pred结点waitStatus为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
`parkAndCheckInterrupt()`方法，调用`LockSupport.park()`阻塞指定线程,在`AQS`中实现
```$xslt
    /**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        //阻塞当前线程，直到其他线程调用LockSupport.unpark(当前线程)，
        //使得调用unpark的线程释放锁后当前线程被唤醒并返回在阻塞期间线程是否被中断
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
`cancelAcquire()`方法，取消获取锁,在`AQS`中实现
```
/**
     * Cancels an ongoing attempt to acquire.
     *
     * @param node the node
     */
    private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;
        
        //节点的线程置null
        node.thread = null;

        // Skip cancelled predecessors

        Node pred = node.prev;
        //如果前驱节点已经取消，那么循环向前找到一个没有取消的节点并设置当前节点的前驱节点
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        //设置当前节点状态为已经取消获取锁
        node.waitStatus = Node.CANCELLED;

        // If we are the tail, remove ourselves.
        //如果节点自身时队尾，则移除节点并使用CAS操作设置队尾
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;

            //前驱节点不是头节点，则当前节点就需要阻塞
            //前驱节点线程不为null，则保证存在前驱节点
            //前驱节点没有被取消，waitStatus可以被设置为SIGNAL保证可以唤醒后继节点
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;

                //写一个节点存在并且没有被取消，则CAS的将前驱节点的后继节点设置为当前节点的后继节点
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //否则唤醒后继线程
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```
至此，公平锁的加锁操作全部结束（或者已经取消获取锁），在同步等待队列中等待的线程，如果对应的节点的先驱节点不是头节点，则线程会被阻塞减少调度。

公平锁的公平的保证时靠加锁时判断当前锁对应的同步等待队列中存在等待的队列，如果有则当前线程入队，排队来保证的；
如果当前锁没有被获取，并且队列不存在或者队列中没有等待的线程则可以直接加锁。回到`acquire()`方法，如果在线程等等待的过程中发生了中断，
那么获取到所之后会还原中断。

#### 公平锁的解锁

通过加锁的过程可以发现，锁被获取的次数通过给`state`字段增加和设置锁所属的线程`exclusiveOwnerThread`来完成加锁操作，
那么当线程需要解锁的时候应该也是对这两个字段的操作，且解锁一定在加锁之后，因此不存在进入同步等待队列等待的过程。

解锁入口：`unlock()`方法；`unlock()`方法定义在`ReentrantLock`类里面，通过调用同步器的解锁操作完成解锁过程，公平机制不影响对外暴露的接口
代码具体解释见注释
```
    /**
     * Attempts to release this lock.
     *
     * <p>If the current thread is the holder of this lock then the hold
     * count is decremented.  If the hold count is now zero then the lock
     * is released.  If the current thread is not the holder of this
     * lock then {@link IllegalMonitorStateException} is thrown.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *         hold this lock
     */
    //尝试释放获取的锁，调用到release方法，这个方法在AQS中实现
    public void unlock() {
        sync.release(1);
    }
```
`release()`方法，在`AQS`中实现
```
    /**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    //释放持有的排他锁
    public final boolean release(int arg) {
        //调用tryRelease()来释放锁，由同步器实现
        //tryRelease方法的解释在同步器章节
        //如果有线程在同步等待队列中等待获取锁，
        //那么还应该唤醒等待的线程
        if (tryRelease(arg)) {
            //如果存在同步等待队列，那么当前节点解锁成功后回将自身节点设置为头节点
            //因此这里的头节点就是自身当前线程的节点
            //但是在加锁成功的时候会将节点的thread字段设置为null，因此无法比对判断
            Node h = head;

            //后继线程在阻塞前以前会将前驱结点的waitStatus设置为SIGNAL=-1，因此不为0即需要唤醒后继节点
            //为什么是不为0，而不是等于-1？？因为node有1，-1，-2，-3和0五种情况
            //0代表默认状态，node节点刚被创建，
            //1代表当前节点已经取消获取锁，
            //-1代表有后继节点需要唤醒
            //-2代表节点在条件等待队列中等待，也就是不会出现在同步等待队列
            //-3代表共享模式，
            //如果在获取到锁并且已经存在后继节点的时候取消获取锁，那么节点就会使1，
            //直接点线程被唤醒完成加锁操作后释放锁，他的waitStatus使1而不是-1，因此使用的是waitStatus != 0
            if (h != null && h.waitStatus != 0)
                //唤醒后继节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
`unparkSuccessor()`方法唤醒后继节点，在`AQS`中实现
```
    /**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    //唤醒头结点的后继节点
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        //头节点的状态<0,则在发出唤醒信号之前尝试清除这个状态，即将头节点的状态设置为0，
        //允许失败或者被等待的线程改变
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        //下一个节点不存在或者取消了获取锁，则沿着队列从后往前找到第一个没有取消的节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒没有取消获取锁的第一个节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
至此可重入锁的公平模式的加锁和解锁全部结束

问题：在解锁的最后一步调用了`LockSupport.unpark()`来解锁，
而一个线程`B`进入同步等待队列阻塞的时候根据先去接点的`waitState==-1`来判断是否需要阻塞，
那么在他判断完前驱节点线程`A` `waitState==-1`成立然后发生系统调度，执行其他线程，
而这时候线程`A`获取锁并解锁调用了`LockSupport.unpark()`，然后执行线程`B`，
线程`B`会执行阻塞的过程，线程`B`会被阻塞掉，然后后面的节点都不能获取锁么？

### 非公平锁

除了公平的加锁方式，可重入锁还提供了非公平模式（默认）的加锁。在非公平模式下只要锁还没有被其他线程获取，就有机会成功获取锁，
当然已加入到队列中的线程还是要按照顺序排队获取。这样做会减少需要阻塞、唤醒的线程，降低由于阻塞、唤醒带来的额外开销，
但是在队列中等待的线程可能会被活活饿死（很惨的那种，出了问题排查的时候）
#### 非公平锁同步器
```
/**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            //和公平锁最大的区别
            //如果当前锁没有被其他线程获取则直接尝试加锁
            //没有被其他线程获取体现在参数值是0
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //调用AQS的acquire方法请求加锁，和公平锁一致
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            //调用父类Sync的nonfairTryAcquire请求加锁
            return nonfairTryAcquire(acquires);
        }
    }
```
####非公平锁加锁
加锁入口同公平模式：
```
    public void lock() {
        //都是调用到同步器的lock方法
        sync.lock();
    }
```
`lock()`直接调用同步器实现的`lock()`
```
        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            //在这里不管是否有线程在排队，直接尝试加锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //加锁失败则调用AQS的acquire方法
                acquire(1);
        }
```
`acquire()`方法在`AQS`中实现，和公平锁的一致，方便阅读就再写一次

具体解释见注释
```
    //以独占的方法加锁，并且忽略中断
    //那是不是还有响应中断的加锁呢？？
    public final void acquire(int arg) {
        //先尝试调用同步器的tryAcquire()方法加锁
        if (!tryAcquire(arg) &&
            //加锁失败的情况下将当前线程放入同步等待队列中
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            
            //acquireQueued返回值是线程在等待期间是否被中断，如果有则还原中断现场
            selfInterrupt();
    }
```
同步器的`tryAcquire()`方法见非公平模式的同步器，会调用到`Sync`的`nonfairTryAcquire()`方法，

如果加锁失败则会依次构建同步等待队列->尝试加锁->失败则判断是否需要进行阻塞->是则阻塞等待前驱节点唤醒->尝试加锁这样的流程

这个流程同公平锁
```
        //默认提供了非公平机制的加锁过程
        //acquires 申请加锁的次数，一般情况下是一次，但是有多次的情况，在Condition中会看到
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            //获取锁的状态，getState在AQS中实现
            int c = getState();
            //锁空闲
            if (c == 0) {
                //加锁，加锁成功设置锁的属于哪个线程信息
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //当前线程已经成功获取了锁，这块也是锁的可重入性的体现
            else if (current == getExclusiveOwnerThread()) {
                //将锁的持有次数加给定的次数即可
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                //设置加锁次数
                setState(nextc);
                return true;
            }
            return false;
        }
```
#### 非公平锁的解锁
非公平锁的解锁过程同公平锁，释放过程不存在公平于非公平，具体逻辑全部由`Sync`和`AQS`实现；

## 公平锁和非公平锁的加锁流程图
---------
![](可重入锁流程图.png)


##ReentrantLock提供的一些其他方法
------
|方法名称|返回值类型|方法描述|
| :---: | :---: | :---: |
|`lockInterruptibly()`|`void`|以响应中断的方式获取锁，如果在锁获取之前发生中断直接抛出中断异常，在从同步等待队列中被唤醒后检查在等待期间是否有中断发生，如果有则抛出中断异常|
|`tryLock()`|`boolean`|尝试一次以非公平模式获取锁，当且仅当锁没有被获取时有可能获取成功，即便锁私用的是公平模式，直接调用 `Sync`的非公平模式获取一次锁，返回获取结果|
|`tryLock(long timeout, TimeUnit unit)`|`boolean`|含有超时等待功能的获取锁，如果线程进入同步等待队列阻塞，则只会阻塞指定的时间，这个功能由`LockSupport`类提供|
|`newCondition()`|`Condition`|获取一个`Condition`对象,可以提供类似`Object Moitor`一样的功能|
|`getHoldCount()`|`int`|获取持有锁的次数，一般用于测试锁|
|`isHeldByCurrentThread`|`boolean`|检查当前线程是否持有这个锁|
|`isLocked`|`boolean`|检查锁是否被任何一a个线程获取|
|`isFair()`|`boolean`|检查当前锁是否是公平锁|
|`getOwner()`|`Thread`|获取持有锁的线程|
|`hasQueuedThreads()`|`boolean`|同步等待队列是否有线程等待获取锁|
|`hasQueuedThread(Thread thread)`|`boolean`|判断指定线程是否在同步等待队列中等待|
|`getQueueLength()`|`int`|获取同步等待对列的长度，队列中的线程不一定都是等待获取锁的线程，还有可能已经取消|
|`hasWaiters(Condition condition)`|`boolean`|判断给定的`condition`中是否由线程等待|
|`getWaitQueueLength(Condition condition)`|`int`|获取给定`condition`中等待队列的长度|
|`getWaitingThreads(Condition condition)`|`Collection<Thread>`|获取给定的`condition`中所有的等待线程|

##Condition
在`ReentrantLock`中提供了一个`newCondition()`方法，
利用返回的`Condition`对象可以实现`Object.wait()、Object.notify()、Object.notifyAll()`等功能，并且具有更强大的功能。
返回的`Condition`的实现是在`AQS`当中实现，所以这个放在`AQS`学习完了写。

-----------
 $\color{#FF3030}{转载请标明出处}$

###附ReentrantLock全部源码理解
--------------
```
/**
 * 一个可以重入的排他锁基本的行为和语义与使用synchronized作为
 * 隐式的锁的监听器是一样的，但是实现了更多的功能 
 *
 * 一个可重入锁后被最后一次成功加锁但是没有解锁的线程持有
 * 当一个锁没有被不被任何一个线程持有，这是一个线程来申请这个锁将会被申请成功
 * 可以使用{@link #isHeldByCurrentThread}, 和 {@link
 * #getHoldCount}检查当前线程是否持有锁
 *
 * 这个类的构造器提供了一个fairness参数来设定是否公平
 * <当设置为true时，是个公平锁，锁总是被当前等待时间最长的线程获取
 * 否则不能保证将按照何种顺序获取该锁
 * 使用公平锁的程序吞吐量比默认（非公平锁）更小
 * 但是在获得此线程和保证保证不被饿死方面会比默认更好
 * 然而锁的g公平性并不保证线程的公平性。因此使用公平锁的
 * 因此使用公平锁的的多个线程可能有一个会多次获得他，而其他线程可能一次也没有
 *
 * 注意：随机的tryLock并不支持公平性，
 * 只要锁可用即使有其他线程正在等待也可以获取他
 *
 * 在使用lock后使用try语句并且将unlock放在finally中
 *
 * 除了实现{@link Lock}接口外，
 * 还有这个类定义了许多检查锁状态的方法。
 * 其中的一些方法方法只对检测和监视有用
 *
 * <p>Serialization of this class behaves in the same way as built-in
 * locks: a deserialized lock is in the unlocked state, regardless of
 * its state when serialized.
 *
 * <p>这个锁最多支持冲入2147483647 次（递归调用）
 *
 */
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

    /**
     * 可重入锁非公平同步器
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * 加锁.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
                
                       //使用cas操作加锁，成功的话将锁的拥有线程置为自己
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                    //排他模式获取，忽略中断，通过再次调用tryAcquire获取锁，
                        //获取失败将进入等待队列，可能会重复的阻塞和取消阻塞，
                        //直到调用tryAcquire获取锁成功
                acquire(1);
        }
                //实现AQS的tryAcquire()方法（该方法进行一次尝试获取锁）
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    /**
     * 可重入锁的公平同步器
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

                 //公平模式与非公平模式的区别，公平模式下所有的所获取都会检查当前的等待队列
                //非公平锁可以进行抢占，即便有等待队列中有等待的线程，只要锁对象没有加锁成功，既可以被加锁
        final void lock() {
            acquire(1);
        }

        /**
         * 实现AQS的tryAcquire方法，加锁时会检查当前锁的等待队列的实际元素
         * 当等待队列为空（头节点==尾节点）或者头结点的下一个节点对应的线程试当前的线程可以加锁成功
                  *   如果当前线程已经持有
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    /**
     * 创建一个可重入锁.非公平模式
     * 与使用 {@code ReentrantLock(false)}.创建效果一样
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * 创建给定模式（公平或者非公平）的可重入锁
     * @param fair {@codetrue} 使用公平模式
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    /**
     * 获取锁.
     *
     * <p>如果这个锁没有被其他线程持有，则获取锁并立刻返回，
     * 将锁的计数器记为1
     *
     * 如果当前线程已经持有锁，则将锁的计数器自增1，
     * 并且立刻返回
     *
     * <p>如果锁被其他线程持有
     * 那么当前线程将会因为线程调度原因设置禁用并且为休眠状态，
     * 并且在锁被获取之前处于休眠状态
     * at which time the lock hold count is set to one.
     */
    public void lock() {
        sync.lock();
    }

    /**
     * 获取锁直到当前线程被阻塞
     *
     * <p>如果锁没被其他的线程获取则获取锁并返回
     *
     * 如果锁已经被当前线程获取，则将锁的计数器自增
     *
     * 如果锁被其他线程持有
     * 处于线程调度的目的会将当前线程设置为禁用并且为休眠状态
     * 直到以下两种情况中的一种发生:
     *
     * <ul>
     *
     * <li>当前线程获取到锁; or
     *
     * <li>其他的线程中断了当前的线程
     *
     * </ul>
     *
     * <p>如果当前线程获取到了锁，那么将锁的计数器设置为1
     *
     * <p>如果当前线程
     *
     * 在进入这个方法的时候中断状态已经被设置; 
     * 或者在获取锁的过程中被中断
     *
     * InterruptedException异常将会被抛出并且清除中断标记位
     *
     * @throws InterruptedException if the current thread is interrupted
     */
    public void lockInterruptibly() throws InterruptedException {
           //该方法会先检查当前线程是否被中断，然后调用tryAcquire获取锁
                //获取失败则调用doAcquireInterruptibly
                //这个方法和忽略中断的加锁acquireQueue基本相同，不同点是
                //当线程在调用LockSupport.park(this)被唤醒后，Lock方法会忽略是否该线程被中断，
                //若被中断该方法会抛出线程中断异常
                sync.acquireInterruptibly(1);
    }

    /**
     * 当且仅当锁没被其他线程获取时才获取锁
     *
     * <p>当且仅当锁没被其他线程获取时获取锁并且返回ture，设置锁的计数器为1
     * 即使这个锁被设定为公平锁，只要没有其他线程获取到锁，使用tryLock也能立刻获取到锁
     * 无论是否有其他线程在排队
     * 即使这种行为破坏了公平，但在某些情况下是有用的
     * 如果想要这个锁的公平设置可以使用
     * {@link #tryLock(long, TimeUnit) tryLock(0, TimeUnit.SECONDS) }
     * 这两者几乎是的等价的 (依然会检测中断).
     *
     * 如果当前线程已经持有锁，那么将锁计数器自增加一
     *
     * 如果当前锁被其他线程持有，那么返回false
     *
     * @return {@code true} if the lock was free and was acquired by the
     *         current thread, or the lock was already held by the current
     *         thread; and {@code false} otherwise
     */
    public boolean tryLock() {
            //与非公平锁tryAcquire方法内容相同
        return sync.nonfairTryAcquire(1);
    }

    /**
     * 在超时等待的时间内获取到锁并且线程没有被中断
     *
     * 如果在超时等待的时间内获取到锁并且没有被中断，锁没有被其他线程持有
         * 将返回true，同时将锁计数器设置为1
     * 如果当前锁使用的是公平锁的模式
     * 有其他的线程在等待这个锁，都不能获取到锁这个功能由（tryAcquire实现）
     * 如果想要在超时等待的时间内破坏锁的公平性获取锁可以和TryLock联合使用
     *
     *  <pre> {@code
     * if (lock.tryLock() ||
     *     lock.tryLock(timeout, unit)) {
     *   ...
     * }}</pre>
     *
     * <p>如果当前的线程已经拥有锁那么将锁计数器自增加一（tryAcquire实现）
     *
     * <p>如果锁被其他线程持有那么处于线程调度的目的会将当前线程置为不可用
     * 并且睡眠直到下面一种事情发生：
     *
     * <ul>
     *
     * <li>锁被当前线程获取，或者
     *
     * <li>其他线程中断了当前线程
     *
     * <li>时间超过了设置的等待时间
     *
     * </ul>
     *
     * <p>如果获取到了锁将返回ture，并且将锁计数器设置为1
     *
     * <p>如果当前线程
     *
     * <ul>
     *
     * <li>在进入这个方法之前，中断标记位被设置
     *
     * 或者获取锁的时候被中断
     *
     * </ul>
     * 会抛出 {@link InterruptedException} 异常，并且清楚中断标记位
     *
     * <p>如果超过了设置的等待时间将会返回false（其实还会尝试获取一次锁）
     * 如果设置的超时等待时间少于等于0，这个方法不会一直等待
     *
     * <p>这个方法会先响应中断，而不是锁的获取和等待
     *
     * @param timeout the time to wait for the lock
     * @param unit the time unit of the timeout argument
     * @return {@code true} if the lock was free and was acquired by the
     *         current thread, or the lock was already held by the current
     *         thread; and {@code false} if the waiting time elapsed before
     *         the lock could be acquired
     * @throws InterruptedException if the current thread is interrupted
     * @throws NullPointerException if the time unit is null
     */
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    /**
     * 试图释放锁
     *
     * <p>如果当前线程是锁的持有者，则锁的计数器自减1，
     * 如果锁的计数器为0，则释放锁
     * 如果当前线程不是锁的持有者，将抛出异常
         * 释放锁成功后如果等待队列中还有等待的线程，那么调用lockSupport.unspark唤醒等待的线程
     * {@link IllegalMonitorStateException}.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *         hold this lock
     */
    public void unlock() {
        sync.release(1);
    }

    /**
     * Returns a {@link Condition} instance for use with this
     * {@link Lock} instance.
     *
     * <p>The returned {@link Condition} instance supports the same
     * usages as do the {@link Object} monitor methods ({@link
     * Object#wait() wait}, {@link Object#notify notify}, and {@link
     * Object#notifyAll notifyAll}) when used with the built-in
     * monitor lock.
     *
     * <ul>
     *
     * <li>If this lock is not held when any of the {@link Condition}
     * {@linkplain Condition#await() waiting} or {@linkplain
     * Condition#signal signalling} methods are called, then an {@link
     * IllegalMonitorStateException} is thrown.
     *
     * <li>When the condition {@linkplain Condition#await() waiting}
     * methods are called the lock is released and, before they
     * return, the lock is reacquired and the lock hold count restored
     * to what it was when the method was called.
     *
     * <li>If a thread is {@linkplain Thread#interrupt interrupted}
     * while waiting then the wait will terminate, an {@link
     * InterruptedException} will be thrown, and the thread's
     * interrupted status will be cleared.
     *
     * <li> Waiting threads are signalled in FIFO order.
     *
     * <li>The ordering of lock reacquisition for threads returning
     * from waiting methods is the same as for threads initially
     * acquiring the lock, which is in the default case not specified,
     * but for <em>fair</em> locks favors those threads that have been
     * waiting the longest.
     *
     * </ul>
     *
     * @return the Condition object
     */
    public Condition newCondition() {
        return sync.newCondition();
    }

    /**
     * 查询锁被当前线程加锁的次数
     *
     * <p>线程对于每个加锁的操作都有一个对应的解锁操作
     *
     * <p>这个操作通常只被用来测试和调试
     * For example, if a certain section of code should
     * not be entered with the lock already held then we can assert that
     * fact:
     *
     *  <pre> {@code
     * class X {
     *   ReentrantLock lock = new ReentrantLock();
     *   // ...
     *   public void m() {
     *     assert lock.getHoldCount() == 0;
     *     lock.lock();
     *     try {
     *       // ... method body
     *     } finally {
     *       lock.unlock();
     *     }
     *   }
     * }}</pre>
     *
     * @return the number of holds on this lock by the current thread,
     *         or zero if this lock is not held by the current thread
     */
    public int getHoldCount() {
        return sync.getHoldCount();
    }

    /**
     * 检查当前线程是否持有这个锁
     *
     * <p>Analogous to the {@link Thread#holdsLock(Object)} method for
     * built-in monitor locks, this method is typically used for
     * debugging and testing. For example, a method that should only be
     * called while a lock is held can assert that this is the case:
     *
     *  <pre> {@code
     * class X {
     *   ReentrantLock lock = new ReentrantLock();
     *   // ...
     *
     *   public void m() {
     *       assert lock.isHeldByCurrentThread();
     *       // ... method body
     *   }
     * }}</pre>
     *
     * <p>It can also be used to ensure that a reentrant lock is used
     * in a non-reentrant manner, for example:
     *
     *  <pre> {@code
     * class X {
     *   ReentrantLock lock = new ReentrantLock();
     *   // ...
     *
     *   public void m() {
     *       assert !lock.isHeldByCurrentThread();
     *       lock.lock();
     *       try {
     *           // ... method body
     *       } finally {
     *           lock.unlock();
     *       }
     *   }
     * }}</pre>
     *
     * @return {@code true} if current thread holds this lock and
     *         {@code false} otherwise
     */
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    /**
     * 检查这个锁是否被任意一个线程持有（检查锁的计数器state是否是0）
     * 这个方法用来监控系统状态，而不是用来同步,
     *
     * @return {@code true} if any thread holds this lock and
     *         {@code false} otherwise
     */
    public boolean isLocked() {
        return sync.isLocked();
    }

    /**
     * 测试当前锁是公平锁还是非公平锁
     *
     * @return {@code true} if this lock has fairness set true
     */
    public final boolean isFair() {
        return sync instanceof FairSync;
    }

    /**
     * 获取当前拥有锁的线程，如果没有线程持有，则返回null
     * 此方法的目的是为了方便构造提供更广泛的锁监视设施的子类。
     *
     * @return the owner, or {@code null} if not owned
     */
    protected Thread getOwner() {
        return sync.getOwner();
    }

    /**
     * 检查是否有现成正在等待获取此锁（head!=tail） Note that
     * 因为线程取消获取锁可能发生在任何时间，所以返回值为true不能保证
     * 一定有其他线程在获取此锁，（比如等待队列中的线程已经取消获取此锁）
     * 这个方法的设计被用来监控系统
     *
     * @return {@code true} if there may be other threads waiting to
     *         acquire the lock
     */
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    /**
     * 判断给定的线程是否在等待这个锁
     * 线程取消获取锁可能发生在任何时间, 
     * true的返回值不代表这个线程还在等待获取这个锁
     * 这个方法的设计被用来监控系统
     *
     * @param thread the thread
     * @return {@code true} if the given thread is queued waiting for this lock
     * @throws NullPointerException if the thread is null
     */
    public final boolean hasQueuedThread(Thread thread) {
        return sync.isQueued(thread);
    }

    /**
     * 返回等待队列中等待获取锁的数量的估计值
     * 这个方法返回的是一个估计值因为队列中的线程数量可能在变化
     * 这个方法的设计被用来监控系统
     *
     * @return the estimated number of threads waiting for this lock
     */
    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    /**
     * 返回一个线程集合，集合中的线程可能在等待获取锁
     * 因为线程获取锁可能被取消所以获取到的集合不是准确的
     * 此方法的目的是为了方便构造提供更广泛的锁监视设施的子类
     *
     * @return the collection of threads
     */
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    /**
     * 查询与这个锁关联的condition是否有线程正在等待，即是否有现成调用过与
     * await方法.因为等待超时和线程中断发生在任何时候
     * 所以返回值true不代表将来会有信号来唤醒线程
     *  这个方法的设计被用来监控系统
     *
     * @param condition the condition
     * @return {@code true} if there are any waiting threads
     * @throws IllegalMonitorStateException if this lock is not held
     * @throws IllegalArgumentException if the given condition is
     *         not associated with this lock
     * @throws NullPointerException if the condition is null
     */
    public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
               //通过检查condition队列是中的节点是否处于condition状态实现
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    /**
     * 返回与锁相关的指定的condition的队列中的线程数量，注意这是个估计值
     * 因为线程等待超时和中断发生在任何时间
     * 因此队列一直在动态的变化，可能还没有统计完就已经发生了变化.
     *
     * @param condition the condition
     * @return the estimated number of waiting threads
     * @throws IllegalMonitorStateException if this lock is not held
     * @throws IllegalArgumentException if the given condition is
     *         not associated with this lock
     * @throws NullPointerException if the condition is null
     */
    public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    /**
     * 返回与锁相关的指定的condition的队列中的线程集合
     * 返回的集合只是一个估计值，
     * 因为线程等待超时和中断发生在任何时间
     *
     * @param condition the condition
     * @return the collection of threads
     * @throws IllegalMonitorStateException if this lock is not held
     * @throws IllegalArgumentException if the given condition is
     *         not associated with this lock
     * @throws NullPointerException if the condition is null
     */
    protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    /**
     * Returns a string identifying this lock, as well as its lock state.
     * The state, in brackets, includes either the String {@code "Unlocked"}
     * or the String {@code "Locked by"} followed by the
     * {@linkplain Thread#getName name} of the owning thread.
     *
     * @return a string identifying this lock, as well as its lock state
     */
    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }
}

```





