Node 是为构建实时 Web 应用而诞生的，可以让 JavaScript 运行在服务端的平台。它具有事件驱动、单线程、异步 I/O 等特性。这些特性不仅带来了巨大的性能提升，有效的解决了高并发问题，还避免了多线程程序设计的复杂性。

本文主要讨论的是 Node 中实现异步 I/O 和事件驱动程序设计的基础——事件循环。

## 事件循环是 Node 的执行模型

事件循环是 Node 自身的执行模型，Node 通过事件循环的方式运行 JavaScript 代码（初始化和回调），并提供了一个线程池处理诸如文件 I/O 等高成本任务。

在 Node 中，有两种类型的线程：一个事件循环线程（也称为主循环、主线程、事件线程等），它负责任务的编排；另一个是工作线程池中的 K 个工作线程（也被称为线程池），它专门处理繁重的任务。

---

_看到这里，可能有同学会有疑问，文章开头说 Node 是单线程的，为什么又存在两种类型的线程呢？_
_事实上，Node 的单线程指的是自身 JavaScript 运行环境的单线程，Node 并没有给 JavaScript 执行时创建新线程的能力，最终的操作，是通过底层的 libuv 及其带来的事件循环来执行的。这也是为什么 JavaScript 作为单线程语言，能在 Node 中实现异步操作的原因。两者并不冲突。_
_下图展示了异步 I/O 中线程的调用模型，可以看到，对于主线程来说，一直都是单线程执行的。_
![image](https://user-images.githubusercontent.com/9818716/60763860-53f81900-a0af-11e9-9fc3-98b7ee19ab28.png)
_【参考文章：[Node.js 探秘：初识单线程的 Node.js — 凌恒](http://taobaofed.org/blog/2015/10/29/deep-into-node-1/)】_

---

Node 的工作线程池是在 libuv 中实现的，它对外提供了通用的任务处理 API — `uv_queue_work`。
![image](https://user-images.githubusercontent.com/9818716/60763971-2eb8da00-a0b2-11e9-8c81-bf8d42e8f291.png)
工作线程池被用于处理一些高成本任务。包括一些操作系统并没有提供非阻塞版本的 I/O 操作，以及一些 CPU 密集型任务，如：

1. I/O 密集型任务
    - DNS：用于 DNS 解析的模块，`dns.lookup()`, `dns.lookupService()` 
    - 文件系统：所有文件系统 API，除了 `fs.FSWatcher()` 和显式调用 API 如 `fs.readFileSync()` 之外
2. CPU 密集型任务
    - Crypto：用于加密的模块
    - Zlib：用于压缩的模块，除了那些显式同步调用的 API 之外

当调用这些 API 时，会进入对应 API 与 C++ 桥接通讯的 Node C++ binding 中，从而向工作线程池提交一个任务。为了达到较高性能，处理这些任务的模块通常都由 C/C++ 编写。

![image](https://user-images.githubusercontent.com/9818716/60763984-87887280-a0b2-11e9-98b0-1e81f33a1152.png)

上图描述了 Node 的运行原理，从左到右，从上到下，Node 被分为了四层：

- 应用层。JavaScript 交互层，常见的就是 Node 的模块，如 http，fs
- V8 引擎层。利用 V8 来解析 JavaScript 语法，进而和下层 API 交互
- Node API 层。为上层模块提供系统调用，和操作系统进行交互
- libuv 层。跨平台的底层封装，实现了事件循环、文件操作等，是 Node 实现异步的核心。它将不同的任务分配给不同的线程，形成一个事件循环（event loop），以异步的方式将任务的执行结果返回给 V8 引擎

## 基于事件循环可以构造高性能服务器

经典的服务器模型有以下几种：

- 同步式。一次只能处理一个请求，其他请求都处于等待状态。
- 每进程/每请求。会为每个请求启动一个进程，这样就可以同时处理多个请求，但由于系统资源有限，不具备扩展性。
- 每线程/每请求。会为每个请求启动一个线程，虽然线程比进程轻量，但是对于大型站点而言，依然不够。因为每个线程都要占用一定内存，当大并发请求到来时，内存将会很快耗光。
- 事件驱动。通过事件驱动的方式处理请求，无需为每个请求创建额外的线程，可以省去创建和销毁线程的开销，同时操作系统在调度任务时因为线程较少，上下文切换的代价较低。这种模式被很多平台所采用，如 Nginx(C)、Event Machine(Ruby)、AnyEvent(Perl)、Twisted(Python)，以及本文讨论的 Node。

事件驱动的实质，是通过主循环加事件触发的方式来运行程序，这种执行模型被称为事件循环。通过事件驱动，可以构建高性能服务器。

既然很多平台都采用了事件驱动的模式，为什么 Ryan Dahl 偏偏选了 JavaScript 呢？在开发 Node 时，Ryan Dahl 曾经评估过多种语言。最终结论为：C 的开发门槛高，可以预见不会有太多开发者将其作为日常的业务开发；Lua 自身已经包含很多阻塞 I/O 库，为其构建非阻塞 I/O 库也无法改变人们继续使用阻塞 I/O 库的习惯；Ruby 的虚拟机性能不够高。相比之下，JavaScript 比 C 开发门槛低，比 Lua 历史包袱少，在浏览器中已经有广泛的事件驱动应用，V8 引擎又具有超高性能，于是，Javascript 就成为了 Node 的开发语言。

例如，使用 Node 进行数据库查询：

```
db.query('SELECT * from some_table', function(res) {
  res.output();
});
```

进程在执行到db.query 的时候，不会等待结果返回，而是直接继续执行后面的语句，直到进入事件循环。当数据库查询结果返回时，会将事件发送到事件队列，等到线程进入事件循环以后，才会调用之前的回调函数继续执行后面的逻辑。

当然，这种事件驱动开发模式的弊端也是显而易见的，它不符合常规的线性思路，需要把一个完整的逻辑拆分为一个个事件，增加了开发和调试的难度。

下面是多线程阻塞式 I/O 和单线程事件驱动的异步式 I/O 的对比：

![image](https://user-images.githubusercontent.com/9818716/60769378-79ae0e00-a101-11e9-92e9-ba9b3d530c99.png)

_【表格来自于 《Node.js 开发指南》 — byvoid】_

## 基于事件循环可以实现异步任务调度

事件循环的使用场景可以分为异步 I/O 和非 I/O 的异步操作两种。

异步 I/O 的主旨是使 I/O 操作与 CPU 操作分离，从而非阻塞的调用底层接口。如前所述，常见的使用场景有网络通信、磁盘 I/O、数据库访问等。当然，Node 也提供了部分同步 I/O 方式，如`fs.readFileSync`，但 Node 并不推荐用户使用它们。

非 I/O 的异步操作有定时器，如 `setTimeout`、`setInterval`，以及 `process.nextTick`、`setImmediate`、`promise`。

`setTimeout`、`setInterval`、`promise` 与浏览器中的 API 一致，在此不再赘述。

`process.nextTick` 的功能是为事件循环设置一项任务， Node 会在下一轮事件循环时调用 callback。

为什么不能在当前循环执行完这项任务，而要交给下次事件循环呢？我们知道，一个 Node 进程只有一个主线程，在任何时刻都只有一个事件在执行。如果这个事件占用大量 CPU 时间，事件循环中的下一个事件就要等待很久。使用 process.nextTick() 可以把复杂的工作拆散，变成一个个较小的事件。例如：

```javascript
function doSomething(args, callback) {
  somethingComplicated(args);
  process.nextTick(callback);
}
doSomething(function onEnd() {
  compute();
});
```

假设 `compute()` 和 `somethingComplicated()` 是两个较为耗时的函数，调用 `doSomething()` 时会先执行 `somethingComplicated()`，如果不使用 `process.nextTick`，会立即调用回调函数，在 `onEnd()` 中会执行 `compute()`，从而会占用较长 CPU 时间，阻塞其他事件的处理。而通过 `process.nextTick` 会把上面耗时的操作拆分至两次事件循环，减少了每个事件的执行时间，避免阻塞其他事件。

另外，需要注意的是，虽然定时器也能将任务拆分至下一次事件循环处理，但并不建议用其代替 `process.nextTick(fn)`，因为定时器的处理涉及到最小堆操作，时间复杂度为 `O(lg(n))`，而 `process.nextTick` 只是把回调函数放入队列之中，时间复杂度为 `O(1)`，更加高效。

`setImmediate()` 和 `process.nextTick()` 类似，也是将回调函数延迟执行。不过 `process.nextTick` 会先于 `setImmediate` 执行。因为 `process.nextTick` 属于 microtask，会在事件循环之初就执行；而 `setImmediate` 在事件循环的 check 阶段才会执行。这部分将在下一小节详述。

## 事件循环的执行机制

Node 中的事件循环是在 libuv 中实现的，libuv 在 Node 中的地位如下图：

![image](https://user-images.githubusercontent.com/9818716/60803034-d9adbe80-a1ac-11e9-8d8f-d5a00625d958.png)

_【图片来自《深入浅出 Node.js》 — 朴灵】_

Node 不是一个从零开始开发的 JavaScript 运行时，它是“站在巨人肩膀上”进行一系列拼凑和封装得到的结果。V8（Chrome V8）是 Node 的 JavaScript 引擎，由谷歌开源，以 C++ 编写，具有高性能和跨平台的特性，同时也用于 Chrome 浏览器。libuv 是专注于异步 I/O 的跨平台类库，实际上它主要就是为 Node 开发的。基于不同平台的异步机制，如 epoll / kqueue / IOCP / event ports，libuv 实现了跨平台的事件循环。作为一个在操作系统之上的中间层，libuv 使开发者不用自己管理线程就能轻松的实现异步。