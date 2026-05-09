# Host 和 IServiceProvider

在 WPF 项目里接触依赖注入时，经常会同时看到两个名字：

* Host
* IServiceProvider

很多人第一次看到会有点混：

* Host 是不是就是容器？
* IServiceProvider 是不是就是依赖注入本身？
* 为什么有时候写 `Host.CreateDefaultBuilder()`，有时候又写 `App.Services.GetRequiredService<T>()`？

如果只记结论，可以先记一句：

**Host 更像应用的“宿主”和启动入口，IServiceProvider 更像“从容器里取对象的入口”。**

这篇就专门把这两个概念在 WPF 里的关系梳理清楚。

## 一、为什么 WPF 里也会看到 Host

很多人对 Host 的第一印象来自 ASP.NET Core，其实 `Microsoft.Extensions.Hosting` 并不只服务于 Web。

它本质上提供的是一套通用的应用启动模型，可以把这些能力统一组织起来：

* 依赖注入；
* 配置读取；
* 日志；
* 应用生命周期管理。

WPF 虽然是桌面程序，但同样也有这些需求：

* 我们希望 Window、ViewModel、Service 都能交给容器管理；
* 我们希望配置和日志不要各写一套散装初始化代码；
* 我们希望应用启动和退出时，有统一的入口去创建和释放资源。

所以在稍微正规一点的 WPF 项目里，引入 Host 是很自然的事。

可以把它理解成：

**原来的 WPF 只有 App 负责启动应用；引入 Host 以后，App 更像是 WPF 壳层，真正的基础设施装配交给 Host。**

## 二、Host 和 IServiceProvider 分别是什么

要先把两个类型的职责拆开看。

### 1. Host 是应用宿主

`IHost` 代表已经构建完成的应用宿主。

它的职责不只是“存放服务”，而是承载一整个应用运行环境，例如：

* 服务容器；
* 配置系统；
* 日志系统；
* 启动和停止生命周期。

常见写法：

```csharp
IHost host = Host.CreateDefaultBuilder()
 .ConfigureServices(services =>
 {
  services.AddSingleton<MainWindow>();
 })
 .Build();
```

上面这段代码做的事情不是“只创建了一个容器”，而是创建了一个完整的应用宿主对象。

### 2. IServiceProvider 是服务解析入口

`IServiceProvider` 是 .NET 依赖注入体系里的一个核心接口。

它最直接的作用就是：

**根据服务类型，返回对应的实例。**

例如：

```csharp
var mainWindow = serviceProvider.GetRequiredService<MainWindow>();
```

这句话关注的不是应用生命周期，而是“给我一个 MainWindow 实例”。

所以从职责上说：

* Host 负责把应用运行环境搭起来；
* IServiceProvider 负责在这个环境里解析服务实例。

## 三、它们之间是什么关系

在 WPF 项目里，这两个对象通常不是平级概念，而是下面这种关系：

1. 先通过 `Host.CreateDefaultBuilder()` 配置应用；
2. 在 `ConfigureServices` 里注册 Window、ViewModel、Service；
3. 调用 `Build()` 生成 `IHost`；
4. 再通过 `host.Services` 拿到 `IServiceProvider`；
5. 最后由 `IServiceProvider` 解析具体对象。

也就是说：

**IServiceProvider 往往是 Host 内部暴露出来的一部分能力。**

最常见的链路就是：

```csharp
IHost host = ...;
IServiceProvider services = host.Services;

var mainWindow = services.GetRequiredService<MainWindow>();
```

因此可以这样记忆：

* Host 是“大管家”；
* IServiceProvider 是“大管家手里的服务分发入口”。

## 四、在 WPF 中典型会怎么写

下面是一套比较典型、也比较容易落地的写法。

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace DemoApp;

public partial class App : Application
{
 public static IHost AppHost { get; private set; } = null!;

 public App()
 {
  AppHost = Host.CreateDefaultBuilder()
   .ConfigureAppConfiguration((context, config) =>
   {
    config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
   })
   .ConfigureLogging(logging =>
   {
    logging.ClearProviders();
    logging.AddDebug();
    logging.AddConsole();
   })
   .ConfigureServices((context, services) =>
   {
    services.AddSingleton<MainWindow>();
    services.AddSingleton<MainViewModel>();

    services.AddTransient<LoginWindow>();
    services.AddTransient<LoginViewModel>();

    services.AddSingleton<IUserService, UserService>();
   })
   .Build();
 }

 protected override async void OnStartup(StartupEventArgs e)
 {
  await AppHost.StartAsync();

  var mainWindow = AppHost.Services.GetRequiredService<MainWindow>();
  mainWindow.Show();

  base.OnStartup(e);
 }

