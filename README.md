# flutter_develop_tips
### 什么是 Isolate
Flutter Isolate 是 Flutter 框架提供的一种并发编程的机制，用于在应用程序中创建多个独立的执行线程，以便在后台执行一些耗时的计算任务、IO 操作或其他需要并行处理的任务。Flutter Isolate 的主要作用包括：

* 允许在 Flutter 应用程序中并发执行代码，提高应用程序的响应性和性能；
* 可以将一些耗时的计算任务或 IO 操作放在后台线程中执行，避免阻塞主线程，从而保证应用程序的流畅性；
* 可以实现跨平台的并发编程，不受操作系统的限制；
* 可以将应用程序的不同功能模块拆分成独立的 Isolate，提高代码的可维护性和可扩展性；
* 可以通过 Isolate 之间的通信机制实现数据共享和协作，提高应用程序的灵活性和可扩展性。

### Isolate 和 future 区别
Flutter 中的 Isolate 和 Future 都涉及到并发编程，但它们是不同的概念。

* Isolate 是一种轻量级的线程，它可以独立于主线程运行。Isolate 可以在自己的内存空间中运行代码，可以运行 CPU 密集型任务而不会阻塞主线程。Isolate 之间是相互独立的，它们不能直接共享状态，但可以通过消息传递进行通信。在 Flutter 中，您可以使用 compute() 方法或 Isolate.spawn() 函数来创建 Isolate 并在其中执行代码。
* Future 是一种异步编程的概念，它代表一个可能在未来完成的操作。Future 允许您在进行耗时操作时不会阻塞 UI 线程，以保持应用程序的响应性。Future 通常用于在后台执行 I/O 操作或从网络获取数据。在 Flutter 中，您可以使用 async/await 语法或 .then() 方法来处理 Future。

### 为什么不用 Future

虽然 Future 是 Flutter 中处理异步操作的一种方式，但在某些情况下使用 Isolate 可以更好地满足特定的需求，而不是使用 Future。以下是一些可能使用 Isolate 而不是 Future 的情况：

* 执行 CPU 密集型任务：如果您需要在后台执行 CPU 密集型任务，例如图像处理或音频处理，那么使用 Isolate 会更好，因为它可以在独立的线程中运行，而不会阻塞 UI 线程。使用 Future 可能会导致 UI 线程被阻塞，从而影响应用程序的响应性。

* 处理大量数据：如果您需要处理大量的数据，例如从网络获取大型数据集，那么使用 Isolate 可以更好，因为它可以在独立的线程中处理数据，而不会阻塞UI线程。使用 Future 可能会导致 UI 线程被阻塞，从而影响应用程序的响应性。

* 并行处理多个任务：如果您需要同时处理多个任务，例如同时下载多个文件或同时处理多个图像，那么使用 Isolate 可以更好，因为它可以在独立的线程中同时处理多个任务，而不会阻塞 UI 线程。使用 Future 可能需要串行处理多个任务，从而导致应用程序的响应性变差。

### isolate 实现

在 Flutter 中使用 Isolate 进行并发编程需要一些步骤。

下面是一个简单的示例，演示如何使用 Isolate 执行计算任务：

```
import 'dart:isolate';

void main() async {
  ReceivePort receivePort = ReceivePort();
  Isolate isolate = await Isolate.spawn(worker, receivePort.sendPort);

  receivePort.listen((message) {
    print('Received result: $message');
    receivePort.close();
    isolate.kill(priority: Isolate.immediate);
  });

  await Future.delayed(Duration(seconds: 1));

  int result = await computeFactorial(10);
  isolate.kill(priority: Isolate.immediate);
  print('Computed result: $result');
}

void worker(SendPort sendPort) {
  ReceivePort receivePort = ReceivePort();
  sendPort.send(receivePort.sendPort);

  receivePort.listen((message) {
    int number = message as int;
    int result = computeFactorial(number);
    sendPort.send(result);
  });
}

int computeFactorial(int number) {
  if (number <= 0) {
    return 1;
  } else {
    return number * computeFactorial(number - 1);
  }
}
```

在这个示例中，我们首先使用 Isolate.spawn() 函数创建了一个新的 Isolate，并在其中执行 worker() 函数。worker() 函数接收一个 SendPort，并创建一个 ReceivePort 用于监听消息。在 main() 函数中，我们创建了一个 ReceivePort 用于监听来自 Isolate 的消息，并将其发送给 Isolate 的 SendPort。然后，我们使用 await Future.delayed() 函数等待一秒钟，以便 Isolate 有足够的时间计算结果。最后，我们终止 Isolate 并输出计算结果。

