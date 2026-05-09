# CogToolBlock 基础

在 VisionPro 中，CogToolBlock 是最核心的容器类型之一。

它的主要职责只有一个：

把多个视觉工具组织成一个可执行、可复用的工具链。

如果只使用单个工具（比如单独用 CogPMAlignTool），当然也可以完成定位。但一旦进入实际项目，大多数检测流程都需要多个工具配合工作——这时 CogToolBlock 的价值就体现出来了。

## 一、为什么需要 CogToolBlock

假设你要做一个产品检测，流程可能是：

1. 采集图像；
2. 用 PMAlign 找到产品位置；
3. 用 FixtureTool 把坐标映射到标准位置；
4. 用 FindCircleTool 测量孔径；
5. 用 BlobTool 检查表面缺陷。

如果不用 CogToolBlock，你需要自己写代码管理这些工具的创建、执行顺序、数据传递和结果读取。

CogToolBlock 的作用就是把这些管理职责标准化：

* 工具按顺序串接；
* 前一个工具的输出自动传给下一个工具；
* 统一执行（Run）和结果访问；
* 支持保存为 .vpp 文件，脱离代码单独配置。

## 二、CogToolBlock 的基本结构

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.PMAlign;
using Cognex.VisionPro.Blob;
using Cognex.VisionPro.ToolBlock;

// 创建 CogToolBlock
CogToolBlock toolBlock = new CogToolBlock();

// 添加工具
CogPMAlignTool pmAlign = new CogPMAlignTool();
toolBlock.Tools.Add(pmAlign);

CogBlobTool blob = new CogBlobTool();
toolBlock.Tools.Add(blob);

// 设置输入图像
toolBlock.Inputs.Add("InputImage", typeof(CogImage8Grey));
toolBlock.Inputs["InputImage"].Value = image;

// 连接工具输入
// toolBlock 会自动维护工具间的依赖关系
```

## 三、运行 CogToolBlock

```csharp
// 执行工具链中的所有工具
toolBlock.Run();

// 执行完成后，按工具名访问结果
CogPMAlignTool pmAlign = toolBlock.Tools["CogPMAlignTool1"] as CogPMAlignTool;
if (pmAlign.Results != null && pmAlign.Results.Count > 0)
{
    double score = pmAlign.Results[0].Score;
    double x = pmAlign.Results[0].GetPose().TranslationX;
    double y = pmAlign.Results[0].GetPose().TranslationY;
}
```

CogToolBlock.Run() 会按工具添加的顺序和依赖关系依次执行。如果一个工具依赖前一个工具的输出，它会等待前一个工具执行完成后再执行。

## 四、工具的添加和命名

添加工具时可以指定名字，方便后续访问：

```csharp
CogPMAlignTool pmAlign = new CogPMAlignTool();
toolBlock.Tools.Add(pmAlign, "定位工具");

CogFindCircleTool findCircle = new CogFindCircleTool();
toolBlock.Tools.Add(findCircle, "测孔径");

// 通过名字访问
CogPMAlignTool locator = toolBlock.Tools["定位工具"] as CogPMAlignTool;
```

建议在实际项目中给工具起有业务含义的名字，而不是保留默认的 "CogPMAlignTool1"。这样保存成 .vpp 文件后，其他人打开时也能很快理解工具链。

## 五、输入和输出

CogToolBlock 支持明确定义输入和输出。

### 定义输入

```csharp
// 添加输入项，供外部设置
toolBlock.Inputs.Add("InputImage", typeof(CogImage8Grey));
toolBlock.Inputs.Add("ProductType", typeof(string));
```

### 定义输出

```csharp
// 设置输出项，外部可以读取结果
toolBlock.Outputs.Add("MatchScore", typeof(double));
toolBlock.Outputs.Add("PassFail", typeof(bool));

