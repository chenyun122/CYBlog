>**Abstract**: The article describes the .Net equivalents of iOS GCD usages. It includes executing the synchronous and asynchronous tasks in the serial and concurrent queue, dispatching the tasks group, semaphore, concurrently iterations performings.

     笔者平时工作是iOS开发，业余时间也有一些PC桌面端的.Net开发，因此想把iOS开发中常用的多线程神器GCD的任务执行方式在.Net中实现一遍，作为多平台开发人员的参考。在以下代码例子中，iOS平台采用语言Swift 4，.Net框架用C# 7。由于.Net历史悠久，创建多线程的方式很多样，为了避免引导大家使用过时的方式，.Net代码用当前官方推荐的“基于任务的异步模式**Task-based Asynchronous Pattern** (TAP)”来实现。我们使用.Net中的**Lambda**表达式来对应Swift中的**Closure**,力求以最简方式来实现各个案例。

### 一.同步执行+串行队列，同步执行+并发队列
     这是最简单常规的执行方式，但在实际开发中它们的意义可能并不大。由于是同步任务，不论串行还是并发，任务只会在当前线程中顺序执行。过程如下图：
<center>[![同步执行](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramSync.png "同步执行")](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramSync.png "同步执行")</br><span style="font-size:12px">图示1  同步执行</span></center>
在Swift和C#中的代码示例如下:
#### Swift：
```swift
func syncExecute() {
    let serialQueue = DispatchQueue(label: "com.example.sq") //串行队列
    serialQueue.sync {
        print("任务1执行(Thread.current),Time:(Date())")
        sleep(2)
    }
    serialQueue.sync {
        print("任务2执行(Thread.current),Time:(Date())")
        sleep(2)
    }
    
    let concurrentQueue = DispatchQueue(label: "com.example.cq", attributes:.concurrent) //并发队列
    concurrentQueue.sync {
        print("任务3执行(Thread.current),Time:(Date())")
        sleep(2)
    }
    concurrentQueue.sync {
        print("任务4执行(Thread.current),Time:(Date())")
        sleep(2)
    }
}
```



>输出：
任务1执行<NSThread: 0x60000330d100>{number = 1, name = main},Time:2019-03-03 08:10:41 +0000
任务2执行<NSThread: 0x60000330d100>{number = 1, name = main},Time:2019-03-03 08:10:43 +0000
任务3执行<NSThread: 0x60000330d100>{number = 1, name = main},Time:2019-03-03 08:10:45 +0000
任务4执行<NSThread: 0x60000330d100>{number = 1, name = main},Time:2019-03-03 08:10:47 +0000

     可以看到，不论串行队列还是并行队列，都是在当前线程顺序执行各个任务。在.Net中没有串行并行队列的概念，下面我们用同步执行的方式来实现一样的效果(尽管这样做有些多此一举的感觉)。
#### C#：
```csharp
private static void syncExecute()
{
    var task1 = new Task(() =>
    {
        Console.WriteLine("任务1执行,Thread:{0},Time:{1}",
            Thread.CurrentThread.ManagedThreadId,
            DateTime.Now);
        Thread.Sleep(2000);
    });
    task1.RunSynchronously(); //同步执行

    var task2 = new Task(() =>
    {
        Console.WriteLine("任务2执行,Thread:{0},Time:{1}",
            Thread.CurrentThread.ManagedThreadId,
            DateTime.Now);
        Thread.Sleep(2000);
    });
    task2.RunSynchronously();
}
```
>输出：
任务1执行,Thread:1,Time:2019/3/3 下午4:22:27
任务2执行,Thread:1,Time:2019/3/3 下午4:22:29

     以上代码，实现了GCD中同步任务一样的效果，都是在当前线程中顺序执行任务。


### 二.异步执行+串行队列
     在串行队列中异步执行任务，表现为在新线程中顺序执行各个任务。当前线程不等待这些任务的执行完成。过程如下图：
<center>[![异步执行串行队列](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramASyncSerial.png "异步执行串行队列")](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramASyncSerial.png "异步执行串行队列")</br><span style="font-size:12px">图示2  异步执行,串行队列</span></center>
     (感谢我的前同事Wally指出，GCD本身有线程池在维护线程。所以这里的新线程，仅指相对于当前线程的另一线程，它不一定是一个全新创建的线程。下同。)
