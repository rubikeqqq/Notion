# Frame 导航、Page 和 CommunityToolkit.Mvvm 配合

前面已经讲了 Window、DataContext、ViewModelLocator、消息通信这些内容。

但在 WPF 项目里，只要一进入多页面结构，通常就会碰到另一类问题：

* Page 的 ViewModel 什么时候创建；
* Frame 导航时页面状态要不要保留；
* 页面切换后消息接收要不要关掉；
* 导航参数怎么传给 ViewModel。

这篇就专门把 Frame、Page 和 CommunityToolkit.Mvvm 放在一起讲清楚。

## 一、先理解 WPF 里常见的导航结构

在很多 WPF 项目里，主界面不是不停弹新窗口，而是：

* 主窗口里放一个 Frame；
* Frame 负责显示不同 Page；
* 左侧菜单或顶部导航切换不同页面。

典型结构可以理解为：

1. ShellWindow 负责整体布局；
2. Frame 负责装载页面；
3. 每个 Page 再绑定自己的 ViewModel。

这和单纯 Window + 单个 ViewModel 的结构不太一样，因为“页面切换”和“页面生命周期”会变成重点。

## 二、最简单的导航示例

### 1. 主窗口 XAML

```xml
<DockPanel>
 <StackPanel DockPanel.Dock="Left" Width="140" Background="#F3F6F8">
  <Button Content="首页" Command="{Binding GoHomeCommand}" Margin="12"/>
  <Button Content="设置" Command="{Binding GoSettingsCommand}" Margin="12,0,12,12"/>
 </StackPanel>

 <Frame x:Name="MainFrame" NavigationUIVisibility="Hidden"/>
</DockPanel>
```

### 2. 主窗口后台代码

```csharp
using Demo.Views;

namespace Demo;

public partial class ShellWindow : Window
{
 public ShellWindow()
 {
  InitializeComponent();
  MainFrame.Navigate(new HomePage());
 }
}
```

这就是最基础的导航模型：

* 点击菜单；
* Frame 导航到不同 Page；
* 每个 Page 再处理自己的绑定。

## 三、Page 和普通 UserControl 的区别

很多人刚开始会混淆 Page 和 UserControl。

可以先简单理解为：

* UserControl 更像一个界面片段；
* Page 更像一个可导航的页面单元。

如果你是通过 Frame.Navigate 切换内容，那么更典型的对象通常是 Page。

## 四、Page 怎么绑定 ViewModel

Page 绑定 ViewModel 的方式，本质上和 Window 类似，仍然是围绕 DataContext。

### 1. 直接在 Page 后台代码设置

```csharp
using Demo.ViewModels;

namespace Demo.Views;

public partial class HomePage : Page
{
 public HomePage()
 {
  InitializeComponent();
  DataContext = new HomeViewModel();
 }
}
```

### 2. Page XAML

```xml
<Page x:Class="Demo.Views.HomePage"
      xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
      xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

 <StackPanel Margin="20">
  <TextBlock FontSize="20" Text="{Binding Title}" Margin="0,0,0,12"/>
  <Button Content="刷新" Command="{Binding RefreshCommand}" Width="100"/>
 </StackPanel>
</Page>
```

这和前面讲过的 Window.DataContext 逻辑是完全一致的。

## 五、配合 CommunityToolkit.Mvvm 的典型 Page ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class HomeViewModel : ObservableObject
{
 [ObservableProperty]
 private string title = "系统首页";

 [ObservableProperty]
 private string statusText = "未刷新";

 [RelayCommand]
 private async Task RefreshAsync()
 {
  StatusText = "刷新中...";
  await Task.Delay(1000);
  StatusText = "刷新完成";
 }
}
```

如果 HomePage 的 DataContext 正确设置为 HomeViewModel，那么：

* Title 会直接显示；
* RefreshCommand 可以直接绑定；
* AsyncRelayCommand 的状态也可以继续参与界面控制。

## 六、导航逻辑放在哪一层更合适

这是一个实际项目里非常关键的问题。

常见有三种放法：

### 1. 直接写在 Window 后台代码里

适合：

* 小项目；
* Demo；
* 导航逻辑很少。

### 2. 放在 ShellViewModel 里，通过服务驱动

适合：

* 中大型项目；
* 需要更清晰的 MVVM 分层；
* 想把按钮点击和导航动作也交给命令管理。

### 3. 单独抽一个导航服务

适合：

* 页面较多；
* 需要统一导航、传参、返回、缓存；
* 需要从多个 ViewModel 发起导航。

如果只是入门学习，不必一开始就把导航服务做得很复杂。

## 七、一个更贴近 MVVM 的导航写法

例如让 ShellViewModel 只发出“导航到哪个页面”的动作。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class ShellViewModel : ObservableObject
{
 public IRelayCommand GoHomeCommand { get; }
 public IRelayCommand GoSettingsCommand { get; }

 private readonly INavigationService _navigationService;

 public ShellViewModel(INavigationService navigationService)
 {
  _navigationService = navigationService;
  GoHomeCommand = new RelayCommand(() => _navigationService.NavigateTo("Home"));
  GoSettingsCommand = new RelayCommand(() => _navigationService.NavigateTo("Settings"));
 }
}
```

