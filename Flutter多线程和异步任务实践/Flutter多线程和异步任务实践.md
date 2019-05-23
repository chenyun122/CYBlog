##前言  
[Flutter](https://flutter.dev/) 作为谷歌推出的跨平台UI开发工具包，为用户创建基于本地编译的移动、Web和桌面应用提供了统一的代码库。它具备热重载能力来提高开发效率，拥有富有表现力和灵活性的用户界面，以及媲美本地原生应用的执行性能(60帧)，这些让它对多端开发者来说是很具有吸引力的。笔者已肤浅地认为它是超越 Reative Native 的存在😄。
多线程和异步任务对大多框架或语言来说是必须掌握的核心技能，本文从实践角度给出Flutter多线程和异步任务的使用。

##基础准备   
###Flutter 与 Dart 语言  
Flutter 作为一个开发工具包或框架，在应用层面，我们使用Dart语言编程。这关系如同iOS App Frameworks和Swift、Android SDK和Kotlin。一般情况下，Flutter里创建线程通过创建[Dart](https://dart.dev/)的Isolates来实现。

###Isolates  
Dart中的[Isolate](https://api.dartlang.org/stable/2.3.1/dart-isolate/dart-isolate-library.html)相当于其他系统中的Thread。但又有不同，各Isolate之间不共享内存，因此不存在线程安全和锁的概念，它们之间使用Port和Message来传递消息。Isolate可执行于不同CPU核心来提高性能。

###Future  
[Future](https://api.flutter.dev/flutter/dart-async/Future-class.html)是一个延后计算的对象，即它的返回值当前并不一定可用，在未来某个时刻它完成计算后便会返回可用的值。比如一个网络请求。通常使用 Future.then 来处理计算完成的场合，用 Future.catchError 来处理发生异常的场合。

###async 和 await 关键字
[async 和 await](https://dart.dev/guides/language/language-tour#asynchrony-support) 关键字用于支持 Dart 语言的异步特性。它的作用和 C# 中的 async 和 await 相同。一个支持异步的方法被标为async, 它的调用者并不会等待它执行完毕。而await关键字必须存在于async方法内，被标为await的语句一般为耗时操作，它后面的语句会等待await语句执行完毕。当耗时操作完成时，await后面的代码便会得到执行(异步)。这对关键字的存在意义就是以同步的编程风格，实现异步的执行。

##轻量异步任务


###Flutter scheduleTask


##重量任务，创建线程

###Flutter Computer函数

###FutureGroup

