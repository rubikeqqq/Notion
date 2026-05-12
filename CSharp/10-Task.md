# Task

很多人第一次系统接触 C# 里的 `Task` 时，通常会卡在这几个问题上：

* `Task` 到底是什么，它和 `Thread` 有什么区别？
* `await` 到底做了什么，为什么写了它就“不阻塞”了？
* `Task` 会不会一定开新线程？
* 为什么有时候用 `await`，界面还是会卡？
* 为什么很多人说不要随便用 `.Result` 和 `.Wait()`？

如果先记一句总定义，可以先记这个：

**Task 表示一个未来会完成的异步操作，它是对“工作”的抽象，不等于线程本身。**

这篇笔记就围绕这条主线来讲：

1. 先分清 `Task`、`Thread`、`ThreadPool`；
2. 为什么要用 `Task`；
3. `Task` 和 `async/await` 的基础用法；
4. `await` 到底做了什么；
5. 多个任务怎么组合；
6. 怎么取消任务；
7. 常见坑和进阶点有哪些。

如果你现在更想先把 `Thread`、`ThreadPool`、`Task`、`async/await` 的整体关系看顺，再回来看这篇细节，可以先看：

* [CSharp/08-Thread、ThreadPool、Task、async和await对比总览.md](08-Thread、ThreadPool、Task、async和await对比总览.md)

---

## 一、先分清 Task、Thread、ThreadPool

理解 `Task` 之前，最好先把它和底层线程概念拆开。

### 1. Thread 是执行线程

`Thread` 是操作系统层面的执行单元。

如果你直接使用 `Thread`，通常意味着你要更明确地参与：

* 线程怎么创建；
* 线程何时开始；
* 线程之间怎么同步；
* 线程资源怎么管理。

这类内容更偏底层，在 [CSharp/09-线程.md](09-线程.md) 里更适合展开。

### 2. ThreadPool 是线程复用池

线程创建和销毁都有成本，所以 .NET 提供了线程池来复用线程。

很多短小的后台工作，不需要每次都手动创建新线程，而是排进线程池，让系统统一调度。

### 3. Task 是对异步工作的抽象

`Task` 更像是“一个工作单元”或者“一个未来结果”的包装。

它关心的是：

* 这件事有没有完成；
* 有没有返回值；
* 有没有异常；
* 能不能继续等待或组合别的任务。

所以可以先这样记：

* `Thread` 更偏“执行资源”；
* `ThreadPool` 更偏“线程调度池”；
* `Task` 更偏“异步工作抽象”。

### 4. Task 不等于一定开新线程

这是最常见的误区之一。

`Task` 很多时候会在线程池线程上执行，但它并不意味着“每个 Task 都会 new 一个线程”。

有些异步操作本身是 I/O 驱动的，在等待期间甚至不会一直占着线程跑。

所以，**不要把 `Task` 直接等同于“开线程”。**

---

## 二、为什么需要 Task

如果只是为了“后台干点活”，好像直接用线程也能做事。

那为什么现代 C# 代码里，`Task` 和 `async/await` 会成为主流？

### 1. 更适合表达异步操作

很多工作并不是“我想控制一个线程”，而是“我发起一个操作，然后等它完成”。

例如：

* 网络请求；
* 文件读取；
* 数据库访问；
* 后台计算。

这些场景更适合用“异步操作”的模型来表达，而不是直接围着线程转。

### 2. 可以统一携带结果、状态和异常

`Task` 不只是表示“在做某事”，它还可以统一描述：

* 是否完成；
* 是否取消；
* 是否失败；
* 返回值是什么。

这比自己手动管理线程状态要整齐很多。

### 3. 更容易组合多个操作

有些场景下，你需要同时做几件异步工作，再等它们一起完成。

例如：

* 同时请求多个接口；
* 同时读取多个文件；
* 多个计算任务并发执行。

`Task.WhenAll()`、`Task.WhenAny()` 这些 API 就是为这种组合场景准备的。

### 4. 更适合 UI 和服务端代码

在 UI 程序里，耗时工作如果堵住主线程，界面就会卡。

在服务端程序里，同步等待 I/O 也会浪费线程资源。

`Task + async/await` 能更自然地表达“发起工作，然后等结果回来”的流程。

所以可以把 `Task` 的价值理解成：

**它不是简单地替代线程，而是把异步工作、结果、异常和组合控制统一进了一套模型。**

---

## 三、Task 的基础用法

先看几个最基本的例子，把直觉建立起来。

