---
layout: post
title: 线程同步
category: Multithreading with C#
tags: Multithreading
keywords: 
description:
---

---
## 一.线程同步的几种方式：

> 原子操作

这意味着一个操作只占用一个量子的时间，一次就可以完成。所以只有当前操作完成后，其他线程才能执行其他操作。因此，你无须实现其他线程等待当前操作完成，这就避免了使用锁，也排除了死锁的情况。

> 上下文切换

上下文切换是指操作系统的线程调度器。该调度器会保存等待的线程的状态，并切换到另外一个线程，依次恢复等待的线程的状态。这需要消耗相当多的资源。然而，如果线程需被挂起很长时间，那么这么做是值得的。这种方式又被称为 **内核模式**

> 用户模式

线程只需要等待一小段的时间，而不用将线程切换成阻塞状态，虽然线程等待会浪费CPU时间，但是我们节省了上下文切换的时间。

> 混合模式

混合模式是指先尝试使用用户模式等待，如果线程等待了足够长的时间，则会切换到阻塞状态以节省CPU资源。



## 二.具体实现方式
### 1.原子操作

```java

 private static void Main(string[] args)
        {
            WriteLine("Correct counter");
            var c1 = new CounterNoLock();
            var t1 = new Thread(() => TestCounter(c1));
            var t2 = new Thread(() => TestCounter(c1));
            var t3 = new Thread(() => TestCounter(c1));
            t1.Start();
            t2.Start();
            t3.Start();
            t1.Join();
            t2.Join();
            t3.Join();
            WriteLine($"Total count: {c1.Count}");
            ReadLine();
        }
        static void TestCounter(CounterBase c)
        {
            for (int i = 0; i < 100000; i++)
            {
                c.Increment();
                c.Decrement();
            }
        }
        class CounterNoLock : CounterBase
        {
            private int _count;
            public int Count => _count;
            public override void Increment()
            {
                Interlocked.Increment(ref _count);
            }
            public override void Decrement()
            {
                Interlocked.Decrement(ref _count);
            }
        }
        abstract class CounterBase
        {
            public abstract void Increment();
            public abstract void Decrement();
        }

```

上述代码的结果为：

![image.png-6.5kB][1]
 这是一个计算器的示例，Interlocked无须锁定任何对象即可获取到正确的结果。Interlocked 提供了Increment，Decrement，Add等基本数学操作的原子的方法。




### 2.Mutex

> “mutex”是术语“互相排斥（mutually exclusive）”的简写形式，也就是互斥量。互斥量跟临界区中提到的Monitor很相似，只有拥有互斥对象的线程才具有访问资源的权限，由于互斥对象只有一个，因此就决定了任何情况下此共享资源都不会同时被多个线程所访问。


#### 2.1构造函数介绍

>* Mutex()：

用无参数的构造函数得到的Mutex没有任何名称，而进程间无法通过变量的形式共享数据，所以没有名称的Mutex也叫做局部（Local）Mutex。另外，这样创建出的Mutex，创建者对这个实例并没有拥有权，仍然需要调用WaitOne()去请求所有权。


>* Mutex(Boolean initiallyOwned)：

与上面的构造函数一样，它只能创建没有名称的局部Mutex，无法用于进程间的同步。Boolean参数用于指定在创建者创建Mutex后，是否立刻获得拥有权，因此Mutex(false)等效于Mutex()。


>* Mutex(Boolean initiallyOwned, String name)：

在这个构造函数里我们除了能指定是否在创建后获得初始拥有权外，还可以为这个Mutex取一个名字。只有这种命名的Mutex才可以被其它应用程序域中的程序所使用.


>* Mutex(Boolean initiallyOwned, String name, out Boolean createdNew)：

头两个参数与上面的构造函数相同，第三个out参数用于表明是否获得了初始的拥有权。这个构造函数应该是我们在实际中使用较多的。


#### 2.2 方法介绍

