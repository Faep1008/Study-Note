# 线程快照分析
## 第一部分：Full thread dump identifier
这部分内容是最开始的部分，展示快照的生成时间及JVM的版本信息。
```
2020-07-02 08:58:16
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.60-b23 mixed mode):
```

## 第二部分：Java EE middleware, third party & custom application Threads
这是整个文件的核心部分，里面展示了JavaEE容器（如tomcat）、自己的程序中所使用的线程信息。

### Thread Dump日志的线程信息
以下面的日志为例：
```
"resin-22129" daemon prio=10 tid=0x00007fbe5c34e000 nid=0x4cb1 waiting on condition [0x00007fbe4ff7c000]
   java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:315)
    at com.caucho.env.thread2.ResinThread2.park(ResinThread2.java:196)
    at com.caucho.env.thread2.ResinThread2.runTasks(ResinThread2.java:147)
    at com.caucho.env.thread2.ResinThread2.run(ResinThread2.java:118)

"Timer-20" daemon prio=10 tid=0x00007fe3a4bfb800 nid=0x1a31 in Object.wait() [0x00007fe3a077a000]
   java.lang.Thread.State: TIMED_WAITING (on object monitor)
    at java.lang.Object.wait(Native Method)
    - waiting on <0x00000006f0620ff0> (a java.util.TaskQueue)
    at java.util.TimerThread.mainLoop(Timer.java:552)
    - locked <0x00000006f0620ff0> (a java.util.TaskQueue)
    at java.util.TimerThread.run(Timer.java:505)
```
以上依次是：
- `"resin-22129"` 线程名称：如果使用 java.lang.Thread 类生成一个线程的时候，线程名称为 Thread-(数字) 的形式，这里是resin生成的线程；
- `daemon` 线程类型：线程分为守护线程 (daemon) 和非守护线程 (non-daemon) 两种，通常都是守护线程；
- `prio=10` 线程优先级：默认为5，数字越大优先级越高；
- `tid=0x00007fbe5c34e000` JVM线程的id：JVM内部线程的唯一标识，通过 java.lang.Thread.getId()获取，通常用自增的方式实现；
- `nid=0x4cb1` 系统线程id：对应的系统线程id（Native Thread ID)，可以通过 top 命令进行查看，线程id是十六进制的形式；
- `waiting on condition` 系统线程状态：这里是系统的线程状态，具体的含义见下面 系统线程状态 部分；
- `[0x00007fbe4ff7c000]` 起始栈地址：线程堆栈调用的内存地址；
- `java.lang.Thread.State: WAITING (parking)` JVM线程状态：这里标明了线程在代码级别的状态，详细的内容见下面的 JVM线程运行状态 部分。
- 线程调用栈信息：下面就是当前线程调用的详细栈信息，用于代码的分析。堆栈信息应该从下向上解读，因为程序调用的顺序是从下向上的。
 
### 系统线程状态 (Native Thread Status)

#### deadlock
死锁线程，一般指多个线程调用期间进入了相互资源占用，导致一直等待无法释放的情况。

#### runnable
一般指该线程正在执行状态中，该线程占用了资源，正在处理某个操作，如通过SQL语句查询数据库、对某个文件进行写入等。

#### blocked
线程正处于阻塞状态，指当前线程执行过程中，所需要的资源长时间等待却一直未能获取到，被容器的线程管理器标识为阻塞状态，可以理解为等待资源超时的线程。

#### waiting on condition
线程正处于等待资源或等待某个条件的发生，具体的原因需要结合下面堆栈信息进行分析。  
  
（1）如果堆栈信息明确是应用代码，则证明该线程正在等待资源，一般是大量读取某种资源且该资源采用了资源锁的情况下，线程进入等待状态，等待资源的读取，或者正在等待其他线程的执行等。  
  
（2）如果发现有大量的线程都正处于这种状态，并且堆栈信息中得知正等待网络读写，这是因为网络阻塞导致线程无法执行，很有可能是一个网络瓶颈的征兆：  
  
