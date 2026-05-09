# CogCalibTool 校准工具

VisionPro 中的视觉工具默认是在像素坐标系中工作的。

但实际项目中，你需要的是物理单位——毫米、厘米、英寸。你需要知道"这个孔的直径是 10.5mm"，而不是"这个孔的直径是 200 像素"。

CogCalibTool 的作用就是：

建立像素坐标和物理坐标之间的映射关系。

## 一、什么时候需要校准

以下场景通常需要校准：

* **需要输出物理单位** —— 测量结果以毫米为单位报告；
* **镜头有畸变** —— 广角镜头或低畸变镜头不够理想时；
* **相机和被测面不平行** —— 安装角度导致透视变形；
* **多相机测量** —— 多个相机的测量结果需要统一到同一个物理坐标系。

如果只是做简单的有无检测、计数，或者产品尺寸非常宽松，不一定需要严格校准。

## 二、校准的基本流程

VisionPro 中最常用的是 CogCalibCheckerboardTool（棋盘格校准）。

### 1. 准备校准板

拍摄一张棋盘格或圆点阵校准板的图像。校准板需要有已知的物理尺寸。

### 2. 执行校准

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.Calib;

CogCalibCheckerboardTool calib = new CogCalibCheckerboardTool();
calib.InputImage = calibrationImage;

// 设置校准板参数
calib.CalibrationPlate = CogCalibCheckerboardPlateConstants.Checkerboard;
calib.CalibrationPlateSpacingX = 2.5;   // 棋盘格间距 X（毫米）
calib.CalibrationPlateSpacingY = 2.5;   // 棋盘格间距 Y（毫米）
calib.CalibrationPlateNumCornersX = 7;   // 角点数 X
calib.CalibrationPlateNumCornersY = 5;   // 角点数 Y

// 运行校准
calib.Run();

if (calib.Calibrated)
{
    Console.WriteLine("校准成功");
}
else
{
    Console.WriteLine("校准失败");
}
```

### 3. 获取校准结果

```csharp
// 校准后的坐标变换对象
CogCoordinateTransform transform = calib.CoordinateTransform;

// 像素转物理
double mmX, mmY;
transform.PixelToWorld(200, 150, out mmX, out mmY);
Console.WriteLine($"像素(200, 150) → 物理({mmX:F3}, {mmY:F3})mm");

// 物理转像素
double pixelX, pixelY;
transform.WorldToPixel(10, 5, out pixelX, out pixelY);
```

## 三、校准参数说明

### 1. 校准板类型

```csharp
// 棋盘格
calib.CalibrationPlate = CogCalibCheckerboardPlateConstants.Checkerboard;

// 圆点阵
calib.CalibrationPlate = CogCalibCheckerboardPlateConstants.DotGrid;
```

棋盘格更常用，角点检测更稳定。圆点阵在某些特定场景中也有优势。

### 2. 间隔和数量

```csharp
calib.CalibrationPlateSpacingX = 2.5;  // 棋盘格之间的实际间距 X（mm）
calib.CalibrationPlateSpacingY = 2.5;  // 棋盘格之间的实际间距 Y（mm）
calib.CalibrationPlateNumCornersX = 7; // X 方向的角点数量
calib.CalibrationPlateNumCornersY = 5; // Y 方向的角点数量
```

这四个参数必须和实际使用的校准板完全一致，否则校准结果会出错。

### 3. 校准模式

```csharp
// 简单线性校准（无畸变校正）
calib.CalibrationType = CogCalibCheckerboardCalibrationTypeConstants.Linear;

// 非线性校准（包含畸变校正）
calib.CalibrationType = CogCalibCheckerboardCalibrationTypeConstants.NonLinear;
```

如果镜头畸变较小，用线性校准就够。如果镜头畸变明显，需要用非线性校准。

## 四、校准结果应用到检测流程

校准后的 CoordinateTransform 可以应用到 FixtureTool，让检测结果直接输出物理单位。

```csharp
// 1. 校准（可以提前完成，保存结果）
calib.Run();
CogCoordinateTransform transform = calib.CoordinateTransform;
// 保存校准结果供后续使用
CogSerializer.SaveObjectToFile(transform, "calibration.vs");