#### Swift：
```swift
func asyncSerial() {
    print("当前线程(Thread.current)")

    let serialQueue = DispatchQueue(label: "com.example.sq") //串行队列
    serialQueue.async { //异步执行
        print("任务1执行(Thread.current),Time:(Date())")
        sleep(2)
    }
    serialQueue.async {
        print("任务2执行(Thread.current),Time:(Date())")
        sleep(2)
    }

    print("当前线程执行结束")
}
```
>输出：
当前线程<NSThread: 0x6000025600c0>{number = 1, name = main}
当前线程执行结束
任务1执行<NSThread: 0x60000257a2c0>{number = 3, name = (null)},Time:2019-03-03 08:43:40 +0000
任务2执行<NSThread: 0x60000257a2c0>{number = 3, name = (null)},Time:2019-03-03 08:43:42 +0000

     从输出中可以看出，任务1、2被放到新线程中按序执行。而当前线程在任务1、2完成前已执行结束，不会等待。这也是异步任务的意义所在。

#### C#：
```csharp
private static void asyncExecute()
{
    Console.WriteLine("当前线程,Thread:{0}", Thread.CurrentThread.ManagedThreadId);

    var taskAsync = new Task(() =>
    {
        Console.WriteLine("新线程,Thread:{0}", Thread.CurrentThread.ManagedThreadId);

        var task1 = new Task(() =>
        {
            Console.WriteLine("任务1执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
            Thread.Sleep(2000);
        });
        task1.RunSynchronously(); //同步执行

        var task2 = new Task(() =>
        {
            Console.WriteLine("任务2执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
            Thread.Sleep(2000);
        });
        task2.RunSynchronously();
    });
    taskAsync.Start(); //异步执行

    Console.WriteLine("当前线程执行结束");
}
```
>输出：
当前线程,Thread:1
当前线程执行结束
新线程,Thread:3
任务1执行,Thread:3,Time:2019/3/3 下午4:57:52
任务2执行,Thread:3,Time:2019/3/3 下午4:57:54

     由于.Net中没有串行队列这个概念，所以我们将两个同步任务放到一个异步任务中执行，达到了一样的效果。当开启异步任务时，便建立了一个新线程。然后在这个线程中，按序执行两个同步任务。同样当前线程也没有等待新线程中的任务执行完毕。

### 三.异步执行+并行队列
     在并行队列异步执行，表现为在多个线程分别执行各个任务。每个任务执行顺序不确定。大量不相互依赖的任务，一般以这种方式执行效率最高。过程如下图：
<center>[![异步执行串行队列](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramASyncConcurrent.png "异步执行串行队列")](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramASyncConcurrent.png "异步执行串行队列")</br><span style="font-size:12px">图示3  异步执行,并行队列</span></center>
#### Swift：
```swift
func asyncConcurrent() {
    print("当前线程(Thread.current)")

    let concurrentQueue = DispatchQueue(label: "com.example.cq", attributes:.concurrent)
    concurrentQueue.async {
        print("任务1执行(Thread.current),Time:(Date())")
        sleep(2)
    }
    concurrentQueue.async {
        print("任务2执行(Thread.current),Time:(Date())")
        sleep(2)
    }

    print("当前线程执行结束")
}
```
>输出：
当前线程<NSThread: 0x6000008a11c0>{number = 1, name = main}
当前线程执行结束
任务2执行<NSThread: 0x6000008b9500>{number = 4, name = (null)},Time:2019-03-03 09:11:19 +0000
任务1执行<NSThread: 0x6000008b9140>{number = 3, name = (null)},Time:2019-03-03 09:11:19 +0000

     从输出看出，两个任务在2个新线程中各自执行，顺序不定。当前线程同样不会等待任务结束。
