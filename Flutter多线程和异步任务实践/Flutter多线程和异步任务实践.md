> 本文介绍了Flutter中多线程和异步任务的实践。首先简要说明了Isolate, Future, async 和 await等概念。然后根据不同场合，给出了轻量异步任务、重量任务建Isolate、线程间通讯、Flutter Computer方法、FutureGroup方法等的代码实现。

## 前言  
[Flutter](https://flutter.dev/) 作为谷歌推出的跨平台UI开发工具包，为用户创建基于本地编译的移动、Web和桌面应用提供了统一的代码库。它具备热重载能力来提高开发效率，拥有富有表现力和灵活性的用户界面，以及媲美本地原生应用的执行性能(60帧)，这些让它对多端开发者来说是很具有吸引力的。笔者已肤浅地认为它是超越 Reative Native 的存在😄。  
多线程和异步任务对大多框架或语言来说是必须掌握的核心技能，本文从实践角度给出Flutter多线程和异步任务的使用。

## 基础准备   
### Flutter 与 Dart 语言  
Flutter 作为一个开发工具包或框架，在应用层面，使用Dart语言编程。这关系如同iOS App Frameworks和Swift、Android SDK和Kotlin。一般情况下，Flutter里创建线程通过创建[Dart](https://dart.dev/)的Isolates来实现。*(本文代码基于 Flutter 1.5.4 和 Dart 2.3 版本)*。

### Isolates  
Dart中的[Isolate](https://api.dartlang.org/stable/2.3.1/dart-isolate/dart-isolate-library.html)类似于其他系统中的Thread。但又有不同，各Isolate之间不共享内存，因此不存在线程安全和锁的概念，它们之间使用Port和Message来传递消息。Isolate可执行于不同CPU核心来提高性能。*(为了方便理解，下文提到的线程即 Isolate )*。

### Future  
[Future](https://api.flutter.dev/flutter/dart-async/Future-class.html)是一个延后计算的对象，即它的返回值当前并不一定可用，在未来某个时刻它完成计算后便会返回可用的值。比如一个网络请求。通常使用 Future.then 来处理计算完成的场合，用 Future.catchError 来处理发生异常的场合。

### async 和 await 关键字
[async 和 await](https://dart.dev/guides/language/language-tour#asynchrony-support) 关键字用于支持 Dart 语言的异步特性。它的作用和 C# 中的 async 和 await 相同。一个支持异步的方法被标为async, 它的调用者并不会等待它执行完毕。而await关键字必须存在于async方法内，被标为await的语句一般为耗时操作，它后面的语句会等待await语句执行完毕。当耗时操作完成时，await后面的代码便会得到执行(异步)。这对关键字的存在意义就是以同步的编程风格，实现异步的执行。

## 轻量异步任务
对于一些轻量异步任务，比如一个小的网络请求，本身的计算量不大，只是我们不知道它的确切完成时间。这种情况我们用 async 和 await 来简单创建一个异步任务即可。这个异步任务并没有创建新线程，而是在原线程执行。只是通过语言机制达到了异步执行而已。一个简单的例子：

```Dart
Future<String>( () async { 
    await Future.delayed(Duration(seconds: 5)); //故意等待5秒(这里只模拟未知时间，并没有多大计算量)
    return "一个假设的计算结果";
}).then((value) {
    print("${DateTime.now()} 返回 :$value");
});

print("${DateTime.now()} 这里没有等待");
```
输出：
>2019-05-24 10:28:59.165291 这里没有等待  
>2019-05-24 10:29:04.301099 返回 :一个假设的计算结果

从输出我们看到一个异步执行机制, 异步方法后面的语句立即得到执行，5秒后，再输出模拟计算的结果。这里的延时5秒实际并没有占用多少CPU资源，所以它属于轻量计算。在实际测试中，轻量的异步计算并不会导致UI卡顿。到底多少计算量会导致UI卡顿？一般情况下，如果计算的CPU耗时超过10毫秒，就有卡顿风险了。那时就需要创建线程了。


### Flutter scheduleTask 方法
极轻量的任务还可以使用Flutter库提供的[`scheduleTask<T>`](https://s0api0flutter0dev.icopy.site/flutter/scheduler/SchedulerBinding/scheduleTask.html)方法。同 async 一样，它也是在同个线程执行任务，不过它可以设定任务的优先级。使用此方法需导入`package:flutter/scheduler.dart` 。
```Dart
SchedulerBinding.instance.scheduleTask<String>( () { 
    return "一个假设的计算结果";
}, Priority.idle)
.then( (value) {
    print("返回 :$value");
});
```
此方法会将任务添加到一个队列，然后根据优先级，在渲染的帧间执行。每个任务的执行耗时应控制在1毫秒以内。优先级说明：
- **touch**: 在有用户交互时依然执行的任务。最高优先级。
- **animation**: 在执行动画时依然执行的任务。优先级不如touch。
- **idle**: 在没有动画，以及其他更高优先级任务在执行时，它才执行。最低优先级。


## 重量任务，创建 Isolate (类似线程)
耗时计算的重量任务，我们创建新的Isolate去执行。使用Isolate需导入`dart:isolate`库。
#### 一个极简的创建线程例子
```Dart
void main() {
    Isolate.spawn(threadTask, "Hello!");
    print("Isolate:${Isolate.current.debugName}, time:${DateTime.now()}");
} 

void threadTask(String message) async {
    await Future.delayed(Duration(seconds: 5)); //等待5秒
    print("Isolate:${Isolate.current.debugName}, received: $message, time:${DateTime.now()}");
}
```
输出：
>Isolate:main, time:2019-05-24 16:29:21.470846  
>Isolate:threadTask, received: Hello!, time:2019-05-24 16:29:27.189168

以上例子中，我们从主线程创建了一个新的Isolate, 并传入了带一个字符串参数的线程方法，以及一个字符串。从输出看出，两个方法已执行于不同的Isolate。线程方法于5秒后打印传入的字符串。为了方便理解，例子中传了字符串相关的参数，但实际上可以传不同的类型。`spawn`方法声明为`static Future<Isolate> spawn<T>(void entryPoint(T message), T message, ...);`, 从声明看出传入的参数支持泛型，所以可以传入带其他类型参数的线程方法。
由于Isolate间不共享内存，所以它们无法访问同个变量，数据交互通过Port进行。
#### 主线程调子线程，子线程完成后回报的例子：
```Dart
void main() {
    final ReceivePort resultPort = ReceivePort();

    Isolate.spawn(threadTask, resultPort.sendPort).then( (isolate) {
      resultPort.listen((data) {
        print("$data, time:${DateTime.now()}"); //3.接收子线程的数据
        resultPort.close();
        isolate.kill();
      });
    });

    print("Job's requested, time:${DateTime.now()}"); //1.主线程不等待
}

void threadTask(SendPort port) async {
  await Future.delayed(Duration(seconds: 5)); 
  port.send("Job's done"); //2.子线程完成任务，回报数据
}

/*代码出处: https://blog.happyyun.com/ 感谢保留！*/
```
输出：
>Job's requested, time:2019-05-24 17:31:27.532293  
>Job's done, time:2019-05-24 17:31:34.386813

以上例子中，我们创建了一个新线程。在子线程完成任务并回报数据后，主线程打印收到的数据，并杀掉子线程。如果有更复杂的场景，比如线程之间要不断相互发送消息，那么要在子线程也创建一个`ReceivePort`对象，并将它的`sendPort`发送给主线程，主线程便可以通过这个sendPort向子线程发送消息。

#### 线程间双向发消息的例子：
在下面例子中，我们开启一个子线程，子线程完成工作后回报，然后主线程让子线程再开始另一项工作。这个过程中便存在2个线程互发消息的行为。*(为了突出关键步骤，程序中省略了一些常规开发中的必要判断)*。程序：

```Dart
const String job1Done = "Job1's done";
const String job2Done = "Job2's done";

void main() {
    final ReceivePort resultPort = ReceivePort();

    Isolate.spawn(threadTask, resultPort.sendPort).then( (isolate) {
      SendPort threadPort;
      resultPort.listen((data) {
        if (data is SendPort) { // 3. 如果子线程发来Port, 保存起来用于后面发消息
          threadPort = data; 
        }
        else{
          print("$data, time:${DateTime.now()}"); //5. 接收子线程的数据
          if (data == job1Done) { // 6. 如果是工作1完成消息，让子线程开始工作2
            threadPort.send("Start Job 2");
          }
          else{
            // 8. 工作1和2都完成了，关闭Port, 结束子线程
            resultPort.close(); 
            isolate.kill();
          }
        }
      });
    });

    print("Job's requested, time:${DateTime.now()}"); //1. 主线程不等待
}

void threadTask(SendPort port) async {
  final ReceivePort threadReceivePort = ReceivePort();
  port.send(threadReceivePort.sendPort); //2. 给主线程一个Port, 让主线程通过这个给自己发消息

  threadReceivePort.listen((data) { //3. 监听主线程消息
    Future.delayed(Duration(seconds: 5)).then((value){
      port.send(job2Done); //7. 完成工作2，回报数据
      threadReceivePort.close();
    });
  });

  await Future.delayed(Duration(seconds: 5)); 
  port.send(job1Done); //4. 完成工作1，回报数据
}

/*代码出处: https://blog.happyyun.com/ 感谢保留！*/
```
输出：
> Job's requested, time:2019-05-25 10:49:33.158430
> Job1's done, time:2019-05-25 10:49:38.387428
> Job2's done, time:2019-05-25 10:49:43.396201

从输出看出，主线程发起子线程后不等待。在主子线程发消息协同中，Job1 和 Job2 被依次完成。


### Flutter Computer方法
相比以上例子中手动创建Isolate，Flutter 库提供了[`computer`](https://api.flutter.dev/flutter/foundation/compute.html)方法来简化建线程操作。它的声明为：
```Dart
Future<R> compute <Q, R>(
    ComputeCallback<Q, R> callback,
    Q message, {
    String debugLabel
})
```
`computer`方法会Spawn一个isolate, 并在isolate中运行`callback`对应的方法。可传递一个泛型参数。最后返回一个Future对象，该对象包含了线程执行的结果。一个常规的网络请求并解析Json到Model的例子：
```Dart
import 'dart:convert';
import 'package:flutter/foundation.dart';
import 'package:http/http.dart';

void main() {
    compute(fetchPhotos, "https://jsonplaceholder.typicode.com/photos").then( (photoList) {
      print("获取了${photoList.length}个Photo对象");
    });
}

//线程方法
Future<List<Photo>> fetchPhotos(url) async {
    final response = await Client().get(url);

    return parsePhotos(response.body);
}

//解析模型，重量计算
List<Photo> parsePhotos(String responseBody) {
    final parsed = json.decode(responseBody).cast<Map<String, dynamic>>();

    return parsed.map<Photo>((json) => Photo.fromJson(json)).toList();
}

//模型
class Photo {
  final int albumId;
  final int id;
  final String title;
  final String url;
  final String thumbnailUrl;

  Photo({this.albumId, this.id, this.title, this.url, this.thumbnailUrl});

  factory Photo.fromJson(Map<String, dynamic> json) {
    return Photo(
      albumId: json['albumId'] as int,
      id: json['id'] as int,
      title: json['title'] as String,
      url: json['url'] as String,
      thumbnailUrl: json['thumbnailUrl'] as String,
    );
  }
}
```
输出：
> 获取了5000个Photo对象

以上例子中，我们通过`compute`方法创建isolate，在isolate中获取json数据，并解析。最后以Future对象返回，主线程获得数据模型对象列表。此例子基于官方文档稍加改动，官方完整版：[《Parsing JSON in the background》](https://flutter.dev/docs/cookbook/networking/background-parsing)。

### FutureGroup

