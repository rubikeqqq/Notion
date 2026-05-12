# CommunityToolkit.Mvvm 常用类型总览

这一组笔记主要围绕 WPF 中最常用的 CommunityToolkit.Mvvm 能力展开。

如果只看学习顺序，可以先这样理解：

1. 02-CommunityToolkit与传统手写MVVM对比：先建立整体认知；
2. 03-ObservableObject：属性通知基础；
3. 04-ObservableProperty：自动生成可通知属性；
4. 05-属性联动通知：属性刷新和命令状态刷新的专项用法；
5. 06-RelayCommand特性：自动生成命令；
6. 07-AsyncRelayCommand：同步命令、异步命令的具体使用；
7. 08-WeakReferenceMessenger：跨模块消息通信；
8. 09-ObservableRecipient：带消息能力的 ViewModel 基类；
9. 10-DataContext和ViewModelLocator配合：解决 View 和 ViewModel 怎么接起来；
10. 11-登录模块完整案例：把属性、命令、消息机制串成一套；
11. 12-Messenger多窗口多页面通信案例：消息机制在桌面程序中的落地方式；
12. 13-Frame导航和Page配合：解决多页面导航场景下的组织问题；
13. 14-依赖注入项目骨架：把 Toolkit 和 DI 组合成更像正式项目的结构；
14. 15-常见坑和排错清单：最后用于快速定位“为什么不工作”。

## 一、各篇笔记分别讲什么

### 1. ObservableObject

重点是：

* 什么是属性通知基类；
* SetProperty 的作用；
* 手写属性时如何触发 UI 更新；
* 什么时候只用 ObservableObject 就够了。

适合在这里开始读：

[03-ObservableObject](03-ObservableObject.md)

### 2. ObservableProperty

重点是：

* 如何把字段自动生成成可绑定属性；
* OnXxxChanging / OnXxxChanged；
* NotifyPropertyChangedFor；
* NotifyCanExecuteChangedFor。

适合在这里继续读：

[04-ObservableProperty](04-ObservableProperty.md)

### 3. RelayCommand 特性

重点是：

* [RelayCommand] 如何自动生成命令；
* 同步方法和异步方法分别生成什么；
* 参数命令和 CanExecute；
* 和 ObservableProperty 如何联动。

[06-RelayCommand特性](06-RelayCommand特性.md)

### 4. AsyncRelayCommand 和 RelayCommand

重点是：

* 同步命令和异步命令如何选择；
* IsRunning、取消、异常处理；
* 实际项目中的登录和刷新列表示例；
* PasswordBox 和异步命令如何配合。

[07-AsyncRelayCommand](07-AsyncRelayCommand.md)

### 5. WeakReferenceMessenger

重点是：

* 什么场景适合消息通信；
* 如何定义消息、发送消息、接收消息；
* 登录成功、主题切换这类广播场景怎么做；
* 消息机制和直接依赖的边界怎么划分。

[08-WeakReferenceMessenger](08-WeakReferenceMessenger.md)

### 6. ObservableRecipient

重点是：

* 它和 ObservableObject 的区别；
* IsActive、OnActivated、OnDeactivated；
* 如何把消息机制和 ViewModel 生命周期结合起来。

[09-ObservableRecipient](09-ObservableRecipient.md)

### 7. 属性联动通知

重点是：

* NotifyPropertyChangedFor 的使用场景；
* NotifyCanExecuteChangedFor 的使用场景；
* 属性联动和命令联动怎么写得更清晰；
* 什么时候该用特性，什么时候该手动通知。

[05-属性联动通知](05-属性联动通知.md)

### 8. 登录模块完整案例

重点是：

* 如何把 ObservableProperty、AsyncRelayCommand、Messenger 串起来；
* 登录按钮启用条件怎么处理；
* 登录成功后如何广播并更新主界面；
* 一个实际 ViewModel 应该怎么拆职责。

[11-登录模块完整案例](11-登录模块完整案例.md)

### 9. Messenger 多窗口、多页面通信案例

重点是：

* 多窗口和多页面同时存在时，消息机制怎么组织；
* 登录窗口、设置页、首页、状态栏如何通过消息解耦；
* 多个接收方如何同时响应同一条消息；
* 生命周期应该怎么配合消息注册。

[12-Messenger多窗口多页面通信案例](12-Messenger多窗口多页面通信案例.md)

### 10. CommunityToolkit 和传统手写 MVVM 对比

重点是：

* Toolkit 和手写 INotifyPropertyChanged、ICommand 的差别；
* 哪些地方省了样板代码；
* 手写模式还有哪些优势；
* 什么时候适合迁移，什么时候不一定要迁移。

[02-CommunityToolkit与传统手写MVVM对比](02-CommunityToolkit与传统手写MVVM对比.md)

### 11. DataContext、ViewModelLocator 和 Toolkit 配合

重点是：

* DataContext 到底在绑定里起什么作用；
* 直接设置 DataContext、用 Locator、用 DI 有什么区别；
* UserControl 最常见的上下文覆盖问题；
* ViewModel 如何真正交给 View。

