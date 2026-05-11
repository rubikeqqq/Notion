# 模板方法模式 Template Method

模板方法模式解决的核心问题是：定义一个操作中的算法骨架，而将一些步骤延迟到子类中实现。

说人话就是：父类把检测流程的"大步骤"定好了（采集→预处理→定位→测量→判定），子类只需要实现每个步骤的具体细节，不能改变流程的骨架。

## 一、模板方法模式解决什么问题

假设有两类产品的检测流程，它们的大步骤是一样的：

```
预处理 → 定位 → 测量 → 判定
```

但每个步骤的具体实现不同：

- 产品 A 的预处理需要高斯滤波，产品 B 需要中值滤波；
- 产品 A 用形状匹配定位，产品 B 用灰度匹配定位。

如果不做抽象，每个产品都要重复写一遍流程控制代码。

## 二、模板方法模式的实现

```csharp
public abstract class ProductInspector
{
    // 模板方法——定义了算法骨架
    public InspectionResult Inspect(ImageData image)
    {
        Preprocess(image);
        
        var pose = Match(image);
        
        var measures = Measure(image, pose);
        
        var result = Judge(measures);
        
        OnCompleted(result);
        
        return result;
    }
    
    // 子类可以重写的步骤（有默认实现）
    protected virtual void Preprocess(ImageData image)
    {
        // 默认预处理——子类可以重写
    }
    
    // 子类必须实现的步骤
    protected abstract MatchResult Match(ImageData image);
    
    protected abstract List<Measurement> Measure(ImageData image, MatchResult pose);
    
    // 有默认实现的步骤
    protected virtual Judgment Judge(List<Measurement> measures)
    {
        return new Judgment { IsPassed = measures.All(m => m.IsInTolerance) };
    }
    
    // 钩子方法——子类可以选择性重写
    protected virtual void OnCompleted(InspectionResult result)
    {
        // 默认什么都不做
    }
}
```

### 具体子类

```csharp
public class PhoneCaseInspector : ProductInspector
{
    protected override void Preprocess(ImageData image)
    {
        // 手机中框需要高斯滤波 + 增强对比度
        image.GaussianBlur(3);
        image.EnhanceContrast();
    }
    
    protected override MatchResult Match(ImageData image)
    {
        // 使用形状匹配
        return _shapeMatcher.Match(image);
    }
    
    protected override List<Measurement> Measure(ImageData image, MatchResult pose)
    {
        // 测量外形尺寸和孔位
        return _measureService.MeasureDimensions(image, pose);
    }
}

public class PCBBoardInspector : ProductInspector
{
    protected override void Preprocess(ImageData image)
    {
        // PCB 需要中值滤波 + 灰度归一化
        image.MedianFilter(5);
        image.Normalize();
    }
    
    protected override MatchResult Match(ImageData image)
    {
        // 使用灰度匹配
        return _grayMatcher.Match(image);
    }
    
    protected override List<Measurement> Measure(ImageData image, MatchResult pose)
    {
        // 测量焊点位置和面积
        return _measureService.MeasureSolderPoints(image, pose);
    }
    
    protected override void OnCompleted(InspectionResult result)
    {
        // PCB 检测完成后额外做数据上传
        _mesService.Upload(result);
    }
}
```

## 三、模板方法的核心机制

模板方法模式有几个关键的设计元素：

### 1. 模板方法（public，不虚）

定义了算法的步骤和顺序。子类不能重写这个模板方法。

在 C# 中可以在基类中把模板方法标记为 sealed：

```csharp
public sealed override InspectionResult Inspect(ImageData image)
```

但不是必需的——约定大于配置。

### 2. 抽象步骤（abstract）

子类必须实现的步骤。

### 3. 可选步骤（virtual）

子类可以选择重写，也可以直接用默认实现。

### 4. 钩子方法（hook）

钩子方法通常是一个空的 virtual 方法，子类可以选择性重写，在特定时机插入额外逻辑。

## 四、模板方法和策略模式的区别

| 模式 | 控制权 | 目的 |
|------|--------|------|
| 模板方法 | 父类控制流程，子类实现细节 | 复用流程骨架 |
| 策略模式 | 上下文选择策略，策略控制算法 | 互换算法族 |

一个简单的区分方式：

- 模板方法：流程一样，步骤实现不同；
- 策略模式：流程完全不一样（整个替换）。

在工控项目中，模板方法适合检测流程骨架固定的场景，策略模式适合整个检测逻辑都能替换的场景。

## 五、工控场景的实际运用

### 1. 检测流程标准化

所有产品都走"采集→预处理→定位→测量→判定"的骨架，不同产品实现各自的步骤细节。

### 2. 标定流程标准化

```csharp
public abstract class CalibrationProcedure
{
    public CalibrationResult Run()
    {
        PrepareCalibrationPlate();
        CollectImages();
        RunCalibration();
        ValidateResult();
        SaveResult();
        return _result;
    }
    
    protected abstract void PrepareCalibrationPlate();
    protected abstract int ImageCount { get; }
    protected abstract void CollectImages();
    // ...
}
```

### 3. 开机自检流程

```csharp
public abstract class StartupProcedure
{
    public StartupResult Run()
    {
        CheckCameraConnection();
        CheckLighting();
        CheckCalibration();
        LoadRecipe();
        return new StartupResult { Ready = true };
    }
    
    protected abstract void CheckCameraConnection();
    protected abstract void CheckLighting();
    protected virtual void CheckCalibration() { /* 默认实现 */ }
    protected abstract void LoadRecipe();
}
```

## 六、模板方法的优势

1. **代码复用**——所有产品的检测流程控制代码都在基类里，子类只需要写步骤实现；
2. **标准化**——所有检测流程都走同样的骨架，不会出现某个产品跳过了某一步；
3. **统一修改**——如果需要在流程中增加一个步骤（如增加标定验证），只需要改基类。

## 七、需要注意的坑

1. **子类过于依赖父类**——如果父类的流程变了，所有子类都可能受影响；
2. **钩子方法过多**——钩子方法多了会让子类难以理解哪些是必须实现的、哪些是可选的；
3. **不要破坏模板方法的流程**——子类不应该通过重写来改变流程的执行顺序。

## 总结

1. 模板方法模式在父类中定义算法骨架，子类实现具体步骤；
2. 模板方法模式实现了"控制反转"的一种形式——父类决定何时调用子类方法；
3. 在工控项目中适合检测流程标准化、标定流程标准化等场景；
4. 模板方法和策略模式的区别是：模板方法固定骨架、子类实现细节，策略模式完全替换算法；
5. 模板方法的关键是骨架稳定，如果骨架频繁变化，就不适合用模板方法。
