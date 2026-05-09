# ObservableRecipient

在 CommunityToolkit.Mvvm 中，ObservableRecipient 可以看成是 ObservableObject 的增强版。

如果只说核心区别，可以直接理解为：

* ObservableObject：负责属性通知；
* ObservableRecipient：在属性通知基础上，进一步支持消息通信。

所以它适合的场景通常不是“普通属性绑定”本身，而是：

1. ViewModel 之间需要解耦通信；
2. 某个页面需要接收全局消息；
3. 登录状态、主题切换、选中项变化这类信息要跨模块广播；
4. 不想让不同 ViewModel 直接互相引用。

## 一、为什么会需要 ObservableRecipient

在 WPF 项目里，如果两个 ViewModel 之间要共享状态，很多人一开始会直接这样写：

```csharp
public class AViewModel
{
 private readonly BViewModel _bViewModel;
}
```

这种写法的问题是：

* ViewModel 之间耦合变高；
* 后续模块增多后，依赖关系会越来越乱；
* 不利于测试和替换实现。

更常见、更松耦合的方式是：

* 发送一个消息；
* 让需要接收它的 ViewModel 自己订阅；
* 彼此不直接引用。

ObservableRecipient 就是围绕这个思路设计的。

## 二、它和 ObservableObject 的关系

ObservableRecipient 继承自 ObservableObject，所以 ObservableObject 的能力它本来就都有。

也就是说，它同样支持：

* 属性变更通知；
* SetProperty；
* OnPropertyChanged；
* 和 ObservableProperty、RelayCommand 等特性配合使用。

只是在此基础上，它还增加了和 Messenger 协作的能力。

## 三、基础写法

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableRecipient
{
 [ObservableProperty]
 private string title = "首页";
}
```

如果只是单看属性绑定，这和 ObservableObject 的写法几乎没区别。

真正的差异点在于：

* 它内部有 Messenger 相关能力；
* 它支持激活和取消激活时的消息注册；
* 它更适合做“可接收消息的 ViewModel”。

## 四、什么是 IsActive

ObservableRecipient 中一个很关键的概念是 IsActive。

它可以理解为：

当前对象是否处于“激活并允许接收消息”的状态。

很多时候，消息接收不是永久开启的，而是跟页面生命周期有关。

例如：

* 页面显示时开始接收消息；
* 页面关闭或切换后停止接收消息；
* 避免已经不在使用的 ViewModel 继续处理广播。

这就是 IsActive 存在的意义。

## 五、OnActivated 和 OnDeactivated

ObservableRecipient 通常配合 OnActivated 和 OnDeactivated 使用。

### 1. 基础结构

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableRecipient
{
 protected override void OnActivated()
 {
  base.OnActivated();
 }

 protected override void OnDeactivated()
 {
  base.OnDeactivated();
 }
}
```

你可以把它理解成：

* OnActivated：对象激活时执行；
* OnDeactivated：对象取消激活时执行。

在实际项目里，最常见的用途就是：

* OnActivated 中注册消息；
* OnDeactivated 中取消注册或释放资源。

## 六、一个最常见的接收消息示例

先定义一个消息类型：

```csharp
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace Demo.Messages;

public sealed class UserLoggedInMessage : ValueChangedMessage<string>
{
 public UserLoggedInMessage(string value) : base(value)
 {
 }
}
```