### 1. Task 表示一个未来会完成的操作

例如：

```csharp
Task task = Task.Run(() =>
{
 Console.WriteLine("后台任务开始");
 Thread.Sleep(1000);
 Console.WriteLine("后台任务结束");
});

await task;
```

这段代码表示：

* 通过 `Task.Run` 启动一个后台工作；
* `task` 代表这项工作；
* `await task` 表示异步等待它完成。

### 2. Task<TResult> 可以返回结果

```csharp
Task<int> task = Task.Run(() =>
{
 Thread.Sleep(1000);
 return 42;
});

int result = await task;
Console.WriteLine(result);
```

这里的 `Task<int>` 表示：

“这是一个未来会完成，并且最终返回 `int` 的异步操作。”

### 3. await 不是启动任务，而是等待任务

这点非常重要。

很多人会误以为 `await` 的作用是“让代码异步执行”。

其实更准确地说：

**`await` 的作用是以非阻塞的方式等待一个任务完成。**

任务通常早就已经开始了，`await` 只是告诉编译器：

“先等这件事完成，完成之后再继续往下执行。”

### 4. 一个最小 async 示例

```csharp
public static async Task DemoAsync()
{
 Console.WriteLine("开始");
 await Task.Delay(1000);
 Console.WriteLine("1 秒后继续");
}
```

这里 `Task.Delay(1000)` 表示异步等待 1 秒。

它和 `Thread.Sleep(1000)` 的关键区别是：

* `Thread.Sleep` 是阻塞当前线程；
* `Task.Delay` 是返回一个会在未来完成的任务，然后由 `await` 异步等待它。

### 5. Task.Run 更适合 CPU 密集型工作

例如：

```csharp
int result = await Task.Run(() =>
{
 int sum = 0;
 for (int i = 0; i < 1000000; i++)
 {
  sum += i;
 }
 return sum;
});
```

这种场景更偏“算得久”，适合丢到线程池后台执行。

但是如果本来就是异步 I/O，比如 `HttpClient.GetAsync()`、`FileStream.ReadAsync()`，通常不需要再额外包一层 `Task.Run()`。

---

## 四、async 和 await 到底做了什么

这是理解 `Task` 最关键的一节。

先看一个例子：

```csharp
public static async Task LoadAsync()
{
 Console.WriteLine("A");
 await Task.Delay(1000);
 Console.WriteLine("B");
}
```

很多人知道它能工作，但不清楚内部逻辑是什么。

### 1. async 方法会先同步执行到第一个 await 前

调用 `LoadAsync()` 时，不是整段代码立刻“飞到后台”。

它会先从头开始执行，直到遇到第一个 `await`。

也就是说：

* `Console.WriteLine("A")` 会先立刻执行；
* 遇到 `await Task.Delay(1000)` 时，再看这个任务是否已经完成。

### 2. 如果任务未完成，方法会先挂起

当 `Task.Delay(1000)` 还没完成时，`LoadAsync()` 不会傻等在原地阻塞线程。

它会把“后面的代码怎么继续执行”记录下来，然后先返回一个 `Task` 给调用者。

可以把它理解成：

“我先停在这里，等这件事完成后，再接着往下跑。”

### 3. 任务完成后，再恢复执行后续代码

等 `Task.Delay(1000)` 完成后，`LoadAsync()` 会继续执行：

```csharp
Console.WriteLine("B");
```

这就是为什么代码写起来像同步流程，但行为上又是异步的。

### 4. async/await 本质上是状态机语法糖

编译器会把异步方法改写成一个状态机。

你可以不必一开始就深挖编译器细节，但要明确一件事：

**`async/await` 不是魔法，而是编译器帮你把“挂起、恢复、异常传播、结果返回”这些琐碎工作包装好了。**

### 5. await 和阻塞等待的区别

这是初学者最容易混淆的点。

```csharp
await task;
task.Wait();
var result = task.Result;
```

虽然看起来都是“等任务完成”，但本质差别很大：

* `await task`：异步等待，不强行堵住当前线程；
* `task.Wait()`：同步阻塞当前线程；
* `task.Result`：同步阻塞并取返回值。

所以 `await` 的关键价值，不是“等”，而是“**不阻塞地等**”。

---

## 五、Task 一定会开新线程吗

这个问题值得单独拿出来讲。

答案是：

**不一定。**

### 1. Task.Run 通常会在线程池线程执行

例如：

```csharp
await Task.Run(() =>
{
 DoCpuWork();
});
```

