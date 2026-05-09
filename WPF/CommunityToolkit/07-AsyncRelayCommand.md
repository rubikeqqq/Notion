# AsyncRelayCommand 和 RelayCommand

在 CommunityToolkit.Mvvm 中，RelayCommand 和 AsyncRelayCommand 都用于把 UI 操作封装成命令，供 Button、MenuItem、CommandBar 等控件绑定。

它们解决的是同一个问题：

1. 把点击事件从后台代码中抽离出来；
2. 配合 MVVM，把视图行为转交给 ViewModel；
3. 统一处理 “能不能执行” 和 “执行什么逻辑”。

它们最大的区别在于：

* RelayCommand 适合同步逻辑；
* AsyncRelayCommand 适合异步逻辑，尤其是网络请求、文件读写、数据库访问这类耗时操作。

## 一、为什么需要 Command

在传统写法中，我们经常会在后台代码里直接写按钮点击事件：

```xml
<Button Content="保存" Click="Button_Click"/>
```

这种写法的问题是：

1. View 和逻辑耦合太紧；
2. 不利于测试；
3. 按钮是否可点击、执行中是否禁用，都要手写控制。

而命令绑定写法更符合 MVVM：

```xml
<Button Content="保存" Command="{Binding SaveCommand}"/>
```

View 只负责绑定，真正的业务逻辑放到 ViewModel 中。

## 二、RelayCommand 是什么

RelayCommand 用于封装同步命令。

最典型的场景：

* 打开弹窗；
* 切换状态；
* 删除当前项；
* 执行纯内存计算；
* 调用一个很快完成的方法。

### 基础示例

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string message = "准备就绪";

 public IRelayCommand SaveCommand { get; }

 public MainViewModel()
 {
  SaveCommand = new RelayCommand(Save);
 }

 private void Save()
 {
  Message = "已保存";
 }
}
```

XAML 绑定：

```xml
<Button Content="保存"
  Command="{Binding SaveCommand}"
  Width="120"
  Height="36"/>

<TextBlock Text="{Binding Message}" Margin="0,12,0,0"/>
```

### 带 CanExecute 的同步命令

如果按钮不是任何时候都允许执行，可以给 RelayCommand 配一个可执行条件：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 [NotifyCanExecuteChangedFor(nameof(DeleteCommand))]
 private bool hasSelection;

 public IRelayCommand DeleteCommand { get; }

 public MainViewModel()
 {
  DeleteCommand = new RelayCommand(Delete, CanDelete);
 }

 private void Delete()
 {
 }

 private bool CanDelete()
 {
  return HasSelection;
 }
}
```

这里的重点是：

* 当 HasSelection 变化时，会自动触发 DeleteCommand 的 CanExecuteChanged；
* Button 会自动跟着变成可用或禁用。

## 三、AsyncRelayCommand 是什么

AsyncRelayCommand 用于封装异步命令，本质上是专门为 async/await 设计的命令类型。

它适合：

* 加载列表数据；
* 提交表单；
* 调接口；
* 异步刷新页面；
* 执行可取消的耗时任务。

如果你把异步方法强行塞进 RelayCommand，虽然有时也能运行，但通常会带来几个问题：

1. 异常不容易被正确观察；
2. 命令执行状态不好管理；
3. 容易出现重复点击、并发执行；
4. 很难优雅地支持取消。

所以，异步逻辑应该优先用 AsyncRelayCommand。

## 四、AsyncRelayCommand 基础写法

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 public ObservableCollection<string> Users { get; } = new();

 [ObservableProperty]
 private string status = "未加载";

 public IAsyncRelayCommand LoadUsersCommand { get; }

 public MainViewModel()
 {
  LoadUsersCommand = new AsyncRelayCommand(LoadUsersAsync);
 }

 private async Task LoadUsersAsync()
 {
  Status = "加载中...";

  await Task.Delay(1500);

  Users.Clear();
  Users.Add("张三");
  Users.Add("李四");
  Users.Add("王五");

  Status = "加载完成";
 }
}
```

对应的 XAML：

```xml
<StackPanel Margin="20">
 <Button Content="加载用户"
   Command="{Binding LoadUsersCommand}"
   Width="120"
   Height="36"/>

 <TextBlock Text="{Binding Status}" Margin="0,12,0,12"/>

 <ListBox ItemsSource="{Binding Users}" Height="160"/>
