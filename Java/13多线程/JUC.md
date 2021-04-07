# 笔记

## 锁的特点

`ReentrantLock`和Synchronized的比较：

<img src="./image/synchronized和reentrantlock.png">

比起Synchronized，是否使用ReentrantLock取决于你是否需要使用到ReentrantLock所提供的特性，但如果说是在性能方面，自从JDK6之后底层对Synchronized的优化，性能已经不再是选择ReentrantLock的一个理由。

`ReentrantLock`保证了只有一个线程可以执行临界区代码，但有的时候需要这样的场景：允许多个线程读取数据，但只要有一个线程在写，其它线程就必须等待。

`ReadWriteLock`可以解决这样的场景：

- 只允许一个线程写入（其它线程既不能写入也不能读取）；
- 没有写入时，多个线程允许同时读（提高性能）。

深入分析`ReadWriteLock`，会发现它有个潜在的问题：如果有线程正在读，写线程需要等待读线程释放锁后才能获取写锁，即读的过程中不允许写，这是一种悲观的读锁。如果此时有大量的读线程，那么写线程就需要等待读线程都结束后才能写。

为了进一步提高并发效率，`StampedLock`的改进之处在于：读的过程中也允许获取写锁后写入！这样一来，我们读的数据就可能不一致，所以，需要一点额外的代码来判断读的过程中是否有写入，这种读锁是一种乐观锁。

**乐观锁的意思就是乐观地估计读的过程中大概率不会有写入，因此被称为乐观锁。反过来，悲观锁则是读的过程中拒绝有写入，也就是写入必须等待。显然乐观锁的并发效率更高，但一旦有小概率的写入导致读取的数据不一致，需要能检测出来，再读一遍就行**。

## AQS

借助UNSAFE类来实现CAS操作，借助LockSupport实现park和unpark操作。

AQS将独占获取资源的方法acquire流程给你实现好，并且不允许重写，在内部它会调用tryAcquire尝试获取资源，如果获取失败，则继续调用acquireQueued将当前线程封装为Node节点插入到阻塞队列的尾部，并调用LockSupport.park(this)挂起线程自己。

acquireQueued会再次尝试进行获取资源，直到真的无法获取资源时才选择阻塞。

其中tryAcquire需要具体的子类来实现，在其中使用CAS算法尝试修改state状态值，成功则返回true，否则返回false。

**因此AQS定义好获取资源成功和获取资源失败后的流程，而具体的子类需要自己定义什么时候获取资源算成功，什么时候获取资源算失败**。

## LockSupport

借助UNSAFE类来实现park和unpark方法。

## ReentrantLock

性能不是使用reentrantlock和synchronized的依据，而是前者所具备的一些特点成为你选择使用它的原因。

## UNSAFE内存屏障

[参考](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)

在JDK8中引入，用于定义内存屏障（也称内存栅栏，内存栅障，屏障指令等，是一类同步屏障指令，**是CPU或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作**），避免代码重排序。

```java
//内存屏障，禁止load操作重排序。屏障前的load操作不能被重排序到屏障后，屏障后的load操作不能被重排序到屏障前
public native void loadFence();
//内存屏障，禁止store操作重排序。屏障前的store操作不能被重排序到屏障后，屏障后的store操作不能被重排序到屏障前
public native void storeFence();
//内存屏障，禁止load、store操作重排序
public native void fullFence();
```

应用场景：

在JDK8中引入了一种锁的新机制——StampedLock，它可以看成是读写锁的一个改进版本。StampedLock提供了一种乐观读锁的实现，这种乐观读锁类似于无锁的操作，完全不会阻塞写线程获取写锁，从而缓解读多写少时写线程“饥饿”现象。由于StampedLock提供的乐观读锁不阻塞写线程获取读锁，当线程共享变量从主内存load到线程工作内存时，会存在数据不一致问题，所以当使用StampedLock的乐观读锁时，需要进行验证来确保数据的一致性。

```java
public class Point {
    private final StampedLock stampedLock = new StampedLock();

    private double x;
    private double y;

    public void move(double deltaX, double deltaY) {
        // 使用写锁修改变量值，独占操作:
        long stamp = stampedLock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            stampedLock.unlockWrite(stamp);
        }
    }

    public double distanceFromOrigin() {
        // 使用乐观读锁读取变量值:
        long stamp = stampedLock.tryOptimisticRead();
        // [copy变量到工作内存，在此处可能发生变量的store操作，因为乐观读锁不阻止写锁的发生]
        double currentX = x;
        double currentY = y;
        // [锁状态验证]，查看乐观读锁之后是否有写锁事件发生:
        if (!stampedLock.validate(stamp)) {
            // 使用悲观读锁重新读取变量值，在这过程中不允许有写锁发生:
            stamp = stampedLock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                stampedLock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

加乐观读锁，拷贝变量到工作内存，判断是否有写锁发生，有则加悲观读锁，再次拷贝最新变量到工作内存。

以下是validate方法的实现，通过锁标记与相关常量进行位运算、比较来校验锁状态，在校验逻辑之前，会通过Unsafe的loadFence方法加入一个load内存屏障：

```java
public boolean validate(long stamp) {
    VarHandle.acquireFence(); // UNSAFE.loadFence(); 禁止之前的load和store操作和之后的load和store操作不会跑到后面和前面去。
    return (stamp & SBITS) == (state & SBITS); // 验证逻辑
}
```

在验证逻辑之前加入内存屏障，防止`[copy变量到工作内存]`和`[锁状态验证]`发生重排序导致校验状态不准确的问题，如果发生重排序，漏判了store操作的发生，将导致逻辑不正确。
