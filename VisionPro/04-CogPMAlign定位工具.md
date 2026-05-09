# CogPMAlign 定位工具

在 VisionPro 中，CogPMAlign 是最常用、最核心的定位工具。

它的主要职责只有一个：

在图像中找到已知图案（模板）的位置。

如果用一个更直观的方式理解：你先给系统看一张"标准样品"的图像，告诉它"这就是你要找的东西"，然后每次运行时，它就去当前的图像里查找和这个模板最匹配的位置。

## 一、PMAlign 的基本原理

PMAlign 使用的是基于灰度值的模板匹配算法。它和简单的像素比对不同：

* 它可以容忍光照变化；
* 支持旋转和缩放；
* 能够给出匹配分数（0~1），方便设定阈值；
* 可以同时找到多个匹配目标。

## 二、基本使用流程

PMAlign 的使用分为两个阶段：训练阶段和运行阶段。

### 1. 训练模板

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.PMAlign;

// 从图像加载或采集
CogImage8Grey image = LoadImage();

// 创建 PMAlign 工具
CogPMAlignTool pmAlign = new CogPMAlignTool();
pmAlign.InputImage = image;

// 设置训练区域（模板区域）
CogRectangleAffine trainRegion = new CogRectangleAffine();
trainRegion.CenterX = 200;
trainRegion.CenterY = 150;
trainRegion.SideX = 100;
trainRegion.SideY = 80;
trainRegion.Rotation = 0;

pmAlign.Pattern.TrainRegion = trainRegion;

// 训练模板
pmAlign.Pattern.Train();

// 保存模板（可选，供后续加载）
pmAlign.Pattern.Save("template.pat");
```

这里有几个关键点：

* 训练区域应该只包含目标特征本身，尽量排除背景噪声；
* 区域太小可能特征不足，区域太大可能包含干扰信息；
* 如果是圆形对称工件，可以适当缩小角度搜索范围。

### 2. 运行时匹配

```csharp
// 设置运行时的输入图像
pmAlign.InputImage = currentImage;

// 执行匹配
pmAlign.Run();

// 读取结果
if (pmAlign.Results != null && pmAlign.Results.Count > 0)
{
    CogPMAlignResult result = pmAlign.Results[0];

    double score = result.Score;              // 匹配分数
    double x = result.GetPose().TranslationX; // 目标位置 X
    double y = result.GetPose().TranslationY; // 目标位置 Y
    double angle = result.GetPose().Rotation; // 目标旋转角度

    Console.WriteLine($"匹配成功：位置({x:F2}, {y:F2})，角度{angle:F2}°，分数{score:F3}");
}
else
{
    Console.WriteLine("匹配失败，未找到目标");
}
```

## 三、常用参数配置

PMAlign 的参数配置会影响匹配的速度和精度。

### 1. 搜索区域

```csharp
// 设置搜索区域（可选，不设置则在全图搜索）
CogRectangle searchRegion = new CogRectangle();
searchRegion.CenterX = 300;
searchRegion.CenterY = 250;
searchRegion.SideX = 400;
searchRegion.SideY = 300;

pmAlign.SearchRegion = searchRegion;
```

如果知道目标的大概位置范围，设置搜索区域可以显著提高速度。

### 2. 角度范围

```csharp
// 限制搜索角度范围（弧度）
pmAlign.Pattern.AcceptAngle = true;
pmAlign.Pattern.AngleStart = -CogMath.PI_OVER_4;
pmAlign.Pattern.AngleEnd = CogMath.PI_OVER_4;
```

### 3. 匹配数量

```csharp
// 设置最大匹配数量
pmAlign.RunParams.NumToFind = 1;  // 只找最好的一个
// pmAlign.RunParams.NumToFind = 0; // 找所有匹配
```

### 4. 匹配分数阈值

```csharp
// 设置分数阈值
pmAlign.RunParams.AcceptThreshold = 0.7;  // 低于 0.7 的匹配结果被忽略
```

## 四、结果解读

PMAlign 的结果中，最重要的几个值：

| 结果项 | 说明 | 典型用途 |
|--------|------|----------|
| Score | 匹配分数（0~1） | 判定是否找到目标 |
| TranslationX/Y | 目标中心在图像中的位置 | 传给 FixtureTool 做坐标映射 |
| Rotation | 目标相对于模板的旋转角度 | 判断产品摆放角度 |
| Scale | 目标相对于模板的缩放比例 | 判断产品尺寸是否异常 |

## 五、PMAlign 和 FixtureTool 的配合

这是实际项目中最常见的组合模式。

```csharp
// 定位
CogPMAlignTool pmAlign = new CogPMAlignTool();
pmAlign.InputImage = image;
pmAlign.Pattern.Train();
pmAlign.Run();