- 网络非常繁忙，几乎消耗了所有的带宽，仍然有大量数据等待网络读写；
- 网络可能是空闲的，但由于路由或防火墙等原因，导致包无法正常到达；  

所以一定要结合系统的一些性能观察工具进行综合分析，比如netstat统计单位时间的发送包的数量，看是否很明显超过了所在网络带宽的限制；观察CPU的利用率，看系统态的CPU时间是否明显大于用户态的CPU时间。这些都指向由于网络带宽所限导致的网络瓶颈。  
  
（3）还有一种常见的情况是该线程在 sleep，等待 sleep 的时间到了，将被唤醒。  

#### waiting for monitor entry 或 in Object.wait()
Moniter 是Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者class的锁，每个对象都有，也仅有一个 Monitor。  
每个Monitor在某个时刻只能被一个线程拥有，该线程就是 "Active Thread"，而其他线程都是 "Waiting Thread"，分别在两个队列 "Entry Set"和"Waint Set"里面等待。其中在 "Entry Set" 中等待的线程状态是 waiting for monitor entry，在 "Wait Set" 中等待的线程状态是 in Object.wait()。  
（1）"Entry Set"里面的线程。  
我们称被 synchronized 保护起来的代码段为临界区，对应的代码如下：
```java
synchronized(obj) {

}
```
当一个线程申请进入临界区时，它就进入了 "Entry Set" 队列中，这时候有两种可能性：  
- 该Monitor不被其他线程拥有，"Entry Set"里面也没有其他等待的线程。本线程即成为相应类或者对象的Monitor的Owner，执行临界区里面的代码；此时在Thread Dump中显示线程处于 "Runnable" 状态。
- 该Monitor被其他线程拥有，本线程在 "Entry Set" 队列中等待。此时在Thread Dump中显示线程处于 "waiting for monity entry" 状态。  

临界区的设置是为了保证其内部的代码执行的原子性和完整性，但因为临界区在任何时间只允许线程串行通过，这和我们使用多线程的初衷是相反的。如果在多线程程序中大量使用synchronized，或者不适当的使用它，会造成大量线程在临界区的入口等待，造成系统的性能大幅下降。如果在Thread Dump中发现这个情况，应该审视源码并对其进行改进。  
  
（2）"Wait Set"里面的线程  
当线程获得了Monitor，进入了临界区之后，如果发现线程继续运行的条件没有满足，它则调用对象（通常是被synchronized的对象）的wait()方法，放弃Monitor，进入 "Wait Set"队列。只有当别的线程在该对象上调用了 notify()或者notifyAll()方法，"Wait Set"队列中的线程才得到机会去竞争，但是只有一个线程获得对象的Monitor，恢复到运行态。"Wait Set"中的线程在Thread Dump中显示的状态为 in Object.wait()。通常来说，当CPU很忙的时候关注 Runnable 状态的线程，反之则关注 waiting for monitor entry 状态的线程。  
  
### JVM线程运行状态 (JVM Thread Status)
在 java.lang.Thread.State 中定义了线程的状态：  
#### NEW
至今尚未启动的线程的状态。线程刚被创建，但尚未启动。

#### RUNNABLE
可运行线程的线程状态。线程正在JVM中执行，有可能在等待操作系统中的其他资源，比如处理器。

#### BLOCKED
受阻塞并且正在等待监视器的某一线程的线程状态。处于受阻塞状态的某一线程正在等待监视器锁，以便进入一个同步的块/方法，或者在调用 Object.wait 之后再次进入同步的块/方法。
在Thread Dump日志中通常显示为 java.lang.Thread.State: BLOCKED (on object monitor) 。

#### WAITING
某一等待线程的线程状态。线程正在无期限地等待另一个线程来执行某一个特定的操作，线程因为调用下面的方法之一而处于等待状态：
- 不带超时的 Object.wait 方法，日志中显示为 java.lang.Thread.State: WAITING (on object monitor)
- 不带超时的 Thread.join 方法
LockSupport.park 方法，日志中显示为 java.lang.Thread.State: WAITING (parking)  