[10-DataContext和ViewModelLocator配合](10-DataContext和ViewModelLocator配合.md)

### 12. 常见坑和排错清单

重点是：

* 绑定不生效时先查什么；
* 命令点不动通常是哪些原因；
* Messenger 为什么收不到消息；
* 源生成器、命名、DataContext 覆盖等高频坑怎么排。

[15-常见坑和排错清单](15-常见坑和排错清单.md)

### 13. Frame 导航、Page 和 Toolkit 配合

重点是：

* Frame 和 Page 在 WPF 中怎么组织；
* Page 的 ViewModel 怎么绑定；
* 导航服务应该放在哪层；
* 导航页使用 Messenger 时要注意什么。

[13-Frame导航和Page配合](13-Frame导航和Page配合.md)

### 14. 依赖注入项目骨架

重点是：

* App.xaml.cs 如何做容器装配；
* View、ViewModel、Service 怎么分层；
* 生命周期怎么大致选；
* Toolkit 和 DI 为什么适合一起用。

[14-依赖注入项目骨架](14-依赖注入项目骨架.md)

## 二、如果按项目实践理解，可以这样分层

### 第一层：属性通知

* ObservableObject
* ObservableProperty

这一层解决的是：

“数据变了，UI 怎么自动更新。”

### 第二层：界面动作

* RelayCommand 特性
* RelayCommand
* AsyncRelayCommand

这一层解决的是：

“按钮点击、刷新、保存、查询这些动作怎么放进 ViewModel。”

### 第三层：跨模块通信

* WeakReferenceMessenger
* ObservableRecipient

这一层解决的是：

“不同页面、不同 ViewModel 之间，怎么低耦合地互相通知。”

## 三、一个最实用的学习顺序

如果你是第一次系统接触 CommunityToolkit.Mvvm，建议按下面顺序看：

1. [02-CommunityToolkit与传统手写MVVM对比](02-CommunityToolkit与传统手写MVVM对比.md)
2. [03-ObservableObject](03-ObservableObject.md)
3. [04-ObservableProperty](04-ObservableProperty.md)
4. [05-属性联动通知](05-属性联动通知.md)
5. [06-RelayCommand特性](06-RelayCommand特性.md)
6. [07-AsyncRelayCommand](07-AsyncRelayCommand.md)
7. [08-WeakReferenceMessenger](08-WeakReferenceMessenger.md)
8. [09-ObservableRecipient](09-ObservableRecipient.md)
9. [10-DataContext和ViewModelLocator配合](10-DataContext和ViewModelLocator配合.md)
10. [11-登录模块完整案例](11-登录模块完整案例.md)
11. [12-Messenger多窗口多页面通信案例](12-Messenger多窗口多页面通信案例.md)
12. [13-Frame导航和Page配合](13-Frame导航和Page配合.md)
13. [14-依赖注入项目骨架](14-依赖注入项目骨架.md)
14. [15-常见坑和排错清单](15-常见坑和排错清单.md)

这样顺序更自然，因为它基本符合从简单到复杂、从单页到跨模块的学习路径。

## 四、常见组合方式

在实际项目里，最常见的组合通常是：

### 1. 普通表单页

* ObservableObject
* ObservableProperty
* [RelayCommand]

例如：查询页、编辑页、设置页。

### 2. 带异步加载的页面

* ObservableObject
* ObservableProperty
* AsyncRelayCommand 或 [RelayCommand] + async Task

例如：登录页、列表加载页、统计页。

### 3. 需要跨模块联动的页面

* ObservableRecipient
* WeakReferenceMessenger

例如：登录成功后更新主界面、主题切换广播、当前项目切换通知。

### 4. 需要完整串联示例时

* 属性联动通知
* AsyncRelayCommand
* WeakReferenceMessenger
* ObservableRecipient

例如：登录页、启动页、主界面联动这类完整流程。

### 5. 需要多窗口和全局广播时

* WeakReferenceMessenger
* ObservableRecipient
* Messenger 多窗口、多页面通信案例

例如：登录弹窗通知主窗口、设置页广播主题切换、多个页面同步状态。

### 6. 需要把 View 和 ViewModel 真正接起来时

* DataContext、ViewModelLocator 和 Toolkit 配合
* 常见坑和排错清单

例如：页面不显示数据、命令不执行、UserControl 绑定错位、Locator 配置失效。

### 7. 需要做正式项目结构时

* Frame 导航、Page 和 Toolkit 配合
* 依赖注入项目骨架

例如：主窗口 + 多 Page 导航、服务注入、导航服务、项目启动装配。

## 总结

1. CommunityToolkit.Mvvm 的常用能力可以分成三层：属性通知、界面动作、跨模块通信；
2. 先理解 ObservableObject 和 ObservableProperty，再学命令，会更顺畅；
3. 消息机制通常放在后面学，因为它处理的是更复杂的跨模块协作；
4. 加上导航和依赖注入骨架后，这一组内容已经覆盖了 WPF 中最常见的一套 CommunityToolkit.Mvvm 基础、落地、接线、导航和排错理解。