</StackPanel>
```

## 五、AsyncRelayCommand 比 RelayCommand 多出的关键能力

### 1. 自动感知执行中状态

AsyncRelayCommand 内部会跟踪当前任务，因此你可以直接使用它的状态属性，比如：

* IsRunning：命令是否正在执行；
* ExecutionTask：当前执行的 Task；
* CanBeCanceled：是否支持取消；
* IsCancellationRequested：是否已请求取消。

例如：

```xml
<Button Content="加载"
  Command="{Binding LoadUsersCommand}"
  IsEnabled="{Binding LoadUsersCommand.IsRunning, Converter={StaticResource InverseBoolConverter}}"/>

<ProgressBar IsIndeterminate="True"
    Visibility="{Binding LoadUsersCommand.IsRunning, Converter={StaticResource BoolToVisibilityConverter}}"/>
```

很多时候，你甚至不需要自己再额外定义一个 IsBusy。

### 2. 支持取消操作

AsyncRelayCommand 支持传入带 CancellationToken 的异步方法。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 public IAsyncRelayCommand DownloadCommand { get; }

 public MainViewModel()
 {
  DownloadCommand = new AsyncRelayCommand(DownloadAsync);
 }

 private async Task DownloadAsync(CancellationToken cancellationToken)
 {
  for (int i = 0; i < 10; i++)
  {
   cancellationToken.ThrowIfCancellationRequested();
   await Task.Delay(500, cancellationToken);
  }
 }
}
```

XAML 中可以绑定一个取消按钮：

```xml
<Button Content="开始下载" Command="{Binding DownloadCommand}"/>
<Button Content="取消下载"
  Command="{Binding DownloadCommand.CancelCommand}"
  Margin="0,8,0,0"/>
```

如果当前 Toolkit 版本没有直接暴露 CancelCommand，也可以自己在 ViewModel 中提供一个普通命令去调用：

```csharp
public IRelayCommand CancelDownloadCommand { get; }

public MainViewModel()
{
 DownloadCommand = new AsyncRelayCommand(DownloadAsync);
 CancelDownloadCommand = new RelayCommand(() => DownloadCommand.Cancel());
}
```

### 3. 更适合和异常处理配合

异步方法中的异常不能像同步方法那样随便忽略。

推荐做法是直接在异步方法内部处理业务异常：

```csharp
private async Task LoadUsersAsync()
{
 try
 {
  Status = "加载中...";
  await Task.Delay(1000);
  Status = "加载成功";
 }
 catch (Exception ex)
 {
  Status = $"加载失败：{ex.Message}";
 }
}
```

这样 UI 状态会更清晰，也更容易维护。

## 六、RelayCommand 和 AsyncRelayCommand 如何选择

可以直接按下面的标准判断：

### 用 RelayCommand 的场景

* 执行逻辑是同步的；
* 方法里不需要 await；
* 执行时间很短，不会阻塞 UI；
* 不需要取消；
* 不需要跟踪任务状态。

### 用 AsyncRelayCommand 的场景

* 方法里有 await；
* 涉及 IO、网络、数据库、磁盘；
* 希望显示加载中状态；
* 要避免重复点击导致并发执行；
* 要支持取消任务。

一个非常实用的经验是：

* 只要命令处理方法是 async Task，优先考虑 AsyncRelayCommand；
* 不要为了偷懒写成 async void 再交给 RelayCommand。

## 七、不要把 async void 塞进 RelayCommand

下面这种写法看起来能跑，但不推荐：

```csharp
public IRelayCommand RefreshCommand { get; }

public MainViewModel()
{
 RefreshCommand = new RelayCommand(async () => await RefreshAsync());
}
```

问题在于：

1. 这里的 lambda 会退化成 async void 风格；
2. 异常传播不可控；
3. 无法直接感知命令是否在执行；
4. 不能方便地取消。

更合适的写法是：

```csharp
public IAsyncRelayCommand RefreshCommand { get; }

public MainViewModel()
{
 RefreshCommand = new AsyncRelayCommand(RefreshAsync);
}
```

## 八、配合源生成器的写法