#### TIMED_WAITING
指定了等待时间的某一等待线程的线程状态。线程正在等待另一个线程来执行某一个特定的操作，并设定了指定等待的时间，线程因为调用下面的方法之一而处于定时等待状态：
- Thread.sleep 方法
- 指定超时值的 Object.wait 方法
- 指定超时值的 Thread.join 方法
- LockSupport.parkNanos
- LockSupport.parkUntil  

#### TERMINATED
线程处于终止状态。  
根据Java Doc中的说明，在给定的时间上，一个JVM线程只能处于上述的一种状态之中，并且这些状态都是JVM的状态，跟操作系统中的线程状态无关。


### 线程状态样例
#### 等待状态样例

```
"IoWaitThread" prio=6 tid=0x0000000007334800 nid=0x2b3c waiting on condition [0x000000000893f000]
   java.lang.Thread.State: WAITING (parking)
                at sun.misc.Unsafe.park(Native Method)
                - parking to wait for  <0x00000007d5c45850> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
                at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
                at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:1987)
                at java.util.concurrent.LinkedBlockingDeque.takeFirst(LinkedBlockingDeque.java:440)
                at java.util.concurrent.LinkedBlockingDeque.take(LinkedBlockingDeque.java:629)
                at com.nbp.theplatform.threaddump.ThreadIoWaitState$IoWaitHandler2.run(ThreadIoWaitState.java:89)
                at java.lang.Thread.run(Thread.java:662) 
```
上面例子中，IoWaitThread 线程保持等待状态并从 LinkedBlockingQueue 接收消息，如果 LinkedBlockingQueue 一直没有消息，该线程的状态将不会改变。

#### 阻塞状态样例

```
"BLOCKED_TEST pool-1-thread-1" prio=6 tid=0x0000000006904800 nid=0x28f4 runnable [0x000000000785f000]
   java.lang.Thread.State: RUNNABLE
                at java.io.FileOutputStream.writeBytes(Native Method)
                at java.io.FileOutputStream.write(FileOutputStream.java:282)
                at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:65)
                at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:123)
                - locked <0x0000000780a31778> (a java.io.BufferedOutputStream)
                at java.io.PrintStream.write(PrintStream.java:432)
                - locked <0x0000000780a04118> (a java.io.PrintStream)
                at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:202)
                at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:272)
                at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:85)
                - locked <0x0000000780a040c0> (a java.io.OutputStreamWriter)
                at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:168)
                at java.io.PrintStream.newLine(PrintStream.java:496)
                - locked <0x0000000780a04118> (a java.io.PrintStream)
                at java.io.PrintStream.println(PrintStream.java:687)
                - locked <0x0000000780a04118> (a java.io.PrintStream)
                at com.nbp.theplatform.threaddump.ThreadBlockedState.monitorLock(ThreadBlockedState.java:44)
                - locked <0x0000000780a000b0> (a com.nbp.theplatform.threaddump.ThreadBlockedState)
                at com.nbp.theplatform.threaddump.ThreadBlockedState$1.run(ThreadBlockedState.java:7)
                at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
                at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
                at java.lang.Thread.run(Thread.java:662)
   Locked ownable synchronizers:
                - <0x0000000780a31758> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
"BLOCKED_TEST pool-1-thread-2" prio=6 tid=0x0000000007673800 nid=0x260c waiting for monitor entry [0x0000000008abf000]
   java.lang.Thread.State: BLOCKED (on object monitor)
                at com.nbp.theplatform.threaddump.ThreadBlockedState.monitorLock(ThreadBlockedState.java:43)
                - waiting to lock <0x0000000780a000b0> (a com.nbp.theplatform.threaddump.ThreadBlockedState)
                at com.nbp.theplatform.threaddump.ThreadBlockedState$2.run(ThreadBlockedState.java:26)
                at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
                at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
                at java.lang.Thread.run(Thread.java:662)
   Locked ownable synchronizers:
                - <0x0000000780b0c6a0> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
"BLOCKED_TEST pool-1-thread-3" prio=6 tid=0x00000000074f5800 nid=0x1994 waiting for monitor entry [0x0000000008bbf000]
   java.lang.Thread.State: BLOCKED (on object monitor)
                at com.nbp.theplatform.threaddump.ThreadBlockedState.monitorLock(ThreadBlockedState.java:42)
                - waiting to lock <0x0000000780a000b0> (a com.nbp.theplatform.threaddump.ThreadBlockedState)
                at com.nbp.theplatform.threaddump.ThreadBlockedState$3.run(ThreadBlockedState.java:34)
                at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886
                at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
                at java.lang.Thread.run(Thread.java:662)
   Locked ownable synchronizers:
                - <0x0000000780b0e1b8> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
```
在上面的例子中，BLOCKED_TEST pool-1-thread-1 线程占用了 <0x0000000780a000b0> 锁，然而 BLOCKED_TEST pool-1-thread-2 和 BLOCKED_TEST pool-1-thread-3 threads 正在等待获取锁。

