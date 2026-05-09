# CogFindCircle 与边缘检测

在工业视觉检测中，边缘检测是最基础也最常用的测量手段之一。

VisionPro 提供了多个边缘检测工具，其中 CogFindCircleTool 是最有代表性的一个。

它的主要职责：

在图像中查找圆弧或圆的边缘，并拟合出几何参数（中心、半径、直线等）。

## 一、边缘检测的基本概念

在 VisionPro 中理解边缘检测，需要先了解三个核心概念：

1. **卡尺（Caliper）** —— 边缘检测的最小单元，每个卡尺沿着扫描线寻找边缘点；
2. **搜索区域（SearchRegion）** —— 告诉工具"去哪里找边缘"；
3. **预期几何（ExpectedGeometry）** —— 告诉工具"你期望找到什么形状"，帮助排除干扰。

边缘检测工具不是在全图中搜索，而是在你指定的区域和预期形状附近查找。

## 二、CogFindCircleTool 的使用

### 1. 基本流程

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.Caliper;

CogFindCircleTool findCircle = new CogFindCircleTool();
findCircle.InputImage = image;

// 设置搜索区域（包含预期圆弧的矩形区域）
CogRectangle searchRegion = new CogRectangle();
searchRegion.SetCenterWidthHeight(300, 250, 200, 150);
findCircle.RunParams.SearchRegion = searchRegion;

// 设置预期圆弧
CogCircle expectedArc = new CogCircle();
expectedArc.CenterX = 300;
expectedArc.CenterY = 250;
expectedArc.Radius = 80;
findCircle.RunParams.ExpectedArc = expectedArc;

// 设置卡尺参数
findCircle.RunParams.CaliperSearchLength = 20;    // 搜索长度
findCircle.RunParams.CaliperProjectionLength = 5; // 投影长度
findCircle.RunParams.NumCalipers = 20;             // 卡尺数量
findCircle.RunParams.CaliperRunParams.ContrastThreshold = 20; // 对比度阈值

// 执行检测
findCircle.Run();

// 读取结果
CogFindCircleResult result = findCircle.Results;
if (result != null)
{
    double radius = result.CircleRadius;
    double centerX = result.CircleCenterX;
    double centerY = result.CircleCenterY;
    double score = result.Score;

    Console.WriteLine($"圆检测：中心({centerX:F2}, {centerY:F2})，半径{radius:F2}，分数{score:F3}");
}
```

### 2. 卡尺参数的理解

卡尺是边缘检测的核心配置项。几个关键参数：

| 参数 | 说明 | 典型值 |
|------|------|--------|
| CaliperSearchLength | 每个卡尺的搜索范围宽度 | 20~50 |
| CaliperProjectionLength | 每个卡尺的投影长度（边缘平滑） | 5~15 |
| NumCalipers | 卡尺数量 | 15~30 |
| ContrastThreshold | 边缘对比度阈值 | 10~30 |
| EdgePosition | 边缘极性选择（亮到暗/暗到亮/任意） | 按实际设置 |

卡尺数量越多，边缘点越多，拟合越稳定，但同时需要的计算量也越大。

## 三、CogFindLineTool —— 直线边缘检测

找直线的使用方式跟找圆非常相似：

```csharp
CogFindLineTool findLine = new CogFindLineTool();
findLine.InputImage = image;

// 设置搜索区域
findLine.RunParams.SearchRegion = searchRegion;

// 设置预期直线
CogLine line = new CogLine();
line.Rotation = CogMath.PI_OVER_2;  // 垂直方向
line.X = 200;
line.Y = 100;
findLine.RunParams.ExpectedLine = line;

// 卡尺配置
findLine.RunParams.NumCalipers = 15;
findLine.RunParams.CaliperSearchLength = 30;

findLine.Run();

CogFindLineResult result = findLine.Results;
if (result != null)
{
    double angle = result.Line.Rotation;  // 直线角度
    double distance = result.Line.DistanceFromOrigin; // 到原点的距离
}
```

## 四、边缘极性（Edge Polarity）

边缘极性描述了像素值的变化方向。在 VisionPro 中有三种选择：

1. **BrightToDark**：从亮到暗（正边缘）
2. **DarkToBright**：从暗到亮（负边缘）
3. **Any**：任意极性

```csharp
// 设置边缘极性
findCircle.RunParams.CaliperRunParams.EdgePosition = CogCaliperEdgePositionConstants.BrightToDark;
```

示例场景：
* 检测黑色背景上的亮色圆环 → 使用 BrightToDark
* 检测亮色背景上的暗色孔洞 → 使用 DarkToBright
* 不确定变化方向 → 使用 Any（但可能引入更多杂讯）

## 五、实际测量场景

### 场景：测量孔径

```csharp
public class HoleMeasurement
{
    private CogFindCircleTool _findCircle = new CogFindCircleTool();

