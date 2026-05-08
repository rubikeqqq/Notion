# ObservableProperty

在 CommunityToolkit.Mvvm 中，ObservableProperty 是一个非常高频的特性。它的核心作用是：

把一个字段自动生成为支持属性变更通知的公开属性。

如果只看结果，它帮我们做的事情其实很明确：

1. 自动生成公开属性；
2. 自动调用 SetProperty；
3. 自动触发 PropertyChanged；
4. 减少 ViewModel 中重复的样板代码。

它通常和 ObservableObject 一起使用，是 CommunityToolkit.Mvvm 里最常见的一组搭配。

## 一、为什么需要 ObservableProperty

在传统 WPF / MVVM 写法里，如果一个属性要参与界面绑定，通常要手动写一套完整代码：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 private string userName = string.Empty;

 public string UserName
 {
  get => userName;
  set => SetProperty(ref userName, value);
 }
}
```

这段代码本身没有问题，但如果一个 ViewModel 有十几个、几十个属性，这种写法会非常重复。

ObservableProperty 的意义，就是把这部分重复劳动交给源生成器自动完成。

## 二、基础写法

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string userName = string.Empty;
}
```

上面只写了一个私有字段，但编译后会自动生成一个公开属性，效果可以理解为：

```csharp
public string UserName
{
 get => userName;
 set => SetProperty(ref userName, value);
}
```

也就是说：

* 你声明的是字段；
* 编译器最终帮你生成的是可绑定属性；
* WPF 绑定时使用的是生成后的 UserName，而不是原始字段。

XAML 中正常绑定即可：

```xml
<TextBox Text="{Binding UserName, UpdateSourceTrigger=PropertyChanged}"/>
<TextBlock Text="{Binding UserName}"/>
```

## 三、它依赖什么

ObservableProperty 不是单独生效的，它通常依赖下面两个前提：

1. 当前类要继承 ObservableObject，或者具备等价通知能力；
2. 当前类一般要声明为 partial，因为源生成器会为它补充代码。

典型写法：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private int count;
}
```

如果你忘了写 partial，源生成器生成代码时就没法和当前类拼接，这通常会直接报错。

## 四、字段名如何生成属性名

ObservableProperty 会根据字段名自动推断属性名。

例如：

```csharp
[ObservableProperty]
private string userName = string.Empty;

[ObservableProperty]
private bool isBusy;

[ObservableProperty]
private int _age;

[ObservableProperty]
private string? m_title;
```

一般会生成：

* UserName
* IsBusy
* Age
* Title

所以实际项目里，最常见的字段风格通常是：

* userName
* isBusy
* _age
* _statusMessage

建议统一团队风格，不要混着写，否则后面看代码会比较乱。

## 五、ObservableProperty 最直接的价值

它最直接的价值不是“少写几行代码”这么简单，而是：

* 减少样板代码；
* 降低手写属性时漏掉 PropertyChanged 的概率；
* 让 ViewModel 更聚焦于业务含义，而不是属性模板；
* 和其他特性联动时非常自然。

在 CommunityToolkit.Mvvm 里，很多能力都是围绕它展开的。

## 六、属性变化时执行附加逻辑

很多时候，属性变化后不仅要通知 UI，还要顺便做一些额外处理。

ObservableProperty 支持通过局部方法扩展生成逻辑。

### 1. OnXxxChanging 和 OnXxxChanged

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string keyword = string.Empty;

 partial void OnKeywordChanging(string value)
 {
 }

 partial void OnKeywordChanged(string value)
 {
 }
}
```

你可以把它理解成：

* OnKeywordChanging：设置新值之前触发；
* OnKeywordChanged：设置新值之后触发。

更实用的示例：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using System.Collections.ObjectModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 public ObservableCollection<string> Results { get; } = new();

 [ObservableProperty]
 private string keyword = string.Empty;

 partial void OnKeywordChanged(string value)
 {
  Results.Clear();

  if (!string.IsNullOrWhiteSpace(value))
  {
   Results.Add($"你输入了：{value}");
  }
 }
}
```

这种写法很适合：

* 输入框联动搜索；
* 属性变化后刷新本地状态；
* 某个字段变化后更新提示文本。

### 2. 带 oldValue 和 newValue 的重载

对于某些场景，你可能既想拿到旧值，也想拿到新值。

可以使用对应重载：

```csharp
partial void OnSelectedUserChanging(User? oldValue, User? newValue)
{
}

partial void OnSelectedUserChanged(User? oldValue, User? newValue)
{
}
```

这类写法适合：

* 切换当前选中项时释放旧资源；
* 从旧对象取消订阅事件，再给新对象订阅；
* 做更精确的状态切换。

## 七、通知其他属性一起刷新

实际项目中，一个属性变化后，往往不只是自己需要通知。

比如：

* FirstName 和 LastName 变化时，FullName 也应该刷新；
* Count 变化时，SummaryText 也应该刷新；
* SelectedItem 变化时，HasSelection 也应该刷新。

这时可以配合 NotifyPropertyChangedFor：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 [NotifyPropertyChangedFor(nameof(FullName))]
 private string firstName = string.Empty;

 [ObservableProperty]
 [NotifyPropertyChangedFor(nameof(FullName))]
 private string lastName = string.Empty;

 public string FullName => $"{FirstName} {LastName}";
}
```

