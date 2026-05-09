# RelayCommand 特性

在 CommunityToolkit.Mvvm 中，除了可以手动 new RelayCommand 或 AsyncRelayCommand，还可以直接使用 [RelayCommand] 特性生成命令。

它的核心作用是：

把一个方法自动生成为可绑定的命令属性。

换句话说，它解决的是下面这类重复代码：

```csharp
public IRelayCommand SaveCommand { get; }

public MainViewModel()
{
 SaveCommand = new RelayCommand(Save);
}

private void Save()
{
}
```

很多时候，你真正关心的只是“方法本身”，而不是命令对象初始化样板。[RelayCommand] 就是把这层样板代码交给源生成器处理。

## 一、基础写法

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [RelayCommand]
 private void Save()
 {
 }
}
```

上面写完后，编译器会自动帮你生成一个可绑定命令，通常可以理解为：

```csharp
public IRelayCommand SaveCommand { get; }
```

所以 XAML 中可以直接绑定：

```xml
<Button Content="保存" Command="{Binding SaveCommand}"/>
```

## 二、为什么它很实用

它最直接的好处主要有几个：

1. 少写构造函数里的命令初始化代码；
2. 命令名自动统一；
3. 和 ObservableProperty、NotifyCanExecuteChangedFor 配合更自然；
4. ViewModel 更聚焦于“有哪些动作”，而不是“怎么 new 命令对象”。

特别是当一个 ViewModel 里有很多按钮操作时，差别会很明显。

## 三、命令名是怎么生成的

一般来说，它会基于方法名生成命令名。

例如：

```csharp
[RelayCommand]
private void Save()
{
}

[RelayCommand]
private void DeleteItem()
{
}

[RelayCommand]
private async Task LoadAsync()
{
 await Task.Delay(1000);
}
```

通常会生成：

* SaveCommand
* DeleteItemCommand
* LoadCommand

注意最后一个例子：

* 方法名是 LoadAsync；
* 生成的命令名通常是 LoadCommand；
* Async 后缀会被去掉。

这也是 Toolkit 很常见的一种命名约定。

## 四、同步方法会生成什么命令

如果被标注的方法是普通同步方法，通常会生成 IRelayCommand。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string status = "未执行";

 [RelayCommand]
 private void Reset()
 {
  Status = "已重置";
 }
}
```

XAML：

```xml
<Button Content="重置" Command="{Binding ResetCommand}"/>
<TextBlock Text="{Binding Status}"/>
```

这种写法适合同步操作，比如：

* 清空表单；
* 切换选中状态；
* 打开本地窗口；
* 本地数据处理。

## 五、异步方法会生成什么命令

如果方法签名是 async Task，通常会生成 IAsyncRelayCommand。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string status = "未加载";

 [RelayCommand]
 private async Task LoadDataAsync()
 {
  Status = "加载中...";
  await Task.Delay(1500);
  Status = "加载完成";
 }
}
```

XAML：

```xml
<Button Content="加载数据" Command="{Binding LoadDataCommand}"/>
<TextBlock Text="{Binding Status}"/>
```

这也是 [RelayCommand] 最方便的一点：

你只需要关心方法签名，生成器会根据方法类型自动选择合适的命令种类。

## 六、带参数的方法怎么生成命令

如果方法带参数，就会生成对应的泛型命令。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string status = "未删除";

 [RelayCommand]
 private void DeleteUser(string userName)
 {
  Status = $"已删除：{userName}";
 }
}
```

XAML：

```xml
<Button Content="删除张三"
        Command="{Binding DeleteUserCommand}"
        CommandParameter="张三"/>
```

如果是异步带参数方法，同理会生成对应的异步泛型命令。

## 七、如何控制 CanExecute

命令不是任何时候都应该可执行的。

[RelayCommand] 支持通过 CanExecute 指定条件方法。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 [NotifyCanExecuteChangedFor(nameof(LoginCommand))]
 private string userName = string.Empty;

 [ObservableProperty]
 [NotifyCanExecuteChangedFor(nameof(LoginCommand))]
 private string password = string.Empty;

 [RelayCommand(CanExecute = nameof(CanLogin))]
 private void Login()
 {
 }

 private bool CanLogin()
 {
  return !string.IsNullOrWhiteSpace(UserName)
   && !string.IsNullOrWhiteSpace(Password);
 }
}
```

这个组合非常常见：

* ObservableProperty 负责属性变化通知；
* NotifyCanExecuteChangedFor 负责属性变化后刷新命令状态；
* [RelayCommand(CanExecute = ...)] 负责定义命令能不能执行。

## 八、一个实际项目示例

下面给一个更完整的查询页示例：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

namespace Demo.ViewModels;

public partial class SearchViewModel : ObservableObject
{
 public ObservableCollection<string> Results { get; } = new();

 [ObservableProperty]
 [NotifyCanExecuteChangedFor(nameof(SearchCommand))]
 private string keyword = string.Empty;

 [ObservableProperty]
 private string statusMessage = "请输入关键字";

 [RelayCommand(CanExecute = nameof(CanSearch))]
 private async Task SearchAsync()
 {
  try
  {
   StatusMessage = "查询中...";
   await Task.Delay(1200);

   Results.Clear();
   Results.Add($"查询结果 1：{Keyword}");
   Results.Add($"查询结果 2：{Keyword}");

   StatusMessage = "查询完成";
  }
  catch (Exception ex)
  {
   StatusMessage = $"查询失败：{ex.Message}";
  }
 }

 [RelayCommand]
 private void Clear()
 {
  Keyword = string.Empty;
  Results.Clear();
  StatusMessage = "已清空";
 }

 private bool CanSearch()
 {
  return !string.IsNullOrWhiteSpace(Keyword);
 }
}
```

