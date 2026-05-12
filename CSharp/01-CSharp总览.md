# CSharp 总览

这一组笔记主要围绕 C# 中几个很常见、也很容易混在一起的基础主题展开。

如果只看学习顺序，可以先这样理解：

1. 常见基础类型与集合总览：先把值类型、引用类型、字符串、数组和常见集合的选择逻辑理顺；
2. 值类型、引用类型、装箱拆箱：把类型行为和传参/装箱问题单独讲透；
3. List、Dictionary、HashSet实战对比：把常见集合的选型逻辑落到实战场景；
4. 文件相关：再把路径、文件、目录、对话框这些常碰到的基础操作理顺；
5. 依赖注入：理解对象创建、解耦、容器和生命周期；
6. Thread、ThreadPool、Task、async和await对比总览：先建立并发与异步的大图景；
7. 线程：看底层线程、同步原语、线程池；
8. Task：看高层异步模型、任务组合、取消与常见坑。

## 一、各篇笔记分别讲什么

### 1. 常见基础类型与集合总览

重点是：

* 值类型和引用类型怎么分；
* 常见基础类型的适用场景；
* `string`、数组、`List<T>`、`Dictionary<TKey, TValue>`、`HashSet<T>` 怎么选；
* `var` 怎么理解更稳。

适合在这里开始读：

* [CSharp/02-常见基础类型与集合总览.md](02-常见基础类型与集合总览.md)

### 2. 值类型、引用类型、装箱拆箱

重点是：

* 值类型和引用类型的行为差异；
* 默认传参与 `ref` / `out` 的区别；
* `string` 的特殊性；
* 装箱拆箱是什么、为什么有成本。

适合在类型总览后继续读：

* [CSharp/03-值类型、引用类型、装箱拆箱.md](03-值类型、引用类型、装箱拆箱.md)

### 3. List、Dictionary、HashSet实战对比

重点是：

* 什么场景适合 `List<T>`；
* 什么场景适合 `Dictionary<TKey, TValue>`；
* 什么场景适合 `HashSet<T>`；
* 实战里怎么按访问模式选集合。

适合在类型总览后继续读：

* [CSharp/04-List、Dictionary、HashSet实战对比.md](04-List、Dictionary、HashSet实战对比.md)

### 4. 文件相关

重点是：

* `Path`、`File`、`Directory` 怎么分工；
* 当前目录、上级目录、系统目录怎么拿；
* `FileInfo` / `DirectoryInfo` 的定位；
* `OpenFileDialog` 和 `SaveFileDialog` 的常见写法。

适合在这里开始读：

* [CSharp/05-文件相关.md](05-文件相关.md)

### 5. 依赖注入

重点是：

* 什么是依赖、IoC、DI；
* 为什么依赖注入能解耦；
* .NET 容器怎么注册和解析对象；
* 构造函数为什么能自动注入；
* 生命周期怎么选。

适合在这里继续读：

* [CSharp/06-依赖注入.md](06-依赖注入.md)

### 6. 并发与异步总览

重点是：

* `Thread`、`ThreadPool`、`Task`、`async/await` 的层级关系；
* 它们分别解决什么问题；
* 什么时候该控制线程，什么时候该写 `Task`；
* `Task.Run()` 和真正异步 I/O 的区别。

适合先建立整体认知：

* [CSharp/08-Thread、ThreadPool、Task、async和await对比总览.md](08-Thread、ThreadPool、Task、async和await对比总览.md)

### 7. 线程

重点是：

* 线程是什么；
* 为什么会有线程安全问题；
* `lock`、`Interlocked`、`Mutex`、`ManualResetEvent`、`AutoResetEvent` 分别解决什么问题；
* 线程池和底层同步工具怎么理解。

适合在总览后深入：

* [CSharp/09-线程.md](09-线程.md)

### 8. Task

重点是：

* `Task` 是什么；
* `async/await` 做了什么；
* 多个任务怎么组合；
* `CancellationToken` 怎么用；
* `Result`、`Wait()`、`async void` 这些高频坑怎么理解。

适合在线程总览后继续：

* [CSharp/10-Task.md](10-Task.md)

## 二、如果按主题理解，可以这样分层