>* close:

释放由当前WaitHandle持有的所有资源。

>* ReleaseMutex()：

释放当前 Mutex 一次。

>* WaitOne() / WaitOne(Int32, Boolean) / WaitOne(TimeSpan, Boolean):

请求所有权，该调用会一直阻塞到当前 mutex 收到信号，或直至达到可选的超时间隔。


#### 2.3实现示例：

示例1：
```java

static void Main(string[] args)
        {
            const string MutexName = "CSharpThreadingCookbook";
            using (var m = new Mutex(false, MutexName))
            {
                if (!m.WaitOne(TimeSpan.FromSeconds(10), false))
                {
                    WriteLine("Second instance is running!");
                    ReadLine();
                }
                else
                {
                    WriteLine("Running!");
                    ReadLine();
                    m.ReleaseMutex();
                   
                }
            }
        }
```

程序介绍：先进来的线程通过WaitOne() 获取到了信号量，按任意键可以释放信号量，第二个进来的线程，需等待10秒钟获取信号量，可以加等待10秒的过程中如果获取信号量和没有获取到信号量的逻辑


结果1：
![image.png-36.2kB][2]


结果2：

![image.png-12.6kB][3]

示例2：

```java
class Program
    {
        private static Mutex mutex = null;  //设为Static成员，是为了在整个程序生命周期内持有Mutex

        static void Main()
        {
            bool firstInstance;
            mutex = new Mutex(true, @"Global\MutexSampleApp", out firstInstance);
            try
            {
                if (!firstInstance)
                {

                    Console.WriteLine("已有实例运行，输入回车退出……");
                    Console.ReadLine();
                    return;

                }
                else
                {
                    Console.WriteLine("我们是第一个实例！");
                    Thread.Sleep(10000);
                }
            }
            finally
            {
                if (firstInstance)
                {

                    mutex.ReleaseMutex();

                }
                mutex.Close();
                mutex = null;
            }
        }
    }


```

![image.png-13.5kB][4]

程序介绍：第一个进来的线程，创建互斥量，并获取控制权，同时可以知道是否获取了初始控制权，直到释放资源，在这个过程中其余的线程判断是否有控制权来处理逻辑。

#### 2.4 使用Mutex需要注意的两个细节

1.可能你已经注意到了，例子中在给Mutex命名的字符串里给出了一个“Global\”的前缀。这是因为在运行终端服务（或者远程桌面）的服务器上，已命名的全局 mutex 有两种可见性。如果名称以前缀“Global\”开头，则 mutex 在所有终端服务器会话中均为可见。如果名称以前缀“Local\”开头，则 mutex 仅在创建它的终端服务器会话中可见，在这种情况下，服务器上各个其他终端服务器会话中都可以拥有一个名称相同的独立 mutex。如果创建已命名 mutex 时不指定前缀，则它将采用前缀“Local\”。在终端服务器会话中，只是名称前缀不同的两个 mutex 是独立的 mutex，这两个 mutex 对于终端服务器会话中的所有进程均为可见。即：前缀名称“Global\”和“Local\”仅用来说明 mutex 名称相对于终端服务器会话（而并非相对于进程）的范围。最后需要注意“Global\”和“Local\”是大小写敏感的。


2.既然父类实现了IDisposalble接口，那么说明这个类一定需要你手工释放那些非托管的资源。所以必须使用try/finally，或者using，调用Close()方法来释放Mutex所占用的所有资源！



### 3.SemaphoreSlim

> SemaphoreSlim 可以限制同一时刻访问统一资源的线程的数量

#### 3.1 程序介绍

