# CogFixtureTool 坐标空间

在工业视觉项目中，有一个非常常见但容易被忽视的问题：

产品在产线上不会每次都停在完全一样的位置。

如果产品位置有偏移、旋转，而你仍然在固定坐标位置检测，那结果肯定不准确。

CogFixtureTool 就是专门解决这个问题的。

它的主要职责：

根据定位结果建立一个标准化的坐标空间，让后续所有检测工具都在这个空间里工作，而不关心产品实际位置在哪里。

## 一、为什么需要坐标空间

先看一个反面示例：

```csharp
// 错误做法：在固定位置检测
CogFindCircleTool findCircle = new CogFindCircleTool();

// 搜索区域是固定的图像坐标
CogRectangle region = new CogRectangle();
region.SetCenterWidthHeight(300, 250, 100, 100);
findCircle.RunParams.SearchRegion = region;

// 如果产品偏移了 50 像素，这里就测不准了
findCircle.Run();
```

正确的思路是：

1. 先用 PMAlign 找到产品位置；
2. 用 FixtureTool 把"标准坐标空间"映射到实际图像位置；
3. 后续工具在"标准坐标空间"里配置检测区域，FixturTool 自动把它们转换到实际图像位置。

## 二、基本用法

```csharp
using Cognex.VisionPro;

CogFixtureTool fixture = new CogFixtureTool();

// 设置输入图像
fixture.InputImage = image;

// 设置参考坐标系（通常来自 PMAlign 的定位结果）
if (pmAlign.Results != null && pmAlign.Results.Count > 0)
{
    // PMAlign 返回的姿态包含了位置和旋转
    fixture.Fixture = pmAlign.Results[0].GetPose();
}

// 执行坐标变换
fixture.Run();

// 获取变换后的图像
CogImage8Grey alignedImage = fixture.OutputImage as CogImage8Grey;
```

## 三、FixturTool 做了什么

FixturTool 本质上做的事情是：

1. 收到一个"参考坐标系"（来自 PMAlign 的定位结果）；
2. 将图像内容和这个坐标系对齐；
3. 输出一张"变换后的图像"，在这张图像上，目标被放在了标准位置。

所以：

* 给 FixtureTool 的图像是原始图像（产品可能是歪的）；
* 从 FixtureTool 拿到的图像是"摆正"的图像（产品在标准位置）。

后续工具使用"摆正"后的图像时，检测区域可以设置在标准坐标位置，不用关心产品实际偏移。

## 四、在多工具链中的典型用法

```csharp
// 工具链示例：定位 → 坐标对齐 → 多个测量工具

// 1. PMAlign 定位
pmAlign.InputImage = originalImage;
pmAlign.Run();

if (pmAlign.Results == null || pmAlign.Results.Count == 0)
{
    // 定位失败，不继续检测
    return;
}

// 2. FixturTool 坐标变换
fixture.InputImage = originalImage;
fixture.Fixture = pmAlign.Results[0].GetPose();
fixture.Run();

CogImage8Grey fixturedImage = fixture.OutputImage as CogImage8Grey;

// 3. 后续工具使用 fixturedImage
// 不管产品在原始图像中是什么位置和角度
// 下面的工具都在"摆正"后的图像上检测

// 测量孔径（搜索区域在标准位置）
findCircle.InputImage = fixturedImage;
// 搜索区域设置在 (300, 250)，因为图像已经被 Fixture 摆正了
findCircle.Run();

// 测量另一个特征（也在标准位置）
findCircle2.InputImage = fixturedImage;
findCircle2.Run();
```

## 五、嵌套 Fixture

在一些复杂的场景中，可能会用到多个 FixtureTool。

例如：先对全局定位，再对局部特征定位，嵌套使用。

