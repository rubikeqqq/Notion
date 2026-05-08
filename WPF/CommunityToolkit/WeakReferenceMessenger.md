# WeakReferenceMessenger

在 CommunityToolkit.Mvvm 中，WeakReferenceMessenger 是默认最常见的消息总线实现。

如果只用一句话概括它的作用，就是：

让对象之间通过消息通信，而不是直接互相引用。

它最适合解决的问题通常是：

1. ViewModel 之间需要解耦通信；
2. 登录状态、主题变化、刷新通知需要跨模块广播；
3. 不想为了传一个状态，就让多个类直接互相持有引用。

## 一、为什么需要 Messenger

假设登录成功后，你希望下面几个地方都做出反应：

* 顶部用户栏显示当前用户；
* 首页重新加载数据；
* 日志页写入一条登录记录。

如果直接让登录页去调用这几个 ViewModel，会带来几个问题：

* 登录页必须知道它们的存在；
* 依赖关系会迅速膨胀；
* 后期改动一个模块，很容易牵连别的模块。

更自然的方式是：

* 登录页只发送“登录成功”这件事；
* 谁关心这个事件，谁自己订阅；
* 发送方和接收方互相不直接依赖。

这就是 Messenger 模式的价值。

## 二、为什么叫 WeakReferenceMessenger

它的名字里最关键的是 WeakReference，也就是弱引用。

这意味着它在管理接收者时，不会像强引用那样更容易把对象“挂住”不释放。

简单理解就是：

* 对象不用时，更不容易因为消息总线而造成额外的持有问题；
* 对于 WPF 这类长时间运行的桌面程序，这一点很实用。

这也是它相对更推荐的原因之一。

## 三、最基础的发送和接收

### 1. 定义消息类型

最常见的方式是继承 `ValueChangedMessage<T>`。

```csharp
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace Demo.Messages;

public sealed class ThemeChangedMessage : ValueChangedMessage<string>
{
 public ThemeChangedMessage(string themeName) : base(themeName)
 {
 }
}
```

这个消息表达的语义就是：

“主题发生变化，值是当前主题名。”

### 2. 发送消息

```csharp
using CommunityToolkit.Mvvm.Messaging;
using Demo.Messages;

WeakReferenceMessenger.Default.Send(new ThemeChangedMessage("Dark"));
```

### 3. 接收消息

```csharp
using CommunityToolkit.Mvvm.Messaging;
using Demo.Messages;

WeakReferenceMessenger.Default.Register<MainViewModel, ThemeChangedMessage>(this, static (recipient, message) =>
{
 recipient.CurrentTheme = message.Value;
});
```

这样一来，发送方和接收方就解耦了。

## 四、一个最常见的登录广播示例

### 1. 定义消息

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

### 2. 登录页发送

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CommunityToolkit.Mvvm.Messaging;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class LoginViewModel : ObservableObject
{
 [ObservableProperty]
 private string userName = string.Empty;

 [RelayCommand]
 private async Task LoginAsync()
 {
  await Task.Delay(1000);
  WeakReferenceMessenger.Default.Send(new LoginSuccessMessage(UserName));
 }
}
```

### 3. 顶部栏接收

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class HeaderViewModel : ObservableObject
{
 [ObservableProperty]
 private string currentUserText = "未登录";

 public HeaderViewModel()
 {
  WeakReferenceMessenger.Default.Register<HeaderViewModel, LoginSuccessMessage>(this, static (recipient, message) =>
  {
   recipient.CurrentUserText = $"当前用户：{message.Value}";
  });
 }
}
```

效果就是：

* 登录页只负责发送登录成功消息；
* HeaderViewModel 只负责接收并更新显示；
* 双方没有直接依赖。

## 五、消息类型怎么设计更合适

很多人刚开始会想直接传字符串，但实际项目里更推荐用明确的消息类型。

例如：

* LoginSuccessMessage
* ThemeChangedMessage
* CurrentProjectChangedMessage
* RefreshDashboardMessage

这样做的好处是：

* 语义更明确；
* 更容易搜索引用；
* 后面扩展字段时也更方便。

如果只是简单值变化，`ValueChangedMessage<T>` 往往就够用了。