#### 死锁状态样例
```
"DEADLOCK_TEST-1" daemon prio=6 tid=0x000000000690f800 nid=0x1820 waiting for monitor entry [0x000000000805f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.goMonitorDeadlock(ThreadDeadLockState.java:197)
                - waiting to lock <0x00000007d58f5e60> (a com.nbp.theplatform.threaddump.ThreadDeadLockState$Monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.monitorOurLock(ThreadDeadLockState.java:182)
                - locked <0x00000007d58f5e48> (a com.nbp.theplatform.threaddump.ThreadDeadLockState$Monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.run(ThreadDeadLockState.java:135)

   Locked ownable synchronizers:
                - None

"DEADLOCK_TEST-2" daemon prio=6 tid=0x0000000006858800 nid=0x17b8 waiting for monitor entry [0x000000000815f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.goMonitorDeadlock(ThreadDeadLockState.java:197)
                - waiting to lock <0x00000007d58f5e78> (a com.nbp.theplatform.threaddump.ThreadDeadLockState$Monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.monitorOurLock(ThreadDeadLockState.java:182)
                - locked <0x00000007d58f5e60> (a com.nbp.theplatform.threaddump.ThreadDeadLockState$Monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.run(ThreadDeadLockState.java:135)

   Locked ownable synchronizers:
                - None

"DEADLOCK_TEST-3" daemon prio=6 tid=0x0000000006859000 nid=0x25dc waiting for monitor entry [0x000000000825f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.goMonitorDeadlock(ThreadDeadLockState.java:197)
                - waiting to lock <0x00000007d58f5e48> (a com.nbp.theplatform.threaddump.ThreadDeadLockState$Monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.monitorOurLock(ThreadDeadLockState.java:182)
                - locked <0x00000007d58f5e78> (a com.nbp.theplatform.threaddump.ThreadDeadLockState$Monitor)
                at com.nbp.theplatform.threaddump.ThreadDeadLockState$DeadlockThread.run(ThreadDeadLockState.java:135)

   Locked ownable synchronizers:
                - None
```
上面的例子中，当线程 A 需要获取线程 B 的锁来继续它的任务，然而线程 B 也需要获取线程 A 的锁来继续它的任务的时候发生的。在 thread dump 中，你能看到 DEADLOCK_TEST-1 线程持有 0x00000007d58f5e48 锁，并且尝试获取 0x00000007d58f5e60 锁。你也能看到 DEADLOCK_TEST-2 线程持有 0x00000007d58f5e60，并且尝试获取 0x00000007d58f5e78，同时 DEADLOCK_TEST-3 线程持有 0x00000007d58f5e78，并且在尝试获取 0x00000007d58f5e48 锁，如你所见，每个线程都在等待获取另外一个线程的锁，这状态将不会被改变直到一个线程丢弃了它的锁。

#### 无限等待的Runnable状态样例

