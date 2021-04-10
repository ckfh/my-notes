# 笔记

## 源码里的例子

```java
class CachedData {
    Object data;
    // 缓存是否有效，失效则需要重新计算data:
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

    void processCachedData() {
        // 读取数据前，加读锁，如果缓存没失效，则跳过判断，读取数据，释放读锁:
        rwl.readLock().lock();
        
        if (!cacheValid) {
            // 获取写锁前必须先释放读锁:
            rwl.readLock().unlock();
            rwl.writeLock().lock();
            try {
                // 判断是否有其它线程已经获取了写锁，更新了缓存，避免重复更新:
                if (!cacheValid) {
                    data = ...
                    cacheValid = true;
                }
                // 降级为读锁，释放写锁，能够让其它线程读取缓存:
                rwl.readLock().lock();
            } finally {
                rwl.writeLock().unlock();
            }
        }
        // 用完数据，释放读锁:
        try {
            use(data);
        } finally {
            rwl.readLock().unlock();
        }
    }
}
```

## 读写锁原理之写锁加锁

```java
// ReentrantReadWriteLock
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 返回state中独占模式的持有数，其实就是获取写锁的个数:
    int w = exclusiveCount(c);
    // state不为零既有可能是已经加了读锁，也有可能是加了写锁:
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // w == 0 表示加的是读锁，那么读-写互斥，直接返回false，
        // 如果 w != 0 表示加的是写锁，那就查看是不是线程自己加的写锁，如果不是同样false:
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 如果写锁部分加1超过最大范围，因为总共只有16位:
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    // 如果是非公平锁，writerShouldBlock总是返回false，公平锁则会对队列节点进行检查:
    if (writerShouldBlock() ||
        // 如果修改失败，则直接返回false:
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

## 读写锁康之读锁加锁

```java
// ReentrantReadWriteLock
public void lock() {
    sync.acquireShared(1);
}
// AQS
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
// ReentrantReadWriteLock
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 如果已经加了写锁，那就再判断是不是当前线程自己加的写锁，此时允许加读锁，如果不是当前线程加的写锁，那就返回-1:
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 获取读锁:
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 以下判断都是为读锁进行计数操作:
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        // 如果获取读锁成功，则返回1:
        return 1;
    }
    return fullTryAcquireShared(current);
}
// AQS
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // “老二”线程允许再次尝试获取一次读锁:
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 获取读锁成功，将前驱节点释放，将当前节点设为头节点:
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
// AQS
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    // 获取读锁成功，将前驱节点释放，将当前节点设为头节点:
    setHead(node);
    // 如果后续节点是SHARED状态，则尝试唤醒后继节点:
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
// AQS
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

## 读写锁之写锁解锁

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

## 读写锁之读锁解锁

```java
// AQS
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        // 如果读锁都释放完毕，则尝试唤醒后继的写锁线程:
        doReleaseShared();
        return true;
    }
    return false;
}
// RRWL
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    // 读锁计数:
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
// AQS
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