这类写法通常会把工作排到线程池线程上执行。

这里你可以近似理解为“确实把工作放到后台线程了”。

### 2. I/O 异步很多时候不是一直占着线程

例如：

```csharp
await httpClient.GetStringAsync(url);
```

这类网络 I/O 异步操作，本质重点不在“开了一个线程帮你等”，而在于：

* 操作发起出去；
* 等外部资源完成；
* 完成后再通知任务结束。

等待期间通常不是一直占着一个线程空转。

### 3. 已完成任务也可能根本不需要新线程

例如：

```csharp
return Task.FromResult(123);
```

这表示直接返回一个已经完成的任务。

此时根本没有“新线程执行工作”这回事。

所以更准确的理解是：

* `Task` 是异步工作的抽象；
* 有些 `Task` 会在线程池线程上跑；
* 有些 `Task` 主要是 I/O 完成通知；
* 有些 `Task` 甚至一创建就是已完成状态。

---

## 六、返回值、异常和等待方式

Task 模型有一个很大的好处，就是结果和异常都能比较统一地流动。

### 1. 使用 await 获取返回值

```csharp
public static async Task<int> GetNumberAsync()
{
 await Task.Delay(500);
 return 100;
}

int value = await GetNumberAsync();
```

这里 `await` 后直接得到最终值，代码看起来会很自然。

### 2. 异常会在 await 时重新抛出

```csharp
public static async Task ThrowAsync()
{
 await Task.Delay(500);
 throw new InvalidOperationException("出错了");
}

try
{
 await ThrowAsync();
}
catch (Exception ex)
{
 Console.WriteLine(ex.Message);
}
```

虽然异常发生在异步方法内部，但你依然可以像同步代码一样通过 `try/catch` 在 `await` 处捕获它。

### 3. Result 和 Wait 会阻塞线程

例如：

```csharp
var task = GetNumberAsync();
int value = task.Result;
```

或者：

```csharp
GetNumberAsync().Wait();
```

这两种写法都会让当前线程停在那里，直到任务完成。

在控制台里有时看起来还能跑，但在 UI 线程或某些上下文里，可能导致卡死、死锁或响应变差。

### 4. 为什么 UI 线程里更要少用 Result / Wait

UI 线程的职责通常是：

* 处理界面消息；
* 响应按钮点击；
* 刷新界面。

如果你在 UI 线程上同步等待一个异步任务：

```csharp
var data = LoadDataAsync().Result;
```

那 UI 线程会被堵住。

而有些异步操作在完成后又想回到 UI 上下文继续执行，这时就容易形成互相等待。

所以更稳的原则是：

**异步链路里尽量一路 async 到底，不要中途强行改回同步等待。**

---

## 七、多个任务怎么组合

实际项目里，很少只有一个任务孤零零地执行。

很多时候你需要把多个异步操作组织起来。

### 1. Task.WhenAll：等待全部完成

例如：

```csharp
Task<int> task1 = GetValueAsync(1);
Task<int> task2 = GetValueAsync(2);
Task<int> task3 = GetValueAsync(3);

int[] results = await Task.WhenAll(task1, task2, task3);
```

适合场景：

* 同时请求多个接口；
* 同时读取多个资源；
* 等所有任务都完成后再汇总结果。

### 2. Task.WhenAny：谁先完成先处理

```csharp
Task<int> task1 = GetValueAsync(1);
Task<int> task2 = GetValueAsync(2);

Task<int> finishedTask = await Task.WhenAny(task1, task2);
int result = await finishedTask;
```

适合场景：

* 谁先返回就先用谁；
* 超时控制；
* 多个来源抢最快结果。

### 3. 顺序 await 和并发启动再 await 是两回事

看下面两种写法：

```csharp
int a = await GetValueAsync(1);
int b = await GetValueAsync(2);
```

这通常是顺序执行。

而下面这样：

```csharp
Task<int> taskA = GetValueAsync(1);
Task<int> taskB = GetValueAsync(2);

int a = await taskA;
int b = await taskB;
```

这两个任务通常会先都启动，再分别等待结果。

如果想明确表达“并发开始，再统一等待”，`Task.WhenAll()` 往往更清晰。

---

## 八、怎么取消任务

取消任务是非常常见的需求，比如：

* 用户点了“取消加载”；
* 页面已经关闭，不想继续请求；
* 后台轮询要停止。

在 .NET 里，取消通常是“协作式取消”，不是强行中断线程。