CommunityToolkit.Mvvm 常和特性一起用。虽然 AsyncRelayCommand 和 RelayCommand 可以手动 new，但在实际项目里也常见下面这种方式：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string keyword = string.Empty;

 [RelayCommand]
 private void ClearKeyword()
 {
  Keyword = string.Empty;
 }

 [RelayCommand]
 private async Task SearchAsync()
 {
  await Task.Delay(1000);
 }
}
```

生成结果可以理解为：

* ClearKeyword 会生成一个同步命令；
* SearchAsync 会生成一个异步命令。

也就是说，[RelayCommand] 特性会根据方法签名自动生成合适的命令类型。

常见生成命令名如下：

* ClearKeyword() -> ClearKeywordCommand
* SearchAsync() -> SearchCommand

XAML 里直接绑定：

```xml
<Button Content="清空" Command="{Binding ClearKeywordCommand}"/>
<Button Content="搜索" Command="{Binding SearchCommand}" Margin="8,0,0,0"/>
```

## 九、常见注意点

### 1. 异步命令的方法签名优先写成 Task

推荐：

```csharp
private async Task LoadAsync()
{
 await Task.Delay(1000);
}
```

不推荐：

```csharp
private async void LoadAsync()
{
 await Task.Delay(1000);
}
```

除了事件处理器，async void 基本都应该避免。

### 2. 注意重复执行问题

某些场景下，用户可能连续点击按钮。如果你不希望命令并发执行，需要明确约束。

实践中常见做法：

* 利用 AsyncRelayCommand 自身的执行状态禁用按钮；
* 或在方法开始前判断业务状态；
* 或通过命令配置控制并发行为。

### 3. UI 绑定时别忘了 DataContext

如果按钮点了没反应，先检查：

1. ViewModel 是否设置为当前视图的 DataContext；
2. 绑定名称是否正确；
3. CanExecute 是否返回 false；
4. 异步方法中是否抛出了异常。

## 十、一个完整示例

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

namespace Demo.ViewModels;

public partial class UserViewModel : ObservableObject
{
 public ObservableCollection<string> Users { get; } = new();

 [ObservableProperty]
 private string status = "等待加载";

 public IAsyncRelayCommand LoadCommand { get; }
 public IRelayCommand ClearCommand { get; }

 public UserViewModel()
 {
  LoadCommand = new AsyncRelayCommand(LoadAsync);
  ClearCommand = new RelayCommand(Clear);
 }

 private async Task LoadAsync()
 {
  try
  {
   Status = "加载中...";
   await Task.Delay(1200);

   Users.Clear();
   Users.Add("A 用户");
   Users.Add("B 用户");
   Users.Add("C 用户");

   Status = "加载完成";
  }
  catch (Exception ex)
  {
   Status = $"出错：{ex.Message}";
  }
 }

 private void Clear()
 {
  Users.Clear();
  Status = "已清空";
 }
}
```

```xml
<StackPanel Margin="20">
 <StackPanel Orientation="Horizontal">
  <Button Content="异步加载"
    Command="{Binding LoadCommand}"
    Width="100"/>
  <Button Content="清空"
    Command="{Binding ClearCommand}"
    Width="100"
    Margin="10,0,0,0"/>
 </StackPanel>

 <TextBlock Text="{Binding Status}" Margin="0,12,0,12"/>

 <ListBox ItemsSource="{Binding Users}" Height="150"/>
</StackPanel>
```

这里可以清楚看到：

* ClearCommand 处理同步清空；
* LoadCommand 处理异步加载；
* 两者职责分明，代码可维护性更高。

## 十一、实际项目示例：登录 + 刷新列表

在真实项目里，经常不是只有一个命令，而是同步命令和异步命令一起出现。

例如：

* 登录按钮要异步调用接口；
* 清空输入框是同步操作；
* 刷新列表通常也是异步操作；
* 执行过程中还要控制按钮禁用和提示文本。

下面给一个更接近业务界面的 ViewModel：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;

namespace Demo.ViewModels;

public partial class LoginViewModel : ObservableObject
{
 [ObservableProperty]
 private string userName = string.Empty;

 [ObservableProperty]
 private string password = string.Empty;

 [ObservableProperty]
 private string statusMessage = "请输入账号和密码";

 public ObservableCollection<string> Logs { get; } = new();

 public IAsyncRelayCommand LoginCommand { get; }
 public IAsyncRelayCommand RefreshLogsCommand { get; }
 public IRelayCommand ClearCommand { get; }