```java

 static void Main(string[] args)
        {
            for (int i = 1; i <= 10; i++)
            {
                string threadName = "Thread " + i;
                int secondsToWait = 2;
                var t = new Thread(() => AccessDatabase(threadName, secondsToWait));
                t.Start();
            }
            ReadLine();
        }

        static SemaphoreSlim _semaphore = new SemaphoreSlim(4);

        static void AccessDatabase(string name, int seconds)
        {
            //WriteLine($"{name} waits to access a database");
            _semaphore.Wait();
            //WriteLine($"{name} was granted an access to a database");
            Sleep(TimeSpan.FromSeconds(seconds));
            WriteLine($"{name} is completed");
            _semaphore.Release();
        }

```

程序介绍：限制了同一时刻访问AccessDatabase 的线程数量为4，当满4个之后其余的线程需要等待，直到调用Release()，其余的才能访问。



### 4.AutoResetEvent
> AutoResetEvent 类可以通知等待的线程有某事件发生，从一个线程向另一个线程发送通知

#### 4.1 代码示例

```java
  static void Main(string[] args)
        {
            var t = new Thread(() => Process(10));
            t.Start();
            WriteLine("Waiting for another thread to complete work");
            _workerEvent.WaitOne();
            WriteLine("First operation is completed!");
            WriteLine("Performing an operation on a main thread");
            Sleep(TimeSpan.FromSeconds(5));
            _mainEvent.Set();
            WriteLine("Now running the second operation on a second thread");
            _workerEvent.WaitOne();
            WriteLine("Second operation is completed!");

            ReadLine();
        }

        private static AutoResetEvent _workerEvent = new AutoResetEvent(false);
        private static AutoResetEvent _mainEvent = new AutoResetEvent(false);

        static void Process(int seconds)
        {
            WriteLine("Starting a long running work...");
            Sleep(TimeSpan.FromSeconds(seconds));
            WriteLine("Work is done!");
            _workerEvent.Set();
            WriteLine("Waiting for a main thread to complete its work");
            _mainEvent.WaitOne();
            WriteLine("Starting second operation...");
            Sleep(TimeSpan.FromSeconds(seconds));
            WriteLine("Work is done!");
            _workerEvent.Set();
        }

```
![image.png-12.2kB][5]

程序介绍：我们向AutoResetEvent构造方法传入false，定义了这两个实例的初始状态为unsigned。这意味着任何线程调用这两个对象中的任何一个WaitOne方法将会被堵塞，直到我们调用了Set方法。如果初始状态为true，那么AutoResetEvent实例的状态为signaled，如果线程调用了WaitOne方法则会被立即处理。然后状态变为unsigned,所以需要再对该实例调用一次Set方法，以便让其他线程对该实例调用WaitOne方法从而继续执行。AutoResetEvent采用的是内核模式，所以等待时间不能太长。


### 5.ManualResetEventSlim
>ManualResetEventSlim 属于混合模式，整个工作方式有点像人群通过大门。上节介绍的AutoResetEvent事件像一个旋转门，一次只允许一人通过。Set() 方法表示开启大门，一直到手动调用Reset为止。


#### 5.1代码介绍

```java
 class Program
    {
        static void Main(string[] args)
        {
            var t1 = new Thread(() => TravelThroughGates("Thread 1", 5));
            var t2 = new Thread(() => TravelThroughGates("Thread 2", 6));
            var t3 = new Thread(() => TravelThroughGates("Thread 3", 12));
            t1.Start();
            t2.Start();
            t3.Start();
            Sleep(TimeSpan.FromSeconds(6));
            WriteLine("The gates are now open!");
            _mainEvent.Set();
            Sleep(TimeSpan.FromSeconds(2));
            _mainEvent.Reset();
            WriteLine("The gates have been closed!");
            Sleep(TimeSpan.FromSeconds(10));
            WriteLine("The gates are now open for the second time!");
            _mainEvent.Set();
            Sleep(TimeSpan.FromSeconds(2));
            WriteLine("The gates have been closed!");
            _mainEvent.Reset();

            Console.ReadLine();
        }

        static void TravelThroughGates(string threadName, int seconds)
        {
            WriteLine($"{threadName} falls to sleep");
            Sleep(TimeSpan.FromSeconds(seconds));
            WriteLine($"{threadName} waits for the gates to open!");
            _mainEvent.Wait();
            WriteLine($"{threadName} enters the gates!");
        }

        static ManualResetEventSlim _mainEvent = new ManualResetEventSlim(false);
    }

```

