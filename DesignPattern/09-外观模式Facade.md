# 外观模式 Facade

外观模式解决的核心问题是：为复杂子系统提供一个统一的、简化的接口。

说人话就是：如果一个检测流程要调 10 个步骤、5 个服务、3 个设备，不要让调用方去了解这些细节。你做一个外观类，把"执行检测"这个操作简化成一个方法调用。

## 一、外观模式解决什么问题

假设一个完整的视觉检测流程涉及：

1. 相机采集（需要管理相机连接、触发、曝光参数）；
2. 图像预处理（滤波、增强、几何校正）；
3. 模板匹配定位；
4. 尺寸测量；
5. 结果判定；
6. 保存图像和日志；
7. 发送结果到 PLC。

如果不做封装，调用方的代码就会变成这样：

```csharp
public void RunInspection()
{
    _camera.Connect();
    _camera.SetExposure(2000);
    _camera.SetTriggerMode(TriggerMode.Hardware);
    var image = _camera.Acquire();
    
    var processed = _preprocessor.MedianFilter(image);
    processed = _preprocessor.EnhanceContrast(processed);
    
    var matchResult = _matcher.Match(processed);
    var aligned = _fixture.Apply(processed, matchResult.Pose);
    
    var measureResult = _measure.Measure(aligned);
    
    var passed = _judge.Evaluate(measureResult);
    
    _logger.SaveImage(image, passed);
    _logger.SaveResult(measureResult);
    
    _plc.SendResult(passed);
}
```

这段代码的问题：

- 调用方需要了解所有子系统的细节；
- 如果检测流程有变化，所有调用方都要改；
- 测试和模拟困难。

## 二、外观模式的实现

```csharp
public class InspectionFacade
{
    private readonly ICamera _camera;
    private readonly IPreprocessor _preprocessor;
    private readonly ITemplateMatcher _matcher;
    private readonly IMeasurement _measurement;
    private readonly IJudge _judge;
    private readonly ILogger _logger;
    private readonly IPlcClient _plc;
    
    public InspectionFacade(
        ICamera camera, IPreprocessor preprocessor,
        ITemplateMatcher matcher, IMeasurement measurement,
        IJudge judge, ILogger logger, IPlcClient plc)
    {
        _camera = camera;
        _preprocessor = preprocessor;
        _matcher = matcher;
        _measurement = measurement;
        _judge = judge;
        _logger = logger;
        _plc = plc;
    }
    
    public InspectionResult Execute()
    {
        try
        {
            // 1. 采集
            var image = _camera.Acquire();
            
            // 2. 预处理
            var processed = _preprocessor.Process(image);
            
            // 3. 定位
            var pose = _matcher.Match(processed);
            var aligned = _preprocessor.Align(processed, pose);
            
            // 4. 测量
            var measures = _measurement.Measure(aligned);
            
            // 5. 判定
            var result = new InspectionResult
            {
                IsPassed = _judge.Evaluate(measures),
                Measures = measures,
                Image = image,
                Timestamp = DateTime.Now
            };
            
            // 6. 保存和通信
            _logger.Save(result);
            _plc.SendResult(result.IsPassed);
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex);
            throw;
        }
    }
}
```

调用方只需要：

```csharp
var result = _inspectionFacade.Execute();
```

## 三、外观模式的真正价值

外观模式不是"加了一个管理层"，它有几个实际的好处：

### 1. 降低使用门槛

调用方不需要了解相机怎么配、预处理做了什么、PLC 怎么通信。他们只需要知道"调 Execute 就能完成一次检测"。

### 2. 集中变化

如果检测流程变了（比如增加了标定步骤、换了判定规则），只需要改 InspectionFacade 内部，所有调用方不受影响。

### 3. 便于测试

```csharp
// 测试时可以替换外观中的子系统
var mockFacade = new InspectionFacade(
    new MockCamera(), new MockPreprocessor(), 
    new MockMatcher(), /* ... */);
```

## 四、外观模式和适配器模式的区别

| 模式 | 目的 | 典型场景 |
|------|------|----------|
| 适配器 | 接口转换——让不兼容的接口能一起工作 | 封装第三方 SDK |
| 外观 | 接口简化——让复杂子系统更容易使用 | 封装整个业务流程 |

适配器是为了"兼容"，外观是为了"简化"。

## 五、外观模式的变体：门面 vs 中介者

外观模式容易被混淆的一个概念是中介者模式（Mediator）：

- 外观：为子系统提供简化接口，子系统之间不需要通过外观通信；
- 中介者：封装对象之间的交互，对象之间不直接通信，都通过中介者。

在工控项目中，外观模式更适合"对外暴露统一入口"的场景，中介者更适合"多个模块之间需要解耦通信"的场景。

## 六、工控场景的实际运用

### 检测站外观

```csharp
public class InspectionStationFacade
{
    public void LoadProduct(int productId) { /* 加载配方 */ }
    public InspectionResult RunOnce() { /* 执行一次完整检测 */ }
    public void Calibrate() { /* 执行标定 */ }
    public string GetStatus() { /* 获取检测站状态 */ }
}
```

### MES 通信外观

对外只暴露 SubmitResult、GetProductionOrder、ReportDefect 几个方法，内部封装了与 MES 系统的复杂通信逻辑。

## 七、使用外观模式时需要注意

1. 外观不应该成为"上帝类"——如果外观变得过大，可以考虑拆分成多个外观；
2. 外观模式不能阻止高级用户直接访问子系统——需要绕过外观时应该允许；
3. 外观层会增加一次调用跳转，但性能损失通常可以忽略。

## 总结

1. 外观模式的核心价值是简化复杂子系统的使用接口；
2. 外观模式让调用方不需要了解子系统的内部细节，降低了使用门槛；
3. 外观模式将变化集中在一点，当检测流程变化时只需要修改外观类；
4. 外观模式和适配器模式不同——前者简化接口，后者转换接口；
5. 注意不要让外观变成过于庞大的"上帝类"。