#### C#：
```csharp
private static void asyncExecuteConcurrent()
{
    Console.WriteLine("当前线程,Thread:{0}", Thread.CurrentThread.ManagedThreadId);

    var task1 = new Task(() =>
    {
        Console.WriteLine("任务1执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
        Thread.Sleep(2000);
    });
    task1.Start();

    var task2 = new Task(() =>
    {
        Console.WriteLine("任务2执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
        Thread.Sleep(2000);
    });
    task2.Start();

    Console.WriteLine("当前线程执行结束");
}
```
>输出：
当前线程,Thread:1
当前线程执行结束
任务2执行,Thread:4,Time:2019/3/3 下午5:15:39
任务1执行,Thread:3,Time:2019/3/3 下午5:15:39

     从输出看出，两个任务在2个新线程中各自执行，顺序不定。当前线程同样不会等待任务结束。由于.Net中的`Task.Start`方法默认就是多线程并发执行，所以实现起来反倒比“异步执行+并行队列”简单了。

### 四.任务组DispatchGroup
     DispatchGroup的作用在于异步执行一组任务，当这组任务执行完成后，某处需要**被通知**到以便后续处理。iOS中一般由`DispatchQueue.async`和`DispatchGroup.notify`方法配对使用，前者用于往队列中加入多个任务作为一个组，后者用于这组任务完成后被通知。如果这组任务本身又是异步，`notify`会立即被调用（因为是异步，很快就调用完成了，而任务本身实际并没有执行完)。此时，可以用`DispatchGroup.enter`和`DispatchGroup.leave`替代`async`来达到更精确的控制。而`DispatchGroup.wait`方法可以用来替代`notify`，同样用于被通知，但它属于阻塞性等待(`notify`不阻塞当前线程)，并且多了一个超时参数，可用于多个任务超时后强制执行后续处理。由于篇幅所限，这里只给出`async`和`notify`的例子以及它们在.Net中的等价做法。对于和Enter、Leave相等的.Net实现，希望以后可以有时间专门写一篇。Group执行过程如下图：
<center>[![Dispatch Group](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramDispatchGroup.png "Dispatch Group")](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramDispatchGroup.png "Dispatch Group")</br><span style="font-size:12px">图示4  Dispatch Group</span></center>
#### Swift：
```swift
func asyncGroup() {
    print("当前线程(Thread.current)")
    
    let concurrentQueue = DispatchQueue(label: "com.example.cq", attributes:.concurrent)
    let group = DispatchGroup()
    
    concurrentQueue.async(group: group, execute: {
        print("任务1执行(Thread.current),Time:(Date())")
        sleep(2)
    })
    
    concurrentQueue.async(group: group, execute: {
        print("任务2执行(Thread.current),Time:(Date())")
        sleep(2)
    })
    
    group.notify(queue: concurrentQueue, execute: {
        print("2项任务都执行完啦(Thread.current),Time:(Date())")
    })
}
```
>输出：
当前线程<NSThread: 0x600002e8d140>{number = 1, name = main}
任务1执行<NSThread: 0x600002e9eec0>{number = 3, name = (null)},Time:2019-03-03 02:56:38 +0000
任务2执行<NSThread: 0x600002e99fc0>{number = 4, name = (null)},Time:2019-03-03 02:56:38 +0000
2项任务都执行完啦<NSThread: 0x600002e9eec0>{number = 3, name = (null)},Time:2019-03-03 02:56:40 +0000

     从输出看出，由于使用了并发队列，任务1、2都起了新线程各自执行。任务1、2执行完成后，某线程如期得到了通知。同时也看出，被通知的线程也并不是主线程，所以在`notify`中要刷新UI的话，需手动调用主线程处理。

