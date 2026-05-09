# CommunityToolkit.Mvvm 和传统手写 MVVM 对比

很多人第一次接触 CommunityToolkit.Mvvm 时，最常见的问题不是“它能不能用”，而是：

如果我本来就会手写 INotifyPropertyChanged、手写 ICommand、手写事件通知，那它到底替我省了什么，又改变了什么。

这篇就专门把 CommunityToolkit.Mvvm 和传统手写 MVVM 放在一起对比。

## 一、先说结论

如果只给一个结论，可以这样理解：

* 传统手写 MVVM：控制更显式，但样板代码更多；
* CommunityToolkit.Mvvm：保持 MVVM 思路不变，只是把大量重复模板抽掉了。

也就是说，它不是换了一套架构思想，而是把原来就该写的那些固定模板做了封装和生成。

## 二、属性通知怎么对比

### 1. 传统手写写法

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace Demo.ViewModels;

public class MainViewModel : INotifyPropertyChanged
{
 public event PropertyChangedEventHandler? PropertyChanged;

 protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
 {
  PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
 }

 private string title = string.Empty;

 public string Title
 {
  get => title;
  set
  {
   if (title != value)
   {
    title = value;
    OnPropertyChanged();
   }
  }
 }
}
```

### 2. CommunityToolkit 属性写法

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string title = string.Empty;
}
```

本质区别不是“有没有属性通知”，而是：

* 传统写法：自己手写通知模板；
* Toolkit 写法：用基类和源生成器把模板收起来。

## 三、命令怎么对比

### 1. 传统手写命令

```csharp
using System;
using System.Windows.Input;

namespace Demo.Commands;

public class RelayCommand : ICommand
{
 private readonly Action _execute;
 private readonly Func<bool>? _canExecute;

 public RelayCommand(Action execute, Func<bool>? canExecute = null)
 {
  _execute = execute;
  _canExecute = canExecute;
 }

 public event EventHandler? CanExecuteChanged;

 public bool CanExecute(object? parameter)
 {
  return _canExecute?.Invoke() ?? true;
 }

 public void Execute(object? parameter)
 {
  _execute();
 }

 public void RaiseCanExecuteChanged()
 {
  CanExecuteChanged?.Invoke(this, EventArgs.Empty);
 }
}
```

然后在 ViewModel 里还要再写：

```csharp
public RelayCommand SaveCommand { get; }

public MainViewModel()
{
 SaveCommand = new RelayCommand(Save, CanSave);
}
```

### 2. CommunityToolkit 命令写法

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [RelayCommand(CanExecute = nameof(CanSave))]
 private void Save()
 {
 }

 private bool CanSave()
 {
  return true;
 }
}
```

差别很明显：

* 传统写法要自己准备命令类和命令初始化；
* Toolkit 写法直接把方法声明成命令。

## 四、异步命令怎么对比

传统手写 MVVM 里，异步命令往往最容易写乱。

常见问题包括：

* async void 到处出现；
* 异常不容易管理；
* 执行状态不好追踪；
* 重复点击难控制。

CommunityToolkit.Mvvm 提供 AsyncRelayCommand 后，这部分会规整很多。

### 传统写法常见样子

```csharp
public RelayCommand LoadCommand { get; }

public MainViewModel()
{
 LoadCommand = new RelayCommand(async () => await LoadAsync());
}
```

### Toolkit 消息写法

```csharp
public IAsyncRelayCommand LoadCommand { get; }

public MainViewModel()
{
 LoadCommand = new AsyncRelayCommand(LoadAsync);
}
```

或者直接：

```csharp
[RelayCommand]
private async Task LoadAsync()
{
 await Task.Delay(1000);
}
```

这会让异步命令的边界更清楚。

## 五、联动通知怎么对比

### 传统手写联动

```csharp
private string firstName = string.Empty;

