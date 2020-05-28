# 多线程

## 多线程基础

- CPU执行代码都是一条一条顺序执行的，但是，即使是单核cpu，也可以同时运行多个任务。因为操作系统执行多任务实际上就是让CPU对多个任务轮流交替执行。即使是多核CPU，因为通常任务的数量远远多于CPU的核数，所以任务也是交替执行的。
- 在计算机中，我们把**一个任务称为一个进程**，浏览器就是一个进程，视频播放器是另一个进程，类似的，音乐播放器和Word都是进程。
- 某些**进程内部还需要同时执行多个子任务**。例如，我们在使用Word时，Word可以让我们一边打字，一边进行拼写检查，同时还可以在后台进行打印，我们**把子任务称为线程**。
- 进程和线程的关系就是：一个进程可以包含一个或多个线程，但至少会有一个线程。  
    ![多线程基础01](./image/多线程基础01.png)
- **操作系统调度的最小任务单位其实不是进程，而是线程**。常用的Windows、Linux等操作系统都采用抢占式多任务，**如何调度线程完全由操作系统决定，程序自己不能决定什么时候执行，以及执行多长时间**。
- 因为同一个应用程序，既可以有多个进程，也可以有多个线程，因此，实现多任务的方法，有以下几种：多进程模式（每个进程只有一个线程）；多线程模式（一个进程有多个线程）；多进程＋多线程模式（复杂度最高）。
- 进程和线程是包含关系，但是多任务既可以由多进程实现，也可以由单进程内的多线程实现，还可以混合多进程＋多线程。
- 和多线程相比，多进程的缺点在于：**创建进程比创建线程开销大**，尤其是在Windows系统上；进程间通信比线程间通信要慢，因为**线程间通信就是读写同一个变量**，速度很快。多进程的优点在于：**多进程稳定性比多线程高**，因为在多进程的情况下，**一个进程崩溃不会影响其他进程**，而在多线程的情况下，**任何一个线程崩溃会直接导致整个进程崩溃**。
- Java语言内置了多线程支持：**一个Java程序实际上是一个JVM进程，JVM进程用一个主线程来执行main()方法，在main()方法内部，我们又可以启动多个线程**。此外，JVM还有负责垃圾回收的其他工作线程等。
- 对于大多数Java程序来说，我们说多任务，实际上是说如何**使用多线程实现多任务**。
- 和单线程相比，多线程编程的特点在于：**多线程经常需要读写共享数据，并且需要同步**。例如，播放电影时，就必须由一个线程播放视频，另一个线程播放音频，两个线程需要协调运行，否则画面和声音就不同步。因此，多线程编程的复杂度高，调试更困难。
- Java多线程编程的特点又在于：多线程模型是Java程序最基本的并发模型；后续读写网络、数据库、Web开发等都依赖Java多线程模型。

## 创建新线程

```Java
public class CreateThreadTest {
    public static void main(String[] args) {
        Thread t = new MyThread();
        t.start();
    }
}
// 从Thread派生一个自定义类，然后覆写run()方法
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("start new thread!");
    }
}
```

```Java
public class CreateThreadTest {
    public static void main(String[] args) {
        // 实现了Runnable接口的子类
        Runnable r1 = new MyRunnable();
        // 有时候为只使用一次的子类单独创建一个.java文件过于浪费
        // 匿名内部类主要用于实现抽象类和接口，说白了就是一个子类，只不过它们new的对象是抽象类和接口
        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                System.out.println("[anonymous subclass] start new thread!");
            }
        };
        // 由于Runnable本身是一个函数式接口，因此我们可以进一步地使用lambda表达式来简化实例过程
        Runnable r3 = () -> System.out.println("[lambda] start new thread!");
        List<Runnable> list = new ArrayList<Runnable>();
        list.add(r1);
        list.add(r2);
        list.add(r3);
        for (Runnable r : list) {
            // 创建Thread实例时，传入一个Runnable实例
            new Thread(r).start();
        }
    }
}
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("[subclass] start new thread!");
    }
}
```

- 程序本身无法确定线程的调度顺序。
- 直接调用run()方法，相当于调用了一个普通的Java方法，当前线程并没有任何改变，也不会启动新线程。
- 必须调用Thread实例的start()方法才能启动新线程，如果我们查看Thread类的源代码，会看到start()方法内部调用了一个private native void start0()方法，**native修饰符**表示这个方法是由JVM虚拟机内部的C代码实现的，不是由Java代码实现的。
- 一个线程对象只能调用一次start()方法。
- 可以对线程设定优先级，优先级高的线程被操作系统调度的优先级较高，操作系统对高优先级线程可能调度更频繁，但我们决不能通过设置优先级来确保高优先级的线程一定会先执行。

## 线程的状态

- 在Java程序中，一个线程对象只能调用一次start()方法启动新线程，并在新线程中执行run()方法。一旦run()方法执行完毕，线程就结束了。因此，Java线程的状态有以下几种：`New`：新创建的线程，尚未执行；`Runnable`：运行中的线程，正在执行run()方法的Java代码；`Blocked`：运行中的线程，因为某些操作被阻塞而挂起；`Waiting`：运行中的线程，因为某些操作在等待中；`Timed Waiting`：运行中的线程，因为执行sleep()方法正在计时等待；`Terminated`：线程已终止，因为run()方法执行完毕。  
    ![线程的状态01](./image/线程的状态01.png)
- 线程终止的原因有：*线程正常终止*：run()方法执行到return语句返回；*线程意外终止*：run()方法因为未捕获的异常导致线程终止；对某个线程的Thread实例调用stop()方法强制终止（强烈不推荐使用）。
- 通过对另一个线程对象调用join()方法可以等待其执行结束；可以指定等待时间，超过等待时间线程仍然没有结束就不再等待；对已经运行结束的线程调用join()方法会立刻返回。

    ```Java
    Thread t = new Thread(() -> {
        System.out.println("hello");
    });
    System.out.println("start");
    t.start();
    // main线程等待t变量表示的线程运行结束，才继续往下执行自身线程
    t.join();
    System.out.println("end");
    ```