 protected override async void OnExit(ExitEventArgs e)
 {
  await AppHost.StopAsync();
  AppHost.Dispose();

  base.OnExit(e);
 }
}
```

这段代码里可以明确看出几层关系：

### 1. App 仍然是 WPF 入口

WPF 还是由 `App` 启动，只不过现在 `App` 不再负责手动 `new MainWindow()`，而是负责初始化 Host。

### 2. Host 负责装配应用环境

在 `ConfigureServices` 中注册所有需要交给容器管理的对象。

### 3. IServiceProvider 负责解析对象

真正取出主窗口的动作发生在：

```csharp
AppHost.Services.GetRequiredService<MainWindow>()
```

这里的 `AppHost.Services` 就是 `IServiceProvider`。

## 五、Host 是怎么做配置读取和日志的

前面提到 Host 不只是装 DI 容器，它还会把配置和日志这两类基础设施一起组织起来。

这一点正是很多 WPF 项目引入 Host 的关键价值。

### 1. 配置读取是怎么进来的

`Host.CreateDefaultBuilder()` 本身就会帮你准备一套默认配置来源，常见包括：

* `appsettings.json`；
* `appsettings.{Environment}.json`；
* 环境变量；
* 命令行参数。

在桌面项目里，最常见的是继续补充或调整 `appsettings.json`。

例如：

```json
{
  "AppSettings": {
    "Theme": "Light",
    "ApiBaseUrl": "https://localhost:5001"
  }
}
```

然后在 Host 构建时追加配置源：

```csharp
AppHost = Host.CreateDefaultBuilder()
 .ConfigureAppConfiguration((context, config) =>
 {
  config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
 })
 .ConfigureServices((context, services) =>
 {
  // context.Configuration 就是拼装好的 IConfiguration
 })
 .Build();
```

这时 `context.Configuration` 就已经可用了，它的类型本质上就是 `IConfiguration`。

如果某个服务需要读取配置，可以直接通过构造函数注入：

```csharp
using Microsoft.Extensions.Configuration;

public class UserService : IUserService
{
 private readonly IConfiguration _configuration;

 public UserService(IConfiguration configuration)
 {
  _configuration = configuration;
 }

 public string GetApiBaseUrl()
 {
  return _configuration["AppSettings:ApiBaseUrl"] ?? string.Empty;
 }
}
```

也就是说，配置不是你自己手动去 new 的，而是 Host 在启动时统一组装好，再通过容器注入给需要的类。

### 2. 日志是怎么进来的

日志的接入思路和配置很像，也是 Host 在启动阶段统一组织。

例如：

```csharp
AppHost = Host.CreateDefaultBuilder()
 .ConfigureLogging(logging =>
 {
  logging.ClearProviders();
  logging.AddDebug();
  logging.AddConsole();
 })
 .ConfigureServices((context, services) =>
 {
  services.AddSingleton<IUserService, UserService>();
 })
 .Build();
```

这样配置之后，容器就能自动解析 `ILogger<T>`。

比如在 ViewModel 或 Service 里这样用：

```csharp
using Microsoft.Extensions.Logging;

public class MainViewModel
{
 private readonly ILogger<MainViewModel> _logger;
 private readonly IUserService _userService;

 public MainViewModel(IUserService userService, ILogger<MainViewModel> logger)
 {
  _userService = userService;
  _logger = logger;
 }

 public void Load()
 {
  _logger.LogInformation("开始加载主页面数据");
 }
}
```

这时日志对象同样不需要你手动创建，Host 会把日志系统注册进容器，然后通过依赖注入给到业务类。

### 3. 配置、日志和 IServiceProvider 之间的关系

这三个概念经常会一起出现，关系可以整理成一条线：

1. Host 在构建阶段收集配置源、日志提供程序和服务注册；
2. Build 之后生成完整的应用宿主；
3. 宿主内部持有服务容器；
4. 通过 `Host.Services` 暴露 `IServiceProvider`；
5. 最终由 `IServiceProvider` 解析 `IConfiguration`、`ILogger<T>`、业务服务、Window、ViewModel 等对象。

所以 `IConfiguration` 和 `ILogger<T>` 本质上也只是容器里可被解析出来的服务之一。

### 4. 在 WPF 项目里这样做的好处

把配置和日志也交给 Host 统一管理，通常会比自己零散处理更稳：

* 启动入口更集中；
* 业务类拿配置和日志的方式一致；
* 后续更换配置源或日志提供程序时，改动集中在 Host 配置阶段；
* 更容易和依赖注入、生命周期管理放在一起维护。

## 六、Window、ViewModel、Service 是怎么串起来的

理解 Host 和 IServiceProvider，最好顺着对象创建链看一遍。

假设主窗口依赖一个 ViewModel：

```csharp
public partial class MainWindow : Window
{
 public MainWindow(MainViewModel viewModel)
 {
  InitializeComponent();
  DataContext = viewModel;
 }
}
```

而 `MainViewModel` 又依赖一个服务：

```csharp
public class MainViewModel
{
 private readonly IUserService _userService;

 public MainViewModel(IUserService userService)
 {
  _userService = userService;
 }
}
```

当执行下面这句时：

```csharp
var mainWindow = AppHost.Services.GetRequiredService<MainWindow>();
```

容器会自动完成这些事情：

1. 创建 `MainWindow`；
2. 发现 `MainWindow` 需要 `MainViewModel`；
3. 创建 `MainViewModel`；
4. 发现 `MainViewModel` 需要 `IUserService`；
5. 找到 `IUserService` 对应的实现并注入进去。

也就是说，`IServiceProvider` 表面上只是在“解析 MainWindow”，实际上会沿着依赖链把整棵对象树组装出来。

## 七、什么时候直接用 IServiceProvider，什么时候不该用

这里很容易走偏。

### 1. 推荐：优先使用构造函数注入

如果一个类需要依赖某个服务，最推荐的方式仍然是构造函数注入。

例如：

```csharp
public class MainViewModel
{
 private readonly IUserService _userService;