在 worker() 函数中，我们监听来自 ReceivePort 的消息，并将其作为整数传递给 computeFactorial() 函数。然后，我们将计算结果发送回主 Isolate 的 SendPort。

computeFactorial() 函数是一个简单的阶乘计算函数，它通过递归方式计算阶乘。

总之，使用 Isolate 进行并发编程需要创建一个新的 Isolate 并在其中执行任务。在 Flutter 中，您可以使用 Isolate.spawn() 函数创建 Isolate，并使用 SendPort 和 ReceivePort 进行消息传递。

### flutter_isolate 的使用

https://pub-web.flutter-io.cn/packages/flutter_isolate

这个插件的好处就是，简化了我们的代码，而且还提供了丰富的 api 让我们更好的进行 isolate 操作。

准备：
pubspec.yaml

```
dependencies:
  ...

  flutter_isolate: ^2.0.4
  path_provider: ^2.0.15
  flutter_downloader: ^1.10.4
```

#### 第一步：简单调用

isolate 不支持热更新，需要重新启动调试

```
@pragma('vm:entry-point')
void someFunction(String arg) {
  print("Running in an isolate with argument : $arg");
}

ElevatedButton(
  child: const Text('01 - 简单的执行顶层函数'),
    onPressed: () {
      FlutterIsolate.spawn(someFunction, "hello world");
    },
),
```

#### 第二步：带返回值

```
@pragma('vm:entry-point')
Future<int> expensiveWork(int arg) async {
  int result = arg + 100;
  return result;
}

ElevatedButton(
  child: const Text('02 - Compute函数'),
    onPressed: () {
      var res = await flutterCompute(expensiveWork, 123);
      print(res);
    },
),
```

#### 第三步：嵌套调用

准备一个显示下载进度的组件

```
class MarkdownView extends StatelessWidget {
  const MarkdownView({super.key});

  static void downloaderCallback(String id, int status, int progress) {
    print("progress: $progress");
  }

  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

下载线程函数

```
@pragma('vm:entry-point')
void isolate2(String arg) {
  getTemporaryDirectory().then((dir) async {
    print("isolate2 temporary directory: $dir");

    await FlutterDownloader.initialize(debug: true);
    FlutterDownloader.registerCallback(MarkdownView.downloaderCallback);

    final taskId = await FlutterDownloader.enqueue(
        url:
            "https://raw.githubusercontent.com/rmawatson/flutter_isolate/master/README.md",
        savedDir: dir.path);
  });
  Timer.periodic(const Duration(seconds: 1),
      (timer) => print("Timer Running From Isolate 2"));
}
```

第一个线程函数，嵌套执行上面的线程函数

```
@pragma('vm:entry-point')
void isolate1(String arg) async {
  await FlutterIsolate.spawn(isolate2, "hello2");

  getTemporaryDirectory().then((dir) {
    print("isolate1 temporary directory: $dir");
  });
  Timer.periodic(const Duration(seconds: 1),
      (timer) => print("Timer Running From Isolate 1"));
}
```

执行线程函数，进行控制 暂停、恢复、杀死

```
ElevatedButton(
      child: const Text('03 - isolate 嵌套执行'),
      onPressed: () async {
        final isolate = await FlutterIsolate.spawn(isolate1, "hello");
        // // 5 秒后暂停
        // Timer(const Duration(seconds: 5), () {
        //   print("Pausing Isolate 1");
        //   isolate.pause();
        // });

        // // 10 秒后恢复
        // Timer(const Duration(seconds: 10), () {
        //   print("Resuming Isolate 1");
        //   isolate.resume();
        // });

        // // 20 秒后杀死
        // Timer(const Duration(seconds: 20), () {
        //   print("Killing Isolate 1");
        //   isolate.kill();
        // });
      },
    );
```

#### 第四步：全部关掉

如果一个界面关闭的时候，你需要全部关闭所有线程
```
ElevatedButton(
            child: const Text('04 - 杀掉全部'),
            onPressed: () async {
              await FlutterIsolate.killAll();
            },
          ),
```

### 小结

总之，尽管 Future 在处理异步操作方面是非常有用的工具，但在某些情况下，使用 Isolate 可以更好地满足特定的需求，例如执行 CPU 密集型任务、处理大量数据或并行处理多个任务。因此，选择使用 Future 还是 Isolate 取决于您的具体需求和场景。