```
"socketReadThread" prio=6 tid=0x0000000006a0d800 nid=0x1b40 runnable [0x00000000089ef000]
   java.lang.Thread.State: RUNNABLE
                at java.net.SocketInputStream.socketRead0(Native Method)
                at java.net.SocketInputStream.read(SocketInputStream.java:129)
                at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:264)
                at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:306)
                at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:158)
                - locked <0x00000007d78a2230> (a java.io.InputStreamReader)
                at sun.nio.cs.StreamDecoder.read0(StreamDecoder.java:107)
                - locked <0x00000007d78a2230> (a java.io.InputStreamReader)
                at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:93)
                at java.io.InputStreamReader.read(InputStreamReader.java:151)
                at com.nbp.theplatform.threaddump.ThreadSocketReadState$1.run(ThreadSocketReadState.java:27)
                at java.lang.Thread.run(Thread.java:662)
```

### 常见的Thread Dump日志案例分析
#### CPU占用率很高，响应很慢
先找到占用CPU的进程，然后再定位到对应的线程，使用jstack定位线程堆栈信息，最后分析出对应的堆栈信息。  
在同一时间多次使用上述的方法，然后进行对比分析，从代码中找到问题所在的原因。如果线程指向的是"VM Thread"或者无法从代码中直接找到原因，就需要进行内存分析
#### CPU占用率不高，但响应很慢
在整个请求的过程中多次执行Thread Dump然后进行对比，取得 BLOCKED 状态的线程列表，通常是因为线程停在了I/O、数据库连接或网络连接的地方。
#### 系统线程状态为 deadlock
线程处于死锁状态，将占用系统大量资源。
#### 系统线程状态为 waiting for monitor entry 或 in Object.wait()
如上所说，系统线程处于这种状态说明它在等待进入一个临界区，此时JVM线程的状态通常都是 java.lang.Thread.State: BLOCKED。  

如果大量线程处于这种状态的话，可能是一个全局锁阻塞了大量线程。如果短期内多次打印Thread Dump信息，发现 waiting for monitor entry 状态的线程越来越多，没有减少的趋势，可能意味着某些线程在临界区里呆得时间太长了，以至于越来越多新线程迟迟无法进入。
#### 系统线程状态为 waiting on condition
系统线程处于此种状态说明它在等待另一个条件的发生来唤醒自己，或者自己调用了sleep()方法。此时JVM线程的状态通常是java.lang.Thread.State: WAITING (parking)（等待唤醒条件）或java.lang.Thread.State: TIMED_WAITING (parking或sleeping)（等待定时唤醒条件）。  

如果大量线程处于此种状态，说明这些线程又去获取第三方资源了，比如第三方的网络资源或读取数据库的操作，长时间无法获得响应，导致大量线程进入等待状态。因此，这说明系统处于一个网络瓶颈或读取数据库操作时间太长。  
#### 系统线程状态为 blocked
线程处于阻塞状态，需要根据实际情况进行判断。

## 第三部分：HotSpot VM Thread
这一部分展示了JVM内部线程的信息，用于执行内部的原生操作。下面常见的集中内置线程：
### "Attach Listener"
该线程负责接收外部命令，执行该命令并把结果返回给调用者，此种类型的线程通常在桌面程序中出现。

