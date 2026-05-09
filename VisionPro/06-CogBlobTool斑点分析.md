# CogBlobTool 斑点分析

在工业视觉检测中，很多时候你关心的不是"东西在哪里"，而是"东西表面有没有异常"。

CogBlobTool 就是专门处理这类场景的工具。

它的主要职责：

把图像中符合灰度条件的连通区域找出来，并计算它们的面积、周长、位置等特征。

## 一、BlobTool 能做什么

Blob（斑点）分析最常见的应用场景：

* **缺陷检测** —— 检测表面划痕、污点、气泡、异物；
* **面积测量** —— 测量焊点、涂层、胶水的面积；
* **目标计数** —— 统计包装内的零件数量；
* **尺寸验证** —— 通过面积判断目标是否在容忍范围内。

它的核心思想很简单：把灰度图像转成二值图像（目标区域设为白，背景设为黑），然后找到所有连通的白色区域。

## 二、基本使用流程

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.Blob;

CogBlobTool blob = new CogBlobTool();
blob.InputImage = image;

// 设置灰度阈值（将图像分割为前景和背景）
blob.RunParams.SegmentationParams.ThresholdMode = CogBlobThresholdModeConstants.Absolute;
blob.RunParams.SegmentationParams.SegmentationThresholds.Add(
    new CogBlobSegmentationThreshold(100, 255) // 灰度值 100~255 为目标区域
);

blob.Run();

// 读取结果
if (blob.Results != null)
{
    Console.WriteLine($"找到 {blob.Results.Count} 个斑点");

    foreach (CogBlobResult r in blob.Results.GetBlobs())
    {
        double area = r.Area;            // 面积（像素）
        double perimeter = r.Perimeter;  // 周长（像素）
        double centerX = r.CenterOfMassX; // 质心 X
        double centerY = r.CenterOfMassY; // 质心 Y

        Console.WriteLine($"斑点：面积{area:F1}，周长{perimeter:F1}，质心({centerX:F1}, {centerY:F1})");
    }
}
```

## 三、阈值分割 —— 最关键的参数

阈值分割决定了"哪些像素属于目标"，这是 BlobTool 最关键的步骤。

### 1. 绝对阈值

```csharp
// 灰度值在 80~255 之间的像素被视为前景
blob.RunParams.SegmentationParams.ThresholdMode = CogBlobThresholdModeConstants.Absolute;
blob.RunParams.SegmentationParams.SegmentationThresholds.Clear();
blob.RunParams.SegmentationParams.SegmentationThresholds.Add(
    new CogBlobSegmentationThreshold(80, 255)
);
```

适合光照稳定的场景。如果光照变化不大，这是最简单直接的方式。

### 2. 相对阈值

```csharp
// 使用相对阈值（基于图像统计）
blob.RunParams.SegmentationParams.ThresholdMode = CogBlobThresholdModeConstants.Relative;
blob.RunParams.SegmentationParams.RelativeThreshold = 30;
```

适合光照整体波动，但目标和背景的对比度关系相对稳定的场景。

### 3. 硬阈值和软阈值

```csharp
// 硬阈值 —— 严格的二元分割
blob.RunParams.SegmentationParams.HardThreshold = true;

// 软阈值 —— 边缘过渡更平滑
blob.RunParams.SegmentationParams.HardThreshold = false;
```

硬阈值更适合目标边缘清晰的场景，软阈值适合边缘模糊或有渐变的目标。

## 四、结果过滤

不是所有检测到的斑点都要关注。实际项目中通常会根据特征值进行过滤。

```csharp
blob.Run();

// 根据面积过滤
CogBlobResultList results = blob.Results;

List<CogBlobResult> validBlobs = new List<CogBlobResult>();

foreach (CogBlobResult r in results.GetBlobs())
{
    // 筛选面积在范围内的斑点
    if (r.Area >= 50 && r.Area <= 5000)
    {
        validBlobs.Add(r);
    }
}

// 判断：如果过滤后还有斑点，说明有缺陷
bool hasDefect = validBlobs.Count > 0;
```

常用的过滤条件：

| 参数 | 说明 | 典型值 |
|------|------|--------|
| Area | 面积（像素） | 按实际尺寸换算 |
| Perimeter | 周长（像素） | 按实际尺寸换算 |
| Length | 长度 | 适合细长缺陷 |
| Width | 宽度 | 适合细长缺陷 |
| Elongation | 伸长率（1=圆形） | 区分圆形和长条形缺陷 |
| AreaPercent | 占 ROI 百分比 | 填充率 |

## 五、极性选择

BlobTool 支持两种极性模式：

```csharp
// 亮目标（前景比背景亮）
blob.RunParams.SegmentationParams.Polarity = CogBlobPolarityConstants.LightBlobs;

