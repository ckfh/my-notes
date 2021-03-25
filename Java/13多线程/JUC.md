# 笔记

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