#### C#：
```csharp
private static void asyncGroup()
{
    Console.WriteLine("当前线程,Thread:{0}", Thread.CurrentThread.ManagedThreadId);

    var taskAsync = new Task(() =>
    {
        var task1 = new Task(() =>
        {
            Console.WriteLine("任务1执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
            Thread.Sleep(2000);
        });
        task1.Start();

        var task2 = new Task(() =>
        {
            Console.WriteLine("任务2执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
            Thread.Sleep(2000);
        });
        task2.Start();

        Task.WaitAll(new Task[] { task1, task2 }); //等待所有任务执行完毕
        Console.WriteLine("2项任务都执行完啦Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);

    });
    taskAsync.Start();
}
```
>输出：
当前线程,Thread:1
任务2执行,Thread:3,Time:2019/3/3 下午3:30:27
任务1执行,Thread:4,Time:2019/3/3 下午3:30:27
2项任务都执行完啦Thread:3,Time:2019/3/3 下午3:30:30

     由于.Net中默认没有类似`DispatchGroup.notify`通知机制，手动实现也比较麻烦。所以这里简单使用`Task.WaitAll`方法等待各个任务的执行完成，来模拟“各个任务完成后进行后续处理”的情况。由于“执行多任务”和“等待”也是在异步任务中进行的，所以并没有阻塞当前线程。
     对于多组任务，Objective-C中用`dispatch_barrier_async`方法进行分隔，Swift中用`queue.async(group: group, flags:.barrier, execute: {})`来分隔。可确保一批任务执行完毕后，再执行下一批任务(栅栏效果)。而.Net中可以在各批任务之间插入`Task.WaitAll`方法，来确保WaitAll之前的那批任务先执行，达到控制先后效果。

### 五.信号量DispatchSemaphore
     信号量的作用在于限制线程执行数量，或者线程同步。当信号量为0时，所有依赖于该信号量的线程都不会执行。当信号量增为1时，有一个线程可以执行，同时该线程也把信号量再次降为0，以阻止其他线程执行。信号量为N时，意味着有N个依赖于该信号量的线程可以执行。信号量的增减控制方法为`signal()`和`wait()`。signal会使信号量增1；wait则是等待信号量，当等待成功线程可以执行时，wait会使信号量减1。在线程同步方面，比如任务B要在任务A完成后才能执行，任务A也是异步任务，我们并不知道它完成的确切时间；那么可以使用一个信号量并置为0，让B wait该信号量，此时B无法执行；当A完成时，执行signal方法使信号量增为1，B便得到了信号开始执行。一般依赖于信号量的任务执行过程如下：
<center>[![信号量](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramSemaphore.png "信号量")](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramSemaphore.png "信号量")</br><span style="font-size:12px">图示5  信号量</span></center>
以下例子演示一个简单的基于信号量的线程同步：
#### Swift：
```swift
func semaphore() {
    let semaphore = DispatchSemaphore(value:0)
    let queue = DispatchQueue(label: "com.example.cq", attributes:.concurrent)
   
    queue.async {
        print("任务A执行(Thread.current),Time:(Date())")
        sleep(2)
        print("任务A完成(Thread.current),Time:(Date())")
        semaphore.signal()
    }
    
    queue.async {
        semaphore.wait()
        print("任务B执行(Thread.current),Time:(Date())")
    }
}
```
>输出：
任务A执行<NSThread: 0x60000338c400>{number = 3, name = (null)},Time:2019-03-03 09:02:03 +0000
任务A完成<NSThread: 0x60000338c400>{number = 3, name = (null)},Time:2019-03-03 09:02:05 +0000
任务B执行<NSThread: 0x60000338e1c0>{number = 6, name = (null)},Time:2019-03-03 09:02:05 +0000

     从输出看到，任务A执行2秒后完成，任务B才得以执行。更复杂的情况，比如任务A、B执行完后，C、D才执行，此时用信号量控制便比较复杂，还是用DispatchGroup的栅栏分批执行比较简单。**一般信号量>1的情况，用作资源访问限制的情况比较多**，比如同时发起网络请求的线程只允许2个。

#### C#：
```csharp
private static void semaphore()
{
    var semaphore = new Semaphore(0, 1);

    var taskA = new Task(() =>
    {
        Console.WriteLine("任务A执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
        Thread.Sleep(2000);
        Console.WriteLine("任务A完成,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
        semaphore.Release();
    });
    taskA.Start();

    var taskB = new Task(() =>
    {
        semaphore.WaitOne();
        Console.WriteLine("任务B执行,Thread:{0},Time:{1}", Thread.CurrentThread.ManagedThreadId, DateTime.Now);
    });
    taskB.Start();
}
```
>输出：
任务A执行,Thread:3,Time:2019/3/3 下午5:26:48
任务A完成,Thread:3,Time:2019/3/3 下午5:26:50
任务B执行,Thread:4,Time:2019/3/3 下午5:26:50

     通过以上例子和输出，我们看到C#中也有Semaphore实现相应的功能。只是`signal()`和`wait()`方法换成了`Release()`和`WaitOne()`。


