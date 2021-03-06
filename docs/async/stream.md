# Stream  
### 1\. 生成器/迭代器（Generator/Iterator）  
`Dart` 语言中提供了两种生成器函数，`同步（Sync`）的生成器返回一个 `可迭代对象（Iterable Object）`，而 `异步（Async`）的生成器返回一个 `流（Stream`）。如下例：  
```dart
// A synchronous generator function generates an Iterator Object
Iterable<int> naturalsTo(int n) sync* {
  int k = 0;
  while (k < n) yield k++;
}

// An asynchronous generator function generates a Stream
Stream<int> asynchronousNaturalsTo(int n) async* {
  int k = 0;
  while (k < n) yield k++;
}
```
如上例所示，生成器的区别在于 `返回的类型` 以及 `函数体前的同步异步（sync*/async*）`。  
同步的生成器返回一个 `Iterable` 对象，该对象拥有 `Iterator` 属性，当对这个 `Iterable` 对象进行迭代时，其内部属性 `Iterator` 会调用 `moveNext()` 方法移动指针到下一个元素，并使用 `current` 属性获取当前指针所对应的值。例：  
```dart
  Iterable<int> natureTo(int n) sync* {
    int k = 0;
    while (k < n) yield k++;
  }

  final a = natureTo(3).iterator;
  print(a.moveNext());      // true
  print(a.current);         // 0
  print(a.moveNext());      // true
  print(a.current);         // 1
  print(a.moveNext());      // true
  print(a.current);         // 2
  print(a.moveNext());      // false
  print(a.current);         // null
```
当下一指针存在时，`moveNext` 返回 `true`，否则返回 `false`。 `current` 返回当前指针的值，否则返回 `null`。  
> `Javascript` 中也存在 `Generator`函数，它没有同步异步之说，返回的不是 `Iterable` 对象，而是 `Iterator`。  
`Iterator` 的 `next()` 方法可以传值，配合 `yield` 关键字 与 `Promise` ，可以简化 JS 中的异步流程， 也就是 JS 中 `async/await` 语法糖的前身。  

异步的生成器返回一个 `流（Stream）`。  
流是一种抽象的概念，它是一段时间内一系列的数据的集合。**它可以被监听转换**。  
举个形象的例子，蒸馏水的生产流程：  
- `流（data）`是流入水管中的水。
- 当经过 `滤器（filter）`时，`过滤` 掉大部分杂质。
- 经过蒸馏装置（map）时，水从液态 `转化` 成气态。
- 经过`冷凝器（merge）`时，蒸汽从气态 `凝结` 成液态。
- 流出管道`（listen）`。  

在 `Dart` 语言中，流可以接收多个异步操作的结果，常用于会多次读取数据的异步任务场景，如网络内容下载、文件读写等。  
```dart
  Stream<int> natureTo(int n) async* {
    int k = 0;
    while (k < n) yield k++;
  }

  final a = natureTo(3);

  Future<void> printStream(Stream<int> intStream) async {
    await for(var i in intStream) {
      print(i);     // 0, 1, 2
    }
  }

  printStream(a);
```

---

### 2\. 单订阅流与多播流（Single-Subscription & Broadcast）  
Dart 中的流有两种，一种是单订阅流，一种是多播流。  
#### 1). 单订阅流（Single-Subscription Stream）  
**单订阅流在它整个生命周期中只允许被一个监听者监听一次**。在没有监听者监听时，它并不产生值，只有在被监听后，才会产出值。当监听者取消监听后，停止产生值，哪怕依然存在可以产出的值。  
监听同一个单订阅流两次以上是不被允许的，哪怕在前一个监听者取消监听后再去监听也是不允许的。如：  
```dart
  final Stream<String> testStream = Stream.periodic(
    Duration(seconds: 1),
    (int i) => 'event$i'
  );

  final StreamSubscription<String> testSubscription1 = testStream.listen(print);
  StreamSubscription<String> testSubscription2;

  Future.delayed(Duration(seconds: 2), () {
    testSubscription1.cancel();
    testSubscription2 = testStream.listen(print);
  });

  Future.delayed(Duration(seconds: 4), () {
    testSubscription2.cancel();
  });

  // output
  // event0
  // event1
  // Uncaught Error: Bad state: Stream has already been listened to.
  // Uncaught Error: NoSuchMethodError: method not found: 'cancel$0' on null
```
上例中，在取消了第一次监听打算开始第二次监听时，会报错说 `流已经被监听了`，而在 `4s` 时，因为第二次没有监听成功，所以会报出 `onCancel` 在 `null` 上不存在的错误。  
单订阅流一般用于大模块数据的传输，比如 `file` 系统的 `I/O`。