    public void Setup(CogImage8Grey image, double expectedX, double expectedY, double expectedRadius)
    {
        _findCircle.InputImage = image;

        // 搜索区域包裹整个圆
        CogRectangle region = new CogRectangle();
        region.SetCenterWidthHeight(expectedX, expectedY, expectedRadius * 2.5, expectedRadius * 2.5);
        _findCircle.RunParams.SearchRegion = region;

        // 预期圆
        CogCircle expected = new CogCircle();
        expected.CenterX = expectedX;
        expected.CenterY = expectedY;
        expected.Radius = expectedRadius;
        _findCircle.RunParams.ExpectedArc = expected;

        _findCircle.RunParams.NumCalipers = 24;
        _findCircle.RunParams.CaliperSearchLength = 30;
        _findCircle.RunParams.CaliperRunParams.ContrastThreshold = 15;
    }

    public double Measure()
    {
        _findCircle.Run();
        if (_findCircle.Results != null)
        {
            return _findCircle.Results.CircleRadius;
        }
        return double.NaN;
    }
}
```

### 场景：测量边缘距离

```csharp
// 找两条直线，计算它们的距离
CogFindLineTool line1 = new CogFindLineTool();
CogFindLineTool line2 = new CogFindLineTool();

// 配置两条直线...
// 执行...

if (line1.Results != null && line2.Results != null)
{
    // 计算点到直线的距离
    CogLine l1 = line1.Results.Line;
    CogLine l2 = line2.Results.Line;

    // 如果两直线平行，计算间距
    double distance = Math.Abs(l1.A * l2.DistanceFromOrigin);
}
```

## 六、边缘检测和 FixtureTool 的配合

当产品位置可能变化时，边缘检测通常跟在 PMAlign + FixtureTool 后面：

```csharp
// 1. 定位
pmAlign.Run();

// 2. 坐标对齐
fixture.Fixture = pmAlign.Results[0].GetPose();
fixture.Run();

// 3. 边缘检测（使用对齐后的图像）
findCircle.InputImage = fixture.OutputImage as CogImage8Grey;
findCircle.Run();
```

这样，即使产品位置有偏移，找圆的搜索区域始终在标准位置，不需要动态调整。

## 七、常见问题排查

### 边缘点找得不够多

* 降低对比度阈值；
* 增加卡尺数量和搜索长度；
* 检查光照条件是否合适。

### 边缘点杂散不集中

* 提高对比度阈值；
* 调整边缘极性（只保留正确的方向）；
* 增加投影长度，平滑噪声。

### 拟合结果不稳定

* 增加卡尺数量；
* 检查搜索区域是否太大，包含了干扰边缘；
* 确保预期几何和真实边缘差距不要太大。

### 圆是找出来了，但位置明显偏了

* 检查是否有更强的边缘在干扰拟合；
* 考虑使用粗过滤（去掉偏差过大的边缘点）。

## 八、综合提示

边缘检测看起来很直接，但实际稳定性往往取决于：

1. **光照** —— 稳定均匀的光照比任何参数调优都重要；
2. **搜索区域** —— 尽量缩小搜索范围，减少干扰边缘进入；
3. **预期几何** —— 预期值越贴近真实情况，结果越稳定；
4. **卡尺参数** —— 优先调节对比度阈值和搜索长度，其他保持默认通常已经够用。

## 总结

1. CogFindCircleTool 和 CogFindLineTool 是 VisionPro 边缘检测的核心工具，通过卡尺扫描找到边缘点，再拟合成几何图形；
2. 卡尺参数（数量、搜索长度、对比度阈值）是影响检测效果的关键配置；
3. 边缘极性选择要注意：选对极性可以减少大量干扰；
4. 边缘检测通常跟在 PMAlign + FixtureTool 之后，让搜索区域固定在标准位置；
5. 稳定的光照往往比完美的参数配置对检测结果的影响更大。