然后在 ViewModel 中接收它：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class HeaderViewModel : ObservableRecipient
{
 [ObservableProperty]
 private string welcomeText = "未登录";

 protected override void OnActivated()
 {
  Messenger.Register<HeaderViewModel, UserLoggedInMessage>(this, static (recipient, message) =>
  {
   recipient.WelcomeText = $"欢迎你，{message.Value}";
  });

  base.OnActivated();
 }

 protected override void OnDeactivated()
 {
  Messenger.UnregisterAll(this);
  base.OnDeactivated();
 }
}
```

这里的关键点是：

1. ViewModel 继承 ObservableRecipient；
2. 在 OnActivated 中注册消息；
3. 在 OnDeactivated 中取消注册；
4. 收到消息后更新绑定属性。

## 七、怎么触发激活

如果你已经写了 OnActivated，但始终没有收到消息，首先要检查的就是：

你有没有把 IsActive 设为 true。

例如：

```csharp
public HeaderViewModel()
{
 IsActive = true;
}
```

或者在页面初始化时设置：

```csharp
viewModel.IsActive = true;
```

如果没有激活，OnActivated 通常不会执行，消息也就不会注册成功。

## 八、结合登录场景的完整示例

下面给一个更贴近项目的示例：

登录页负责发送登录成功消息，主页头部负责接收并更新显示。

### 1. 消息类型

```csharp
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace Demo.Messages;

public sealed class LoginSuccessMessage : ValueChangedMessage<string>
{
 public LoginSuccessMessage(string userName) : base(userName)
 {
 }
}
```

### 2. 发送方 ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class LoginViewModel : ObservableRecipient
{
 [ObservableProperty]
 private string userName = string.Empty;

 [RelayCommand]
 private async Task LoginAsync()
 {
  await Task.Delay(1000);
  Messenger.Send(new LoginSuccessMessage(UserName));
 }
}
```

### 3. 接收方 ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class ShellViewModel : ObservableRecipient
{
 [ObservableProperty]
 private string currentUserText = "当前未登录";

 public ShellViewModel()
 {
  IsActive = true;
 }

 protected override void OnActivated()
 {
  Messenger.Register<ShellViewModel, LoginSuccessMessage>(this, static (recipient, message) =>
  {
   recipient.CurrentUserText = $"当前用户：{message.Value}";
  });

  base.OnActivated();
 }
}
```

这个例子说明了 ObservableRecipient 的主要价值：

* 登录页不需要知道 ShellViewModel 的存在；
* ShellViewModel 只关心自己要响应的消息；
* 模块之间通过消息解耦。

## 九、什么时候用 ObservableRecipient，什么时候用 ObservableObject

可以直接按下面的思路判断：

### 用 ObservableObject 的场景

* 只是普通的属性通知；
* ViewModel 不需要接收消息；
* 页面比较简单，没有跨模块广播需求。

### 用 ObservableRecipient 的场景

* 需要接收或发送消息；
* 希望通过 Messenger 解耦模块；
* 希望利用 IsActive 管理消息注册生命周期。

一个很实用的经验是：

* 默认先用 ObservableObject；
* 只有明确需要消息机制时，再升级成 ObservableRecipient。

## 十、常见注意点

### 1. 不要忘记激活

如果 IsActive 没有设为 true，OnActivated 往往不会执行。

### 2. 不要让不再使用的对象继续收消息

如果某个 ViewModel 已经不该继续工作，就应该及时取消激活，或者在 OnDeactivated 中取消注册。

### 3. Messenger 适合解耦，不适合滥用

消息机制很方便，但如果所有状态传递都靠消息，会让调用关系变得隐蔽。

更合适的边界通常是：

* 明确的依赖，用构造注入；
* 跨模块广播，用 Messenger；
* 局部页面状态，不要硬上消息。

### 4. 消息类型要尽量语义化

不要只写一个模糊的字符串消息。

更好的方式是：

* LoginSuccessMessage
* ThemeChangedMessage
* CurrentProjectChangedMessage

这样后面维护时更清楚。

## 总结

1. ObservableRecipient 可以理解为带消息能力的 ObservableObject；
2. 它适合处理 ViewModel 之间的解耦通信；
3. IsActive、OnActivated、OnDeactivated 是它的关键使用点；
4. 默认场景先用 ObservableObject，明确需要消息机制时再考虑 ObservableRecipient；
5. 它和 Messenger 配合后，特别适合登录状态、主题切换、选中项变更这类跨模块通知。