 public MainViewModel(IUserService userService)
 {
  _userService = userService;
 }
}
```

这种写法有几个好处：

* 依赖关系清晰；
* 更利于测试；
* 不会把业务类写成“到处找服务”的样子。

### 2. 可以接受：在应用入口或基础设施层用 IServiceProvider

比如下面这些位置，直接使用 `IServiceProvider` 往往是合理的：

* `App.xaml.cs` 中解析主窗口；
* 导航服务里按需创建页面；
* 某些工厂类里按类型解析对象。

因为这些位置本来就承担“创建对象”或“组织对象”的职责。

### 3. 不推荐：在业务代码里到处保存 IServiceProvider

例如下面这种写法一般不建议：

```csharp
public class MainViewModel
{
 private readonly IServiceProvider _serviceProvider;

 public MainViewModel(IServiceProvider serviceProvider)
 {
  _serviceProvider = serviceProvider;
 }

 public void Load()
 {
  var service = _serviceProvider.GetRequiredService<IUserService>();
 }
}
```

这会带来几个问题：

* 依赖变得不透明；
* 类真正需要什么服务，从构造函数表面看不出来；
* 容易退化成 Service Locator 风格。

所以实践里通常遵循一个原则：

**能注入具体依赖，就不要注入 IServiceProvider。**

## 八、WPF 中常见的生命周期选择

虽然这篇重点不是讲生命周期，但 Host 和 IServiceProvider 的使用一定会碰到它。

桌面项目里常见的注册思路通常是：

### 1. Singleton

适合：

* 主窗口；
* 全局配置服务；
* 日志服务；
* 整个应用期都共享的状态对象。

### 2. Transient

适合：

* 临时弹窗；
* 每次打开都希望是新实例的页面或 ViewModel；
* 短生命周期服务。

### 3. Scoped

在 Web 项目中常见，在 WPF 里不是不能用，而是通常没有那么强的默认场景。

如果没有明确的作用域边界设计，先用 `Singleton` 和 `Transient` 往往更直观。

## 九、一个更容易记住的理解方式

如果把整个应用比作一家公司：

* Host 像公司整体运行框架；
* IServiceProvider 像前台或调度台；
* ServiceCollection 像入职登记表；
* 各个 Service、ViewModel、Window 像具体员工。

流程就是：

1. 先登记有哪些岗位和实现；
2. 再让 Host 把整个运行框架搭起来；
3. 需要谁时，通过 IServiceProvider 去拿；
4. 由容器自动把依赖关系组装好。

这个比喻不一定严谨到类型层面，但对理解角色分工很有帮助。

## 十、实际开发中的几个注意点

### 1. 不要把 Host 和 IServiceProvider 当成同一个东西

`Host` 是宿主对象，`IServiceProvider` 是服务解析入口。

虽然平时经常通过 `host.Services` 使用它们，但两者职责并不相同。

### 2. 不要在 View 里手动 new ViewModel

如果已经引入 Host 和依赖注入，就尽量让容器负责对象创建。

这样依赖关系、生命周期和后续替换实现都会更清晰。

### 3. 不要把静态全局 Services 用成万能入口

很多项目会写一个静态属性，例如：

```csharp
public static IServiceProvider Services => AppHost.Services;
```

这个写法在入口层很方便，但如果业务代码到处直接访问，就会让结构慢慢失控。

静态访问可以有，但最好控制在少量基础设施位置。

### 4. 启动和退出时要考虑资源释放

既然 Host 负责应用运行环境，就不要只管创建不管释放。

在 `OnExit` 中调用 `StopAsync()` 和 `Dispose()`，是比较完整的做法。

## 十一、总结

回到最开始的问题，在 WPF 里可以把 Host 和 IServiceProvider 这样理解：

* `IHost` 负责承载应用级基础设施，是应用宿主；
* `IServiceProvider` 负责解析服务实例，是取服务的入口；
* 在实际代码里，通常通过 `Host.Services` 拿到 `IServiceProvider`；
* 推荐把 `IServiceProvider` 用在应用入口和基础设施位置，把业务依赖尽量写成构造函数注入。

如果你已经理解了依赖注入的基础概念，那么再看 WPF 里的 Host，其实就是多了一层“应用运行环境管理”的外壳。

也可以把它简化成一句话：

**Host 负责把舞台搭起来，IServiceProvider 负责把舞台上的角色按需请出来。**

如果你还想继续往下接，可以把这篇和下面两篇连起来看：

* `CSharp/依赖注入.md`：先打基础，理解服务注册、生命周期和容器解析。
* `WPF/CommunityToolkit/14-依赖注入项目骨架.md`：再看一个更接近实战项目结构的 WPF 写法。
