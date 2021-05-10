# 笔记

## ReentrantLock

### 加锁(非公平)

```java
// ReentrantLock
final void lock() {
    // 如果成功从0改成1，表示加锁成功，将独占线程设置为当前线程:
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    // 失败则调用AQS的acquire方法:
    else
        acquire(1);
}
// AbstractQueuedSynchronizer
public final void acquire(int arg) {
    // tryAcquire尝试加锁，用的是NonfairSync中的tryAcquire方法，竞争失败的线程在此处会再次尝试加锁:
    if (!tryAcquire(arg) &&
    // 如果失败，则将当前线程封装成一个节点加入到等待队列中去等待:
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

AQS的等待队列是一个双向链表，有head和tail两个头尾指针，并且第一个节点被称为dummy节点或者哨兵节点，它用于占位，不关联任何线程，节点对象中包含一个waitStatus属性，它表示状态，节点创建时默认为0，正常状态。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取前驱节点:
            final Node p = node.predecessor();
            // 判断前驱节点是否为哨兵节点，如果是那么当前节点就处于第二个节点，它可以再次尝试加锁:
            if (p == head && tryAcquire(arg)) {
                // 如果判断成功，将当前节点设为哨兵节点，将原来的哨兵节点从链表中断开:
                setHead(node);
                p.next = null; // help GC
                failed = false;
                // 等待线程即使被打断，也还是需要获得锁之后，才能返回打断状态:
                return interrupted;
            }
            // 如果当前节点既不是第二节点，或者尝试加锁失败，就会执行到该代码块，
            // 如果shouldParkAfterFailedAcquire返回真，那就执行parkAndCheckInterrupt将当前线程阻塞，
            // 如果shouldParkAfterFailedAcquire返回假，那它就会再次循环，再次执行上面的判断，
            // shouldParkAfterFailedAcquire方法逻辑就是将前驱节点的waitStatus状态改为-1，返回false，
            // 状态为-1的节点表示该节点有责任唤醒后续节点:
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 如果线程从阻塞状态被唤醒，则又返回到外层循环，
                // 被唤醒后的尝试加锁仍可能失败，因为此时可能有非等待队列中的线程来竞争锁:
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
        return true;
    if (ws > 0) {
        /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

### 解锁(非公平)

```java
// ReentrantLock
public void unlock() {
    sync.release(1);
}
// AQS
public final boolean release(int arg) {
    // tryRelease用的是ReentrantLock中Sync的tryRelease方法:
    if (tryRelease(arg)) {
        Node h = head;
        // 如果线程解锁成功(不成功是因为如果锁可重入，需要多次解锁操作)，此时队列不为空并且头节点状态不为0，
        // 需要唤醒后继节点:
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
// ReentrantLock
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
// AQS
// 找到队列中离head最近的一个节点(未被取消的)，unpark恢复运行，恢复运行后的执行逻辑可以看上述acquireQueued:
private void unparkSuccessor(Node node) {
    /*
        * If status is negative (i.e., possibly needing signal) try
        * to clear in anticipation of signalling.  It is OK if this
        * fails or if status is changed by waiting thread.
        */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
        * Thread to unpark is held in successor, which is normally
        * just the next node.  But if cancelled or apparently null,
        * traverse backwards from tail to find the actual
        * non-cancelled successor.
        */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

### 可重入

```java
static final class NonfairSync extends Sync {
    // ...

    protected final boolean tryAcquire(int acquires) {
        // 该方法直接从Sync继承过来:
        return nonfairTryAcquire(acquires);
    }
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    // ...

    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 获得锁成功:
        if (c == 0) {
            // 直接修改状态，不在乎等待队列，此处体现了非公平性:
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果锁已经被获得，并且当前线程就是获得线程，表示发生了锁重入:
        else if (current == getExclusiveOwnerThread()) {
            // state++
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
        // state--
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 支持锁重入，只有state减为0，才释放成功:
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
}
```

### 不可打断模式

```java
private final boolean parkAndCheckInterrupt() {
    // park在线程被调用interrupt后会失效:
    LockSupport.park(this);
    // interrupted()返回线程的中断标志位，但是调用后将清除中断标志位，
    // 由于后续线程在判断中断逻辑时需要使用isInterrupted()方法来获得中断标志位的状态，
    // 因此后续逻辑需要重新产生一次中断:
    return Thread.interrupted();
}

public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 如果线程获得锁之后返回true，将执行该方法:
        selfInterrupt();
}

static void selfInterrupt() {
    // 重新产生一次中断，设置中断标志位为true:
    Thread.currentThread().interrupt();
}
```

在不可打断模式下，即使线程中断标志位被设置为true，它仍会驻留在AQS队列中，**直到获得锁后才能继续运行去判断中断标志，进行中断响应**。

### 可打断模式

```java
// RL
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
// AQS
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
// AQS
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 如果阻塞过程中发生中断，则park失效，直接抛出异常:
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 公平锁

```java
static final class FairSync extends Sync {
    // ...

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 先检查AQS队列中是否有前驱节点，没有才会去竞争:
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
// AQS
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

### 条件变量之await

每个条件变量对应一个等待队列，其实现类是ConditionObject，它也有头尾指针，但没有占位节点。

某个线程在调用某个条件变量的await()方法前，发生了锁重入，那么在进入等待队列时，需要释放自己所持有的锁。

```java
// AQS
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将调用线程封装为一个节点添加到等待队列中:
    Node node = addConditionWaiter();
    // 因为当前线程可能发生锁重入，因此需要按照锁重入的次数将当前锁释放掉，并唤醒后继节点:
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 在条件变量等待队列中等待的线程节点状态为-2:
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}

final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

### 条件变量之signal

```java
// AQS
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 唤醒条件变量等待队列中的队首线程:
    Node first = firstWaiter;
    if (first != null)
        // 将队首线程转移到AQS队列的尾部:
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    // transferForSignal转移线程到AQS队列尾部，转移失败可能是发生超时，中断等情况，因此没必要加入到等待队列中:
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
    /*
        * If cannot change waitStatus, the node has been cancelled.
        */
    // 加入到AQS队列中的最后一个节点的状态为0:
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
        * Splice onto queue and try to set waitStatus of predecessor to
        * indicate that thread is (probably) waiting. If cancelled or
        * attempt to set waitStatus fails, wake up to resync (in which
        * case the waitStatus can be transiently and harmlessly wrong).
        */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

### 节点状态变化

节点刚被创建时，其waitStatus被设置为0，当节点位于等待队列中，如果是最后一个节点，其waitStatus仍然为0，但是位于最后一个节点之前的节点其waitStatus都被设置为-1，表示有责任唤醒后继节点，而在条件变量等待队列中等待的节点，其waitStatus都为-2。