 public LoginViewModel()
 {
  LoginCommand = new AsyncRelayCommand(LoginAsync, CanLogin);
  RefreshLogsCommand = new AsyncRelayCommand(RefreshLogsAsync);
  ClearCommand = new RelayCommand(Clear);
 }

 partial void OnUserNameChanged(string value)
 {
  LoginCommand.NotifyCanExecuteChanged();
 }

 partial void OnPasswordChanged(string value)
 {
  LoginCommand.NotifyCanExecuteChanged();
 }

 private bool CanLogin()
 {
  return !string.IsNullOrWhiteSpace(UserName)
   && !string.IsNullOrWhiteSpace(Password)
   && !LoginCommand.IsRunning;
 }

 private async Task LoginAsync()
 {
  try
  {
   StatusMessage = "登录中...";
   await Task.Delay(1500);

   Logs.Add($"{DateTime.Now:HH:mm:ss} 用户 {UserName} 登录成功");
   StatusMessage = "登录成功";
  }
  catch (Exception ex)
  {
   StatusMessage = $"登录失败：{ex.Message}";
  }
  finally
  {
   LoginCommand.NotifyCanExecuteChanged();
  }
 }

 private async Task RefreshLogsAsync()
 {
  try
  {
   StatusMessage = "刷新日志中...";
   await Task.Delay(1000);

   if (Logs.Count == 0)
   {
    Logs.Add("暂无日志记录");
   }

   StatusMessage = "刷新完成";
  }
  catch (Exception ex)
  {
   StatusMessage = $"刷新失败：{ex.Message}";
  }
 }

 private void Clear()
 {
  UserName = string.Empty;
  Password = string.Empty;
  Logs.Clear();
  StatusMessage = "已清空输入和日志";
 }
}
```

对应 XAML：

```xml
<Grid Margin="20">
 <Grid.RowDefinitions>
  <RowDefinition Height="Auto"/>
  <RowDefinition Height="Auto"/>
  <RowDefinition Height="Auto"/>
  <RowDefinition Height="*"/>
 </Grid.RowDefinitions>

 <TextBox Grid.Row="0"
   Margin="0,0,0,10"
   Text="{Binding UserName, UpdateSourceTrigger=PropertyChanged}"/>

 <PasswordBox Grid.Row="1" Margin="0,0,0,10"/>

 <StackPanel Grid.Row="2" Orientation="Horizontal">
  <Button Content="登录"
    Width="100"
    Command="{Binding LoginCommand}"/>
  <Button Content="刷新日志"
    Width="100"
    Margin="10,0,0,0"
    Command="{Binding RefreshLogsCommand}"/>
  <Button Content="清空"
    Width="100"
    Margin="10,0,0,0"
    Command="{Binding ClearCommand}"/>
 </StackPanel>

 <StackPanel Grid.Row="3" Margin="0,12,0,0">
  <TextBlock Text="{Binding StatusMessage}" Margin="0,0,0,8"/>

  <ProgressBar IsIndeterminate="True"
      Height="4"
      Visibility="{Binding LoginCommand.IsRunning, Converter={StaticResource BoolToVisibilityConverter}}"/>

  <ListBox Margin="0,12,0,0" ItemsSource="{Binding Logs}"/>
 </StackPanel>
</Grid>
```

这个例子里有几个点比较关键：

1. 登录和刷新日志是异步命令，因为它们通常要访问服务端或本地资源；
2. 清空输入框是同步命令，因为它只是修改内存状态；
3. LoginCommand 的 CanExecute 会根据输入内容和运行状态变化；
4. UI 可以直接绑定 LoginCommand.IsRunning 来控制进度条显示。

如果你项目里使用 PasswordBox，通常还要额外处理密码绑定问题，因为它不是普通依赖属性，不能像 TextBox 那样直接双向绑定。

## 十二、项目中更实用的写法建议

### 1. 命令负责流程，业务放到服务层

ViewModel 中的命令最好只负责：

* 收集输入；
* 调用服务；
* 更新界面状态；
* 处理错误提示。

不要把复杂数据库逻辑、HTTP 细节、文件解析都堆到命令方法里，否则后面会越来越难维护。

更合理的结构通常是：

```csharp
private readonly IUserService _userService;