```csharp
// 第一层 Fixture：全局定位
fixtureGlobal.InputImage = originalImage;
fixtureGlobal.Fixture = pmAlignGlobal.Results[0].GetPose();
fixtureGlobal.Run();

// 局部定位：在全局摆正后的图像上找局部特征
pmAlignLocal.InputImage = fixtureGlobal.OutputImage as CogImage8Grey;
pmAlignLocal.Run();

// 第二层 Fixture：基于局部特征进一步对齐
fixtureLocal.InputImage = fixtureGlobal.OutputImage as CogImage8Grey;
fixtureLocal.Fixture = pmAlignLocal.Results[0].GetPose();
fixtureLocal.Run();

// 最终对齐后的图像
CogImage8Grey finalAligned = fixtureLocal.OutputImage as CogImage8Grey;
```

但在实际项目中，嵌套 Fixture 会增加复杂性和计算量。大多数场景只用一个 FixtureTool 就够了。

## 六、Fixture 的不同来源

FixtureTool 的坐标参考不一定非得来自 PMAlign。

```csharp
// 手动创建坐标系（适合产品位置固定，只需要旋转校正）
CogTransform2D transform = new CogTransform2D();
transform.TranslationX = 0;
transform.TranslationY = 0;
transform.Rotation = 0.05; // 轻微旋转校正

fixture.Fixture = new CogFixture(transform);

// 来自校准结果
fixture.Fixture = calibTool.CoordinateTransform;

// 来自多个定位结果的平均
fixture.Fixture = averagePose;
```

## 七、FixtureTool 的性能影响

FixturTool 会对整张图像做坐标变换，因此：

* **图像越大，变换越耗时** —— 如果图像分辨率很高，FixturTool 的耗时会很明显；
* **只对检测区域做变换** —— 如果后续工具只使用了较小的检测区域，可以考虑只对 ROI 做变换；
* **权衡精度和速度** —— 如果产品位置变化很小，不一定每次都要做 Fixture。

## 八、何时可以省略 FixtureTool

FixturTool 不是所有场景都需要。以下情况可以省略：

* **产品位置完全固定** —— 机械定位足够精确，不需要每次调整；
* **检测只在相对坐标内完成** —— 某些测量工具支持动态搜索区域；
* **检测项不依赖绝对位置** —— 比如只做全局是否存在判定。

但一旦发现"产品稍微偏一点就不准了"，优先考虑加 FixtureTool。

## 九、常见问题

### 1. FixturTool 的输出图像看起来变形了

这是正常的。因为 FixtureTool 要做旋转和仿射变换，输出图像可能包含旋转后的内容。后续工具关心的是"变换后的坐标"，不是"看起来有没有歪"。

### 2. FixturTool 之后找不到目标

检查 FixturTool 的输入：如果 PMAlign 定位结果偏差很大，那 FixturTool 会把这个偏差带进输出图像，导致后续测量偏得更远。

### 3. 定位不准

Fixture 的精度上限取决于 PMAlign 的定位精度。定位本身不准，再好的 Fixture 也无济于事。

### 4. 变换后图像有黑边

由于旋转和变换，输出图像的边缘区域可能会出现空白。如果后续工具的检测区域靠近边缘，要注意避开这些区域。

## 十、关于 Fixture 和 Calibration 的区别

很多初学者容易把 Fixture 和 Calibration 搞混。它们解决的是两个不同的问题：

| 工具 | 解决什么问题 |
|------|------------|
| CogFixtureTool | 产品位置偏移后，把检测区域映射到实际位置 |
| CogCalibTool | 把像素坐标映射到物理单位（毫米） |

简单理解：

* Fixture = "目标跑到哪里了，我要跟着它";
* Calibration = "像素距离等于多少毫米"。

两者经常一起用，但职责不同。

## 总结

1. CogFixtureTool 负责将"标准坐标空间"映射到实际图像位置，解决产品位置偏移的问题；
2. FixtureTool 通常接收 PMAlign 的定位结果作为输入，输出"摆正"后的图像；
3. 后续工具在摆正后的图像上配置检测区域，不需要关心产品实际位置；
4. 大多数场景一个 FixtureTool 就够了，嵌套使用会增加复杂性；
5. Fixture 解决的是"位置跟随"问题，和 Calibration（像素↔毫米）是两回事。
