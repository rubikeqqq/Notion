# 多工具串联与 CogToolBlock 流程设计

前面几篇分别介绍了定位、测量、识别、采集等单项能力。

但在实际项目中，很少只用单一工具。大多数检测场景需要多个工具配合工作。

这一篇的重点不是单个工具怎么用，而是：

多个工具如何组织成一条完整的检测流程。

## 一、工具串联的基本模式

一个典型的 VisionPro 检测流程通常是这样的：

```
采集图像 → 定位 → 坐标对齐 → 多项检测 → 结果判定
```

对应到工具上就是：

```
CogAcqFifoTool → CogPMAlignTool → CogFixtureTool → FindCircle / Blob / OCR → 输出结果
```

## 二、在 CogToolBlock 中组织工具链

CogToolBlock 是组织多工具串联的推荐方式。

### 1. 添加工具并建立连接

```csharp
CogToolBlock toolBlock = new CogToolBlock();

// 1. 定位工具
CogPMAlignTool pmAlign = new CogPMAlignTool();
toolBlock.Tools.Add(pmAlign, "定位");

// 2. 坐标对齐工具
CogFixtureTool fixture = new CogFixtureTool();
toolBlock.Tools.Add(fixture, "坐标对齐");

// 3. 测量孔径
CogFindCircleTool findCircle = new CogFindCircleTool();
toolBlock.Tools.Add(findCircle, "测孔径");

// 4. 缺陷检测
CogBlobTool blob = new CogBlobTool();
toolBlock.Tools.Add(blob, "缺陷检测");

// 定义输入输出
toolBlock.Inputs.Add("InputImage", typeof(CogImage8Grey));
toolBlock.Outputs.Add("PassFail", typeof(bool));
toolBlock.Outputs.Add("Results", typeof(string));
```

### 2. 建立数据连接

在 CogToolBlock 中，工具之间的数据传递通过"连接"实现。

需要手动连接的地方通常是：

* PMAlign 的输出图像 → FixturTool 的输入图像；
* PMAlign 的定位结果 → FixturTool 的 Fixture；
* FixturTool 的输出图像 → 各测量工具的输入图像。

```csharp
// 设置工具之间的数据连接
// PMAlign 的输出图像传给 Fixture
fixture.InputImage = pmAlign.OutputImage as CogImage8Grey;

// PMAlign 的定位结果传给 Fixture 作为参考坐标系
fixture.Fixture = pmAlign.Results[0].GetPose();

// Fixture 的输出图像传给后续测量工具
findCircle.InputImage = fixture.OutputImage as CogImage8Grey;
blob.InputImage = fixture.OutputImage as CogImage8Grey;
```

### 3. 执行工具链

```csharp
// 设置输入
toolBlock.Inputs["InputImage"].Value = image;

// 执行整个工具链
toolBlock.Run();

// 读取输出
bool pass = (bool)toolBlock.Outputs["PassFail"].Value;
string result = (string)toolBlock.Outputs["Results"].Value;
```

## 三、工具链的执行顺序

CogToolBlock 会自动分析工具间的数据依赖关系，并按照依赖顺序执行。

但理解执行顺序对调试很重要。典型的执行顺序：

1. **输入赋值** —— 设置输入图像和其他参数；
2. **PMAlign.Run()** —— 定位，找到产品位置；
3. **Fixture.Run()** —— 坐标变换，基于定位结果对齐图像；
4. **FindCircle.Run()** —— 在摆正后的图像上测量；
5. **Blob.Run()** —— 在摆正后的图像上检测缺陷；
6. **结果汇总** —— 根据各工具的输出判定 OK/NG。

如果一个工具不依赖前一个工具的输出，CogToolBlock 可能会并行执行它们。

## 四、条件执行

有些场景下，某些工具可能只需要在特定条件下执行。

例如：如果定位失败，后续的测量和识别就不需要执行了。

在 CogToolBlock 中实现条件执行有几种方式：

### 1. 在 Run 之前检查

```csharp
pmAlign.Run();

if (pmAlign.Results == null || pmAlign.Results.Count == 0)
{
    // 定位失败，跳过后续检测
    toolBlock.Outputs["PassFail"].Value = false;
    toolBlock.Outputs["Results"].Value = "定位失败";
    return;
}

// 定位成功，继续执行
fixture.Run();
findCircle.Run();
blob.Run();
```

### 2. 使用 Enabled 属性

```csharp
// 根据前一步结果决定是否执行某个工具
findCircle.Enabled = (pmAlign.Results != null && pmAlign.Results.Count > 0);
```

### 3. 在 CogToolBlock 中通过脚本（CogScript）控制

对于更复杂的条件逻辑，可以在 CogToolBlock 中加入 CogScript 工具来处理。

## 五、支持多产品型号

实际项目中，同一台设备可能需要检测多种产品型号。

不同型号之间，定位模板、检测区域、判定标准可能都不同。