private async Task LoginAsync()
{
 var result = await _userService.LoginAsync(UserName, Password);
 StatusMessage = result.Message;
}
```

### 2. 尽量让 CanExecute 和界面状态一致

很多命令“看起来没问题”，实际上体验很差，原因通常是：

* 方法执行中按钮还可以反复点；
* 输入为空时按钮没有禁用；
* 执行失败后 UI 没有提示。

所以命令设计时，除了“能不能执行”，还要考虑：

* 执行中是否要禁用；
* 是否需要显示 Busy 状态；
* 失败后是否需要重试；
* 是否需要取消。

### 3. 区分“命令状态”和“页面状态”

AsyncRelayCommand 已经能提供 IsRunning，但页面里有时还会有更宽的状态，比如：

* 当前是否已登录；
* 当前页面是否正在初始化；
* 当前是否处于编辑模式。

这类状态仍然建议单独建属性，不要把所有状态都强行塞进命令本身。

## 十三、PasswordBox 在 MVVM 中怎么处理

前面登录示例里故意只放了一个简单的 PasswordBox，因为它和 TextBox 不一样：

* TextBox.Text 是依赖属性，可以直接双向绑定；
* PasswordBox.Password 不是依赖属性，不能直接写成普通的 TwoWay 绑定。

这也是很多人刚写登录页时最容易卡住的地方。

### 1. 最直接的做法：在命令执行时读取 PasswordBox

如果你只是做一个简单登录页，最省事的方法通常不是强行双向绑定，而是在命令参数里把 PasswordBox 传进来：

```xml
<PasswordBox x:Name="PwdBox" Grid.Row="1" Margin="0,0,0,10"/>

<Button Content="登录"
        Command="{Binding LoginCommand}"
        CommandParameter="{Binding ElementName=PwdBox}"/>
```

ViewModel 可以把命令定义成带参数形式：

```csharp
public IAsyncRelayCommand<PasswordBox> LoginCommand { get; }

public LoginViewModel()
{
 LoginCommand = new AsyncRelayCommand<PasswordBox>(LoginAsync, CanLogin);
}

private async Task LoginAsync(PasswordBox? passwordBox)
{
 if (passwordBox is null)
 {
  StatusMessage = "未获取到密码输入框";
  return;
 }

 string password = passwordBox.Password;

 StatusMessage = "登录中...";
 await Task.Delay(1000);
 StatusMessage = $"用户 {UserName} 登录成功";
}
```

这种方式的优点是：

* 实现简单；
* 不需要额外附加属性；
* 对单个登录页很实用。

缺点也很明确：

* ViewModel 直接依赖了 PasswordBox；
* 纯粹性不如标准 MVVM；
* 单元测试不如字符串属性方案方便。

所以它适合“小而快”的页面，不一定适合要求严格分层的项目。

### 2. 更常见的 MVVM 做法：附加属性同步 Password

如果你希望 ViewModel 里仍然只保留字符串 Password 属性，常见做法是写一个附加属性，把 PasswordBox.Password 同步到绑定源。

示例辅助类：

```csharp
using System.Windows;
using System.Windows.Controls;

namespace Demo.Helpers;

public static class PasswordBoxHelper
{
 public static readonly DependencyProperty BoundPasswordProperty =
  DependencyProperty.RegisterAttached(
   "BoundPassword",
   typeof(string),
   typeof(PasswordBoxHelper),
   new PropertyMetadata(string.Empty, OnBoundPasswordChanged));

 public static readonly DependencyProperty BindPasswordProperty =
  DependencyProperty.RegisterAttached(
   "BindPassword",
   typeof(bool),
   typeof(PasswordBoxHelper),
   new PropertyMetadata(false, OnBindPasswordChanged));

 private static readonly DependencyProperty UpdatingPasswordProperty =
  DependencyProperty.RegisterAttached(
   "UpdatingPassword",
   typeof(bool),
   typeof(PasswordBoxHelper),
   new PropertyMetadata(false));

 public static string GetBoundPassword(DependencyObject obj)
 {
  return (string)obj.GetValue(BoundPasswordProperty);
 }

 public static void SetBoundPassword(DependencyObject obj, string value)
 {
  obj.SetValue(BoundPasswordProperty, value);
 }

 public static bool GetBindPassword(DependencyObject obj)
 {
  return (bool)obj.GetValue(BindPasswordProperty);
 }

 public static void SetBindPassword(DependencyObject obj, bool value)
 {
  obj.SetValue(BindPasswordProperty, value);
 }