### 六.延时执行DispatchQueue.asyncAfter
     延时执行的比较简单，用于在指定队列中延时执行异步任务，当前线程不等待。
#### Swift：
```swift
func delay() {
    let concurrentQueue = DispatchQueue(label: "com.example.cq", attributes:.concurrent)
    print("当前线程时间:(Date()), 线程:(Thread.current)")

    concurrentQueue.asyncAfter(deadline: DispatchTime.now() + 2) {
        print("任务1执行时间:(Date()), 线程:(Thread.current)")
    }
    
    print("当前线程结束")
}
```
>输出：
当前线程时间:2019-03-03 07:54:28 +0000, 线程:<NSThread: 0x600001ae91c0>{number = 1, name = main}
当前线程结束
任务1执行时间:2019-03-03 07:54:30 +0000, 线程:<NSThread: 0x600001af2600>{number = 4, name = (null)}

     从输出看出，任务1延时2秒后, 在新线程中执行。当前线程已结束，不等待。
#### C#：
```csharp
private static void delayAsync()
{
    Console.WriteLine("当前线程时间:{0},Thread:{1}", DateTime.Now, Thread.CurrentThread.ManagedThreadId);

    var task1 = new Task(async () =>
    {
        await Task.Delay(2000); // Thread.Sleep(2000); 效果雷同，只是Task.Delay多了可取消功能
        Console.WriteLine("任务1执行时间:{0}, Thread:{1}", DateTime.Now, Thread.CurrentThread.ManagedThreadId);
    });
    task1.Start();

    Console.WriteLine("当前线程结束");
}
```
>输出：
当前线程时间:2019/3/3 16:22:15,Thread:1
当前线程结束
任务1执行时间:2019/3/3 16:22:17, Thread:6

     从输出看，取得了和Swift一样的效果。


### 七.并发迭代DispatchQueue.concurrentPerform
     并发迭代是我自己理解和翻译的一个称谓，顾名思义就是将迭代放到各个线程中并发执行，以此利用CPU多核优势来提高效率。进一步说明下它存在的理由，当面对大量迭代项时，传统的迭代方式比如for、while循环，一个一个顺序处理性能比较低。但如果在循环中创建大量线程，通过并发来提高效率，则又可能导致系统崩溃。试想下以下场景：
#### 一个不合理的并发迭代：
```swift
func performUnreasonably() {
    let queue = DispatchQueue(label: "com.example.cq", attributes:.concurrent)
    //应避免这种方式
    for i in 0...999 {
        queue.async {
            print("任务(i)执行(Thread.current),Time:(Date())")
            sleep(2)
        }
    }
}
```
     以上例子的弊端是不言而喻的。虽然GCD的线程池会对最大线程数做一定限制，但大量线程对系统来说负担还是比较大的，并且提高了死锁的概率。其它系统或框架是否会有GCD那样的线程限制也不确定，所以上面这种做法应当避免。那么把大量任务放到线程中处理，又要避免大量线程，要怎么做？合理的“并发迭代”由此而生：
