# Messenger 多窗口、多页面通信案例

前面的 WeakReferenceMessenger 和 ObservableRecipient 笔记，已经分别讲了消息总线和接收消息的 ViewModel 基类。

但在真实项目里，很多人真正遇到的不是“怎么发一条消息”，而是：

多窗口、多页面、多模块同时存在时，消息机制到底怎么落地。

这篇就用一个更贴近桌面程序的案例，把下面几个问题串起来：

1. 一个弹出的登录窗口如何通知主窗口；
2. 一个设置页如何让多个页面同步主题变化；
3. 多个接收方如何各自响应同一条消息；
4. 生命周期和消息注册应该怎么配合。

## 一、案例目标

假设项目里有下面几个部分：

* 主窗口 ShellWindow
* 登录窗口 LoginWindow
* 首页 HomePage
* 设置页 SettingsPage
* 顶部状态栏 HeaderViewModel

现在希望实现两个通信场景：

### 场景 1：登录成功后联动

登录窗口登录成功后：

* 主窗口顶部显示当前用户；
* 首页刷新欢迎信息；
* 日志区追加一条登录记录。

### 场景 2：主题切换后联动

设置页切换主题后：

* 主窗口背景样式更新；
* 首页提示文字更新；
* 状态栏显示当前主题。

这些模块都不希望直接互相引用，因此比较适合用 Messenger。

## 二、为什么这个场景特别适合消息机制

如果不用消息机制，很多人一开始会写成这样：

* 登录窗口拿到主窗口引用；
* 设置页拿到首页和状态栏引用；
* 每个模块互相调用别人的方法。

问题是：

1. 窗口之间耦合越来越高；
2. 页面一多以后依赖关系很乱；
3. 后面换页面结构时，修改范围会很大。

而消息机制的思路是：

* 发送方只负责发送“发生了什么”；
* 接收方自己决定要不要响应；
* 多个接收方可以并行处理同一条消息。

## 三、定义消息类型

先把语义明确的消息类型定义好。

### 1. 登录成功消息

```csharp
using CommunityToolkit.Mvvm.Messaging.Messages;

namespace Demo.Messages;

public sealed class UserLoggedInMessage : ValueChangedMessage<string>
{
 public UserLoggedInMessage(string userName) : base(userName)
 {
 }
}
```

### 2. 主题切换消息

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

这样后面搜索引用、扩展字段、定位业务都更清楚。

## 四、登录窗口发送消息

登录窗口的 ViewModel 负责登录成功后广播当前用户。

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

 [ObservableProperty]
 private string password = string.Empty;

 [ObservableProperty]
 private string statusMessage = "请输入用户名和密码";

 [RelayCommand]
 private async Task LoginAsync()
 {
  StatusMessage = "登录中...";
  await Task.Delay(1200);

  StatusMessage = "登录成功";
  WeakReferenceMessenger.Default.Send(new UserLoggedInMessage(UserName));
 }
}
```

关键点是：

* 登录窗口不需要知道主窗口是谁；
* 它只广播“用户已登录”；
* 谁关心，谁自己去接收。

## 五、主窗口顶部接收登录消息

顶部状态栏一般非常适合用 ObservableRecipient。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class HeaderViewModel : ObservableRecipient
{
 [ObservableProperty]
 private string currentUserText = "当前未登录";

 [ObservableProperty]
 private string currentThemeText = "当前主题：Light";

 public HeaderViewModel()
 {
  IsActive = true;
 }

 protected override void OnActivated()
 {
  Messenger.Register<HeaderViewModel, UserLoggedInMessage>(this, static (recipient, message) =>
  {
   recipient.CurrentUserText = $"当前用户：{message.Value}";
  });

  Messenger.Register<HeaderViewModel, ThemeChangedMessage>(this, static (recipient, message) =>
  {
   recipient.CurrentThemeText = $"当前主题：{message.Value}";
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

这里可以看到，一个 ViewModel 完全可以同时接收多种消息。

## 六、首页接收同一条登录消息

首页也可以接收同样的 UserLoggedInMessage，但做不一样的事。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class HomeViewModel : ObservableRecipient
{
 [ObservableProperty]
 private string welcomeMessage = "欢迎进入系统";

 [ObservableProperty]
 private string pageThemeName = "Light";

 public HomeViewModel()
 {
  IsActive = true;
 }

 protected override void OnActivated()
 {
  Messenger.Register<HomeViewModel, UserLoggedInMessage>(this, static (recipient, message) =>
  {
   recipient.WelcomeMessage = $"欢迎回来，{message.Value}";
  });

  Messenger.Register<HomeViewModel, ThemeChangedMessage>(this, static (recipient, message) =>
  {
   recipient.PageThemeName = message.Value;
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

这说明了消息机制的一个核心优势：

同一条消息，可以被多个接收方独立响应。

## 七、日志面板如何接收并记录

日志区域通常只是把消息转成一条文本记录。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using System.Collections.ObjectModel;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class LogViewModel : ObservableRecipient
{
 public ObservableCollection<string> Logs { get; } = new();

 public LogViewModel()
 {
  IsActive = true;
 }

 protected override void OnActivated()
 {
  Messenger.Register<LogViewModel, UserLoggedInMessage>(this, static (recipient, message) =>
  {
   recipient.Logs.Add($"{DateTime.Now:HH:mm:ss} 用户登录：{message.Value}");
  });

  Messenger.Register<LogViewModel, ThemeChangedMessage>(this, static (recipient, message) =>
  {
   recipient.Logs.Add($"{DateTime.Now:HH:mm:ss} 切换主题：{message.Value}");
  });

  base.OnActivated();
 }
}
```

