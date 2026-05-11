# 观察者模式 Observer

观察者模式解决的核心问题是：当一个对象的状态发生变化时，自动通知所有依赖它的对象。

说人话就是：事件发布-订阅机制。某个事件发生了（比如检测完成、NG 报警、设备断连），所有关心这个事件的地方都能收到通知。

## 一、观察者模式在 C# 中的天然实现

C# 对观察者模式有原生的语言支持——event 和 delegate。

### 最基础的形式

```csharp
public class VisionSystem
{
    // 声明一个事件
    public event EventHandler<InspectionEventArgs> InspectionCompleted;
    
    public void RunInspection()
    {
        var result = new InspectionResult();
        // ... 检测逻辑 ...
        
        // 触发事件
        InspectionCompleted?.Invoke(this, new InspectionEventArgs(result));
    }
}

public class InspectionEventArgs : EventArgs
{
    public InspectionResult Result { get; }
    public InspectionEventArgs(InspectionResult result) => Result = result;
}
```

订阅事件：

```csharp
var system = new VisionSystem();
system.InspectionCompleted += (sender, e) =>
{
    Console.WriteLine($"检测完成：{e.Result.IsPassed}");
};
```

## 二、观察者模式解决什么问题

想象一个检测系统在 NG 时需要做很多事情：

- 在界面上显示 NG 图标；
- 保存 NG 图片；
- 发送报警信号到 PLC；
- 记录日志；
- 停线或排除。

如果不使用观察者模式，检测代码会变成：

```csharp
public void RunInspection()
{
    var result = DoInspection();
    
    // 检测逻辑和后续处理混在一起
    _ui.ShowResult(result);
    _logger.Save(result);
    _plc.SendAlarm(result);
    if (!result.IsPassed) _alarm.StopLine();
}
```

每加一个处理动作都要修改 RunInspection 方法。观察者模式可以把这些处理动作解耦出去。

## 三、观察者模式的自定义实现

如果不想使用 C# 原生的 event（比如需要跨进程、跨线程），也可以手动实现：

```csharp
public interface IInspectionObserver
{
    void OnInspectionCompleted(InspectionResult result);
}

public class UIObserver : IInspectionObserver
{
    public void OnInspectionCompleted(InspectionResult result)
    {
        // 更新 UI 显示
    }
}

public class LoggerObserver : IInspectionObserver
{
    public void OnInspectionCompleted(InspectionResult result)
    {
        // 记录检测日志
    }
}

public class PlcObserver : IInspectionObserver
{
    public void OnInspectionCompleted(InspectionResult result)
    {
        // 发送结果到 PLC
    }
}

public class InspectionSubject
{
    private readonly List<IInspectionObserver> _observers = new();
    
    public void Attach(IInspectionObserver observer) => _observers.Add(observer);
    public void Detach(IInspectionObserver observer) => _observers.Remove(observer);
    
    public void Notify(InspectionResult result)
    {
        foreach (var observer in _observers)
            observer.OnInspectionCompleted(result);
    }
}
```

## 四、工控场景的实际运用

### 1. 检测结果通知

订阅检测完成的多个观察者：

- UIObserver：更新检测结果界面显示；
- LoggerObserver：保存检测日志；
- PlcObserver：发送 OK/NG 信号到 PLC；
- ImageSaveObserver：保存检测图片；
- MesObserver：上传检测结果到 MES 系统。

### 2. 设备状态监控

```csharp
public class CameraMonitor
{
    public event EventHandler<CameraStatusEventArgs> StatusChanged;
    public event EventHandler CameraDisconnected;
    public event EventHandler CameraReconnected;
}
```

### 3. 生产数据统计

```csharp
public class ProductionCounter
{
    public event EventHandler<int> TotalCountChanged;
    public event EventHandler<int> NGCountChanged;
    public event EventHandler<double> YieldRateChanged;
}
```

## 五、观察者模式的注意事项

### 1. 事件泄露

如果观察者没有正确取消订阅（-=），被观察者持有观察者的引用，可能导致内存泄漏。

```csharp
// 一定要取消订阅
system.InspectionCompleted -= OnInspectionCompleted;
```

在 WPF 中，WeakEvent 模式或者 CommunityToolkit 的 WeakReferenceMessenger 可以避免这个问题。

### 2. 异常传播

一个观察者抛异常会影响其他观察者：

```csharp
// 不好的写法——一个观察者抛异常会中断通知链
foreach (var observer in _observers)
    observer.OnInspectionCompleted(result);

// 好的写法——每个观察者独立处理异常
foreach (var observer in _observers)
{
    try { observer.OnInspectionCompleted(result); }
    catch (Exception ex) { /* 记录日志，继续通知下一个 */ }
}
```

### 3. 通知顺序

观察者的执行顺序通常不应该依赖。如果顺序很重要，说明设计上可能有问题（比如观察者之间有依赖关系）。

### 4. 线程安全

如果事件在后台线程触发，但观察者需要更新 UI，需要用 Dispatcher 或者 SynchronizationContext 做线程切换。

## 六、观察者模式和 Messenger 的关系

在 WPF 的 CommunityToolkit.Mvvm 中提供的 WeakReferenceMessenger，本质上就是一个高级的观察者模式变体。

- 观察者模式：一对多，一个主题发布事件给多个观察者；
- Messenger：多对多，任何地方都可以发消息，任何地方都可以收消息。

如果只是"一个发布者通知多个订阅者"的场景，用 event 或观察者模式就够了。如果需要更复杂的跨模块通信，Messenger 更灵活。

## 七、INotifyPropertyChanged 也算观察者模式

在 WPF 中，`INotifyPropertyChanged` 接口的本质就是观察者模式——当属性变化时通知 Binding 框架刷新 UI。

```csharp
public class ProductViewModel : INotifyPropertyChanged
{
    private string _productName;
    public string ProductName
    {
        get => _productName;
        set
        {
            _productName = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(ProductName)));
        }
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
}
```

WPF 的 Binding 引擎自动订阅了这个 PropertyChanged 事件，在属性变化时更新 UI。

## 总结

1. 观察者模式定义了一对多的依赖关系，一个对象变化时自动通知所有依赖对象；
2. C# 原生支持观察者模式——event 和 delegate 就是它的天然实现；
3. 观察者模式在工控项目中广泛用于检测结果通知、设备状态监控、生产数据统计；
4. 使用观察者模式时注意事件泄露、异常传播和线程安全问题；
5. WPF 的 INotifyPropertyChanged 本质上也是观察者模式的一种应用。
