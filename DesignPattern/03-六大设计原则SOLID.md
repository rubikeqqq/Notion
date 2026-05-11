# 六大设计原则 SOLID

在学习具体的设计模式之前，先理解设计原则更重要。

如果说设计模式是"武功招式"，那设计原则就是"内功心法"。

招数学得再多，没有内功支撑，遇到真实场景还是不知道怎么变通。反过来，内功扎实了，很多模式自己就能推导出来。

## 一、单一职责原则（SRP）

### 定义

一个类应该只有一个引起它变化的原因。

### 理解

说人话就是：一个类只做一件事。

如果一个类既负责图像采集，又负责算法处理，又负责结果保存，那任何一个需求的改动都会影响到这个类。

### 违反的例子

```csharp
public class CameraService
{
    public void AcquireImage() { /* ... */ }
    public void ProcessImage() { /* ... */ }
    public void SaveResult() { /* ... */ }
}
```

### 遵循的例子

```csharp
public class CameraAcquisition { public ImageData Acquire() { /* ... */ } }
public class ImageProcessor { public ProcessResult Process(ImageData data) { /* ... */ } }
public class ResultRepository { public void Save(ProcessResult result) { /* ... */ } }
```

### 怎么判断

如果有一个理由让你想修改这个类，这个类的职责就是合理的。如果你因为"采集接口变了"和"保存格式变了"两个不相关的理由去修改同一个类，说明它违反了 SRP。

## 二、开闭原则（OCP）

### 定义

对扩展开放，对修改关闭。

### 理解

说人话就是：加功能的时候，尽量通过新增代码来实现，而不是去改已有的代码。

### 违反的例子

```csharp
public class Inspector
{
    public void Run(string productType)
    {
        if (productType == "TypeA") { /* A 的检测逻辑 */ }
        else if (productType == "TypeB") { /* B 的检测逻辑 */ }
    }
}
```

每次加新产品类型都要改 Inspector 类。

### 遵循的例子

```csharp
public interface IProductInspector
{
    bool Inspect(ImageData image);
}

public class TypeAInspector : IProductInspector { /* ... */ }
public class TypeBInspector : IProductInspector { /* ... */ }
```

加新产品时只需要新增一个实现类，不需要改已有代码。

### 怎么判断

如果每次需求变更都有一个或多个已有的类被修改，说明系统没有遵循开闭原则。

## 三、里氏替换原则（LSP）

### 定义

子类对象应该能够替换父类对象，并且程序的行为不变。

### 理解

说人话就是：子类不能破坏父类的约定。

如果你有一个"矩形"的基类，然后继承了"正方形"类，但正方形的宽度和高度必须相等，这样用正方形的代码就不能无差别地替换矩形——这就违反了 LSP。

### 违反的例子

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
}

public class Square : Rectangle
{
    public override int Width
    {
        set { base.Width = value; base.Height = value; }
    }
    // ...
}
```

### 正确做法

如果不满足"is-a"的行为一致性，就不要用继承。宁可抽取共同的抽象接口。

## 四、接口隔离原则（ISP）

### 定义

不应该强迫客户端依赖它不需要的接口。

### 理解

说人话就是：接口要小而专，不要大而全。

### 违反的例子

```csharp
public interface IVisionSystem
{
    void Acquire();
    void Match();
    void Measure();
    void ReadCode();
    void SaveImage();
    void PrintReport();
}
```

如果某一个模块只需要 Acquire 和 SaveImage，也被迫依赖 Match、Measure、ReadCode 等方法。

### 遵循的例子

```csharp
public interface IAcquisition { void Acquire(); }
public interface ITemplateMatch { void Match(); }
public interface IMeasurement { void Measure(); }
public interface ICodeReader { void ReadCode(); }
```

### 怎么判断

如果你的接口中有方法在特定实现中是空的或抛出 NotImplementedException，说明接口可能太大了。

## 五、依赖反转原则（DIP）

### 定义

高层模块不应该依赖低层模块，两者都应该依赖抽象。抽象不应该依赖细节，细节应该依赖抽象。

### 理解

说人话就是：不要 new 出具体类，要依赖接口或抽象类。

### 违反的例子

```csharp
public class Inspector
{
    private HalconMatcher _matcher = new HalconMatcher();
    public void Run() { _matcher.Match(); }
}
```

如果将来要从 Halcon 换成 VisionPro，Inspector 类必须修改。

### 遵循的例子

```csharp
public class Inspector
{
    private readonly IMatcher _matcher;
    public Inspector(IMatcher matcher) { _matcher = matcher; }
    public void Run() { _matcher.Match(); }
}
```

依赖注入（DI）的核心思想就来自 DIP。

### 怎么判断

如果你在一个类里看到了很多 new 关键字创建具体对象，很可能违反了 DIP。

## 六、迪米特法则（LoD）

### 定义

一个对象应该对其他对象保持最少的了解。

### 理解

说人话就是：不要和"陌生人"说话，只和你直接相关的对象交互。

### 违反的例子

```csharp
// 在一个处理类里，通过多层访问去拿数据
var result = inspector.Runner.Camera.FrameGrabber.Status;
```

这个调用链说明了 Inspector 直接依赖了 Runner → Camera → FrameGrabber，其中任何一层变化都会影响到 Inspector。

### 遵循的例子

```csharp
var status = inspector.GetFrameGrabberStatus();
```

用封装把调用链藏起来，减少类之间的直接依赖。

## 七、原则和模式的关系

SOLID 原则是设计模式的基础：

| 原则 | 对应的模式思路 |
|------|---------------|
| SRP | 每个模式都有明确的职责范围 |
| OCP | 策略模式、模板方法、装饰器等都是典型的扩展方式 |
| LSP | 工厂模式、策略模式都依赖子类能安全替换父类 |
| ISP | 适配器模式把一个接口转成另一个接口 |
| DIP | 所有依赖抽象的模式都自然遵循 DIP |

理解原则之后，很多模式的产生原因就是"为了解决某个原则被违反时的问题"。

## 总结

1. SOLID 是面向对象设计的五大基本原则（+ 迪米特定律共六条）；
2. 单一职责确保类不会因为多个不相关的原因被修改；
3. 开闭原则让系统可以通过扩展而非修改来适应变化；
4. 里氏替换保证继承关系不会破坏程序逻辑；
5. 接口隔离避免强迫客户端依赖不需要的方法；
6. 依赖反转通过依赖抽象来解耦高层和低层模块。