这样登录页和设置页都不需要知道日志模块存在，但日志仍然能自动记录。

## 八、设置页发送主题消息

设置页只负责发出主题变化消息。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CommunityToolkit.Mvvm.Messaging;
using Demo.Messages;

namespace Demo.ViewModels;

public partial class SettingsViewModel : ObservableObject
{
 [ObservableProperty]
 private string selectedTheme = "Light";

 [RelayCommand]
 private void ApplyTheme()
 {
  WeakReferenceMessenger.Default.Send(new ThemeChangedMessage(SelectedTheme));
 }
}
```

它不直接去改 HeaderViewModel，也不直接通知 HomeViewModel，而是只广播主题变化。

## 九、多窗口场景里怎么理解生命周期

桌面项目和单页页面不一样，多窗口经常会出现：

* 窗口打开又关闭；
* 页面被切换掉；
* 临时弹窗只活几秒。

所以要特别注意消息接收生命周期。

更稳妥的做法通常是：

1. 需要长期接收消息的对象，在构造后激活；
2. 页面销毁或失活时，取消注册；
3. 临时窗口不要长期保持激活状态；
4. 尽量把接收逻辑放在 ObservableRecipient 的激活周期里。

## 十、这个案例里谁该发消息，谁该直接调用

这点很关键。

### 适合发消息的地方

* 登录成功；
* 退出登录；
* 主题切换；
* 当前项目切换；
* 全局刷新通知。

### 更适合直接调用的地方

* 登录按钮内部调用认证服务；
* 设置页保存配置到本地文件；
* 页面内部两个方法之间的正常协作；
* 单一模块内部明确的一对一调用。

不要把所有调用都改成 Messenger，否则后面很难追踪程序流向。

## 十一、一个完整的联动链路怎么读

以“登录成功”这件事为例，完整链路其实是：

1. LoginViewModel 执行 LoginAsync；
2. 登录成功后发送 UserLoggedInMessage；
3. HeaderViewModel 收到消息后更新当前用户文本；
4. HomeViewModel 收到消息后更新欢迎语；
5. LogViewModel 收到消息后追加日志。

整个过程中：

* 发送方不知道有几个接收方；
* 接收方也不需要彼此认识；
* 模块扩展时，只要新增一个接收者即可。

## 十二、常见注意点

### 1. 消息名要表达业务含义

尽量不要只发一个裸字符串，而是定义语义清晰的消息类型。

### 2. 页面关闭时要考虑取消接收

尤其是临时窗口、弹窗、可切换页面，最好把接收逻辑放在激活和取消激活流程里。

### 3. 不要在接收回调里堆太多业务逻辑

接收消息后更适合：

* 更新界面状态；
* 调用一个明确的方法；
* 触发一次刷新。

如果回调里塞了大量业务判断，后面会很难维护。

### 4. Messenger 是解耦手段，不是万能总线

适合广播型事件，不适合替代所有依赖关系。

## 总结

1. 多窗口、多页面场景是 Messenger 最能体现价值的地方；
2. 发送方只负责广播发生了什么，接收方各自决定如何响应；
3. 同一条消息可以被多个模块同时接收；
4. ObservableRecipient 很适合管理接收生命周期；
5. 登录成功、主题切换、全局刷新这类场景尤其适合用消息机制。