### 第一层：基础类型与数据组织

* 常见基础类型与集合总览
* 值类型、引用类型、装箱拆箱
* List、Dictionary、HashSet实战对比

这一层解决的是：

“数据该用什么类型表示，数据集合该怎么组织。”

### 第二层：文件与路径基础

* 文件相关

这一层解决的是：

“程序怎么找到文件、读写文件、管理目录。”

### 第三层：对象组织与工程结构

* 依赖注入

这一层解决的是：

“对象怎么创建、怎么解耦、怎么统一管理生命周期。”

### 第四层：并发与异步

* Thread、ThreadPool、Task、async和await对比总览
* 线程
* Task

这一层解决的是：

“代码如何并发执行、如何异步等待、如何避免阻塞和竞态。”

## 三、一个最实用的学习顺序

如果你是第一次系统整理这组 C# 基础主题，建议按下面顺序看：

1. [CSharp/02-常见基础类型与集合总览.md](02-常见基础类型与集合总览.md)
2. [CSharp/03-值类型、引用类型、装箱拆箱.md](03-值类型、引用类型、装箱拆箱.md)
3. [CSharp/04-List、Dictionary、HashSet实战对比.md](04-List、Dictionary、HashSet实战对比.md)
4. [CSharp/05-文件相关.md](05-文件相关.md)
5. [CSharp/06-依赖注入.md](06-依赖注入.md)
6. [CSharp/08-Thread、ThreadPool、Task、async和await对比总览.md](08-Thread、ThreadPool、Task、async和await对比总览.md)
7. [CSharp/09-线程.md](09-线程.md)
8. [CSharp/10-Task.md](10-Task.md)

这样看比较稳，因为顺序是：

* 先把数据类型和集合选择理顺；
* 再把类型行为和集合选型单独讲透；
* 先把日常基础操作理顺；
* 再理解对象和工程结构；
* 再进入并发与异步这组更容易混乱的话题。

## 四、如果按问题找文档，可以这样查

如果你当前遇到的是下面这些问题，可以直接跳转：

* 不知道值类型、引用类型、数组、List、Dictionary、HashSet 该怎么选：看 [CSharp/02-常见基础类型与集合总览.md](02-常见基础类型与集合总览.md)
* 不清楚赋值、传参、装箱拆箱为什么会出现不同表现：看 [CSharp/03-值类型、引用类型、装箱拆箱.md](03-值类型、引用类型、装箱拆箱.md)
* 不知道 `List<T>`、`Dictionary<TKey, TValue>`、`HashSet<T>` 实战里怎么选：看 [CSharp/04-List、Dictionary、HashSet实战对比.md](04-List、Dictionary、HashSet实战对比.md)
* 不知道怎么拼路径、找父目录、设置文件对话框筛选器：看 [CSharp/05-文件相关.md](05-文件相关.md)
* 不理解容器、注册、生命周期、构造函数自动注入：看 [CSharp/06-依赖注入.md](06-依赖注入.md)
* 不知道 `Thread`、`Task`、`await` 之间到底是什么关系：看 [CSharp/08-Thread、ThreadPool、Task、async和await对比总览.md](08-Thread、ThreadPool、Task、async和await对比总览.md)
* 不知道怎么理解锁、原子操作、事件等待句柄、线程池：看 [CSharp/09-线程.md](09-线程.md)
* 不知道 `await` 做了什么、为什么 `Result` 容易卡、怎么取消任务：看 [CSharp/10-Task.md](10-Task.md)

## 五、总结

这一组文档现在可以先按下面这套记忆框架理解：

* 常见基础类型与集合：解决数据表示和集合选择；
* 值类型、引用类型、装箱拆箱：解决类型行为和装箱成本理解；
* List、Dictionary、HashSet：解决集合选型和访问模式判断；
* 文件相关：解决路径、文件、目录操作；
* 依赖注入：解决对象创建和解耦；
* 线程：解决底层执行和同步；
* Task：解决高层异步工作和流程编排；
* 总览页：负责把这些主题串起来。

如果先把这张地图建立起来，后面再往任何一篇里深入，都会更容易定位它在整套知识里的位置。
