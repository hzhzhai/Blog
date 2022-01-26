[AQS简介](#AQS简介)  
[AQS数据结构](#AQS数据结构)  
[框架](#框架)  
[独占模式](#独占模式) 

### AQS简介
AbstractQueuedSynchronizer，抽象队列同步器，简称为AQS，是构建阻塞锁或者其他相关同步器的基础框架，是Java并发包的基础工具类。通过AQS这个框架可以对同步状态原子性管理、线程的阻塞和解除阻塞、队列的管理进行统一管理。  
AQS是抽象类，并不能直接实例化。  
当需要使用AQS的时候需要继承AQS抽象类并且重写指定的方法，这些方法包括线程获取资源和释放资源的方式(如ReentractLock通过分别重写线程获取和释放资源的方式实现了公平锁和非公平锁)。  
同时子类还需要负责共享变量state的维护，如当state为0时表示该锁没有被占，大于0时候代表该锁被一个或多个线程占领(重入锁)，而队列的维护(获取资源失败入队、线程唤醒、线程的状态等)不需要我们考虑，AQS已经帮我们实现好了。  
AQS的这种设计模式采用的正是模板方法模式。

总结起来子类的任务有：
- 通过CAS操作维护共享变量state；
- 重写资源的获取方式；
- 重写资源的释放方式。

完成以上三个任务即可实现自己的锁。  
AQS作为J.U.C的工具类，面向的是需要实现锁的实现者，而锁面向的是锁的使用者，这两者的区别还是需要搞清楚的。

### AQS数据结构
**AQS的成员变量**
```java
//头结点，当前持有锁的线程
private transient volatile Node head;

//阻塞的尾节点，每个新的节点进来都插入到最后，也就形成了一个链表
private transient volatile Node tail;

//当前锁的状态，0代表没有被占用，大于0代表有线程持有当前锁
//之所以说大于0，而不是等于1，是因为锁可以重入，每次重入都加上1
private volatile int state;

//代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
//reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
//if(currentThread == getExclusiveOwnerThread()) {state++}
private transient Thread exclusiveOwnerThread; //继承自AbstractOwnableSynchronizer
```

**内部类Node**
```java
static final class Node {
    //代表当前节点(线程)是共享模式
    static final Node SHARED = new Node();
    //代表当前节点(线程)是独占模式
    static final Node EXCLUSIVE = null;
    //代表当前节点(线程)已被取消调度。当timeout或被中断(响应中断的情况下)，会触发变更为此状态，进入该状态后的结点将不会再变化。
    static final int CANCELLED = 1;
    //代表当前节点(线程)的后继节点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。
    static final int SIGNAL = -1;
    //代表节点(线程)在Condition queue中。当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
    static final int CONDITION = -2;
    //代表当前节点的后继节点(线程)会传播唤醒的操作，仅在共享模式下才有作用
    static final int PROPAGATE = -3;
    //代表当前节点的状态，它的取值除了以上说的CANCELLED、SIGNAL、CONDITION、PROPAGATE，同时
    //还可能为0(新结点入队时的默认状态)，代表当前节点在sync队列中，阻塞着排队获取锁。
    //注意，负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0、<0来判断结点的状态是否正常。
    volatile int waitStatus;
    //当前节点的前驱节点
    volatile Node prev;
    //当前节点的后继节点
    volatile Node next;
    //当前节点关联的线程
    volatile Thread thread;
    //在condition队列中的后继节点
    Node nextWaiter;
    
    //判断当前节点是否为共享模式
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
    
    //返回当前节点的前驱节点 没有前驱节点则抛出异常
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }    
}
```

### 框架
![](../../resources/concurrent/AQS.png)  
它维护了一个volatile int state(代表共享资源)和一个FIFO线程等待队列(多线程争用资源被阻塞时会进入此队列)。  
state的访问方式有三种:
- getState()
- setState()
- compareAndSetState()

AQS定义两种资源共享方式：
- Exclusive(独占，只有一个线程能执行，如ReentrantLock)
- Share(共享，多个线程可同时执行，如Semaphore/CountDownLatch)

不同的自定义同步器争用共享资源的方式也不同，在实现时只需要实现共享资源state的获取与释放方式即。至于具体线程等待队列的维护(如获取资源失败入队/唤醒出队等)，AQS已经在顶层实现好了。  
自定义同步器实现时主要实现以下几种方法：
- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。
再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N(注意N要与线程个数一致)。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

### 独占模式
独占模式即一个线程获取到资源后，其他线程不能再对资源进行任何操作，只能阻塞获得资源。
![](../../resources/concurrent/EXCLUSIVE.png)  

**获取资源**
(1)线程调用子类重写的tryAcquire方法获取资源，如果获取成功，则流程结束，否则继续往下执行。
(2)调用addWaiter方法(详细过程看下面的源码解析)，将该线程封装成Node节点，并添加到队列队尾。
(3)调用acquireQueued方法让节点以”死循环”方式进行获取资源，为什么死循环加了双引号呢？因为循环并不是一直让节点无间断的去获取资源，节点会经历 获取资源->失败->线程进入等待状态->唤醒->获取资源……，线程在死循环的过程会不断等待和唤醒，节点进入到自旋状态(详细过程看下面的源码解析)，再循环过程中还会将标识为取消的前驱节点移除队列，同时标识前驱节点状态为SIGNAL。
线程的等待状态是通过调用LockSupport.lock()方法实现的，这个方法会响应Thread.interrupt，但是不会抛出InterruptedException异常，这点与Thread.sleep、Thread.wait不一样。
```java
/**
此方法是独占模式下线程获取共享资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响。获取到资源后，线程就可以去执行其临界区代码了。
函数流程如下：
1.tryAcquire()尝试直接去获取资源，如果成功则直接返回(这里体现了非公平锁，每个线程获取锁时会尝试直接抢占加塞一次，而CLH队列中可能还有别的线程在等待)；
2.如果获取失败，则调用addWaiter()，将线程封装到Node节点中，然后再将该Node节点加入等待队列的尾部，并标记为独占模式；
3.调用acquireQueued()使线程阻塞在等待队列中等待获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。
需要注意的是，tryAcquire()就是实现AQS的子类需要去重写的方法。
*/
public final void acquire(int arg) {
    if(!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)){
        selfInterrupt();
    }
}

/**
此方法尝试去获取独占资源。如果获取成功，则直接返回true，否则直接返回false。
AQS这里只定义了一个接口，具体资源的获取方式交由自定义同步器去实现(通过state的get/set/CAS)！！！至于能不能重入，能不能加塞，那就看具体的自定义同步器怎么去设计了！！！当然，自定义同步器在进行资源访问时要考虑线程安全的影响。
这里之所以没有定义成abstract，是因为独占模式下只用实现tryAcquire-tryRelease，而共享模式下只用实现tryAcquireShared-tryReleaseShared。如果都定义成abstract，那么每个模式也要去实现另一模式下的接口。说到底，这样可以尽量减少不必要的工作量。
*/
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

/**
方法用于将当前线程加入到等待队列的队尾，并返回当前线程所在的结点。
*/
private Node addWaiter(Node mode) {
    //以给定模式构造结点。mode有两种：EXCLUSIVE(独占)和SHARED(共享)
    Node node = new Node(Thread.currentThread(), mode);

    //尝试快速方式直接放到队尾。
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }

    //如果上面的CAS操作插入不成功，则调用enq方法 死循环插入 直到成功。
    enq(node);
    return node;
}

/**
此方法用于将node加入队尾。
*/
private Node enq(final Node node) {
    //CAS"自旋"，直到成功加入队尾
    for (;;) {
        Node t = tail;
        if (t == null) { //队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {//正常流程，放入队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

/**
通过tryAcquire()和addWaiter()，该线程获取资源失败，已经被放入等待队列尾部进入等待状态休息，直到其他线程彻底释放资源后唤醒自己。
节点以“死循环”的方式去获取资源，为什么死循环加了双引号呢？
因为循环并不是一直让节点无间断的去获取资源，节点会经历 获取资源->失败->线程进入等待状态->唤醒->获取资源......，
线程在死循环的过程会不断等待和唤醒，即节点的自旋。
*/
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//标记是否成功拿到资源
    try {
        boolean interrupted = false;//标记等待过程中是否被中断过

        //又是一个“自旋”！
        for (;;) {
            final Node p = node.predecessor();//拿到前驱
            //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
            if (p == head && tryAcquire(arg)) {
                setHead(node);//拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                p.next = null; // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                failed = false; // 成功获取资源
                return interrupted;//返回等待过程中是否被中断过
            }

            //如果自己可以休息了，就通过park()进入waiting状态，直到被unpark()。如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
        }
    } finally {
        if (failed) // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
            cancelAcquire(node);
    }
}

/**
如果前驱结点的状态不是SIGNAL，那么自己就不能安心去休息，需要去找个安心的休息点。
此方法主要用于检查状态，看看自己是否真的可以去休息了(进入waiting状态)。
*/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;//拿到前驱的状态
    if (ws == Node.SIGNAL)
        //如果已经告诉前驱拿完号后通知自己一下，那就可以安心休息了
        return true;
    if (ws > 0) {
        /*
         * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
         * 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被保安大叔赶走了(GC回收)！
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //如果前驱节点没有做好准备(标志状态为SIGNAL)、前驱节点也没有被取消，
        //则使用CAS操作将前驱节点的状态更新为SIGNAL，然后返回false，为什么
        //是返回false呢？因为CAS操作并不保证一定能更新成功，返回false的目的
        //是让acquireQueued函数再执行一次for循环，这个循环第一可以让该节点
        //再尝试获取资源(万一成功了呢)，第二是让acquireQueued函数再调用
        //一次shouldParkAfterFailedAcquire函数(即本函数)判断节点的前驱节点是
        //否已经设置为SIGNAL状态了。
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

/**
如果线程找好安全休息点后，那就可以安心去休息了。此方法就是让线程去休息，真正进入等待状态。
park()会让当前线程进入waiting状态。
在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()。
需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。
*/
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//调用park()使线程进入waiting状态
    return Thread.interrupted();//如果被唤醒，查看自己是不是被中断的。
}

/**
看了shouldParkAfterFailedAcquire()和parkAndCheckInterrupt()，现在让我们再回到acquireQueued()，总结下该函数的具体流程：
1.结点进入队尾后，检查状态，找到安全休息点；
2.调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
3.被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。
*/
```

**释放资源**
(1)线程调用子类重写的tryRelease方法进行释放资源，如果释放成功则继续检查线程(节点)是否有后继节点，有后继几点则去唤醒。
(2)调用unparkSuccessor方法进行后继节点的唤醒，如果后继节点为取消状态，则从队列的队尾往前遍历，找到一个离节点最近且不为取消状态的节点进行唤醒，如果后继节点不为取消状态则直接唤醒。
```java
/**
上一小节已经把acquire()说完了，这一小节就来讲讲它的反操作release()吧。此方法是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。
它调用tryRelease()来释放资源。有一点需要注意的是，它是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自定义同步器在设计tryRelease()的时候要明确这一点！！
*/
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;//找到头结点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);//唤醒等待队列里的下一个线程
        return true;
    }
    return false;
}

/**
此方法尝试去释放指定量的资源。下面是tryRelease()的源码：
跟tryAcquire()一样，这个方法是需要独占模式的自定义同步器去实现的。正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false。
*/
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}

/**
此方法用于唤醒等待队列中下一个线程。
这个函数并不复杂。一句话概括：用unpark()唤醒等待队列中最前边的那个未放弃线程，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了！！And then, DO what you WANT!
*/
private void unparkSuccessor(Node node) {
    //这里，node一般为当前线程所在的结点。
    int ws = node.waitStatus;
    if (ws < 0)//置零当前线程所在的结点状态，允许失败。
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;//找到下一个需要唤醒的结点s
    if (s == null || s.waitStatus > 0) {//如果为空或已取消
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev) // 从后向前找。
            if (t.waitStatus <= 0)//从这里可以看出，<=0的结点，都是还有效的结点。
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒
}
```