# 策略模式 Strategy

策略模式解决的核心问题是：定义一组算法，把它们封装起来，并且在运行时可以互相替换。

说人话就是：一个任务有多种做法（比如不同的检测算法、不同的标定方法、不同的光源控制方式），把每种做法封装成独立的策略类，在运行时根据条件选择使用哪一种。

## 一、策略模式解决什么问题

假设一个检测系统需要在多种"缺陷检测算法"之间选择：

```csharp
public class DefectDetector
{
    public List<Defect> Detect(ImageData image, string method)
    {
        if (method == "Blob")
        {
            // Blob 分析检测
        }
        else if (method == "TemplateDiff")
        {
            // 模板差分检测
        }
        else if (method == "ML")
        {
            // 机器学习检测
        }
        // 每加一种方法都要加一个 else if
    }
}
```

这种写法的核心问题：

- 每次新增检测算法都要改 DefectDetector；
- 所有算法耦合在同一个方法里，互相影响；
- 算法的选择逻辑和执行逻辑混在一起。

## 二、策略模式的实现

```csharp
// 策略接口
public interface IDefectDetectionStrategy
{
    List<Defect> Detect(ImageData image);
}

// 具体策略
public class BlobDetectionStrategy : IDefectDetectionStrategy
{
    public List<Defect> Detect(ImageData image)
    {
        // Blob 分析实现
        return new List<Defect>();
    }
}

public class TemplateDiffStrategy : IDefectDetectionStrategy
{
    public List<Defect> Detect(ImageData image)
    {
        // 模板差分实现
        return new List<Defect>();
    }
}

public class MLDetectionStrategy : IDefectDetectionStrategy
{
    public List<Defect> Detect(ImageData image)
    {
        // 机器学习实现
        return new List<Defect>();
    }
}

// 上下文
public class DefectDetector
{
    private readonly IDefectDetectionStrategy _strategy;
    
    public DefectDetector(IDefectDetectionStrategy strategy)
    {
        _strategy = strategy;
    }
    
    public List<Defect> Detect(ImageData image)
    {
        return _strategy.Detect(image);
    }
}
```

使用方式：

```csharp
var strategy = new BlobDetectionStrategy();  // 可以根据配置选择
var detector = new DefectDetector(strategy);
var defects = detector.Detect(image);
```

现在 Detector 不再关心"具体用什么算法"，只关心"调用策略接口"。

## 三、和简单 if/else 的区别

策略模式不是简单地把 if/else 变成了多态，它更重要的是让：

- 每种算法可以独立测试；
- 新增算法不需要修改现有代码（开闭原则）；
- 算法可以在运行时切换。

## 四、工控场景的实际运用

### 1. 不同产品的检测策略

```csharp
public interface IProductInspector
{
    InspectionResult Inspect(ImageData image);
}

public class PhoneCaseInspector : IProductInspector { /* ... */ }
public class PCBBoardInspector : IProductInspector { /* ... */ }
public class GlassInspector : IProductInspector { /* ... */ }
```

根据产品型号注入不同的检测策略：

```csharp
var inspector = productType switch
{
    "Phone" => new PhoneCaseInspector(),
    "PCB" => new PCBBoardInspector(),
    "Glass" => new GlassInspector(),
    _ => throw new ArgumentException()
};
```

### 2. 光源控制策略

```csharp
public interface ILightingStrategy
{
    void Apply(ILightController controller);
}

public class RingLightStrategy : ILightingStrategy { /* 环形光配置 */ }
public class BackLightStrategy : ILightingStrategy { /* 背光配置 */ }
public class MultiAngleStrategy : ILightingStrategy { /* 多角度组合 */ }
```

### 3. 标定方法策略

```csharp
public interface ICalibrationStrategy
{
    CalibrationResult Calibrate(ImageData[] images);
}

public class CheckerboardCalibration : ICalibrationStrategy { /* 棋盘格标定 */ }
public class DotArrayCalibration : ICalibrationStrategy { /* 圆点阵标定 */ }
```

## 五、策略模式 vs 工厂模式

策略模式和工厂模式经常一起出现，但关注点不同：

- 工厂模式：关注"创建哪个对象"（对象创建）；
- 策略模式：关注"执行哪个算法"（行为选择）。

常见的组合用法是：

```csharp
// 工厂负责创建策略
var strategy = new DetectionStrategyFactory().Create("Blob");

// 策略负责执行算法
var detector = new DefectDetector(strategy);
```

## 六、策略模式的代价

策略模式不是没有代价的：

1. **类数量增加**：每个策略一个类，策略多了维护成本上升；
2. **调用方需要知道策略的区别**：如果调用方不知道 BlobDetectionStrategy 和 TemplateDiffStrategy 有什么不同，就没法选；
3. **策略之间的重复代码**：不同策略之间可能有共同的逻辑，需要提取到基类或工具类中。

## 七、什么时候不适合策略模式

- 算法只有一两种，且不会扩展；
- 算法选择逻辑极其简单（一个 if/else 足够）；
- 算法之间差异太大，无法抽象出统一的接口。

## 八、C# 中的简化方案

如果策略逻辑非常简单，C# 的委托也可以作为轻量级策略：

```csharp
public class DefectDetector
{
    private readonly Func<ImageData, List<Defect>> _strategy;
    
    public DefectDetector(Func<ImageData, List<Defect>> strategy)
    {
        _strategy = strategy;
    }
    
    public List<Defect> Detect(ImageData image) => _strategy(image);
}

// 使用
var detector = new DefectDetector(img => BlobAnalysis(img));
```

对于复杂策略还是建议用正式的接口实现，代码结构更清晰。

## 总结

1. 策略模式把一组算法封装成可互换的策略类，实现算法和调用方的解耦；
2. 策略模式遵循开闭原则——新增算法不需要修改现有代码；
3. 在工控项目中常用于产品检测策略、光源控制策略、标定方法策略等场景；
4. 策略模式经常和工厂模式配合使用——工厂选策略，策略执行业务；
5. 策略模式会增加类的数量，简单场景不要强行套用。