**- 在Objective-C版的GCD中，它叫`dispatch_apply`方法;**
**- 在Swift版的GCD中，它叫`DispatchQueue.concurrentPerform`方法;**
**- 在.Net中, 它叫`Parallel.For`或`Parallel.Foreach`。**
合理的并发迭代过程如下图：
<center>[![并发迭代](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramApply.png "并发迭代")](https://blog.happyyun.com/wp-content/uploads/2019/03/diagramApply.png "并发迭代")</br><span style="font-size:12px">图示7  并发迭代</span></center>

#### 合理的Swift版本：
```swift
func concurrentPerform() {
    DispatchQueue.concurrentPerform(iterations: 999) { (index) in
        print("任务(index)执行(Thread.current),Time:(Date())")
        sleep(2)
    }
}
```
>输出：
任务3执行<NSThread: 0x600002ff94c0>{number = 3, name = (null)},Time:2019-03-03 06:52:41 +0000
任务1执行<NSThread: 0x600002ffeb00>{number = 4, name = (null)},Time:2019-03-03 06:52:41 +0000
任务0执行<NSThread: 0x600002fe68c0>{number = 1, name = main},Time:2019-03-03 06:52:41 +0000
任务2执行<NSThread: 0x600002ffebc0>{number = 5, name = (null)},Time:2019-03-03 06:52:41 +0000
任务6执行<NSThread: 0x600002ffebc0>{number = 5, name = (null)},Time:2019-03-03 06:52:43 +0000
任务4执行<NSThread: 0x600002fe68c0>{number = 1, name = main},Time:2019-03-03 06:52:43 +0000
任务7执行<NSThread: 0x600002ff94c0>{number = 3, name = (null)},Time:2019-03-03 06:52:43 +0000
任务5执行<NSThread: 0x600002ffeb00>{number = 4, name = (null)},Time:2019-03-03 06:52:43 +0000
任务9执行<NSThread: 0x600002ffebc0>{number = 5, name = (null)},Time:2019-03-03 06:52:45 +0000
任务8执行<NSThread: 0x600002fe68c0>{number = 1, name = main},Time:2019-03-03 06:52:45 +0000
任务10执行<NSThread: 0x600002ff94c0>{number = 3, name = (null)},Time:2019-03-03 06:52:45 +0000
......

     由于迭代999次太多，这里只给出有代表性的输出。可以看出，各个任务在新起的线程中无序并发执行，提高了效率。同时，新起的线程数量并没有很多，而是限制在4个，重复利用。比如线程3一开始处理任务3，后面处理任务7，以及任务10。这样在提高性能的同时，有效避免了线程爆炸的情况。

#### 合理的C#版本：
```csharp
private static void execParallelFor()
{
    Parallel.For(0, 999, (i) =>
    {
        Console.WriteLine("任务{0}执行,Thread:{1},Time:{2}", i+1, Thread.CurrentThread.ManagedThreadId, DateTime.Now);
        Thread.Sleep(2000);
    });
}
```
>输出：
任务250执行,Thread:3,Time:2019/3/3 下午3:55:52
任务997执行,Thread:6,Time:2019/3/3 下午3:55:52
任务499执行,Thread:5,Time:2019/3/3 下午3:55:52
任务748执行,Thread:4,Time:2019/3/3 下午3:55:52
任务1执行,Thread:1,Time:2019/3/3 下午3:55:52
任务2执行,Thread:7,Time:2019/3/3 下午3:55:53
任务251执行,Thread:8,Time:2019/3/3 下午3:55:53
任务500执行,Thread:9,Time:2019/3/3 下午3:55:54
任务501执行,Thread:5,Time:2019/3/3 下午3:55:54
任务749执行,Thread:4,Time:2019/3/3 下午3:55:54
任务998执行,Thread:6,Time:2019/3/3 下午3:55:54
任务252执行,Thread:3,Time:2019/3/3 下午3:55:54
任务3执行,Thread:1,Time:2019/3/3 下午3:55:54
任务5执行,Thread:7,Time:2019/3/3 下午3:55:55
任务254执行,Thread:8,Time:2019/3/3 下午3:55:55
任务503执行,Thread:9,Time:2019/3/3 下午3:55:56
任务253执行,Thread:3,Time:2019/3/3 下午3:55:56
任务750执行,Thread:4,Time:2019/3/3 下午3:55:56
......

     我们同样迭代执行999个任务，它们无序异步执行于各个线程，线程数被限制为9个。线程3一开始处理任务250，之后处理任务252和253。基本和Swift版本效果相同。

### 八.只执行一次dispatch_once(Objective-C)
     在语言Objective-C中利用GCD的dispatch_once来确保相关代码只执行一次，很多时候用在单例的初始化上。从Swift3开始，dispatch_once已被废弃，用语言本身的静态常量等来实现线程安全的单例。同样C#大多也在语言层面实现单例。所以这里对dispatch_once不再展开。

### 总结
     通过以上各项例子，我们做到了大部分GCD功能在.Net中的对等实现。从中也可以看出，多线程、异步、并发等等概念，在各个平台中的思路其实是通的。感谢阅读，希望此文能帮助到您！

>欢迎转载，请保留出处：[https://blog.happyyun.com](https://blog.happyyun.com "https://blog.happyyun.com")