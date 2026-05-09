# DataContext、ViewModelLocator 和 CommunityToolkit.Mvvm 配合

前面几篇笔记主要讲的是 ViewModel 自己内部怎么写。

但在 WPF 项目里，真正让 ViewModel 跑起来，还差一个关键问题：

View 到底怎么拿到 ViewModel。

这件事通常会落在几个关键词上：

* DataContext
* ViewModelLocator
* 依赖注入
* 窗口或页面初始化

如果这些地方没接好，前面写得再完整的 ObservableProperty、RelayCommand、AsyncRelayCommand 都不会真正生效。

## 一、先理解 DataContext 是什么

在 WPF 里，大多数绑定默认都会从当前元素的 DataContext 往下找。

例如：

```xml
<TextBlock Text="{Binding Title}"/>
```

这句能不能工作，关键不是 TextBlock 本身，而是当前界面有没有一个 DataContext，并且这个 DataContext 里是否存在 Title 属性。

你可以把它先简单理解成：

DataContext 就是当前这块界面默认绑定到哪个对象。

## 二、最直接的写法：在代码后置里设置 DataContext

这是最容易理解的方式。

### 1. ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string title = "首页标题";
}
```

### 2. Window 后置代码

```csharp
using Demo.ViewModels;

namespace Demo.Views;

public partial class MainWindow : Window
{
 public MainWindow()
 {
  InitializeComponent();
  DataContext = new MainViewModel();
 }
}
```

### 3. XAML

```xml
<Window x:Class="Demo.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MainWindow"
        Height="300"
        Width="400">

    <Grid>
        <TextBlock Text="{Binding Title}" />
    </Grid>
</Window>
```

这种方式的优点是：

* 非常直观；
* 上手最快；
* 对小页面、示例代码很合适。

缺点是：

* View 直接 new 了 ViewModel；
* 依赖注入和测试替换不方便；
* 页面一多以后，初始化风格可能不统一。

## 三、在 XAML 里直接设置 DataContext

另一种常见写法，是在 XAML 中直接声明 DataContext。

```xml
<Window x:Class="Demo.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:Demo.ViewModels">

    <Window.DataContext>
        <vm:MainViewModel/>
    </Window.DataContext>

    <Grid>
        <TextBlock Text="{Binding Title}" />
    </Grid>
</Window>
```

这种方式也能工作，但要注意它本质上还是在 View 侧直接创建 ViewModel。

因此它更适合：

* Demo；
* 学习示例；
* 很简单的页面。

对中大型项目来说，通常会更偏向统一的注入或 Locator 方式。

## 四、什么是 ViewModelLocator

ViewModelLocator 不是 WPF 的强制机制，而是一种组织方式。

它的核心思想是：

给 View 提供一个统一入口，让 View 从这里拿到自己对应的 ViewModel，而不是各个页面自己随便 new。

你可以把它简单理解为：

“专门用来给界面分发 ViewModel 的一个中间层。”

它常见的价值有：

1. 统一 ViewModel 获取方式；
2. 避免每个页面都自己写一套初始化；
3. 更容易结合依赖注入容器；
4. 在设计器和运行时之间更容易切换数据源。

## 五、一个最基础的 ViewModelLocator 示例

### 1. Locator 类

```csharp
using Demo.ViewModels;

namespace Demo;

public class ViewModelLocator
{
 public MainViewModel MainViewModel { get; } = new();
 public SettingsViewModel SettingsViewModel { get; } = new();
}
```

### 2. 在 App.xaml 中注册资源

```xml
<Application x:Class="Demo.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:Demo">
    <Application.Resources>
        <local:ViewModelLocator x:Key="Locator"/>
    </Application.Resources>
</Application>
```

### 3. 在 View 中绑定 Locator

```xml
<Window x:Class="Demo.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        DataContext="{Binding MainViewModel, Source={StaticResource Locator}}">

    <Grid>
        <TextBlock Text="{Binding Title}" />
    </Grid>
</Window>
```

这个结构里：

* Window 不直接 new MainViewModel；
* DataContext 通过 Locator 统一拿；
* 后续切换获取方式时更容易集中调整。

## 六、CommunityToolkit.Mvvm 和 ViewModelLocator 的关系

CommunityToolkit.Mvvm 本身主要负责：

* 属性通知；
* 命令；
* 消息通信；
* ViewModel 编写体验。

它并不强制规定你必须怎么创建 ViewModel。

也就是说：

* 你可以直接 new ViewModel；
* 可以用 ViewModelLocator；
* 也可以用依赖注入容器；
* 这些都能和 CommunityToolkit.Mvvm 配合。

Toolkit 解决的是“ViewModel 怎么写”，而 DataContext / Locator 解决的是“ViewModel 怎么交给 View”。

## 七、和依赖注入一起用时的常见方式

如果项目已经用了依赖注入，通常不建议 Locator 自己再 new ViewModel，而是从容器里解析。

例如：

```csharp
using Microsoft.Extensions.DependencyInjection;
using Demo.ViewModels;