![image.png-14.2kB][6]


程序介绍：程序运行的时候开启三个线程，在主线程调用Set()之前，线程调用Wait()会处于等待，在调用Set()表示开启大门,无论开启的线程等待多久都可以通过。调用Reset() 表示关闭大门，拒绝通过。


### 6.CountDownEvent
>CountDownEvent可以初始化指定需要完成的操作个数，当操作完成之后需要调用Signal(),当不足完成的个数时，将会一直等待,不过可以设置等待的时长,也可以重置。

#### 6.1代码介绍
```java
 static void Main(string[] args)
        {
            WriteLine("Starting two operations");
            var t1 = new Thread(() => PerformOperation("Operation 1 is completed", 4));
            var t2 = new Thread(() => PerformOperation("Operation 2 is completed", 8));
            t1.Start();
            t2.Start();
            _countdown.Wait();
            WriteLine("Both operations have been completed.");
            _countdown.Dispose();
        }
        static CountdownEvent _countdown = new CountdownEvent(2);

        static void PerformOperation(string message, int seconds)
        {
            Sleep(TimeSpan.FromSeconds(seconds));
            WriteLine(message);
            _countdown.Signal();
        }

```
![image.png-5.6kB][7]


### 7.Barrier

> 在创建Barrier类时，指定同步的线程的个数，以及回调函数。在两个线程中的一个调用了SignalAndWait方法后，会触发回调函数。


#### 7.1 代码示例

```java
static void Main(string[] args)
        {
            var t1 = new Thread(() => PlayMusic("the guitarist", "play an amazing solo", 5));
            var t2 = new Thread(() => PlayMusic("the singer", "sing his song", 2));

            t1.Start();
            t2.Start();

            ReadLine();
        }

        static Barrier _barrier = new Barrier(2,
    b => WriteLine($"End of phase {b.CurrentPhaseNumber + 1}"));

        static void PlayMusic(string name, string message, int seconds)
        {
            for (int i = 1; i < 3; i++)
            {
                WriteLine("----------------------------------------------");
                Sleep(TimeSpan.FromSeconds(seconds));
                WriteLine($"{name} starts to {message}");
                Sleep(TimeSpan.FromSeconds(seconds));
                WriteLine($"{name} finishes to {message}");
                _barrier.SignalAndWait();
            }
        }

```

![image.png-12.6kB][8]
程序介绍：初始创建了两个线程，每个线程将向Barrier发送两次信号，所以会存在两个阶段。每次这两个线程调用SignalAndWait()方法时,Barrier将执行回调函数。


### 8.ReaderWriterLockSlim 类
>ReaderWriterLockSlim 代表了一个管理资源访问的锁，允许多个线程同时读取，以及独占写。