// 在工具执行后设置输出值
toolBlock.Outputs["PassFail"].Value = isOK;
```

输入输出的好处是：

* 外部调用者不需要关心工具链内部细节；
* 只需要设置输入，读取输出；
* 如果换产品型号，只需要换一个 .vpp 文件，代码不用改。

## 六、保存和加载 .vpp 文件

CogToolBlock 可以序列化成 .vpp 文件（VisionPro 的文件格式），这是实际项目中最常用的方式。

### 保存

```csharp
// 将配置好的工具链保存为 .vpp 文件
toolBlock.Save("ProductA.vpp");
```

### 加载

```csharp
// 从 .vpp 文件加载工具链
CogToolBlock toolBlock = CogSerializer.LoadObjectFromFile("ProductA.vpp") as CogToolBlock;
```

这样做的好处非常明显：

* 现场调试时，可以在 Cognex Explorer 中调整参数，然后保存 .vpp；
* 程序重启后自动加载最新配置；
* 换产品型号时，只需切换不同的 .vpp 文件。

## 七、一个实际示例

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.PMAlign;
using Cognex.VisionPro.ToolBlock;

public class VisionInspector
{
    private CogToolBlock _toolBlock;

    public VisionInspector(string vppFilePath)
    {
        _toolBlock = CogSerializer.LoadObjectFromFile(vppFilePath) as CogToolBlock;
    }

    public bool RunInspection(CogImage8Grey image)
    {
        // 设置输入图像
        _toolBlock.Inputs["InputImage"].Value = image;

        // 执行检测
        _toolBlock.Run();

        // 读取结果
        bool pass = (bool)_toolBlock.Outputs["PassFail"].Value;
        return pass;
    }

    public object GetOutput(string name)
    {
        return _toolBlock.Outputs[name].Value;
    }
}
```

这样调用者只需要关心"传图像进去，拿结果出来"，完全不用关心内部用了哪些工具。

## 八、单个工具和 CogToolBlock 的选择

### 什么时候可以直接用单个工具

* 只需要一个工具完成检测；
* 快速验证某个算法的效果；
* 临时脚本或调试场景。

### 什么时候用 CogToolBlock

* 需要多个工具串联；
* 需要保存/加载配置（.vpp）；
* 需要支持多个产品型号切换；
* 工具之间有数据依赖关系；
* 需要给操作员提供可视化配置能力。

## 九、常见注意点

### 1. 工具执行顺序

CogToolBlock 会根据工具的输入依赖自动决定执行顺序。但如果工具之间没有明确的依赖关系，它会按添加顺序执行。

### 2. 重复使用时的状态重置

每次 Run 之前，CogToolBlock 不会自动清除上次的结果。如果你需要确保结果是最新的，建议在 Run 之前确认输入图像是最新的。

### 3. .vpp 文件的版本兼容性

不同版本的 VisionPro 生成的 .vpp 文件不一定完全兼容。跨版本使用时建议先测试。

### 4. 不要在一个 Run 周期中修改工具参数

如果在工具链执行过程中修改了某个工具的配置参数，可能会导致结果不一致。建议在 Run 之前完成所有参数配置。

## 十、CogToolBlock 的性能考虑

当检测流程中有多个工具时，CogToolBlock 的执行时间是所有工具执行时间的总和。如果关注性能：

* 尽量复用已经构建好的工具链，而不是每次都重新创建；
* 如果某些工具不是每次都需要执行，可以在配置阶段按条件跳过；
* 在实时性要求高的场景下，尽量精简工具链，去掉不必要的工具。

## 总结

1. CogToolBlock 是 VisionPro 中组织多工具链的核心容器；
2. 工具按顺序串联，前一个的输出自动传递给后一个；
3. 支持 .vpp 文件保存和加载，是实现配置与代码分离的关键；
4. 明确定义输入输出接口，让调用者不关心内部工具细节；
5. 在正式项目中，大多数检测流程都适合放进 CogToolBlock 统一管理。