```
"Attach Listener" daemon prio=5 tid=0x00007fc6b6800800 nid=0x3b07 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
### "DestroyJavaVM"
执行main()的线程在执行完之后调用JNI中的 jni_DestroyJavaVM() 方法会唤起DestroyJavaVM 线程。在JBoss启动之后，也会唤起DestroyJavaVM线程，处于等待状态，等待其它线程（java线程和native线程）退出时通知它卸载JVM。
```
"DestroyJavaVM" prio=5 tid=0x00007fc6b3001000 nid=0x1903 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
### "Service Thread"
用于启动服务的线程
```
"Service Thread" daemon prio=10 tid=0x00007fbea81b3000 nid=0x5f2 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
### "CompilerThread"
用来调用JITing，实时编译装卸CLASS。通常JVM会启动多个线程来处理这部分工作，线程名称后面的数字也会累加，比如CompilerThread1。
```
"C1 CompilerThread3" #53 daemon prio=9 os_prio=2 tid=0x000000001cd6b800 nid=0x37b8 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread2" #52 daemon prio=9 os_prio=2 tid=0x000000001cd5f800 nid=0xc8c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" #51 daemon prio=9 os_prio=2 tid=0x000000001cd5d000 nid=0x25a4 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #50 daemon prio=9 os_prio=2 tid=0x000000001cd5c800 nid=0x351c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
### "Signal Dispatcher"
Attach Listener线程的职责是接收外部jvm命令，当命令接收成功后，会交给signal dispather 线程去进行分发到各个不同的模块处理命令，并且返回处理结果。
signal dispather线程也是在第一次接收外部jvm命令时，进行初始化工作。
```
"Signal Dispatcher" #4 daemon prio=9 os_prio=2 tid=0x0000000019208800 nid=0x3330 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
### "Finalizer"
这个线程也是在main线程之后创建的，其优先级为10，主要用于在垃圾收集前，调用对象的finalize()方法；关于Finalizer线程的几点：
- （1）只有当开始一轮垃圾收集时，才会开始调用finalize()方法；因此并不是所有对象的finalize()方法都会被执行；
- （2）该线程也是daemon线程，因此如果虚拟机中没有其他非daemon线程，不管该线程有没有执行完finalize()方法，JVM也会退出；
- （3）JVM在垃圾收集时会将失去引用的对象包装成Finalizer对象（Reference的实现），并放入ReferenceQueue，由Finalizer线程来处理；最后将该Finalizer对象的引用置为null，由垃圾收集器来回收；
- （4）JVM为什么要单独用一个线程来执行finalize()方法呢？
如果JVM的垃圾收集线程自己来做，很有可能由于在finalize()方法中误操作导致GC线程停止或不可控，这对GC线程来说是一种灾难。
```
"Finalizer" #3 daemon prio=8 os_prio=1 tid=0x0000000017e05800 nid=0x3458 in Object.wait() [0x000000001a1ff000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x00000000838bca08> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)
```
### "Reference Handler"
JVM在创建main线程后就创建Reference Handler线程，其优先级最高，为10，它主要用于处理引用对象本身（软引用、弱引用、虚引用）的垃圾回收问题 。

```
"Reference Handler" #2 daemon prio=10 os_prio=2 tid=0x0000000017e02800 nid=0x2ac in Object.wait() [0x000000001a0ff000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:157)
	- locked <0x0000000083874160> (a java.lang.ref.Reference$Lock)
```
### "VM Thread"
JVM中线程的母体，根据HotSpot源码中关于vmThread.hpp里面的注释，它是一个单例的对象（最原始的线程）会产生或触发所有其他的线程，这个单例的VM线程是会被其他线程所使用来做一些VM操作（如清扫垃圾等）。  
在 VM Thread 的结构体里有一个VMOperationQueue列队，所有的VM线程操作(vm_operation)都会被保存到这个列队当中，VMThread 本身就是一个线程，它的线程负责执行一个自轮询的loop函数(具体可以参考：VMThread.cpp里面的void VMThread::loop()) ，该loop函数从VMOperationQueue列队中按照优先级取出当前需要执行的操作对象(VM_Operation)，并且调用VM_Operation->evaluate函数去执行该操作类型本身的业务逻辑。
VM操作类型被定义在vm_operations.hpp文件内，列举几个：ThreadStop、ThreadDump、PrintThreads、GenCollectFull、GenCollectFullConcurrent、CMS_Initial_Mark、CMS_Final_Remark…..

```
"VM Thread" os_prio=2 tid=0x0000000017e00000 nid=0x238c runnable 
```

## 第四部分：HotSpot GC Thread
JVM中用于进行资源回收的线程，包括以下几种类型的线程：
### "VM Periodic Task Thread"
该线程是JVM周期性任务调度的线程，它由WatcherThread创建，是一个单例对象。该线程在JVM内使用得比较频繁，比如：定期的内存监控、JVM运行状况监控。

```
"VM Periodic Task Thread" os_prio=2 tid=0x000000001d3db800 nid=0x39c waiting on condition 
```
可以使用jstat 命令查看GC的情况，比如查看某个进程没有存活必要的引用可以使用命令 jstat -gcutil <pid> 250 7 参数中pid是进程id，后面的250和7表示每250毫秒打印一次，总共打印7次。
这对于防止因为应用代码中直接使用native库或者第三方的一些监控工具的内存泄漏有非常大的帮助。
### "GC task thread#0 (ParallelGC)"
垃圾回收线程，该线程会负责进行垃圾回收。通常JVM会启动多个线程来处理这个工作，线程名称中#后面的数字也会累加。
```
"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x0000000002e4c000 nid=0x2198 runnable 