## 六、发送方和接收方的典型边界

WeakReferenceMessenger 很适合做“广播式通知”，但不是所有调用都该走消息。

比较合理的边界通常是：

### 适合用消息的场景

* 登录成功后，多个模块要响应；
* 主题切换后，多个页面要刷新样式；
* 当前项目变化后，多个视图要联动；
* 某些全局状态变化需要广播。

### 不适合硬用消息的场景

* 一个页面内部两个对象的直接协作；
* 明确的一对一依赖；
* 普通服务调用；
* 本来就应该通过构造注入传入的对象关系。

经验上可以这样判断：

如果这是“谁关心谁就订阅”的广播型事件，消息很合适；如果这是明确调用关系，就不要绕成消息。

## 七、如何取消注册

虽然 WeakReferenceMessenger 使用弱引用更安全，但在一些明确的生命周期场景里，仍然建议主动管理注册关系。

例如：

```csharp
WeakReferenceMessenger.Default.UnregisterAll(this);
```

或者按消息类型取消：

```csharp
WeakReferenceMessenger.Default.Unregister<LoginSuccessMessage>(this);
```

这在下面这些场景里比较有价值：

* 页面关闭时；
* 临时 ViewModel 销毁时；
* 某个对象只想在一段时间内接收消息时。

## 八、和 ObservableRecipient 的关系

WeakReferenceMessenger 和 ObservableRecipient 经常一起出现。

可以这样理解：

* WeakReferenceMessenger：消息总线本身；
* ObservableRecipient：更方便地接入消息机制的 ViewModel 基类。

也就是说：

* 你可以直接使用 WeakReferenceMessenger；
* 也可以在 ObservableRecipient 中通过 Messenger 属性和激活机制来使用它。

如果你的 ViewModel 本来就要接收消息，而且还要结合生命周期管理，那么 ObservableRecipient 往往更自然。

## 九、一个更贴近项目的综合示例

假设项目里有三个模块：

* 登录模块；
* 顶部状态栏；
* 工作台首页。

登录成功后，顶部状态栏要显示用户名，工作台首页要刷新统计数据。

### 1. 统一消息类型

```csharp
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace Demo.Messages;

public sealed class UserChangedMessage : ValueChangedMessage<string>
{
 public UserChangedMessage(string userName) : base(userName)
 {
 }
}
```

### 2. 登录模块发送消息

```csharp
WeakReferenceMessenger.Default.Send(new UserChangedMessage(UserName));
```

### 3. 顶部栏接收消息

```csharp
WeakReferenceMessenger.Default.Register<HeaderViewModel, UserChangedMessage>(this, static (recipient, message) =>
{
 recipient.CurrentUserText = $"当前用户：{message.Value}";
});
```

### 4. 首页接收消息并刷新

```csharp
WeakReferenceMessenger.Default.Register<HomeViewModel, UserChangedMessage>(this, static async (recipient, message) =>
{
 recipient.CurrentUserName = message.Value;
 await recipient.ReloadDashboardAsync();
});
```

这个例子里，发送方完全不需要知道谁会处理消息，这就是弱耦合通信的价值。

## 十、常见注意点

### 1. 消息类型不要设计得太随意

消息名应该表达业务含义，而不是只传一个模糊字符串。

### 2. 不要把所有交互都改成消息

消息适合广播，不适合替代所有正常调用。

### 3. 接收逻辑不要写得过重

如果接收到消息后直接做大量复杂业务，后面会很难排查问题。

更稳妥的方式通常是：

* 接收消息；
* 更新状态；
* 调用明确的方法继续处理。

### 4. 生命周期明确时，仍然建议主动取消注册

尤其是临时对象、弹窗、一次性页面，最好不要完全依赖弱引用机制“自动处理一切”。

## 总结

1. WeakReferenceMessenger 是 CommunityToolkit.Mvvm 中最常见的消息总线实现；
2. 它让对象之间通过消息通信，而不是直接持有彼此引用；
3. 它特别适合登录状态、主题变化、当前项目变化这类跨模块广播；
4. 它经常和 ObservableRecipient 一起使用；
5. 消息机制适合解耦，但不适合替代所有正常调用关系。
