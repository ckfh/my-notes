# 笔记

## acquire

竞争成功，修改state值，竞争失败，进入等待队列等待，节点状态为SHARED，共享锁类型，因此一个线程被唤醒，之后的等待线程都会被唤醒:

```java
// S
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
// AQS
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 线程竞争成功，tryAcquireShared返回非负数，竞争失败，tryAcquireShared返回负数，但state依旧为0，执行doAcquireSharedInterruptibly方法:
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
// S
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
// S
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
// AQS
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

## release

```java
// S
public void release() {
    sync.releaseShared(1);
}
// AQS
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
// S
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        // 释放时修改state值:
        if (compareAndSetState(current, next))
            return true;
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