"GC task thread#1 (ParallelGC)" os_prio=0 tid=0x0000000002e4d800 nid=0x1430 runnable 

"GC task thread#2 (ParallelGC)" os_prio=0 tid=0x0000000002e4f000 nid=0x20d0 runnable 

"GC task thread#3 (ParallelGC)" os_prio=0 tid=0x0000000002e50800 nid=0x3690 runnable 

"GC task thread#4 (ParallelGC)" os_prio=0 tid=0x0000000002e54000 nid=0x3f8c runnable 

"GC task thread#5 (ParallelGC)" os_prio=0 tid=0x0000000002e55000 nid=0x938 runnable 

"GC task thread#6 (ParallelGC)" os_prio=0 tid=0x0000000002e58000 nid=0x818 runnable 

"GC task thread#7 (ParallelGC)" os_prio=0 tid=0x0000000002e59800 nid=0x3d7c runnable 
```

## 第五部分：JNI global references count
这一部分主要回收那些在native代码上被引用，但在java代码中却没有存活必要的引用，对于防止因为应用代码中直接使用native库或第三方的一些监控工具的内存泄漏有非常大的帮助。

```
JNI global references: 36773
```

## 案例分析
### waiting for monitor entry 和 java.lang.Thread.State: BLOCKED
```
"DB-Processor-13" daemon prio=5 tid=0x003edf98 nid=0xca waiting for monitor entry [0x000000000825f000]
java.lang.Thread.State: BLOCKED (on object monitor)
                at beans.ConnectionPool.getConnection(ConnectionPool.java:102)
                - waiting to lock <0xe0375410> (a beans.ConnectionPool)
                at beans.cus.ServiceCnt.getTodayCount(ServiceCnt.java:111)
                at beans.cus.ServiceCnt.insertCount(ServiceCnt.java:43)

"DB-Processor-14" daemon prio=5 tid=0x003edf98 nid=0xca waiting for monitor entry [0x000000000825f020]
java.lang.Thread.State: BLOCKED (on object monitor)
                at beans.ConnectionPool.getConnection(ConnectionPool.java:102)
                - waiting to lock <0xe0375410> (a beans.ConnectionPool)
                at beans.cus.ServiceCnt.getTodayCount(ServiceCnt.java:111)
                at beans.cus.ServiceCnt.insertCount(ServiceCnt.java:43)

"DB-Processor-3" daemon prio=5 tid=0x00928248 nid=0x8b waiting for monitor entry [0x000000000825d080]
java.lang.Thread.State: RUNNABLE
                at oracle.jdbc.driver.OracleConnection.isClosed(OracleConnection.java:570)
                - waiting to lock <0xe03ba2e0> (a oracle.jdbc.driver.OracleConnection)
                at beans.ConnectionPool.getConnection(ConnectionPool.java:112)
                - locked <0xe0386580> (a java.util.Vector)
                - locked <0xe0375410> (a beans.ConnectionPool)
                at beans.cus.Cue_1700c.GetNationList(Cue_1700c.java:66)
                at org.apache.jsp.cue_1700c_jsp._jspService(cue_1700c_jsp.java:120)
