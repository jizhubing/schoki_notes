## AbstractQueuedSynchronizer 初识队列同步器方法
# 在了解AbstractQueuedSynchronizer主要方法时，需要先弄清楚Java并发中的独占式和共享式的区别
1. 不论是独占式还是共享式，都是Java并发包下提供的加锁方式
2. 独占锁模式：
- 每次只能有一个线程持有锁
- 属于悲观保守的加锁策略，避免了读/读冲突，如果某个只读线程获取共享状态，其他所有只读线程只能等待，势必影响性能
3. 共享锁模式
- 允许多个线程同时获取锁，因此可以并发访问共享资源
- 属于乐观锁，放宽了加锁策略，允许多个只读线程同时访问共享资源，也可以被一个写线程访问，但是不能两个写线程同时进行

# Node节点

```/
 * @since 1.5
 * @author Doug Lea 有必要了解一下作者
 */ 
   /**
   * Node节点，包括
   * 1、获取同步状态失败的线程引用
   * 2、等待状态
   * 3、前驱节点
   * 4、后继节点
   * 5、节点的属性类型、名称以及描述
   * 6、源码中的数据结构图
   *        <pre>
   *            +------+  prev +-----+       +-----+
   *        head |      | <---- |     | <---- |     |  tail
   *            +------+       +-----+       +-----+
   *        </pre>
   */
    static final class Node {
       //标记表明该节点为共享模式下等待获取同步状态
        static final Node SHARED = new Node();

       //标记表明该节点为独占模式下等待获取同步状态
        static final Node EXCLUSIVE = null;

        //等待状态的节点被取消
        static final int CANCELLED =  1;

        //后继节点处于处于等待状态，当前节点如果释放了同步状态或者被取消(当前节点状态值为-1)
        //会通知后继节点，使后继节点得以运行
        static final int SIGNAL    = -1;

        /**
        *节点处于等待队列中，节点线程等待在Condition上，其他线程对Condition调用了Signal()
        * 后，该节点会从等待队列中转到到同步队列中，加入到同步状态的获取
        * 这里先解释一下这个吧，听起来很迷糊。该状态主要和源码中的ConditionObject相对应
        * 如果有线程1和2
        * 线程1调用lock方法持有锁
        * 线程1调用await方法进入[条件等待对联]，同时释放锁
        * 线程1获取到线程2的signal信号，从[条件等待队列中]进入[同步等待队列]
        * 而对于线程2
        * 在获取锁时，由于线程1持有锁，则进入[同步等待队列]中
        * 1释放锁，2从[同步等待队列]中移除，获取锁，2再调用signal方法，导致1被唤醒
        * 2调用unlock，1获取锁
        *详细参考本类中的ConditionObject和
        *https://blog.csdn.net/u012420654/article/details/56496631
        *由此可见，AQS中目前来看，最少维护两个队列，
        * 1、等待获取同步状态的等待队列
        * 2、条件队列，一个节点要么在条件队列中，要么在等待队列中。两者不可同时存在    
        */
        static final int CONDITION = -2;
     
        //下一次的共享状态会被无条件的传播下去
        static final int PROPAGATE = -3;

       
       //等待状态
        volatile int waitStatus;

        //前驱节点
        volatile Node prev;

       //后继节点
        volatile Node next;

       //有类型Thread可知，该参数应该就是节点中获取同步状态的线程了
        volatile Thread thread;

        /**等待节点的后继节点，为什么要这个属性呢？
        * 如果当前等待节点为共享模式或者毒战模式，则后继节点的值为SHARED 或者null
        */
        Node nextWaiter;

       
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }```
    
    ```
    //头结点
    private transient volatile Node head;

   //未节点
    private transient volatile Node tail;

   
    private volatile int state;

    //返回同步状态的当前值
    protected final int getState() {
        return state;
    }

    //设置当前同步状态
    protected final void setState(int newState) {
        state = newState;
    }

    //使用CAS设置当前状态，使用CAS可以保证操作的原子性
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

    // Main exported methods

   //独占式获取同步状态，获取成功后，其他线程需要等待该线程释放同步状态(锁)才可以获取
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    //独占式释放同步状态
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }

    //共享式获取同步状态，返回值大于0标识获取成功，否则失败
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    //共享式释放同步状态
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

    //当前同步器是否是独占模式下被线程占有(可以理解成同步状态是否被当前线程所占有)
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }

   /**独占模式获取锁，如果当前线程获取同步状态，则返回 
   * 否则，将会进入同步队列进行等待，而该方法又调用了tryAcquire
   */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    /**与acquire相同，只是该方法响应中断。
    * 如果当前线程在获取同步状态过程中，进入了同步队列中
    * 如果被中断，则会抛出InterruptedException异常
    */
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

    /**
    *超时获取同步队列，如果当前线程在nanosTimeout内获取到同步状态
    *返回true,否则返回false
    * 同样，如果因获取同步状态进入了同步队列，如果被中断
    *InterruptedException异常
    */
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

   //独占式释放同步状态，在释放同步状态成功后，会唤醒同步队列中第一个节点包含的线程
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

   /**
   * 共享模式下获取同步状态(锁)，如果当前线程获取同步状态失败，则进入同步队列等待
   * 共享式与独占式主要区别在于，同一时刻可以有多个线程持有共享状态
   */
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    //共享式获取同步状态，同时响应中断
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

    //共享模式下获取同步状态，增加超时限制
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }

   //共享式释放同步状态
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }


   /**
   * 和Node节点中的Condition条件对应，即AQS中维护着这个Condition队列
   /
    public class ConditionObject implements Condition, java.io.Serializable {

    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;

    /**
    *在类加载的时候，就加载进方法区中的常量池中，并进行初始化操作
    */
    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));

        } catch (Exception ex) { throw new Error(ex); }
    }
}
```