这时 ViewModel 不直接操作 Frame，而是通过导航服务表达意图。

## 八、一个简单导航服务可以长什么样

```csharp
namespace Demo.Services;

public interface INavigationService
{
 void NavigateTo(string pageKey);
}
```

```csharp
using Demo.Views;

namespace Demo.Services;

public class NavigationService : INavigationService
{
 private readonly Frame _frame;

 public NavigationService(Frame frame)
 {
  _frame = frame;
 }

 public void NavigateTo(string pageKey)
 {
  switch (pageKey)
  {
   case "Home":
    _frame.Navigate(new HomePage());
    break;
   case "Settings":
    _frame.Navigate(new SettingsPage());
    break;
  }
 }
}
```

这个写法虽然简单，但已经能体现一个关键思想：

* 页面切换的具体实现放到服务层；
* ViewModel 只描述导航意图。

## 九、Page 导航时状态要不要保留

这是使用 Frame 时非常常见的实际问题。

例如：

* 从首页切到设置页；
* 再切回首页；
* 首页输入内容还要不要保留。

常见策略通常有两种：

### 1. 每次都新建页面和 ViewModel

特点：

* 状态干净；
* 逻辑简单；
* 但页面返回时会丢失先前输入和数据。

### 2. 复用页面或复用 ViewModel

特点：

* 状态可保留；
* 用户体验可能更连续；
* 但生命周期管理会更复杂。

这没有绝对标准，取决于页面性质。

经验上：

* 查询页、编辑页通常更在意状态保留；
* 一次性流程页、启动页则更适合每次新建。

## 十、导航参数怎么传给 ViewModel

Page 导航里常见需求是：

* 打开详情页时传入 ID；
* 打开编辑页时传入当前对象；
* 打开日志页时传入过滤条件。

常见做法通常有三类：

### 1. 通过 Page 构造函数传

```csharp
public partial class DetailPage : Page
{
 public DetailPage(int userId)
 {
  InitializeComponent();
  DataContext = new DetailViewModel(userId);
 }
}
```

### 2. 通过导航服务传

```csharp
void NavigateTo(string pageKey, object? parameter);
```

### 3. 通过消息或共享状态对象传

适合跨模块场景，但不适合滥用。

如果只是简单的详情跳转，构造函数或导航服务参数通常更直接。

## 十一、ObservableRecipient 在导航页里的注意点

如果某个 Page 的 ViewModel 继承 ObservableRecipient，那么页面切换时要特别注意它的激活状态。

例如：

* 页面进入时 IsActive = true；
* 页面离开时取消激活；
* 避免已经不可见的页面继续接收全局消息。

否则就可能出现：

* 你已经切到别的页面；
* 旧页面的 ViewModel 还在处理消息；
* 状态莫名其妙被更新。

## 十二、Page + Messenger 的一个实际场景

例如设置页切换主题后，首页和顶部状态栏都要变。

这时完全可以这样做：

1. SettingsPage 对应的 ViewModel 发送 ThemeChangedMessage；
2. HomePage 对应的 ViewModel 接收并更新本页状态；
3. HeaderViewModel 也接收并更新顶部文本。

这个模型和前面多窗口案例是一样的，只不过这里的承载对象从 Window 变成了 Page。

## 十三、Frame 导航里最常见的坑

### 1. 以为 Page 会自动复用

实际上如果你每次都是 new 一个 Page，自然就是新状态。

### 2. 页面离开了，消息接收还开着

如果用了 ObservableRecipient，要注意页面生命周期和 IsActive 的关系。

### 3. DataContext 设置位置不统一

有的页面在 XAML 里设，有的页面在后台代码设，有的页面走 DI，后期会很乱。

最好统一团队约定。

### 4. 把导航逻辑全塞进后台代码

小项目问题不大，但页面一多以后会越来越难维护。

## 总结

1. Frame + Page 是 WPF 中很常见的多页面导航结构；
2. Page 和 Window 一样，核心仍然是把正确的 ViewModel 交给 DataContext；
3. CommunityToolkit.Mvvm 主要负责 ViewModel 逻辑，本身不限制你如何实现导航；
4. 页面一多时，导航服务通常比把所有 Navigate 都写在代码后置里更清晰；
5. 如果页面 ViewModel 会接收消息，切换页面时一定要关注激活和失活状态。
