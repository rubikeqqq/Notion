# 适配器模式 Adapter

适配器模式解决的核心问题是：让不兼容的接口能够协同工作。

在工业软件项目中，这个模式几乎每天都在用——因为你经常需要把第三方的 SDK（Halcon、VisionPro、OpenCV）封装成项目内部的统一接口。

## 一、适配器模式解决什么问题

假设你的项目定义了一个统一的相机接口：

```csharp
public interface ICamera
{
    ImageData Acquire();
    void Start();
    void Stop();
}
```

但现在需要接入一个第三方相机，它的 SDK 接口是这样的：

```csharp
public class ThirdPartyCamera
{
    public byte[] CaptureFrame() { /* ... */ }
    public void Connect() { /* ... */ }
    public void Disconnect() { /* ... */ }
}
```

接口不匹配，你不能直接把 ThirdPartyCamera 当成 ICamera 用。

适配器模式就是用来解决这个问题的——在不修改 ThirdPartyCamera 的前提下，让它适配你的 ICamera 接口。

## 二、对象适配器（推荐）

对象适配器使用组合方式：

```csharp
public class ThirdPartyCameraAdapter : ICamera
{
    private readonly ThirdPartyCamera _camera;
    
    public ThirdPartyCameraAdapter(ThirdPartyCamera camera)
    {
        _camera = camera;
    }
    
    public ImageData Acquire()
    {
        var frame = _camera.CaptureFrame();
        return ConvertToImageData(frame);
    }
    
    public void Start() => _camera.Connect();
    
    public void Stop() => _camera.Disconnect();
    
    private ImageData ConvertToImageData(byte[] frame) { /* ... */ }
}
```

使用方式：

```csharp
ICamera camera = new ThirdPartyCameraAdapter(new ThirdPartyCamera());
```

现在 ThirdPartyCamera 就可以在项目中和所有依赖 ICamera 的代码一起工作了。

## 三、类适配器（使用继承）

类适配器使用继承方式，在 C# 中比较少用，因为 C# 不支持多继承。

```csharp
public class ThirdPartyCameraAdapter : ThirdPartyCamera, ICamera
{
    public ImageData Acquire()
    {
        var frame = CaptureFrame();
        return ConvertToImageData(frame);
    }
    
    public void Start() => Connect();
    public void Stop() => Disconnect();
}
```

类适配器的局限：需要能继承第三方类（sealed 类不行），而且继承引入了额外的耦合。

在 C# 中优先使用对象适配器（组合）。

## 四、工控场景中的高频使用场景

### 1. 封装不同品牌的相机

```csharp
// 项目内部统一接口
public interface ICamera { /* ... */ }

// 两个不同品牌的相机适配器
public class BaslerCameraAdapter : ICamera { /* 封装 Basler SDK */ }
public class HikCameraAdapter : ICamera { /* 封装 海康 SDK */ }
public class CognexCameraAdapter : ICamera { /* 封装 Cognex SDK */ }
```

### 2. 封装不同视觉算法的统一调用

```csharp
public interface ITemplateMatcher
{
    MatchResult Match(ImageData image);
}

public class HalconMatcherAdapter : ITemplateMatcher
{
    private HShapeModel _model;
    public MatchResult Match(ImageData image) { /* 调用 Halcon 匹配 */ }
}

public class VisionProMatcherAdapter : ITemplateMatcher
{
    private CogPMAlignTool _tool;
    public MatchResult Match(ImageData image) { /* 调用 VisionPro 匹配 */ }
}
```

### 3. 封装不同格式的图像数据

```csharp
public interface IImageSource
{
    ImageData GetImage();
}

public class HalconImageAdapter : IImageSource { /* HImage → ImageData */ }
public class OpenCVImageAdapter : IImageSource { /* Mat → ImageData */ }
public class FileImageAdapter : IImageSource { /* 从文件读取 → ImageData */ }
```

## 五、适配器模式的代价

适配器模式在带来灵活性的同时，也有一些需要注意的地方：

1. **额外的封装层**：每个适配器多一层调用，有轻微的性能损失（通常可忽略）；
2. **接口映射的匹配度**：如果被适配的接口和项目接口在语义上差异太大，适配器可能变得很复杂；
3. **适配器的维护**：第三方 SDK 升级时，适配器可能需要同步更新。

## 六、适配器和外观模式的区别

这两个模式容易混淆：

| 模式 | 目的 | 举例 |
|------|------|------|
| 适配器 | 让不兼容的接口能协同工作 | 把第三方 SDK 接口转成项目统一接口 |
| 外观 | 为复杂子系统提供简化接口 | 封装整个检测流程为一个 Inspect 方法 |

适配器是"接口转换"，外观是"接口简化"。

## 七、适配器模式在 C# 中的简化写法

如果只是适配一个方法的签名，C# 的 lambda 和委托也可以作为轻量级适配器：

```csharp
// 第三方方法
byte[] ThirdPartyCapture() { /* ... */ }

// 适配为目标委托
Func<ImageData> acquire = () => Convert(ThirdPartyCapture());
```

不过对于完整的服务适配，还是建议用正式的适配器类。

## 总结

1. 适配器模式让不兼容的接口可以协同工作，是工业软件中最常用的设计模式之一；
2. C# 中推荐使用对象适配器（组合方式），而非类适配器（继承方式）；
3. 适配器在封装第三方 SDK（相机、算法库、图像源）时非常实用；
4. 适配器专注于"接口转换"，和外观模式（接口简化）有不同的关注点；
5. 适配器不是万能的——如果被适配的接口差异过大，适配器会变得复杂且脆弱。