#### 2). 多播流（Broadcast Stream）  
多播流准许任意数量的监听者，且无论是否有监听者，它都能产生值。如果在流已经被监听的途中加入另外一个监听者，那么新加入的监听者只能接收到该流后续产出的值。  
```dart
  final Stream<String> testStream = Stream.periodic(Duration(seconds: 1), (int i) => 'event $i').asBroadcastStream();

  final StreamSubscription<String> testSubscription1 = testStream.listen((value) => print('listener1: $value'));

  StreamSubscription<String> testSubscription2;

  Future.delayed(Duration(seconds: 4), () {
    testSubscription2 = testStream.listen((value) => print('listener2: $value'));
  });

  Future.delayed(Duration(seconds: 8), () {
    testSubscription1.cancel();
    testSubscription2.cancel();
  });

  // output
  // listener1: event 0
  // listener1: event 1
  // listener1: event 2
  // listener1: event 3
  // listener1: event 4
  // listener2: event 4
  // listener1: event 5
  // listener2: event 5
  // listener1: event 6
  // listener2: event 6
  // listener1: event 7
  // listener2: event 7
```
第二个监听者在 `4s` 时开始监听，所以此时它收到第一个值是从 `4` 开始的。  
在上例中，使用了 `asBroadcastStream()` 方法，它可以将单订阅流转成多播流。  
每个监听者对于多播流的监听都可以看做对一个新流的监听（当然已经产出的值除外），因为每一个监听者对流的控制都是独立的，不会相互影响的。  
比如，两个监听者同时监听了一个多播流，其中的一个监听者对该流进行了 `暂停（Pause）` 操作，但是另一个监听者的监听行为依然在继续着。
```dart
  final Stream<String> testStream = Stream.periodic(Duration(seconds: 1), (int i) => 'event $i').asBroadcastStream();

  final StreamSubscription<String> testSubscription1 = testStream.listen((value) => print('listener1: $value'));

  final StreamSubscription<String> testSubscription2 = testStream.listen((value) => print('listener2: $value'));

  Future.delayed(Duration(seconds: 4), () => testSubscription2.pause());

  Future.delayed(Duration(seconds: 6), () => testSubscription2.resume());

  Future.delayed(Duration(seconds: 8), () {
    testSubscription1.cancel();
    testSubscription2.cancel();
  });

  // output
  // listener1: event 0
  // listener2: event 0
  // listener1: event 1
  // listener2: event 1
  // listener1: event 2
  // listener2: event 2
  // listener1: event 3
  // listener2: event 3
  // listener1: event 4
  // listener1: event 5
  // listener2: event 4
  // listener2: event 5
  // listener1: event 6
  // listener2: event 6
  // listener1: event 7
  // listener2: event 7
```
`4s` 时，`listener2` 暂停了对流的监听，此时它所监听的流的值开始进入缓冲，而 `listener1` 并没有受到影响，依然继续接收值。  
`6s` 时，`listnener2` 恢复了对流的监听，此时缓冲区的值瞬间被监听者接收，在控制台打印日志，而后面则跟 `listener1` 一起接收后面的值。  

当 `流（Stream）` 完成了所有的值得产出后，会发出 `done` 事件，然后结束该流，流就没有监听者了。在流结束后依然可以监听流，只不过会立刻接收到 `done` 事件而已。

如果要在 `Stream` 类上继承多播流，记得重写 `isBroadcast` 属性，因为默认的值是 `false`。

---

### 3\. 创建流（Create Stream）  
创建流有三种方法，分别是创建 `async*/yield/yield* 函数`， `Stream 的构造方法`， `StreamController`。  