 private static bool GetUpdatingPassword(DependencyObject obj)
 {
  return (bool)obj.GetValue(UpdatingPasswordProperty);
 }

 private static void SetUpdatingPassword(DependencyObject obj, bool value)
 {
  obj.SetValue(UpdatingPasswordProperty, value);
 }

 private static void OnBoundPasswordChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
 {
  if (d is not PasswordBox passwordBox)
  {
   return;
  }

  passwordBox.PasswordChanged -= PasswordBox_PasswordChanged;

  if (!GetUpdatingPassword(passwordBox))
  {
   passwordBox.Password = e.NewValue?.ToString() ?? string.Empty;
  }

  passwordBox.PasswordChanged += PasswordBox_PasswordChanged;
 }

 private static void OnBindPasswordChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
 {
  if (d is not PasswordBox passwordBox)
  {
   return;
  }

  bool wasBound = (bool)e.OldValue;
  bool needToBind = (bool)e.NewValue;

  if (wasBound)
  {
   passwordBox.PasswordChanged -= PasswordBox_PasswordChanged;
  }

  if (needToBind)
  {
   passwordBox.PasswordChanged += PasswordBox_PasswordChanged;
  }
 }

 private static void PasswordBox_PasswordChanged(object sender, RoutedEventArgs e)
 {
  if (sender is not PasswordBox passwordBox)
  {
   return;
  }

  SetUpdatingPassword(passwordBox, true);
  SetBoundPassword(passwordBox, passwordBox.Password);
  SetUpdatingPassword(passwordBox, false);
 }
}
```

然后在 XAML 中使用：

```xml
<Window
    xmlns:helpers="clr-namespace:Demo.Helpers">

    <PasswordBox
        helpers:PasswordBoxHelper.BindPassword="True"
        helpers:PasswordBoxHelper.BoundPassword="{Binding Password, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>
</Window>
```

这样 ViewModel 中就可以正常使用字符串属性：

```csharp
[ObservableProperty]
private string password = string.Empty;
```

这种方式的优点是：

* ViewModel 不直接依赖具体控件；
* 更接近标准 MVVM；
* 和 AsyncRelayCommand、CanExecute 搭配更自然。

### 3. 结合 AsyncRelayCommand 的典型用法

当 Password 属性被同步到 ViewModel 后，就可以直接参与命令的可执行判断：

```csharp
public IAsyncRelayCommand LoginCommand { get; }

public LoginViewModel()
{
 LoginCommand = new AsyncRelayCommand(LoginAsync, CanLogin);
}

partial void OnUserNameChanged(string value)
{
 LoginCommand.NotifyCanExecuteChanged();
}

partial void OnPasswordChanged(string value)
{
 LoginCommand.NotifyCanExecuteChanged();
}

private bool CanLogin()
{
 return !string.IsNullOrWhiteSpace(UserName)
  && !string.IsNullOrWhiteSpace(Password)
  && !LoginCommand.IsRunning;
}
```

这时按钮状态就会比较符合真实业务：

* 用户名为空，不能登录；
* 密码为空，不能登录；
* 正在登录时，不能重复点击；
* 登录结束后，再重新恢复可执行状态。

### 4. 关于安全性的说明

PasswordBox 之所以没有直接把 Password 做成普通依赖属性，部分原因就是出于安全层面的考虑。

因此要注意：

* 把密码绑定成普通字符串后，会在内存里以字符串形式存在；
* 这对一般业务系统是常见做法，但不代表完全没有风险；
* 如果项目安全要求很高，需要结合具体规范评估是否要改用更严格的处理方式。

对大多数企业内部 WPF 系统来说，上面的附加属性方案已经足够常见且实用。

## 总结

1. RelayCommand 用于同步命令，适合轻量、快速完成的操作；
2. AsyncRelayCommand 用于异步命令，适合 await、耗时操作、取消控制和加载状态管理；
3. 遇到 async Task 方法时，优先使用 AsyncRelayCommand，而不是把异步 lambda 塞给 RelayCommand；
4. 在实际项目里，通常会让同步命令负责本地状态切换，让异步命令负责接口调用和数据加载；
5. 在 CommunityToolkit.Mvvm 中，配合 ObservableObject、ObservableProperty、RelayCommand 特性，能让 WPF 的 MVVM 写法更简洁。