### 1. CancellationTokenSource 负责发出取消请求

```csharp
var cts = new CancellationTokenSource();
CancellationToken token = cts.Token;
```

这里：

* `CancellationTokenSource` 是取消发起者；
* `CancellationToken` 是传给任务的取消令牌。

### 2. 任务内部要主动检查 token

```csharp
public static async Task DoWorkAsync(CancellationToken token)
{
 for (int i = 0; i < 10; i++)
 {
  token.ThrowIfCancellationRequested();
  Console.WriteLine($"执行中: {i}");
  await Task.Delay(500, token);
 }
}
```

重点不是“外部把任务杀掉”，而是任务自己在合适时机看一下：

“有没有人请求我停下？”

### 3. 调用方发出取消请求

```csharp
var cts = new CancellationTokenSource();

Task task = DoWorkAsync(cts.Token);

cts.Cancel();
```

调用 `Cancel()` 后，不代表任务瞬间消失，而是表示：

“现在请求取消，请你尽快按约定退出。”

### 4. 取消通常会表现为异常或取消状态

如果任务内部调用了 `ThrowIfCancellationRequested()`，通常会抛出 `OperationCanceledException`。

这不是普通失败，而是任务按取消路径结束。

所以取消和异常失败要区分开理解。

### 5. 一个完整示例

```csharp
public static async Task RunLoopAsync(CancellationToken token)
{
 while (true)
 {
  token.ThrowIfCancellationRequested();
  Console.WriteLine("轮询中...");
  await Task.Delay(1000, token);
 }
}

var cts = new CancellationTokenSource();

try
{
 Task loopTask = RunLoopAsync(cts.Token);

 await Task.Delay(3000);
 cts.Cancel();

 await loopTask;
}
catch (OperationCanceledException)
{
 Console.WriteLine("任务已取消");
}
```

---

## 九、几个常见实战场景

把概念落到场景里，会更容易记住什么时候该用什么。

### 1. UI 中加载数据，避免卡界面

例如按钮点击后去加载数据：

```csharp
private async Task LoadDataAsync()
{
 Status = "加载中...";
 var data = await _service.GetDataAsync();
 Items = data;
 Status = "加载完成";
}
```

这里重点是：

* 不要在 UI 线程里同步堵住耗时操作；
* 用 `await` 让界面保持可响应。

### 2. CPU 密集型计算放到后台

```csharp
int result = await Task.Run(() => CalculateHugeData());
```

这适合明确的 CPU 密集型工作，比如大批量计算、复杂转换、图像处理前置运算等。

### 3. 并发拉取多个结果再汇总

```csharp
Task<User> userTask = _userService.GetUserAsync();
Task<Order[]> orderTask = _orderService.GetOrdersAsync();

await Task.WhenAll(userTask, orderTask);

User user = await userTask;
Order[] orders = await orderTask;
```

这类写法比一个一个顺序等待更高效。

### 4. 带取消按钮的异步操作

例如界面上有“开始”和“取消”按钮：

* 开始时创建 `CancellationTokenSource`；
* 任务执行时检查 `token`；
* 点击取消时调用 `Cancel()`。

这是桌面程序和后台服务里都很常见的模式。

---

## 十、常见误区和坑

`Task` 和 `async/await` 很好用，但坑也很多。

### 1. async void 只适合事件处理器

例如：

```csharp
private async void Button_Click(object sender, EventArgs e)
{
 await LoadDataAsync();
}
```

事件处理器通常只能这样写，所以没问题。

但普通业务方法如果写成：

```csharp
public async void SaveAsync()
{
 await Task.Delay(1000);
}
```

通常不推荐，因为：

* 调用方没法 `await` 它；
* 异常处理更麻烦；
* 流程控制更难做。

普通异步方法一般应该返回 `Task` 或 `Task<TResult>`。

### 2. 不要在异步链路里乱用 Wait() / Result

前面已经提到，这会阻塞线程，尤其在 UI 或带上下文的环境里更容易出问题。

经验上，能 `await` 就优先 `await`。

### 3. Task.Run 不是万能加速器

很多人会写成：

```csharp
await Task.Run(async () => await httpClient.GetStringAsync(url));
```

这通常没有必要。

因为 `GetStringAsync()` 本来就是异步 I/O，不需要你再硬塞进线程池。

`Task.Run` 更适合 CPU 密集型工作，而不是所有异步操作都包一层。

### 4. 忘记 await 会导致流程失控

例如：