// 2. 在检测时加载校准结果并应用到 Fixture
CogCoordinateTransform loadedTransform =
    CogSerializer.LoadObjectFromFile("calibration.vs") as CogCoordinateTransform;

fixture.InputImage = image;
fixture.Fixture = pmAlign.Results[0].GetPose();
fixture.CoordinateTransform = loadedTransform; // 叠加校准
fixture.Run();

// 3. 后续工具的测量结果会自动转换为物理单位
findCircle.InputImage = fixture.OutputImage as CogImage8Grey;
findCircle.Run();

// 此时读取的结果已经是物理单位（毫米）
double radiusMm = findCircle.Results.CircleRadius;
Console.WriteLine($"孔径：{radiusMm:F3}mm");
```

## 五、校准精度验证

校准完成后，建议做精度验证：

```csharp
// 验证校准精度：用已知距离的两个特征点来验证
double knownDistance = 10.0; // 已知物理距离（mm）

// 测量两个特征点在物理坐标系下的距离
double measuredDistance = Math.Sqrt(
    Math.Pow(mmX2 - mmX1, 2) + Math.Pow(mmY2 - mmY1, 2)
);

double error = Math.Abs(measuredDistance - knownDistance);
Console.WriteLine($"校准误差：{error:F4}mm");

// 如果误差超过可接受范围，需要重新校准或检查校准板参数
```

## 六、校准的常见问题

### 1. 校准失败

最常见的原因：

* 校准板图像质量不好（模糊、光照不均、反光）；
* 校准板参数设置错误（间距、角点数量不对）；
* 校准板没有充满视野，或者只拍到部分校准板；
* 校准板倾斜角度过大。

### 2. 校准精度不达标

可能的原因：

* 校准板本身精度不够（建议使用 Ceramic 或玻璃校准板）；
* 校准板图像数量太少（非线性校准需要多个角度和位置的图像）；
* 镜头固定不牢，在校准过程中发生了位移。

### 3. 什么时候需要重新校准

* 相机位置发生了变动；
* 镜头焦距或光圈被调整；
* 更换了相机或镜头；
* 温度和机械结构发生了明显变化。

## 七、校准和 Fixture 的关系

校准和 Fixture 经常一起使用，但职责不同：

```csharp
// Fixture：将检测区域映射到产品实际位置
fixture.Fixture = pmAlign.Results[0].GetPose();

// Calibration：像素单位 ↔ 物理单位的转换
fixture.CoordinateTransform = calibTransform;

// 两者可以叠加使用
fixture.Run();
```

更直白的理解：

* Fixture = "跟着产品走"；
* Calibration = "把像素变成毫米"。

两者重叠使用时，检测链通常是：

PMAlign（定位） → Fixture（坐标对齐） + Calibration（像素→毫米） → 其他检测工具

## 八、一个实用建议

对于大多数视觉检测项目，建议这样做：

1. **安装调试阶段** 完成校准，保存校准文件和变换参数；
2. **日常生产阶段** 直接加载校准文件，不需要反复校准；
3. **定期验证** 用标准件验证校准精度是否有漂移；
4. **维护记录** 记录每次校准的误差，提前发现机械或相机松动。

## 总结

1. CogCalibTool 建立像素坐标到物理坐标的映射关系，让检测结果以毫米为单位输出；
2. 棋盘格校准是最常用、最稳定的方式，关键参数是校准板的间距和角点数量；
3. 校准结果可以叠加到 FixtureTool 上，让工具链同时支持位置跟随和单位转换；
4. 校准精度取决于校准板质量、图像质量和参数设置的准确性；
5. 校一般在安装调试阶段完成，日常生产时直接加载校准文件，不需要反复执行。
