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

下图是官方文档中给出的 libuv 结构图：

![image](https://user-images.githubusercontent.com/9818716/60973805-829a1c00-a35b-11e9-84bf-66d546db2846.png)

可以看出，除了事件循环外，libuv 还提供了计时器、网络操作、文件操作、子进程等功能。

在 Node 中，就是直接使用 libuv 中的事件循环：

https://github.com/nodejs/node/blob/master/src/node.cc#L1059

![image](https://user-images.githubusercontent.com/9818716/60974007-dd337800-a35b-11e9-9444-8d1f39a7b493.png)

下面是 libuv 中事件循环的详细流程：

![image](https://user-images.githubusercontent.com/9818716/60974016-e15f9580-a35b-11e9-86de-80883b828616.png)

如上图所示，libuv 中的事件循环主要有 7 个阶段，它们按照执行顺序依次为：

- timers 阶段：这个阶段执行 `setTimeout` 和 `setInterval` 预定的回调函数；
- pending callbacks 阶段：这个阶段会执行除了 close 事件回调、被 timers 设定的回调、`setImmediate` 设定的回调之外的回调函数；
- idle、prepare 阶段：供 node 内部使用；
- poll 阶段：获取新的 I/O 事件，在某些条件下 node 将阻塞在这里；
- check 阶段：执行 `setImmediate` 设定的回调函数；
- close callbacks 阶段：执行 `socket.on('close', ...)` 之类的回调函数

除了 libuv 中的七个阶段外，Node 中还有一个特殊的阶段，它一般被称为 microtask，它由 V8 实现，被 Node 调用。包括了 `process.nextTick`、`Promise.resolve` 等微任务，它们会在 libuv 的七个阶段之前执行，而且 `process.nextTick` 的优先级高于 `Promise.resolve`。值得注意的是，在浏览器环境下，我们常说事件循环中包括宏任务（macrotask 或 task）和微任务（microtask），这两个概念是在 HTML 规范中制定，由浏览器厂商各自实现的。而在 Node 环境中，是没有宏任务这个概念的，至于前面所说的微任务，则是由 V8 实现，被 Node 调用的；虽然名字相同，但浏览器中的微任务和 Node 中的微任务实际上不是一个东西，当然，不排除它们间有相互借鉴的成分。

让我们通过 libuv 中控制事件循环的核心代码，近距离观察这几个阶段。在 libuv v1.x 版本中，事件循环的核心函数 `uv_run()` 分别在 `src/unix/core.c` 和 `src/win/core.c` 中：

https://github.com/libuv/libuv/blob/v1.x/src/unix/core.c#L348

```C
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

	// 判断事件循环继续还是启动新一轮循环
  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);  // timers 阶段
    ran_pending = uv__run_pending(loop);  // pending 阶段
    uv__run_idle(loop);  // idle 阶段
    uv__run_prepare(loop);  // prepare 阶段

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);  // poll 阶段
    uv__run_check(loop);  // check 阶段
    uv__run_closing_handles(loop);  // close 阶段

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

从上述代码中可以清楚的看到 timers、pending、idle、prepare、poll、check、close 这七个阶段的调用。下面，让我们详细看看这几个阶段。

### timers 阶段

https://github.com/libuv/libuv/blob/v1.x/src/timer.c#L159

```C
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min(timer_heap(loop));  // 最小堆
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)  // 如果遇到第一个还未到触发时间的事件回调，退出循环
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
```

可以看出，timers 阶段使用的数据结构是最小堆。这个阶段会在事件循环的一个 tick 中不断循环，把超时时间和当前的循环时间（`loop -> time`）进行比较，执行所有到期回调；如果遇到第一个还未到期的回调，则退出循环，不再执行 timers queue 后面的回调。

这里为什么用最小堆而不用队列？因为 timeout 回调需要按照超时时间的顺序来调用，而不是先进先出的队列逻辑。所以这里用了最小堆。

### pending 阶段

https://github.com/libuv/libuv/blob/v1.x/src/unix/core.c#L792

```C
static int uv__run_pending(uv_loop_t* loop) {
  QUEUE* q;
  QUEUE pq;
  uv__io_t* w;

  if (QUEUE_EMPTY(&loop->pending_queue))
    return 0;

  QUEUE_MOVE(&loop->pending_queue, &pq);

  while (!QUEUE_EMPTY(&pq)) {
    q = QUEUE_HEAD(&pq);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);
    w = QUEUE_DATA(q, uv__io_t, pending_queue);
    w->cb(loop, w, POLLOUT);
  }

  return 1;
}
```

这里使用的是队列。一些应该在上轮循环 poll 阶段执行的回调，如果因为某些原因不能执行，就会被延迟到这一轮循环的 pending 阶段执行。也就是说，这个阶段执行的回调都是上一轮残留的。

### idle、prepare、check 阶段

这三个阶段都由同一个函数定义

https://github.com/libuv/libuv/blob/v1.x/src/unix/loop-watcher.c#L48

```C
void uv__run_##name(uv_loop_t* loop) {                                      \
  uv_##name##_t* h;                                                         \
  QUEUE queue;                                                              \
  QUEUE* q;                                                                 \
  QUEUE_MOVE(&loop->name##_handles, &queue);                                \
  while (!QUEUE_EMPTY(&queue)) {                                            \
    q = QUEUE_HEAD(&queue);                                                 \
    h = QUEUE_DATA(q, uv_##name##_t, queue);                                \
    QUEUE_REMOVE(q);                                                        \
    QUEUE_INSERT_TAIL(&loop->name##_handles, q);                            \
    h->name##_cb(h);                                                        \
  }                                                                         \
}
```

这里用了宏以实现代码的复用，但同时也降低了可读性。这部分的逻辑和 pending 阶段很像，遍历队列，执行回调，直至队列为空。

### poll 阶段

https://github.com/libuv/libuv/blob/v1.x/src/unix/linux-core.c#L196

poll 阶段较为复杂，一共有 400+ 行代码，这里只截取部分，完整逻辑请自行查看源码。

```C
void uv__io_poll(uv_loop_t* loop, int timeout) {
	// ...
	// 处理观察者队列
  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    // ...
    if (w->events == 0)
      op = EPOLL_CTL_ADD;  // 新增监听事件
    else
      op = EPOLL_CTL_MOD;  // 修改事件
	
	// ...
  for (;;) {
    /* See the comment for max_safe_timeout for an explanation of why
     * this is necessary.  Executive summary: kernel bug workaround.
     */
		// 计算好 timeout 以防 uv_loop 一直阻塞
    if (sizeof(int32_t) == sizeof(long) && timeout >= max_safe_timeout)
      timeout = max_safe_timeout;

    nfds = epoll_pwait(loop->backend_fd,
                       events,
                       ARRAY_SIZE(events),
                       timeout,
                       psigset);

    /* Update loop->time unconditionally. It's tempting to skip the update when
     * timeout == 0 (i.e. non-blocking poll) but there is no guarantee that the
     * operating system didn't reschedule our process while in the syscall.
     */
    SAVE_ERRNO(uv__update_time(loop));

    if (nfds == 0) {
      assert(timeout != -1);

      if (timeout == 0)
        return;

      /* We may have been inside the system call for longer than |timeout|
       * milliseconds so we need to update the timestamp to avoid drift.
       */
      goto update_timeout;
    }

    if (nfds == -1) {
      if (errno != EINTR)
        abort();

      if (timeout == -1)
        continue;

      if (timeout == 0)
        return;

      /* Interrupted by a signal. Update timeout and poll again. */
      goto update_timeout;
    }

    have_signals = 0;
    nevents = 0;

    assert(loop->watchers != NULL);
    loop->watchers[loop->nwatchers] = (void*) events;
    loop->watchers[loop->nwatchers + 1] = (void*) (uintptr_t) nfds;
    for (i = 0; i < nfds; i++) {
      pe = events + i;
      fd = pe->data.fd;

      /* Skip invalidated events, see uv__platform_invalidate_fd */
      if (fd == -1)
        continue;

      assert(fd >= 0);
      assert((unsigned) fd < loop->nwatchers);

      w = loop->watchers[fd];

      if (w == NULL) {
        epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, pe);
        continue;
      }
      pe->events &= w->pevents | POLLERR | POLLHUP;
      if (pe->events == POLLERR || pe->events == POLLHUP)
        pe->events |=
          w->pevents & (POLLIN | POLLOUT | UV__POLLRDHUP | UV__POLLPRI);

      if (pe->events != 0) {
        /* Run signal watchers last.  This also affects child process watchers
         * because those are implemented in terms of signal watchers.
         */
        if (w == &loop->signal_io_watcher)
          have_signals = 1;
        else
          w->cb(loop, w, pe->events);

        nevents++;
      }
    }
		// ...
}
```

poll 阶段的任务是阻塞以等待监听事件的来临，然后执行对应的回调。其中阻塞是有超时时间，在某些条件下超时时间会被置为 0。此时，就会进入下一阶段，而本 poll 阶段未执行的回调会在下一循环的 pending 阶段执行。

### close 阶段

https://github.com/libuv/libuv/blob/v1.x/src/unix/core.c#L291

```C
static void uv__run_closing_handles(uv_loop_t* loop) {
  uv_handle_t* p;
  uv_handle_t* q;

  p = loop->closing_handles;
  loop->closing_handles = NULL;

  while (p) {
    q = p->next_closing;
    uv__finish_close(p);
    p = q;
  }
}
```

close 阶段的逻辑非常简单，就是循环关闭所有的 closing handles，其中的回调被 `uv__finish_close` 调用。

上面便是 libuv 关于事件循环七个阶段的简单源码解读。之前我们还提到，在 microtask 中，存在着 `process.nextTick` 和 `Promise.resolve` 等阶段。这部分已经超出了 libuv 的范围，我们可以在 node 源码中找到它们的调用路径。

### process.nextTick

https://github.com/nodejs/node/blob/master/lib/internal/process/task_queues.js#L109

```C++
const {
  // For easy access to the nextTick state in the C++ land,
  // and to avoid unnecessary calls into JS land.
  tickInfo,
  // Used to run V8's micro task queue.
  runMicrotasks,
  setTickCallback,
  enqueueMicrotask
} = internalBinding('task_queue');
// ...
// Should be in sync with RunNextTicksNative in node_task_queue.cc
function runNextTicks() {
  if (!hasTickScheduled() && !hasRejectionToWarn())
    runMicrotasks();
  if (!hasTickScheduled() && !hasRejectionToWarn())
    return;

  processTicksAndRejections();
}
// ...
function nextTick(callback) {
  if (typeof callback !== 'function')
    throw new ERR_INVALID_CALLBACK(callback);

  if (process._exiting)
    return;

  var args;
  switch (arguments.length) {
    case 1: break;
    case 2: args = [arguments[1]]; break;
    case 3: args = [arguments[1], arguments[2]]; break;
    case 4: args = [arguments[1], arguments[2], arguments[3]]; break;
    default:
      args = new Array(arguments.length - 1);
      for (var i = 1; i < arguments.length; i++)
        args[i - 1] = arguments[i];
  }

  if (queue.isEmpty())
    setHasTickScheduled(true);
  queue.push(new TickObject(callback, args));
}
```

可以看到，nextTick 就是向 queue 队列压入 callback，然后 `nextTick` 调用后得到的队列被 `runNextTick` 使用，触发 `runMicrotasks` 函数，这个函数通过 `internalBiding` 绑定至 `node_task_queue.cc` 中的同名函数。最终，会触发 V8 中的 microtask 队列处理。

同理，promise 的相应调用路径可以从 https://github.com/nodejs/node/blob/master/lib/internal/process/promises.js 中追踪得到，篇幅有限，就不再赘述了。

## 浏览器事件循环 vs Node 事件循环

浏览器中的事件循环是 HTML 规范中制定的，由不同浏览器厂商自行实现；而 Node 中则由 libuv 库实现。因此，浏览器和 Node 中的事件循环在实现原理和执行流程上都存在差异。

### 浏览器环境

在浏览器中，JavaScript 执行为单线程（不考虑 web worker），所有代码均在主线程调用栈完成执行。当主线程任务清空后才会去轮循任务队列中的任务。

异步任务分为 task（宏任务，也可以被称为 macrotask）和 microtask（微任务）两类。关于事件循环的权威定义可以在 HTML 规范文档中查到：https://html.spec.whatwg.org/multipage/webappapis.html#event-loops。

当满足执行条件时，task 和 microtask 会被放入各自的队列中，等待进入主线程执行，这两个队列被称为 task queue（或 macrotask queue）和 microtask queue。

- task：包括 script 中的代码、`setTimeout`、`setInterval`、`I/O`、UI render
- microtask：包括 `promise`、`Object.observe`、`MutationObserver`

不过，正如规范强调的，这里的 task queue 并非是队列，而是集合（sets），因为事件循环的执行规则是执行第一个可执行的任务，而不是将第一个任务出队并执行。

详细的执行规则可以在 https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model 查询，一共有 15 个步骤。

可以将执行步骤不严谨的归纳为：

1. 执行完主线程中的任务
2. 清空 microtask queue 中的任务并执行完毕
3. 取出 macrotask queue 中的一个任务执行
4. 清空 microtask queue 中的任务并执行完毕
5. 重复 3、4

进一步归纳，就是：一个宏任务，所有微任务；一个宏任务，所有微任务...

### Node 环境

Node 中的事件循环流程已经在前面详述（参见_事件循环的执行机制_一节），这里就不再赘述了。一图以蔽之：

![image](https://user-images.githubusercontent.com/9818716/61017139-b1001180-a3c4-11e9-8860-1c5e54d6ce5c.png)

下面，让我们看一些容易产生误解的情况：

#### `process.nextTick` 造成的 *starve* 现象

```javascript
const fs = require('fs');

function addNextTickRecurs(count) {
  let self = this;
  if (self.id === undefined) {
    self.id = 0;
  }

  if (self.id === count) return;

  process.nextTick(() => {
    console.log(`process.nextTick call ${++self.id}`);
    addNextTickRecurs.call(self, count);
  });
}

addNextTickRecurs(Infinity);
setTimeout(console.log.bind(console, 'omg! setTimeout was called'), 10);
setImmediate(console.log.bind(console, 'omg! setImmediate also was called'));
fs.readFile(__filename, () => {
  console.log('omg! file read complete callback was called!');
});

console.log('started');
```

在这段代码中，由于递归调用了 `process.nextTick`，`setTimeout`、`setImmediate` 以及文件 I/O 的回调将永远不会得到执行。因为执行 libuv 中七个阶段前，会清空 microtask 中的任务。所谓的清空是指，执行完 microtask 队列中已有的任务之后，准备执行 libuv 中的任务之前，会再次确认 microtask 中的任务是否为空，若还有任务，会继续执行。由于递归调用了 `process.nextTick`，会不断往 microtask 中添加任务，从而造成了这种其他队列的饥饿（starve）现象。

当然，在 Node v0.12 之前，存在 `process.maxTickDepth` 属性，用于限制 `process.nextTick` 的执行深度。但是在 v0.12 之后，出于某些原因，这个属性被移除了。此后只能建议开发者避免写出这种代码。

执行结果为：

```
started
process.nextTick call 1
process.nextTick call 2
process.nextTick call 3
...
```

#### `setTimeout` vs `setImmediate`

`setTimeout` 和 `setImmediate` 的回调哪个会先执行呢？有同学可能会说，我知道啊，`setTimeout` 属于 timers 阶段，`setImmediate` 属于 check 阶段，所以会先执行 `setTimeout`。错~，正确答案是，我们无法保证它们的先后顺序。

```javascript
setTimeout(function() {
  console.log('setTimeout')
}, 0);
setImmediate(function() {
  console.log('setImmediate')
});
```

多次执行这段代码，可以看到，我们会得到两种不同的输出结果。

这是由 `setTimeout` 的执行特性导致的，`setTimeout` 中的回调会在超时时间后被执行，但是具体的执行时间却不是确定的，即使设置的超时时间为 0。所以，当事件循环启动时，定时任务可能尚未进入队列，于是，`setTimeout` 被跳过，转而执行了 check 阶段的任务。

换句话说，这种情况下，`setTimeou` 和 `setImmediate` 不一定处于同一个循环内，所以它们的执行顺序是不确定的。

事情到这里并没有结束：

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
    setTimeout(() => {
        console.log('timeout')
    }, 0);
    setImmediate(() => {
        console.log('immediate')
    })
});
```

对于这种情况，immediate 将会永远先于 timeout 输出。

让我们捋一遍这段代码的执行过程：

1. 执行 `fs.readFile`，开始文件 I/O
2. 事件循环启动
3. 文件读取完毕，相应的回调会被加入事件循环中的 I/O 队列
4. 事件循环执行到 pending 阶段，执行 I/O 队列中的任务
5. 回调函数执行过程中，定时器被加入 timers 最小堆中，`setImmediate` 的回调被加入 immediates 队列中
6. 当前事件循环处于 pending 阶段，接下来会继续执行，到达 check 阶段。这是，发现 immediates 队列中存在任务，从而执行 `setImmediate` 注册的回调函数
7. 本轮事件循环执行完毕，进入下一轮，在 timers 阶段执行 `setTimeout` 注册的回调函数

#### `promise` vs `process.nextTick`

`promise` 和 `process.nextTick` 组合使用的情况比较好理解，`nextTick` 会优于 `promise` 执行，microtask 会优于 7 个阶段执行，在执行 7 个阶段前，会进一步确认 microtask 队列是否为空。例如：

```javascript
Promise.resolve().then(() => console.log('promise1 resolved'));
Promise.resolve().then(() => console.log('promise2 resolved'));
Promise.resolve().then(() => {
    console.log('promise3 resolved');
    process.nextTick(() => console.log('next tick inside promise resolve handler'));
});
Promise.resolve().then(() => console.log('promise4 resolved'));
Promise.resolve().then(() => console.log('promise5 resolved'));
setImmediate(() => console.log('set immediate1'));
setImmediate(() => console.log('set immediate2'));

process.nextTick(() => console.log('next tick1'));
process.nextTick(() => console.log('next tick2'));
process.nextTick(() => console.log('next tick3'));

setTimeout(() => console.log('set timeout'), 0);
setImmediate(() => console.log('set immediate3'));
setImmediate(() => console.log('set immediate4'));
```

执行结果将为：

```
next tick1
next tick2
next tick3
promise1 resolved
promise2 resolved
promise3 resolved
promise4 resolved
promise5 resolved
next tick inside promise resolve handler
set timeout
set immediate1
set immediate2
set immediate3
set immediate4
```

## 总结

本文介绍了 Node 中事件循环的作用、执行机制以及与浏览器中事件循环的区别。事件循环是事件驱动编程模式的基础，通过事件驱动模式，可以构建异步非阻塞的高性能服务器，非常适合 I/O 密集型 web 应用。在 Node 中，事件循环是由 libuv 实现的，`uv_run()` 函数中定义了事件循环的七个阶段。在 HTML 规范中，同样也对事件循环做了定义，并由各个浏览器厂商各自实现，实现原理和运行机制都与 Node 中的事件循环有一定的区别。同时，由于 Node 是在不断迭代的，目前最新已经到了 v12.6.0 版本，不同版本间也会存在一定差异，所以本文也无法涵盖关于事件循环的所有内容。当我们讨论关于事件循环的具体问题时，可能会发现许多与之前经验不符的现象。对于这些问题，首先要确定 Node 版本；然后，多动手实验、多看源码、多读规范，形成自己的正确认识。