#### 1). `async* yield/yield* 函数`  
使用 `async*` 函数创建流，如下所示：  
```dart
  Stream<int> natureToThree() async* {
    yield 1;
    yield 2;
    yield 3;
  }

  final a = natureToThree();
  a.listen(print);      // 1, 2, 3
```
上例中，在函数体前加上关键字 `async*`，在函数体内部则使用关键字 `yield` 返回每一个需要返回的值，然后执行这个函数，函数就会返回一个流。**而这个流在被监听前都没有执行**，只有在被监听时，这个流才开始执行。  
`yield*` 关键字允许在流中执行另外的一个流（当然在同步迭代器函数中执行的是一个可迭代对象）：  
```dart
  Stream<int> natureFourToFive() async* {
    yield 4;
    yield 5;
  }

  Stream<int> natureToFive() async* {
    yield 1;
    yield 2;
    yield 3;
    yield* natureFourToFive();
  }

  final a = natureToFive();
  a.listen(print);      // 1, 2, 3, 4, 5

  Iterable<int> natureTo() sync* {
    yield 1;
    yield 2;
    yield 3;
    yield* [4, 5, 6];
  }

  final b = natureTo();
  b.forEach(print);     // 1, 2, 3, 4, 5, 6
```

#### 2). Stream 的构造方法  
`Stream` 有很多构造方法，这里仅列出几种常用的：  
- `Stream<T>.value(T value)`  
创建一个产出单个值的单订阅流，当值被产出时，这个流也 `宣告结束（Completed）`。  
```dart
  Stream<String>.value('test').listen(print);       // test
```
- `Stream<T>.periodic(Duration period, [T computation(int computationCount)])`  
创建一个根据周期自动产出值的单订阅流。值是通过传入的回调函数的返回值产出的，而回调函数的参数是从 `0` 开始的 `int` 类型值。如果不传入回调的话，会一直返回 `null`。
```dart
  Stream<String>.periodic(
    Duration(seconds: 1),
    (value) {
      print(value.runtimeType);     // init, init, ...
      return value.toString();      // 0, 1, ...
    }
  ).listen(print);
```
- `Stream<T>.fromIterable(Iterable<T> elements)`  
创建一个通过传入的可迭代对象产出值的单订阅流。该流在被监听时开始产出值，在被取消监听，或者可迭代对象的 `Iterator.moveNext()` 方法返回 `false` 甚至报错时，停止执行。可以通过 `StreamSubscription` 对象的 `pause` 方法，挂起产出的执行。  
```dart
  final testStream = Stream<String>.fromIterable(['Just', 'A', 'Test']);
  testStream.listen(print);     // Just, A, Test
```
- `Stream<T>.fromFuture(Future<T> future)`  
创建一个根据传入的 `future` 产出值的流。当 `future` 变为 `Completed` 状态时，该流则产出这个值（不管是 `Completed(Value)` 还是 `Completed(Error)`），然后该流结束。  
```dart
  final testStream = Stream<String>.fromFuture(
      Future.delayed(Duration(seconds: 2), () => 'future')
  );
  testStream.listen(print);     // future
```
- `Stream<T>.fromFutures(Iterable<Future<T>> futures)`  
通过一组 `future` 来创建一个单订阅流。该流根据 `future` 的完成顺序（即变为 `Completed` 状态，`value` 或者 `error`）来产出值。当所有 `future` 都完成时，流也变为 `Completed` 并关闭。如果 `futures` 是空的，则流立即关闭。
```dart
  final testStream = Stream<String>.fromFutures([
    Future.delayed(Duration(seconds: 1), () => 'future1'),
    Future.delayed(Duration(seconds: 3), () => 'future2'),
    Future.delayed(Duration(seconds: 2), () => 'future3'),
    Future.delayed(Duration(seconds: 4), () => 'future4'),
    Future.value('future5'),
    Future.value('future6')
  ]);
  testStream.listen(print);     // future5, future6, future1, future3, future2, future4
```

#### 3). StreamController 创建流  
`Dart` 提供了 `StreamController` 来创建流，使用 `StreamController` 创建的流可以自己来控制数据产出的时机。  
```dart
  final StreamController<int> testStreamController = StreamController();

  final subscription = testStreamController.stream.listen(print);   // 1, 2

  testStreamController.add(1);
  testStreamController.add(2);
```
如上例，通过调用 `StreamController` 提供的 `add` 方法，可以更新 `流的值`。  
此处可以看出，`Dart` 中的流具有 `事件机制（观察者模式）`，当触发事件时，`监听者可以更新自身的状态`。`Flutter` 状态管理库 `Bloc`，底层的实现机制就是利用流的事件机制。  