```java
 static void Main(string[] args)
        {
            new Thread(Read) { IsBackground = true }.Start();
            new Thread(Read) { IsBackground = true }.Start();
            new Thread(Read) { IsBackground = true }.Start();

            new Thread(() => Write("Thread 1")) { IsBackground = true }.Start();
            new Thread(() => Write("Thread 2")) { IsBackground = true }.Start();

            Sleep(TimeSpan.FromSeconds(30));
        }

        static ReaderWriterLockSlim _rw = new ReaderWriterLockSlim();
        static Dictionary<int, int> _items = new Dictionary<int, int>();

        static void Read()
        {
            WriteLine("Reading contents of a dictionary");
            while (true)
            {
                try
                {
                    _rw.EnterReadLock();
                    foreach (var key in _items.Keys)
                    {
                        Sleep(TimeSpan.FromSeconds(0.1));
                        WriteLine($" key {key} value {_items[key]}");
                    }
                }
                finally
                {
                    _rw.ExitReadLock();
                }
            }
        }

        static void Write(string threadName)
        {
            while (true)
            {
                try
                {
                    int newKey = new Random().Next(250);
                    _rw.EnterUpgradeableReadLock();
                    if (!_items.ContainsKey(newKey))
                    {
                        try
                        {
                            _rw.EnterWriteLock();
                            _items[newKey] = 1;
                            WriteLine($"New key {newKey} is added to a dictionary by a {threadName}");
                        }
                        finally
                        {
                            _rw.ExitWriteLock();
                        }
                    }
                    Sleep(TimeSpan.FromSeconds(0.1));
                }
                finally
                {
                    _rw.ExitUpgradeableReadLock();
                }
            }
        }

```
程序介绍：读锁允许多线程读取数据，写锁在被释放前会阻塞了其他线程的所有操作。获取读锁时还有一个有意思的场景，即从集合中读取数据时，根据当前数据而决定是否获取一个写锁并修改该集合。一旦得到写锁，会阻止阅读者读取数据，从而浪费了大量的时间，因此获取写锁后会处于阻塞状态。为了最小化阻塞时间。可以使用EnterUpgradeableReadLock和ExitUpgradeableReadLock方法。先获取读锁后读取数据。如果发现必须修改底层集合，只需要使用EnterWriteLock方法升级锁，然后快速执行一次写操作，最后使用ExitWriteLock释放写锁。

### 9.SpinWait

>SpinWait 是一个混合同步构造，被设计为使用用户模式等待一段时间，然后切换到内核模式以节省CPU时间。


#### 9.1 程序示例：

```java

 static void Main(string[] args)
        {
            var t1 = new Thread(UserModeWait);
            var t2 = new Thread(HybridSpinWait);

            WriteLine("Running user mode waiting");
            t1.Start();
            Sleep(20);
            _isCompleted = true;
            Sleep(TimeSpan.FromSeconds(1));
            _isCompleted = false;
            WriteLine("Running hybrid SpinWait construct waiting");
            t2.Start();
            Sleep(5);
            _isCompleted = true;
        }

        static volatile bool _isCompleted = false;

        static void UserModeWait()
        {
            while (!_isCompleted)
            {
                Write(".");
            }
            WriteLine();
            WriteLine("Waiting is complete");
        }

        static void HybridSpinWait()
        {
            var w = new SpinWait();
            while (!_isCompleted)
            {
                w.SpinOnce();
                WriteLine(w.NextSpinWillYield);
            }
            WriteLine("Waiting is complete");
        }

```
程序介绍：本示例主要演示了一种是普通的线程无止境循环和混合模式循环在CPU资源消耗上的对比，切换到内核模式之后变成了阻塞状态，在任务管理器将看不到任何CPU的使用。


  [1]: http://static.zybuluo.com/qxjbeyond/xs67jijmdwjza1mhhrwidoyh/image.png
  [2]: http://static.zybuluo.com/qxjbeyond/owcike79irllrgqhzcn4zsoq/image.png
  [3]: http://static.zybuluo.com/qxjbeyond/kqfd7zfqzepn5ulxsz8nk1hc/image.png
  [4]: http://static.zybuluo.com/qxjbeyond/xea7pu5921dpoe8npl02rj2i/image.png
  [5]: http://static.zybuluo.com/qxjbeyond/q57xqcoyetqcetohli1pzldm/image.png
  [6]: http://static.zybuluo.com/qxjbeyond/2z6sptutj7oxzjfrmdjiqvxl/image.png
  [7]: http://static.zybuluo.com/qxjbeyond/9dqq020fwovpf4kxou2c7ypo/image.png
  [8]: http://static.zybuluo.com/qxjbeyond/iz56960q3dtr0bhugt5cwswl/image.png
