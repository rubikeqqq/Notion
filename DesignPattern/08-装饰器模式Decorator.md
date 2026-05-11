# 装饰器模式 Decorator

装饰器模式解决的核心问题是：在不修改现有类的情况下，动态地给对象添加新的行为。

说人话就是：如果你有一个基础的检测服务，想在不改它代码的前提下加上日志、缓存、重试、超时等功能，装饰器模式就是最合适的方案。

## 一、装饰器模式解决什么问题

假设你有一个基础的检测服务：

```csharp
public interface IInspectionService
{
    InspectionResult Inspect(ImageData image);
}

public class BasicInspectionService : IInspectionService
{
    public InspectionResult Inspect(ImageData image)
    {
        // 核心检测逻辑
        return new InspectionResult();
    }
}
```

现在你需要加一个功能：每次检测完成后记录日志。

传统做法是直接改 BasicInspectionService，或者继承它做 override：

- 直接改：违反开闭原则，而且加了日志的检测代码和核心检测逻辑混在一起；
- 继承：如果后面还要加缓存、还要加重试、还要加性能统计，继承层次会爆炸。

装饰器模式用组合的方式解决这个问题。

## 二、装饰器模式的实现

```csharp
public abstract class InspectionServiceDecorator : IInspectionService
{
    protected readonly IInspectionService _inner;
    
    public InspectionServiceDecorator(IInspectionService inner)
    {
        _inner = inner;
    }
    
    public virtual InspectionResult Inspect(ImageData image)
    {
        return _inner.Inspect(image);
    }
}
```

### 日志装饰器

```csharp
public class LoggingInspectionDecorator : InspectionServiceDecorator
{
    private readonly ILogger _logger;
    
    public LoggingInspectionDecorator(IInspectionService inner, ILogger logger) 
        : base(inner) 
    {
        _logger = logger;
    }
    
    public override InspectionResult Inspect(ImageData image)
    {
        _logger.LogInfo("Starting inspection...");
        var result = base.Inspect(image);
        _logger.LogInfo($"Inspection completed: {result.IsPassed}");
        return result;
    }
}
```

### 性能统计装饰器

```csharp
public class PerformanceDecorator : InspectionServiceDecorator
{
    public PerformanceDecorator(IInspectionService inner) : base(inner) { }
    
    public override InspectionResult Inspect(ImageData image)
    {
        var sw = Stopwatch.StartNew();
        var result = base.Inspect(image);
        sw.Stop();
        Console.WriteLine($"Inspection took {sw.ElapsedMilliseconds}ms");
        return result;
    }
}
```

### 重试装饰器

```csharp
public class RetryDecorator : InspectionServiceDecorator
{
    private readonly int _maxRetries;
    
    public RetryDecorator(IInspectionService inner, int maxRetries = 3) 
        : base(inner) 
    {
        _maxRetries = maxRetries;
    }
    
    public override InspectionResult Inspect(ImageData image)
    {
        for (int i = 0; i < _maxRetries; i++)
        {
            try { return base.Inspect(image); }
            catch when (i < _maxRetries - 1) { /* retry */ }
        }
        return base.Inspect(image);
    }
}
```

## 三、装饰器的组合使用

装饰器模式真正的威力在于可以任意组合：

```csharp
IInspectionService service = new BasicInspectionService();
service = new LoggingInspectionDecorator(service, logger);
service = new PerformanceDecorator(service);
service = new RetryDecorator(service, 3);

// 调用时，多个装饰器会按添加顺序依次执行
var result = service.Inspect(image);
```

调用链的执行顺序：

1. RetryDecorator（最外层）
2. PerformanceDecorator
3. LoggingInspectionDecorator
4. BasicInspectionService（最内层，核心逻辑）

这种组合方式比继承灵活得多——你可以按需选择加哪些装饰，而不需要为每种组合创建一个新的子类。

## 四、装饰器模式在 C# 中的天然应用

C# 中很多特性本身就是装饰器思想的体现：

### using + 资源管理

```csharp
// 标准的装饰器模式写法
public class TimestampedStream : Stream
{
    private readonly Stream _inner;
    public TimestampedStream(Stream inner) { _inner = inner; }
    
    public override void Write(byte[] buffer, int offset, int count)
    {
        AddTimestamp();
        _inner.Write(buffer, offset, count);
    }
}
```

### ASP.NET Core 中间件

ASP.NET Core 的中间件管道本质上也是一个装饰器链——每个中间件在请求前后添加行为，然后传递给下一个中间件。

## 五、工控场景的实际运用

### 相机采集装饰

```csharp
ICamera camera = new BaslerCamera();
camera = new CameraRetryDecorator(camera);      // 采集失败时重试
camera = new CameraLoggingDecorator(camera);     // 记录采集日志
camera = new CameraTimeoutDecorator(camera, 5000); // 采集超时保护
```

### 检测流程装饰

```csharp
IInspectionService inspection = new ProductInspectionService();
inspection = new ImageSaveDecorator(inspection);      // 保存检测图片
inspection = new ResultPublishDecorator(inspection);  // 发布检测结果到 MES
inspection = new NotificationDecorator(inspection);   // NG 时发送通知
```

## 六、装饰器模式和继承、AOP 的对比

| 方式 | 灵活性 | 复杂度 | 适用场景 |
|------|--------|--------|----------|
| 继承 | 低（编译期固定） | 低 | 行为固定不变 |
| 装饰器 | 高（运行时组合） | 中 | 行为需要灵活组合 |
| AOP（切面） | 高 | 高 | 横切关注点（日志、事务、权限） |

装饰器模式是 AOP 的一种手动实现方式。如果项目已经使用了 AOP 框架（如 Castle DynamicProxy），也可以用 [Interceptor] 自动实现类似效果。

## 七、需要避免的坑

1. 装饰器链过长会影响性能和调试的难度；
2. 装饰器之间的顺序有依赖时（比如日志应该在性能统计内层还是外层），需要清楚每个装饰器的职责；
3. 不是所有"加一个功能"都适合用装饰器——如果新增功能是业务逻辑本身的一部分，应该直接写在核心类里。

## 总结

1. 装饰器模式可以在不修改原有类的情况下动态扩展功能；
2. C# 中通过组合（持有内部接口引用）实现装饰器；
3. 装饰器可以多层嵌套组合，是继承的灵活替代方案；
4. 在工控项目中常用于给采集、检测、通信服务添加日志、重试、超时等横切关注点；
5. 装饰器链不要太长，且顺序可能影响执行结果，需要明确每层的职责。