> StreamController 的构造方法  
```dart
  StreamController({
    void onListen(),
    void onPause(),
    void onResume(),
    void onCancel(),
    bool sync: false
  })
```
- `StreamController`：创建一个单订阅流，只能被一个监听者监听。  
- `onListen`：当有监听者监听该流时被调用。  
- `onCancel`：监听者取消监听该流时被调用。  
- `onPause`：监听者暂停对流的监听时被调用。  
- `onResume`：监听者恢复流的暂停状态时被调用。  
- `sync`：流是否是同步的，默认为 `false`。  

```dart
  StreamController.broadcast({
    void onListen(),
    void onCancel(),
    bool sync: false
  })
```
多播流中不存在 `onPause` 和 `onResume` 方法。并且 `onListen` 是在第一次被监听时触发，同样的， `onCancel` 是在最后一个监听者取消监听后触发。  
在多播流中，每一个监听者的订阅都是独立运行的，比如对该流的一个订阅者使用 `cancel` 方法，取消的只是当前的订阅，而其他的订阅则不会受到影响。  
```dart
  // create a broadcast stream
  final StreamController<int> testStreamController = StreamController.broadcast();

  // subscribe the stream
  final test1 = testStreamController.stream.listen((value) {
    print('test1');
    print(value);
  });
  final test2 = testStreamController.stream.listen((value) {
    print('test2');
    print(value);
  });
  StreamSubscription test3;

  Future.delayed(Duration(seconds: 1), () => testStreamController.add(1));
  Future.delayed(Duration(seconds: 2), () => testStreamController.add(2));
  Future.delayed(Duration(seconds: 2), () {
    test1.cancel();
    test3 = testStreamController.stream.listen((value) {
      print('test3');
      print(value);
    });
  });
  Future.delayed(Duration(seconds: 3), () => testStreamController.add(3));
  Future.delayed(Duration(seconds: 4), () => testStreamController.add(4));
  Future.delayed(Duration(seconds: 5), () {
    test2.cancel();
    test3.cancel();
    testStreamController.close();
  });

  // output
  // test1
  // 1
  // test2
  // 1
  // test1
  // 2
  // test2
  // 2
  // test2
  // 3
  // test3
  // 3
  // test2
  // 4
  // test3
  // 4
```
如上例，在一开始 `test1` 和 `test2` 对 `testStreamController` 中的 `stream` 进行订阅，两秒后，`test1` 取消了对 `stream` 的订阅，`test3` 则在此时对 `stream` 进行了订阅。而此时 `test2` 依然对 `stream` `正常的进行着监听。test3` 因为在 `2s` 后才对 `stream` 进行监听，所以它只能接收到 `2s` 以后 `stream` 中发出的值。  

> StreamController 常用的属性和方法  
- `StreamController.stream`  
`stream` 是只读属性，它返回这个 `controller` 所控制的 `stream。`  
- `StreamController.add(T event)`  
发送一个 `data` 事件，监听者将在下一个 `微任务（MicroTask）`收到这个事件。  
- `StreamController.addStream(Stream<T> source, { bool cancelOnError })`  
接收 `source` 中的 `data` 并将它们放入当前的 `controller` 中，只有当 `source` 发送了 `done` 事件之后，才能使用 `StreamController` 中的其它方法。返回一个 `Future`。  
```dart
  final Stream<int> testStream = Stream.periodic(Duration(seconds: 1), (int value) => value).take(3);

  StreamController<int> streamCtrl;
  streamCtrl = StreamController.broadcast(
    onCancel: () {
      print(streamCtrl.hasListener);
      streamCtrl.close();
    }
  );
  final lsn1 = streamCtrl.stream.listen((v) => print('lsn1: $v'));

  streamCtrl.add(1);
  streamCtrl.add(2);

  final Future<dynamic> isCompleted = streamCtrl.addStream(testStream);

  // streamCtrl.add(5);
  // streamCtrl.add(6);

  isCompleted.then((value) {
    print('value: $value');
    streamCtrl.add(7);
    streamCtrl.add(8);
  });

  Timer(Duration(seconds: 10), () => lsn1.cancel());
```
此处，如果将注释打开将会报错说在 `addStream` 未完成时无法添加其他 `data`。正确的使用方法应该是在 `Future` 后继续添加，或者使用 `async` 函数去添加。
- `StreamController.close()`  
发送一个 `done` 事件，并关闭该 `stream。`  

---

