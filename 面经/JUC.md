## ReentrantLock

### 加锁（非公平）

- 借助 ReentrantLock.NonfairSync 的 lock 方法，通过 CAS 尝试将 state 变量从 0 改为 1，如果成功，则将当前线程设为独占线程，失败则进入到 AQS 的 acquire 方法；
- acquire 方法先执行 ReentrantLock.NonfairSync 的 tryAcquire 方法尝试加锁，该方法逻辑为先判断当前 state 变量是否为 0，如果为 0，则通过 CAS 尝试加锁，如果不为 0，则判断当前线程是否就为独占线程，如果是独占线程，则对 state 变量进行加操作，表示**可重入**，如果前面条件都不满足，则进入到 AQS 的 addWaiter 方法；
- addWaiter 方法将当前线程封装为一个 Node 对象，其 waitStatus 状态初始时为 0，并放到虚拟双向链表的尾部，所谓“虚拟”指的是 AQS 自身并没有链表属性，而是通过 head 和 tail 这两个指针指向头尾节点，从而持有了一个双向链表，随后进入到 AQS 的 acquireQueued 方法；
- acquireQueued 方法内部是一个死循环，死循环的目的是为了不断尝试去获取锁：
  - 循环一开始先判断当前节点的前驱节点是否为哨兵节点，如果是，则可以尝试进行加锁，加锁成功则将当前节点设置为哨兵节点，然后将原来的哨兵节点断开；
  - 前驱节点是哨兵节点，但加锁失败，或者前驱节点不是哨兵节点，则进入到 AQS 的 shouldParkAfterFailedAcquire 方法，该方法会将前驱节点的 waitStatus 状态修改为 -1，-1 表示此节点有责任唤醒它的后继节点，修改之后返回 false，将跳过后续的阻塞方法，再度回到循环开头，如果前面的判断仍失败，则再次进入该方法，此时因为前驱节点已经为 -1，那么此时将直接返回 true，进入后续 AQS 的 parkAndCheckInterrupt 方法；
  - parkAndCheckInterrupt 方法将调用 LockSupport.park 方法将当前线程阻塞。

### 解锁（非公平）

- 借助 ReentrantLock 的 unlock 方法，直接调用 AQS 的 release 方法；
- release 方法先执行 ReentrantLock.Sync 的 tryRelease 方法尝试释放锁（注意公平和非公平解锁流程一致，设计者选择将该流程放到了两者的父类 Sync 中），该方法对 state 变量进行减操作，如果 state 变量减为 0，则将独占线程设为 null，并返回 true，返回 true 将判断此时链表头节点是否不为空，并且头节点状态不为 0，如果条件成立，表示后续有节点等待被唤醒，将执行 AQS 的 unparkSuccessor 方法；
- unparkSuccessor 方法找到最近一个状态为**非取消状态**的节点，调用 LockSupport.unpark 唤醒该节点所封装的线程对象，恢复运行，也就是让线程对象从 parkAndCheckInterrupt 方法阻塞处继续向下运行，将回到 acquireQueued 的循环开头处；
- 由于是**非公平锁**实现，当被唤醒的线程对象尝试获取锁时，此时一个未加入到链表的线程抢先获得了锁，那么当前被唤醒的线程对象只能执行后续的 shouldParkAfterFailedAcquire 方法和 parkAndCheckInterrupt 方法。

### 可重入体现

- 加锁时如果是当前线程再次加锁，则对 state 变量进行加操作；
- 解锁时只有在 state 变量被减为 0 时，才会尝试唤醒等待线程。

### 可打断原理

- 因为采用 LockSupport.park 的方式来阻塞线程，当其它线程通过 Thread.interrupt 方法来打断当前线程时，将恢复线程的运行，而 parkAndCheckInterrupt 方法的返回值是 Thread.interrupted 方法的返回值，它将返回当前线程是否被打断，**并且会清除中断标记**，如果不清除打断标记，当下次再调用 park 方法时，由于打断标记的存在，将使得线程无法被阻塞；

#### 不可打断模式

- 当一个线程对象是被打断而恢复运行时，会将 acquireQueued 方法内的一个局部变量 interrupted 改为 true，该局部变量会在线程成功获取锁时返回；
- 从 acquireQueued 方法返回时，如果返回为 true，表示线程之前被打断过，由于之前在 parkAndCheckInterrupt 方法内部清除过中断标记，此时将调用 selfInterrupt 方法将当前线程对象的中断标记设为 true，确保逻辑的正确性；
- **综上可以看出，在不可打断模式下，仅是将一个中断标记置为 true，线程对象本身仍会驻留在 AQS 链表中，直到获取锁，才能去判断中断标记，进行中断响应**。

#### 可打断模式

- **在可打断模式下，如果线程被打断，将直接抛出 InterruptedException 异常，终止线程运行，不会回到循环开始处**。

### 加锁（公平）

- 借助 ReentrantLock.FairSync 的 lock 方法，它将直接调用 AQS 的 acquire 方法，从而执行 ReentrantLock.FairSync 的 tryAcquire 方法；
- tryAcquire 方法内部，和非公平锁不一样的地方在于，尝试进行 CAS 操作前，**会判断链表中是否有前驱节点**，没有才会进行 CAS 操作；

### 解锁（公平）

- 同非公平解锁实现。

### 条件变量

