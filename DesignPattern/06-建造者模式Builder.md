# 建造者模式 Builder

建造者模式解决的核心问题是：将一个复杂对象的构建过程与它的表示分离。

说人话就是：如果一个对象的构造过程很复杂（需要很多参数、很多步骤），建造者模式可以让你分步构建，并且可以复用同样的构建过程得到不同的对象。

## 一、什么时候需要建造者模式

### 构造函数参数过多

```csharp
public class InspectionReport
{
    public InspectionReport(
        string productName,
        string batchNumber,
        DateTime inspectionTime,
        bool isPassed,
        double score,
        string defectType,
        string operatorName,
        string cameraConfig,
        string algorithmVersion,
        List<MeasurementResult> measurements)
    { /* ... */ }
}
```

这种构造函数的问题：

- 调用方搞不清楚每个参数是什么意思；
- 很多参数其实有默认值，但调用时还得全传；
- 新增参数会影响所有调用方。

### 构建步骤复杂

创建一个检测报告需要：设置基本信息 → 添加测量结果 → 计算判定 → 生成图片 → 格式化输出。

如果这些步骤都放在一个构造函数里或一个方法里，代码会变得非常臃肿。

## 二、建造者模式的实现

```csharp
public class InspectionReport { /* report 的属性 */ }

public class ReportBuilder
{
    private readonly InspectionReport _report = new InspectionReport();
    
    public ReportBuilder WithProductInfo(string name, string batch)
    {
        _report.ProductName = name;
        _report.BatchNumber = batch;
        return this;
    }
    
    public ReportBuilder WithInspectionTime(DateTime time)
    {
        _report.InspectionTime = time;
        return this;
    }
    
    public ReportBuilder AddMeasurement(MeasurementResult result)
    {
        _report.Measurements ??= new List<MeasurementResult>();
        _report.Measurements.Add(result);
        return this;
    }
    
    public ReportBuilder SetResult(bool passed, double score)
    {
        _report.IsPassed = passed;
        _report.Score = score;
        return this;
    }
    
    public InspectionReport Build()
    {
        // 在 Build 之前可以做最终校验
        if (_report.Measurements == null || !_report.Measurements.Any())
            throw new InvalidOperationException("No measurements added");
        return _report;
    }
}
```

使用方式：

```csharp
var report = new ReportBuilder()
    .WithProductInfo("ProductA", "BATCH001")
    .WithInspectionTime(DateTime.Now)
    .AddMeasurement(measure1)
    .AddMeasurement(measure2)
    .SetResult(true, 0.95)
    .Build();
```

## 三、链式调用的实现关键

建造者模式中常见的链式调用，关键在于每个设置方法都返回 `this`。

这样调用方可以用一行连续的代码完成对象的构建，可读性非常好。

## 四、工控场景的实际运用

### 检测流程配置

```csharp
public class InspectionPipelineBuilder
{
    private readonly List<IInspectionStep> _steps = new();
    
    public InspectionPipelineBuilder AddStep<T>() where T : IInspectionStep, new()
    {
        _steps.Add(new T());
        return this;
    }
    
    public InspectionPipelineBuilder WithPreprocessing()
    {
        _steps.Add(new PreprocessingStep());
        return this;
    }
    
    public InspectionPipelineBuilder WithTemplateMatch()
    {
        _steps.Add(new TemplateMatchStep());
        return this;
    }
    
    public InspectionPipelineBuilder WithMeasurement()
    {
        _steps.Add(new MeasurementStep());
        return this;
    }
    
    public InspectionPipelineBuilder WithCodeReading()
    {
        _steps.Add(new CodeReadingStep());
        return this;
    }
    
    public InspectionPipeline Build()
    {
        return new InspectionPipeline(_steps);
    }
}

// 使用
var pipeline = new InspectionPipelineBuilder()
    .WithPreprocessing()
    .WithTemplateMatch()
    .WithMeasurement()
    .Build();
```

### 相机配置构建

```csharp
var config = new CameraConfigBuilder()
    .WithIpAddress("192.168.1.100")
    .WithExposureTime(2000)        // 2000 μs
    .WithGain(1.5)
    .WithTriggerMode(TriggerMode.Hardware)
    .WithResolution(2448, 2048)
    .Build();
```

## 五、建造者模式和构造函数的对比

| 对比项 | 构造函数 | 建造者模式 |
|--------|----------|------------|
| 可读性 | 参数多时易混淆 | 方法名说明意图 |
| 可选参数 | 需要重载或默认值 | 只设置需要的参数 |
| 参数校验 | 构造函数内校验 | Build 时统一校验 |
| 不可变性 | 可以在构造函数内赋值后 readonly | 需要内部属性可写，但返回的对象可以改为不可变 |

## 六、建造者模式和其他模式的关系

建造者模式和工厂模式都是创建型模式，但关注点不同：

- 工厂模式：关注"创建哪个类的实例"（产品选择）；
- 建造者模式：关注"怎么构建这个复杂对象"（构建过程）。

它们可以组合使用：用工厂生产原材料组件，用建造者装配成最终产品。

## 总结

1. 建造者模式解决的是复杂对象的分步构建问题，让代码可读性更好；
2. 链式调用（fluent interface）是建造者模式的典型风格；
3. 构造函数参数过多、构建步骤复杂时，考虑用建造者模式重构；
4. 工厂模式和建造者模式关注点不同，可以配合使用；
5. 建造者模式不适合简单对象——简单场景直接用构造函数或对象初始化器即可。