### 方案 1：多个 .vpp 文件

```csharp
public class MultiProductInspector
{
    private Dictionary<string, CogToolBlock> _toolBlocks = new Dictionary<string, CogToolBlock>();

    public void LoadProduct(string productType, string vppFile)
    {
        _toolBlocks[productType] = CogSerializer.LoadObjectFromFile(vppFile) as CogToolBlock;
    }

    public bool Inspect(string productType, CogImage8Grey image)
    {
        if (!_toolBlocks.ContainsKey(productType))
            return false;

        var tb = _toolBlocks[productType];
        tb.Inputs["InputImage"].Value = image;
        tb.Run();
        return (bool)tb.Outputs["PassFail"].Value;
    }
}
```

### 方案 2：一个 .vpp 多组参数

```csharp
// 在工具链中加载对应的 PMAlign 模板和检测参数
public void SwitchProduct(string productType)
{
    pmAlign.Pattern.Load($"{productType}.pat");
    // 切换检测区域参数
}
```

方案 1 更彻底（工具链完全独立），方案 2 更轻量（共享工具框架）。

## 六、结果汇总

多工具检测后，需要把每个工具的结果汇总起来做总体判定。

```csharp
// 使用一个结果汇总类
public class InspectionResult
{
    public bool OverallPass { get; set; }
    public bool LocatePass { get; set; }
    public double HoleDiameter { get; set; }
    public bool HolePass { get; set; }
    public int DefectCount { get; set; }
    public bool DefectPass { get; set; }
    public string SerialNumber { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;

    public bool IsOK => LocatePass && HolePass && DefectPass;
}
```

在 CogToolBlock 的输出中设置汇总结果：

```csharp
// 在执行完工具链后，汇总结果
var result = new InspectionResult
{
    LocatePass = pmAlign.Results != null && pmAlign.Results.Count > 0,
    HoleDiameter = findCircle.Results?.CircleRadius * 2 ?? 0,
    HolePass = holeDiameter >= minDiameter && holeDiameter <= maxDiameter,
    DefectCount = validDefects.Count,
    DefectPass = validDefects.Count == 0,
};

toolBlock.Outputs["PassFail"].Value = result.IsOK;
toolBlock.Outputs["Results"].Value = JsonSerializer.Serialize(result);
```

## 七、性能考虑

### 1. 尽量减少不必要的大图变换

FixturTool 会对整张图像做变换，如果图像很大（如 500 万像素），这个操作的耗时可能很明显。

如果后续工具有些不需要全图变换，可以只对需要变换的工具接 Fixture 的输出。

### 2. 合理设置搜索区域

每个工具都在全图中搜索会非常耗时。合理设置搜索区域可以显著提升速度。

### 3. 考虑使用区域处理

```csharp
// 只对图像中的感兴趣区域做处理
fixture.RunParams.RegionMode = CogFixtureRegionModeConstants.InputImage;
```

### 4. 工具链长度和检测节拍

工具链越长，单次检测的总耗时越长。如果产线节拍要求很高（如每分钟 60 个），需要在工具数量和精度之间做权衡。

## 八、CogToolBlock 的 .vpp 调试

用 CogToolBlock 组织工具链最大的好处之一，就是可以在 Cognex Explorer 中打开 .vpp 文件进行可视化调试：

1. 保存工具链为 .vpp 文件；
2. 在 Cognex Explorer 中打开；
3. 逐工具检查参数和结果；
4. 调整参数后重新保存；
5. 应用程序重新加载即可更新。

这样就把"调参数 → 编译 → 运行 → 看结果"的循环，缩短为"调参数 → 保存 → 立刻看结果"。

## 九、一个完整的工具链设计思路

设计一条检测工具链时，可以按这个顺序来思考：

1. **图像从哪里来** —— 实时采集还是读取文件；
2. **先找到目标** —— PMAlign 负责定位；
3. **坐标对齐** —— FixturTool 保证后续工具不受位置偏移影响；
4. **列出所有检测项** —— 测量、缺陷、识别等，每个检测项对应一个工具；
5. **汇总判定** —— 所有检测项的结果合并，得出 OK/NG；
6. **输出** —— 把结果传给上位机、PLC 或保存到数据库。

这个顺序是从上到下的，每一步都依赖上一步的输出。设计好每一步的输入输出接口，工具链就会很清晰。

## 总结

1. 多工具串联的典型模式：采集 → 定位 → 坐标对齐 → 多项检测 → 结果判定；
2. CogToolBlock 是组织多工具链的首选方式，支持依赖分析和顺序执行；
3. 工具链可以支持多型号切换（多 .vpp 或动态切换参数）；
4. 结果汇总要考虑每个检测项的 OK/NG，然后做总体判定；
5. 使用 CogToolBlock 的 .vpp 文件可以在 Cognex Explorer 中可视化调试，大幅缩短调参周期。