```csharp
SaveAsync();
Console.WriteLine("保存完成");
```

如果这里本来希望“保存结束后再继续”，但你忘了 `await`，那后面的代码就会过早执行。

异常也可能漂得更远，排查起来更麻烦。

### 5. 异步方法建议用 Async 后缀

例如：

* `LoadAsync()`
* `SaveAsync()`
* `RefreshDataAsync()`

这样调用方一眼就知道：

“这是个异步方法，需要考虑 await。”

这不是语法要求，但非常值得养成。

---

## 十一、几个进阶点

主线理解清楚后，可以再知道几个常见进阶概念。

### 1. TaskCompletionSource<T>

有些旧代码不是 `Task` 风格，而是回调风格。

这时 `TaskCompletionSource<T>` 可以帮你把“外部事件完成时再回填结果”的模式包装成一个 `Task`。

可以把它理解成：

**我手动控制一个 Task 什么时候完成、返回什么结果、抛什么异常。**

它在桥接旧接口时很有用。

### 2. ConfigureAwait(false)

默认情况下，`await` 完成后可能会尝试回到原来的上下文继续执行。

在 UI 程序里，这通常和“回到 UI 线程继续更新界面”有关。

而在一些类库代码里，如果后续逻辑并不依赖特定上下文，常会看到：

```csharp
await task.ConfigureAwait(false);
```

这背后的重点是“上下文捕获”概念。

初学阶段不需要把它复杂化，但至少要知道：

* 它不是“让代码更异步”；
* 它主要和恢复执行时是否回到原上下文有关。

### 3. ValueTask

有些异步操作经常很快完成，如果每次都创建 `Task`，可能会有额外分配成本。

`ValueTask` 就是在这种背景下出现的。

但它并不是“Task 的全面替代品”。

通常在性能敏感、并且明确理解其使用边界时才会考虑。

### 4. 聚合异常

当多个任务并发执行时，异常处理有时会表现成聚合形式。

初学阶段你可以先记住：

* 单个任务用 `await` 时，异常通常会像普通异常一样抛出来；
* 多任务场景下，要有“可能不止一个任务出错”的意识。

---

## 十二、把整条主线串起来

把前面的内容压缩成一条完整链路，大概是这样：

1. `Task` 表示一个未来会完成的异步操作；
2. `async` 方法会返回 `Task` 或 `Task<TResult>`；
3. `await` 用来非阻塞地等待这个任务完成；
4. `Task.Run` 常用于 CPU 密集型后台工作；
5. `Task.WhenAll` 和 `Task.WhenAny` 用于组合多个任务；
6. `CancellationToken` 用于协作式取消；
7. `.Result` 和 `.Wait()` 是同步阻塞等待，能不用就尽量不用。

所以这套知识最好不要记成零散 API，而要记成：

**Task 负责表示工作，await 负责等待工作，组合 API 负责组织多个工作，取消令牌负责结束工作。**

---

## 十三、和线程那篇的关系

这篇主要讲的是：

* `Task`
* `async/await`
* 异步等待
* 多任务组合
* 取消
* 常见异步陷阱

如果你想进一步看这些更底层的话题：

* `Thread`
* `Mutex`
* `Interlocked`
* `ManualResetEvent`
* `AutoResetEvent`
* `ThreadPool`

那更适合继续看 [CSharp/09-线程.md](09-线程.md)。

可以把两篇理解成：

* 线程那篇负责“底层执行与同步工具”；
* 这篇负责“高层异步工作模型”。

---

## 十四、总结

最后把这篇内容压成几句最重要的话。

### 1. Task 是什么

**Task 是对异步工作的抽象，不等于线程。**

### 2. await 做了什么

**await 的本质是非阻塞地等待一个任务完成。**

### 3. Task.Run 适合什么

**Task.Run 主要适合 CPU 密集型工作，不是所有异步场景都要包一层。**

### 4. 取消是怎么做的

**CancellationToken 是协作式取消，不是强行杀掉线程。**

### 5. 最后记一套判断标准

你可以把 Task 主线记成下面这五句：

* `Task`：表示未来会完成的工作；
* `await`：非阻塞等待；
* `Task.Run`：把 CPU 密集型工作丢到线程池；
* `WhenAll / WhenAny`：组合多个任务；
* `CancellationToken`：请求任务优雅结束。

如果这五句能记住，后面再看 UI 异步、服务端异步、异步命令、异步数据加载时，就会更容易判断该怎么写，而不是只会机械地套 API。