public string FirstName
{
 get => firstName;
 set
 {
  if (firstName != value)
  {
   firstName = value;
   OnPropertyChanged();
   OnPropertyChanged(nameof(FullName));
   SaveCommand.RaiseCanExecuteChanged();
  }
 }
}
```

### Toolkit 联动

```csharp
[ObservableProperty]
[NotifyPropertyChangedFor(nameof(FullName))]
[NotifyCanExecuteChangedFor(nameof(SaveCommand))]
private string firstName = string.Empty;
```

本质还是一样的通知，只是 Toolkit 把联动规则显式写成了特性。

## 六、消息通信怎么对比

传统项目里，跨 ViewModel 通信常见几种方式：

* 直接互相持有引用；
* 自己封装事件聚合器；
* 用全局事件或中介类；
* 手写一个 Messenger。

而 CommunityToolkit.Mvvm 已经直接提供了 WeakReferenceMessenger 和 ObservableRecipient。

### 传统自写事件聚合器思路

```csharp
public static class EventHub
{
 public static event Action<string>? UserLoggedIn;
}
```

### Toolkit 写法

```csharp
WeakReferenceMessenger.Default.Send(new UserLoggedInMessage(userName));
```

接收方：

```csharp
Messenger.Register<ShellViewModel, UserLoggedInMessage>(this, static (recipient, message) =>
{
 recipient.CurrentUserText = message.Value;
});
```

它的优势是：

* 现成可用；
* 语义更清晰；
* 生命周期管理更规范；
* 不用每个项目再自己造一套轮子。

## 七、到底省掉了哪些样板代码

如果从日常开发体感上看，Toolkit 主要省掉的是这几类重复劳动：

1. INotifyPropertyChanged 模板代码；
2. 属性 setter 中重复的通知逻辑；
3. ICommand / RelayCommand 初始化模板；
4. 属性变化后手动刷新命令状态的重复代码；
5. 自己搭一套简单消息总线的基础设施代码。

所以它节省的不是一两行，而是一整类重复模式。

## 八、传统手写模式还有哪些优势

说清楚优点也要说清楚边界。

传统手写模式并不是“落后写法”，它仍然有自己的优势：

### 1. 控制非常显式

每一行通知、每一个命令实例、每一次事件抛出，都是自己手写的。

### 2. 不依赖源生成器特性风格

有些团队对源生成器比较保守，更倾向于手写显式代码。

### 3. 复杂 setter 或复杂流程时有时更直观

如果属性赋值逻辑特别复杂，纯手写反而容易看清流程。

## 九、CommunityToolkit.Mvvm 的主要优势

### 1. 更适合中大型 ViewModel

属性和命令一多，样板代码会急剧增加，Toolkit 的收益会越来越明显。

### 2. 团队代码风格更统一

大家都用同一套 ObservableProperty、RelayCommand、Messenger 约定，代码一致性更高。

### 3. 更适合快速搭建 WPF MVVM 基础设施

不用每个项目从头封装基类、命令类、消息总线。

### 4. 和异步、联动通知、消息机制的配套更完整

这点很关键，因为实际项目最麻烦的地方往往不是单一属性通知，而是异步命令、联动刷新、跨模块通信一起出现时。

## 十、什么时候更适合用 Toolkit

下面这些情况尤其适合：

* 新项目准备标准化 MVVM 写法；
* 界面较多、ViewModel 较多；
* 有大量命令和属性联动；
* 需要异步命令和消息通信；
* 希望减少重复代码，提高可维护性。

## 十一、什么时候不一定非要迁移

下面这些情况就不一定非要强行改：

* 老项目已经有稳定的一套 MVVM 基类；
* 团队已有成熟的命令和消息基础设施；
* 项目规模很小，手写成本本来就不高；
* 当前痛点并不在样板代码，而在业务分层混乱。

也就是说，Toolkit 很好，但不是所有项目都必须“立刻全面重构”。

## 十二、一个实用判断标准

可以用一句很实际的话判断：

如果你在项目里频繁重复写这些内容：

* 属性通知模板；
* 命令初始化模板；
* RaiseCanExecuteChanged；
* 自己的消息中介类；

那 CommunityToolkit.Mvvm 通常值得上。

如果你当前项目的主要问题是模块耦合、服务层混乱、职责不清，那先解决架构边界，收益可能比换 Toolkit 更大。

## 总结

1. CommunityToolkit.Mvvm 没有改变 MVVM 的本质，只是把大量重复模板做了封装和生成；
2. 传统手写模式更显式，但样板代码更多；
3. CommunityToolkit.Mvvm 在属性通知、命令、异步命令、联动通知和消息通信上都更省力；
4. 对新项目和中大型 WPF 项目来说，Toolkit 往往更容易形成统一、整洁的 MVVM 写法；
5. 是否采用 Toolkit，最终还是要看项目规模、团队习惯和现有基础设施。