这样当 FirstName 或 LastName 变化时，FullName 绑定也会自动刷新。

这在 WPF 里非常常见，因为很多属性本来就是“计算属性”。

## 八、属性变化后通知命令状态更新

ObservableProperty 和命令特性经常一起用。

例如：

* 用户名为空时，登录按钮禁用；
* 是否选中项变化时，删除按钮禁用；
* 输入条件变化时，查询按钮可执行状态更新。

这时可以配合 NotifyCanExecuteChangedFor：

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

 public IRelayCommand LoginCommand { get; }

 public MainViewModel()
 {
  LoginCommand = new RelayCommand(Login, CanLogin);
 }

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

这样字段变化后，命令的 CanExecuteChanged 会自动触发，不需要你手动反复写 NotifyCanExecuteChanged。

## 九、一个更完整的实际示例

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class UserEditorViewModel : ObservableObject
{
 [ObservableProperty]
 [NotifyPropertyChangedFor(nameof(FullName))]
 [NotifyCanExecuteChangedFor(nameof(SaveCommand))]
 private string firstName = string.Empty;

 [ObservableProperty]
 [NotifyPropertyChangedFor(nameof(FullName))]
 [NotifyCanExecuteChangedFor(nameof(SaveCommand))]
 private string lastName = string.Empty;

 [ObservableProperty]
 private string statusMessage = "未保存";

 public string FullName => $"{FirstName} {LastName}";

 public IRelayCommand SaveCommand { get; }

 public UserEditorViewModel()
 {
  SaveCommand = new RelayCommand(Save, CanSave);
 }

 private void Save()
 {
  StatusMessage = $"已保存：{FullName}";
 }

 private bool CanSave()
 {
  return !string.IsNullOrWhiteSpace(FirstName)
   && !string.IsNullOrWhiteSpace(LastName);
 }
}
```

对应 XAML：

```xml
<StackPanel Margin="20">
 <TextBox Margin="0,0,0,10"
   Text="{Binding FirstName, UpdateSourceTrigger=PropertyChanged}"/>

 <TextBox Margin="0,0,0,10"
   Text="{Binding LastName, UpdateSourceTrigger=PropertyChanged}"/>

 <TextBlock Text="{Binding FullName}" Margin="0,0,0,10"/>

 <Button Content="保存"
   Width="100"
   Command="{Binding SaveCommand}"/>

 <TextBlock Text="{Binding StatusMessage}" Margin="0,10,0,0"/>
</StackPanel>
```

这个例子里能看到 ObservableProperty 的几个典型作用：

1. 自动生成 FirstName、LastName、StatusMessage；
2. 自动驱动 FullName 刷新；
3. 自动驱动 SaveCommand 的可执行状态刷新；
4. 让 ViewModel 保持简洁。

## 十、和手写属性相比的区别

ObservableProperty 不是能力变强了，而是把“手写通知属性”的固定模板抽掉了。

也就是说：

* 它不是替代属性通知机制；
* 它是帮你更方便地使用属性通知机制；
* 底层本质仍然是属性变化通知。

所以如果你本来就理解 INotifyPropertyChanged，那么理解 ObservableProperty 通常不会有障碍。

## 十一、常见注意点

### 1. 字段要写在类里，不是直接写属性上

ObservableProperty 标注的是字段，不是普通属性。

正确写法：

```csharp
[ObservableProperty]
private string title = string.Empty;
```

不要写成：

```csharp
[ObservableProperty]
public string Title { get; set; }
```

### 2. 类通常要声明为 partial

因为源生成器会把生成代码并到同一个类里，所以通常必须是 partial class。

### 3. 不要一边手写同名属性，一边又让它生成

例如你已经有：

```csharp
public string UserName { get; set; }
```

这时又写：

```csharp
[ObservableProperty]
private string userName = string.Empty;
```

就会冲突，因为最终会生成同名的 UserName。

### 4. 复杂逻辑不要全塞进 OnXxxChanged

OnXxxChanged 很方便，但如果你把数据库访问、网络请求、复杂业务流程都放进去，后面会很难追踪。

比较合理的做法是：

* 简单联动放 OnXxxChanged；
* 复杂逻辑交给命令或服务层；
* 不要把属性 setter 变成“隐式业务入口”。

### 5. 命名最好统一

虽然源生成器能从多种字段风格里推断属性名，但团队里最好统一：

* 要么统一用 userName；
* 要么统一用 _userName；
* 不要一个类里混着 userName、_age、m_title。

## 十二、什么时候特别适合用 ObservableProperty

下面这些场景尤其适合：

* 表单输入；
* 查询条件；
* 当前选中项；
* 页面状态文本；
* 是否忙碌、是否可编辑、是否选中这类布尔状态；
* 需要和命令、计算属性联动的 ViewModel 字段。

如果一个属性就是给 UI 绑定用，并且需要通知更新，那么它往往都适合考虑 ObservableProperty。

## 总结

1. ObservableProperty 的核心作用是把字段自动生成成可通知的属性；
2. 它能显著减少 ViewModel 中重复的属性模板代码；
3. 它可以和 NotifyPropertyChangedFor、NotifyCanExecuteChangedFor 等特性配合使用；
4. OnXxxChanging 和 OnXxxChanged 适合处理轻量的联动逻辑；
5. 在 CommunityToolkit.Mvvm 中，它通常是最值得优先掌握的基础特性之一。