```
上面系统线程的状态是 waiting for monitor entry，说明此线程通过 synchronized(obj) { } 申请进入临界区，但obj对应的 Monitor 被其他线程所拥有，所以 JVM线程的状态是 java.lang.Thread.State: BLOCKED (on object monitor)，说明线程等待资源超时。  
  
下面的 waiting to lock <0xe0375410> 说明线程在等待给 0xe0375410 这个地址上锁（trying to obtain 0xe0375410 lock），如果在日志中发现有大量的线程都在等待给 0xe0375410 上锁的话，这个时候需要在日志中查找那个线程获取了这个锁 locked <0xe0375410>，如上面的例子中是 "DB-Processor-14" 这个线程，这样就可以顺藤摸瓜了。上面的例子是因为获取数据库操作等待的时间太长所致的，这个时候就需要修改数据库连接的配置信息。  
  
如果两个线程相互都被对方的线程锁锁住，这样就造成了 死锁 现象。  
  
### waiting on condition 和 java.lang.Thread.State: TIMED_WAITING

```
"RMI TCP Connection(idle)" daemon prio=10 tid=0x00007fd50834e800 nid=0x56b2 waiting on condition [0x00007fd4f1a59000]
java.lang.Thread.State: TIMED_WAITING (parking)
                at sun.misc.Unsafe.park(Native Method)
                - parking to wait for  <0x00000000acd84de8> (a java.util.concurrent.SynchronousQueue$TransferStack)
                at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:198)
                at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:424)
                at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:323)
                at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:874)
                at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:945)
                at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:907)
                at java.lang.Thread.run(Thread.java:662)
```
JVM线程的状态是 java.lang.Thread.State: TIMED_WAITING (parking)，说明线程处于定时等待的状态，parking指线程处于挂起中。  
  
waiting on condition需要结合堆栈中的 parking to wait for <0x00000000acd84de8> (a java.util.concurrent.SynchronousQueue$TransferStack) 一起来分析。首先，本线程肯定是在等待某个条件的发生来把自己唤醒。其次，SynchronousQueue并不是一个队列，只是线程之间移交信息的机制，当我们把一个元素放入到 SynchronousQueue 中的时候必须有另一个线程正在等待接受移交的任务，因此这就是本线程在等待的条件。

### in Object.wait() 和 java.lang.Thread.State: TIMED_WAITING

```
"RMI RenewClean-[172.16.5.19:28475]" daemon prio=10 tid=0x0000000041428800 nid=0xb09 in Object.wait() [0x00007f34f4bd0000]
java.lang.Thread.State: TIMED_WAITING (on object monitor)
                at java.lang.Object.wait(Native Method)
                - waiting on <0x00000000aa672478> (a java.lang.ref.ReferenceQueue$Lock)
                at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:118)
                - locked <0x00000000aa672478> (a java.lang.ref.ReferenceQueue$Lock)
                at sun.rmi.transport.DGCClient$EndpointEntry$RenewCleanThread.run(DGCClient.java:516)
                at java.lang.Thread.run(Thread.java:662)        
```
本例中JVM线程的状态是 java.lang.Thread.State: TIMED_WAITING (on object monitor)，说明线程调用了 java.lang.Object.wait(long timeout) 方法而进入了等待状态。  
  
"Wait Set"中等待的线程状态就是 in Object.wait()，当线程获得了 Monitor进入临界区之后，如果发现线程继续运行的条件没有满足，它就调用对象（通常是被 synchronized 的对象）的wait()方法，放弃了Monitor，进入 "Wait Set" 队列中。只有当别的线程在该对象上调用了 notify()或notifyAll()方法， "Wait Set" 队列中线程才得到机会去竞争，但是只有一个线程获得对象的 Monitor，恢复到的运行态。  
  
另外需要注意的是，是先 locked <0x00000000aa672478> 然后再 waiting on <0x00000000aa672478>，之所以如此，可以通过下面的代码进行演示： 
```
static private class  Lock { };
private Lock lock = new Lock();
public Reference<? extends T> remove(long timeout) {
    synchronized (lock) {
        Reference<? extends T> r = reallyPoll();
        if (r != null) return r;
        for (;;) {
            lock.wait(timeout);
            r = reallyPoll();
            // ……
       }
}
```
线程在执行的过程中，先用 synchronized 获得了这个对象的 Monitor（对应 locked <0x00000000aa672478>），当执行到 lock.wait(timeout); 的时候，线程就放弃了Monitor的所有权，进入 "Wait Set" 队列（对应 waiting on <0x00000000aa672478>）。