// 创建 FixtureTool，将定位结果转为坐标空间
CogFixtureTool fixture = new CogFixtureTool();
fixture.InputImage = image;

if (pmAlign.Results != null && pmAlign.Results.Count > 0)
{
    // 将定位结果设为 Fixture 的参考坐标系
    fixture.Fixture = pmAlign.Results[0].GetPose();

    // FixtureTool 运行后，后续工具的坐标系就自动对齐了
    fixture.Run();

    // 后续工具可以直接使用 Fixture 的输出图像
    CogImage8Grey alignedImage = fixture.OutputImage as CogImage8Grey;
}
```

这种模式的优点：

* 即使产品位置有偏移，后续工具也不需要调整检测区域；
* 换产品型号时，只需要更新模板和检测参数；
* 可读性好，工具链的职责清晰。

## 六、多个模板切换

在实际项目中，经常需要在不同产品型号之间切换。

```csharp
public class MultiProductLocator
{
    private CogPMAlignTool _pmAlign = new CogPMAlignTool();
    private Dictionary<string, string> _templateFiles;

    public void LoadTemplate(string productType)
    {
        if (_templateFiles.ContainsKey(productType))
        {
            _pmAlign.Pattern.Load(_templateFiles[productType]);
        }
    }

    public bool Locate(CogImage8Grey image)
    {
        _pmAlign.InputImage = image;
        _pmAlign.Run();
        return _pmAlign.Results != null && _pmAlign.Results.Count > 0;
    }
}
```

## 七、注意事项

### 1. 训练和运行时使用相同类型的图像

如果训练时用的是 8bit 灰度图，运行时也应该用相同类型。图像类型不一致会影响匹配效果。

### 2. 匹配分数不是越高越好

0.9 以上的分数当然很好，但有时分数 0.7 就已经足够稳定了。关键是看实际检测的稳定性，而不是一味追求高分。

### 3. 使用前确认模板是否已训练

```csharp
if (!pmAlign.Pattern.Trained)
{
    pmAlign.Pattern.Train();
}
```

### 4. 模板更新时机

如果产线上的产品批次有微小变化，可能需要在新的样本上重新训练模板。但不要在生产运行时频繁更新模板，除非你明确知道原因。

## 八、PMAlign 常见问题排查

### 匹配分数低

* 光照是否变化较大；
* 目标是否有遮挡或外观变化；
* 模板区域是否包含过多背景；
* 角度范围是否设置过于严格。

### 匹配速度慢

* 搜索区域是否设置得太大；
* 图像分辨率是否过高；
* 角度范围和精度要求是否偏高。

### 误匹配

* 模板特征是否不够独特；
* 是否有类似图案的干扰物；
* 匹配分数阈值是否设得太低。

## 九、和深度学习定位的对比

VisionPro 也提供了基于深度学习的定位工具（CogDeepLearning），但在大多数常规工业场景中：

* PMAlign 已经能胜任 90% 以上的定位需求；
* 深度学习更适合外观变化大、背景复杂的场景；
* PMAlign 的部署和调试比深度学习简单得多。

## 总结

1. CogPMAlign 是 VisionPro 中最基础的定位工具，通过模板匹配找到目标位置；
2. 使用分训练和运行两个阶段，训练好的模板可以保存成文件重复使用；
3. 匹配结果包括位置、旋转角度和分数，可以直接用于 Fixture 坐标变换；
4. 和 FixtureTool 组合是"一次定位，后续测量工具不用再调整位置"的通用模式；
5. 在正式项目中，PMAlign 通常是整个检测流程的第一步，它的稳定性直接影响后续所有工具的效果。