namespace Demo;

public class ViewModelLocator
{
 public MainViewModel MainViewModel => App.Current.Services.GetRequiredService<MainViewModel>();
 public SettingsViewModel SettingsViewModel => App.Current.Services.GetRequiredService<SettingsViewModel>();
}
```

App.xaml.cs 中大概会有类似结构：

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace Demo;

public partial class App : Application
{
 public static IServiceProvider Services { get; private set; } = null!;

 public App()
 {
  var services = new ServiceCollection();
  services.AddSingleton<MainViewModel>();
  services.AddTransient<SettingsViewModel>();
  Services = services.BuildServiceProvider();
 }
}
```

这样做的优点是：

* ViewModel 的依赖可以统一走容器；
* 服务、仓储、消息组件都更容易注入；
* 大型项目更容易管理对象生命周期。

## 八、Page、UserControl、Window 的常见差异

虽然它们都能绑定 DataContext，但实际使用时关注点不完全一样。

### 1. Window

最常见的是：

* 自己在构造时设置 DataContext；
* 或从 Locator / 容器中解析 ViewModel。

### 2. Page

如果是导航页，通常要考虑：

* 页面重复进入时是否要复用 ViewModel；
* 导航参数怎么传给 ViewModel；
* 页面离开时消息接收是否需要关闭。

### 3. UserControl

这里最容易踩坑。

很多 UserControl 不是独立页面，而是父页面的一部分，这时不要轻易在 UserControl 内部强行覆盖 DataContext，否则会把父级继承链打断。

## 九、UserControl 最常见的 DataContext 坑

例如下面这种写法，经常会让父页面绑定失效：

```xml
<UserControl ...>
    <UserControl.DataContext>
        <vm:ChildViewModel/>
    </UserControl.DataContext>
</UserControl>
```

为什么它容易出问题：

* UserControl 默认会继承父级 DataContext；
* 你一旦在内部重新指定，就把外部传入的绑定上下文覆盖了；
* 后面复用这个控件时，容易出现“明明绑定了但没值”的情况。

所以更常见的建议是：

* 页面级 View 自己持有 DataContext；
* UserControl 优先复用父级 DataContext；
* 如果必须独立上下文，要非常明确这个控件是“独立组件”，不是普通模板片段。

## 十、一个更贴近项目的组合方式

对于一个普通 WPF + CommunityToolkit.Mvvm 项目，比较常见的实践是：

### 小型项目

* Window 直接设置 DataContext；
* ViewModel 用 ObservableObject、ObservableProperty、RelayCommand。

### 中型项目

* 用 Locator 或简单容器统一解析 ViewModel；
* 页面和窗口保持一致的初始化方式；
* Messenger 和 AsyncRelayCommand 开始参与更多场景。

### 较大项目

* 直接用依赖注入容器管理 ViewModel 和服务；
* Locator 只做展示层桥接，或者干脆不再单独保留；
* View 负责拿 ViewModel，ViewModel 负责状态和行为。

## 十一、什么时候该用 ViewModelLocator

更适合用 Locator 的情况通常是：

* 你想让 XAML 层就能直接拿 ViewModel；
* 页面较多，想统一写法；
* 需要配合设计时数据；
* 暂时没有引入完整 DI，但又不想每页都手写 new。

## 十二、什么时候直接用依赖注入更合适

如果项目已经有这些特点，通常可以直接往 DI 方向走：

* 服务较多；
* ViewModel 构造依赖明显增加；
* 模块化程度高；
* 需要更清晰的生命周期管理。

这时 Locator 不是不能用，而是它更像一个桥接层，不再是核心。

## 十三、常见排查思路

如果按钮不响应、文本不显示、绑定没效果，先检查这几件事：

1. 当前 View 是否真的设置了正确的 DataContext；
2. 绑定路径是否和 ViewModel 中的属性名一致；
3. UserControl 是否意外覆盖了父级 DataContext；
4. ViewModel 是否成功创建；
5. 如果走 Locator，资源是否注册成功；
6. 如果走 DI，服务是否注册成功。

## 总结

1. CommunityToolkit.Mvvm 负责把 ViewModel 写得更高效，但不决定 ViewModel 如何注入到 View；
2. DataContext 决定绑定默认从哪个对象取值；
3. ViewModelLocator 适合统一 ViewModel 获取方式；
4. 中大型项目里，Locator 往往会和依赖注入一起使用，或者逐步过渡到以 DI 为主；
5. UserControl 覆盖 DataContext 是 WPF 里非常常见的坑，需要特别注意。