对应 XAML：

```xml
<StackPanel Margin="20">
 <TextBox Margin="0,0,0,10"
   Text="{Binding Keyword, UpdateSourceTrigger=PropertyChanged}"/>

 <StackPanel Orientation="Horizontal">
  <Button Content="查询"
    Width="100"
    Command="{Binding SearchCommand}"/>
  <Button Content="清空"
    Width="100"
    Margin="10,0,0,0"
    Command="{Binding ClearCommand}"/>
 </StackPanel>

 <TextBlock Margin="0,10,0,10" Text="{Binding StatusMessage}"/>
 <ListBox ItemsSource="{Binding Results}"/>
</StackPanel>
```

这个例子里可以清楚看到：

1. SearchAsync 自动生成 SearchCommand；
2. Clear 自动生成 ClearCommand；
3. Keyword 变化时，会自动刷新 SearchCommand 的 CanExecute；
4. ViewModel 代码比手动 new 命令更紧凑。

## 九、和手动 new RelayCommand 的区别

两种方式本质上都能完成命令绑定，只是写法不同。

### 手动写法

```csharp
public IRelayCommand SaveCommand { get; }

public MainViewModel()
{
 SaveCommand = new RelayCommand(Save);
}
```

### 特性写法

```csharp
[RelayCommand]
private void Save()
{
}
```

可以这样理解：

* 手动写法更显式；
* 特性写法更简洁；
* 小型和中型 ViewModel 中，特性写法通常更省事；
* 如果命令初始化逻辑非常特殊，手动写法有时更灵活。

## 十、常见注意点

### 1. 当前类通常要写成 partial

因为命令属性是源生成器生成的，所以类一般要声明成 partial。

### 2. 方法本身才是核心，不要只盯着生成结果

使用 [RelayCommand] 时，真正需要你好好设计的是：

* 这个命令方法要做什么；
* 它是否应该异步；
* 它是否需要参数；
* 它是否应该有 CanExecute。

特性只是帮你减少样板，不会替你设计命令边界。

### 3. async void 仍然不推荐

推荐：

```csharp
[RelayCommand]
private async Task LoadAsync()
{
 await Task.Delay(1000);
}
```

不推荐：

```csharp
[RelayCommand]
private async void LoadAsync()
{
 await Task.Delay(1000);
}
```

除了事件处理器，async void 基本都应该避免。

### 4. CanExecute 依赖属性变化时，要记得刷新命令状态

最省事的做法通常不是手动在 setter 里写 NotifyCanExecuteChanged，而是直接配合：

* ObservableProperty
* NotifyCanExecuteChangedFor

这样结构会更清晰。

### 5. 不要和同名命令属性冲突

如果你已经手写了：

```csharp
public IRelayCommand SaveCommand { get; }
```

这时又写：

```csharp
[RelayCommand]
private void Save()
{
}
```

就会冲突，因为生成器也会尝试生成 SaveCommand。

## 十一、什么时候特别适合用 [RelayCommand]

下面这些场景尤其适合：

* 表单按钮命令；
* 查询、重置、刷新、删除这类标准动作；
* 一个 ViewModel 里命令比较多；
* 命令逻辑本身清晰，不需要复杂初始化；
* 项目已经大量使用 CommunityToolkit.Mvvm 特性风格。

如果你的 ViewModel 已经在使用 ObservableProperty，那么 [RelayCommand] 通常也很适合一起使用。

## 总结

1. [RelayCommand] 的核心作用是把方法自动生成成可绑定命令；
2. 同步方法通常生成 RelayCommand，异步 Task 方法通常生成 AsyncRelayCommand；
3. 带参数方法会生成对应的泛型命令；
4. 它和 ObservableProperty、NotifyCanExecuteChangedFor 配合使用时效果最好；
5. 在 CommunityToolkit.Mvvm 中，它是减少 ViewModel 样板代码最有效的特性之一。