### 4\. 监听流（Listen Stream）  
`Dart` 中，想要监听流的变化有两种方式：  
- 使用 `await for/in` 来接收流数据  
- 使用 `listen()` 方法监听流的变化  

#### 1). await for/in  
使用循环的方式可以接收流的数据：  
```dart
  final Stream<int> testStream = Stream.fromIterable([1, 2, 3]);

  Future<void> recieveData(Stream<int> stream) async {
    await for(var data in stream) {
      print(data);
    }
  }

  final result = recieveData(testStream);
  print(result);
  Future.delayed(Duration(seconds: 0), () => print(result));

  // output
  // Instance of '_Future<void>'
  // 1
  // 2
  // 3
  // Instance of '_Future<void>'
```
在函数体前添加关键字 `async`，而在正常的 `for` 循环前加上 `await` 关键字，返回一个 `Future` 对象。每一次循环都将获取一个流中的 `data`。当流中的所有数据都被产出之后，也就是流结束时，循环就会退出。  

#### 2). listen()  
监听一个流最常用的方式就是使用 `Dart` 提供的 `listen` 方法。  
```dart
  StreamSubscription<T> listen(
    void onData(T event),
    {
      Function onError,
      void onDone,
      bool cancelOnError
    }
  )
```
`listen` 方法返回一个 `Subscription` 对象，可以通过它来对流进行解除监听，`暂停/恢复` 流的运行等功能。  
`listen` 方法拥有一个必须的参数 `onData()` 回调，以它来接收流中产出的值。如果给 `onData` 回调设置为 `null`，则将不再触发产出值的监听。其余三个可选参数 `onError` 在收到错误时触发，`onDone` 在流完成时触发，`cancelOnError` 表示的是当该流接收到第一个 `Error` 时，是否继续监听流，默认是 `false。`  

---

### 5\. 取消 & 关闭（Cancel & Close）  
#### 1). 取消
当使用 `listen()` 方法对流进行监听时， 流会返回一个 `StreamSubscription` 对象，可以使用此对象的 `cancel()` 方法对流的监听进行取消。  
而根据流种类的不同（单订阅流和多播流），取消的效果也不一样。
- 单订阅流  
> 当监听者取消监听时，流停止产生值，即使它依然可以产生更多的值。  
- 多播流  
> 多播流准许多个监听者监听，且无论是否有监听者，它都会产生值。当流发出 done 事件时，监听者们在接收到该事件之前取消对流的监听。  

1. 流在没有被监听前是不产出值的。  
2. 流被监听后，不管是否有监听者，它依然在产出新的值，直到该流结束。

```dart
  Stream<int> testStream = Stream.periodic(Duration(seconds: 1), (value) => value).take(20).asBroadcastStream();

  StreamSubscription subscription1 = testStream.listen((v) {
    print('lsn1: $v');
  });
  StreamSubscription subscription2 = testStream.listen((v) {
    print('lsn2: $v');
  });
  StreamSubscription subscription3;

  Timer(Duration(seconds: 5), () => subscription1.cancel());

  Timer(Duration(seconds: 10), () => subscription2.cancel());

  Timer(Duration(seconds: 18), () => subscription3 = testStream.listen((v) {
    print('lsn3: $v');
  }));

  Timer(Duration(seconds: 22), () => subscription3.cancel());
```
由上例可见，多播流在被监听过一次以后，哪怕没有监听者，其依然在内存中继续运行，直到流结束。  

#### 2). 关闭  
普通方法创建的流是没有关闭方式的，关闭的方法只存在于 `StreamController`。
```dart
  StreamController<int> streamCtrl;
  streamCtrl = StreamController.broadcast(
    onCancel: () {
      print(streamCtrl.hasListener);
      streamCtrl.close();
    }
  );
  final lsn1 = streamCtrl.stream.listen((v) => print('lsn1: $v'));
  final lsn2 = streamCtrl.stream.listen((v) => print('lsn2: $v'));

  streamCtrl.add(1);
  streamCtrl.add(2);

  Timer(Duration(seconds: 4), () {
    lsn1.cancel();
    streamCtrl.add(3);
    streamCtrl.add(4);
  });

  Timer(Duration(seconds: 8), () {
    lsn2.cancel();
    streamCtrl.add(5);
  });
```
上例中，在创建 `StreamController` 时，为 `onCancel` 事件添加函数，使流在没有监听者时，自己关闭。当流关闭后，就无法再产生值。