// 暗目标（前景比背景暗）
blob.RunParams.SegmentationParams.Polarity = CogBlobPolarityConstants.DarkBlobs;
```

选择正确的极性很重要：

* 在亮色表面上查找暗色缺陷 → DarkBlobs；
* 在暗色表面上查找亮色异物 → LightBlobs；
* 同时查找亮和暗 → 可能需要运行两次 BlobTool。

## 六、一个实际缺陷检测示例

```csharp
public class SurfaceDefectDetector
{
    private CogBlobTool _blob = new CogBlobTool();

    public bool DetectDefects(CogImage8Grey image, out int defectCount, out double maxArea)
    {
        defectCount = 0;
        maxArea = 0;

        _blob.InputImage = image;

        // 阈值配置：查找灰度值偏暗的区域（假设表面是亮的）
        _blob.RunParams.SegmentationParams.Polarity = CogBlobPolarityConstants.DarkBlobs;
        _blob.RunParams.SegmentationParams.ThresholdMode = CogBlobThresholdModeConstants.Absolute;
        _blob.RunParams.SegmentationParams.SegmentationThresholds.Clear();
        _blob.RunParams.SegmentationParams.SegmentationThresholds.Add(
            new CogBlobSegmentationThreshold(0, 80)
        );

        _blob.Run();

        if (_blob.Results == null)
            return true; // 无斑点，OK

        foreach (CogBlobResult r in _blob.Results.GetBlobs())
        {
            if (r.Area >= 20) // 面积超过 20 像素视为缺陷
            {
                defectCount++;
                if (r.Area > maxArea)
                    maxArea = r.Area;
            }
        }

        return defectCount == 0;
    }
}
```

## 七、BlobTool 和 PMAlign 配合

在大多数场景下，BlobTool 不是单独使用的。通常是先定位，再检测缺陷：

```csharp
// 1. PMAlign 定位
pmAlign.InputImage = image;
pmAlign.Run();

if (pmAlign.Results == null || pmAlign.Results.Count == 0)
    return false; // 定位失败

// 2. Fixture 坐标对齐
fixture.Fixture = pmAlign.Results[0].GetPose();
fixture.InputImage = image;
fixture.Run();

// 3. Blob 缺陷检测（在对齐后的图像上）
blob.InputImage = fixture.OutputImage as CogImage8Grey;
blob.Run();
```

这样不管产品位置如何变化，Blob 的检测区域都能跟随到正确位置。

## 八、注意事项

### 1. 阈值的影响

阈值直接决定了哪些像素被当成前景。阈值设得太高，可能漏掉真实缺陷；设得太低，会把正常纹理误判为缺陷。

### 2. 光照均匀性

BlobTool 对光照均匀性比较敏感。光照不均匀会导致同一张图像上不同位置的阈值效果不一致。如果光照难以做到均匀，可以考虑预处理（光照校正）或使用相对阈值。

### 3. 目标和背景的对比度

BlobTool 本质依赖灰度对比度来分割。如果对比度太低（比如透明气泡在透明背景上），BlobTool 可能不适用，需要考虑其他工具或补光方案。

### 4. 结果过滤先于判定

不要直接拿 Blob 工具的原始结果做判定。一定要通过面积、伸长率等条件做二次过滤，否则轻微的图像噪声就会导致误判。

## 九、BlobTool 的限制

BlobTool 虽然很实用，但也有它的边界：

* **重叠目标** —— 如果多个目标重叠在一起，BlobTool 无法区分；
* **对比度过低** —— 目标和背景灰度值相近时效果不好；
* **复杂纹理背景** —— 背景本身的纹理会被分割成很多斑点，过滤成本很高；
* **需要形状识别** —— BlobTool 只能提供几何特征，不能识别"这是什么形状"。

在这些场景下，可以考虑深度学习工具（CogDeepLearning）或组合多个工具来处理。

## 总结

1. CogBlobTool 通过灰度阈值将图像分割为前景和背景，然后分析连通区域的特征；
2. 阈值分割是最关键的参数——它决定了哪些像素属于目标区域；
3. 结果一定要做二次过滤（面积、伸长率等），不要直接用原始结果判定；
4. BlobTool 通常跟在 PMAlign + FixtureTool 后面，保证检测区域跟随产品位置；
5. BlobTool 对光照均匀性和对比度要求较高，局限性在于无法处理重叠目标和复杂纹理。
