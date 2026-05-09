# CogAcqFifoTool 图像采集

在工业视觉项目中，第一件事通常是：

把相机的图像拿过来。

CogAcqFifoTool 是 VisionPro 中负责图像采集的核心工具。它封装了相机连接、触发、采集、缓冲等工业场景中常见但容易出错的环节。

## 一、AcqFifoTool 的基本职责

* 连接相机（GigE Vision、USB3 Vision、Camera Link 等）；
* 配置采集模式（连续采集、硬件触发、软件触发）；
* 管理图像缓冲区；
* 处理采集超时和丢帧；
* 将采集到的图像输出给后续工具。

## 二、基本使用流程

### 1. 初始化相机

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.Acquisition;

CogAcqFifoTool acqTool = new CogAcqFifoTool();
acqTool.Operator = new CogAcqFifoOperator();

// 初始化相机（使用相机名称或 IP 地址）
acqTool.Operator.Init("Cognex GigE Vision Camera");

// 或者使用相机序列号
// acqTool.Operator.Init("SN: 12345678");
```

### 2. 配置触发模式

```csharp
// 自由运行模式（连续采集）
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.FreeRun;

// 软件触发模式（调用 Acquire 时触发）
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.Software;

// 硬件触发模式（外部信号触发）
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.Hardware;
```

### 3. 采集图像

```csharp
// 配置超时
acqTool.Operator.OwnedTimeoutEnabled = true;
acqTool.Operator.OwnedTimeout_ms = 5000; // 5 秒超时

// 执行采集
acqTool.Run();

// 获取图像
CogImage8Grey image = acqTool.OutputImage as CogImage8Grey;

if (image != null)
{
    // 图像可用于后续检测
}
```

## 三、采集模式选择

### 1. 自由运行模式（FreeRun）

```csharp
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.FreeRun;
```

相机以固定帧率持续采集。适合：
* 调试和开发阶段；
* 需要实时显示视频流的场景；
* 不需要外部同步的场景。

### 2. 软件触发

```csharp
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.Software;

// 调用 Run() 时触发一次采集
acqTool.Run();
```

每次调用 Run() 时相机采集一帧。适合：
* 由程序逻辑控制采集时机；
* 不需要外部硬件信号；
* 需要精确控制每帧图像处理完成后再采集下一帧。

### 3. 硬件触发

```csharp
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.Hardware;

// 配置触发信号参数（可选）
acqTool.Operator.Trigger.TriggerEdge = CogAcqTriggerEdgeConstants.Rising; // 上升沿触发
```

由外部硬件信号控制采集时机。适合：
* 产线自动化场景，需要和 PLC 配合；
* 产品到达检测位置时触发拍照；
* 需要精确的帧同步。

## 四、相机参数配置

不同的相机支持的参数不同，但 VisionPro 提供了统一的方式配置：

```csharp
// 设置曝光时间（微秒）
acqTool.Operator.Exposure = 5000; // 5ms

// 设置增益
acqTool.Operator.Gain = 1.0;

// 设置帧率（自由运行模式）
acqTool.Operator.AcquisitionFPS = 30; // 30 fps

// 获取相机支持的分辨率
int width = acqTool.Operator.Width;
int height = acqTool.Operator.Height;
```

如果需要访问相机特有的参数：

```csharp
// 通过特征名设置
acqTool.Operator.SetFeatureValue("GainAuto", "Continuous");
acqTool.Operator.SetFeatureValue("BalanceWhiteAuto", "Continuous");
```

## 五、多相机采集

```csharp
// 两个相机，两个采集工具
CogAcqFifoTool camera1 = new CogAcqFifoTool();
CogAcqFifoTool camera2 = new CogAcqFifoTool();

camera1.Operator.Init("Camera1");
camera2.Operator.Init("Camera2");

// 并行触发
camera1.Run();
camera2.Run();

// 获取图像
CogImage8Grey image1 = camera1.OutputImage as CogImage8Grey;
CogImage8Grey image2 = camera2.OutputImage as CogImage8Grey;
```

注意多相机采集时，多个相机的采集过程可能会消耗系统带宽（特别是 GigE 相机），需要合理配置带宽和缓冲区。

## 六、采集和检测的异步模式

在连续检测场景中，建议将采集和检测分离在不同的线程或流程中：

```csharp
// 采集线程：持续采集图像并放入队列
// 检测线程：从队列取出图像进行检测

// 这样可以做到：采集不等待检测完成，检测不阻塞下一次采集

// 简化的异步采集循环
public void AcquisitionLoop()
{
    while (_isRunning)
    {
        acqTool.Run();
        CogImage8Grey image = acqTool.OutputImage as CogImage8Grey;
        if (image != null)
        {
            _imageQueue.Enqueue(image); // 放入队列
        }
    }
}

// 检测线程
public void InspectionLoop()
{
    while (_isRunning)
    {
        if (_imageQueue.TryDequeue(out CogImage8Grey image))
        {
            bool result = _inspector.RunInspection(image);
            // 处理检测结果
        }
    }
}
```

## 七、错误处理

工业相机经常因为环境因素（线缆松动、光照变化、温度等）出现异常。生产中应该有完备的错误处理：

```csharp
try
{
    acqTool.Run();
    CogImage8Grey image = acqTool.OutputImage as CogImage8Grey;
    if (image == null)
    {
        // 采集成功但图像为空
        _logger.Warn("采集成功但图像为空");
    }
}
catch (CogAcqException ex)
{
    // 采集失败
    _logger.Error($"采集异常：{ex.Message}");

    // 尝试重新初始化相机
    try
    {
        acqTool.Operator.Shutdown();
        Thread.Sleep(100);
        acqTool.Operator.Init("CameraName");
    }
    catch (Exception reinitEx)
    {
        _logger.Error($"重新初始化相机失败：{reinitEx.Message}");
    }
}
```

## 八、常用场景配置

### 场景 1：产线在线检测（硬件触发）

```csharp
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.Hardware;
acqTool.Operator.Exposure = 3000; // 3ms
acqTool.Operator.OwnedTimeoutEnabled = true;
acqTool.Operator.OwnedTimeout_ms = 10000; // 10s 内没有触发信号则超时
```

### 场景 2：离线调试（自由运行）

```csharp
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.FreeRun;
acqTool.Operator.AcquisitionFPS = 15; // 15 fps，降低调试时的带宽占用
```

### 场景 3：手动按钮采集（软件触发）

```csharp
acqTool.Operator.Trigger.TriggerModel = CogAcqTriggerModelConstants.Software;
```

## 九、采集性能考虑

### 1. 缓冲区设置

```csharp
// 设置缓冲区数量
acqTool.Operator.NumBuffers = 4; // 增加缓冲区防止丢帧
```

如果采集速度快而处理速度慢，适当增加缓冲区可以减少丢帧。

### 2. 带宽管理

GigE 相机占用的网络带宽可以通过 VisionPro 的带宽管理功能调整。多台 GigE 相机同时采集时尤其需要注意。

### 3. ROI 采集

```csharp
// 只采集部分区域，减少数据传输量
acqTool.Operator.Width = 640;
acqTool.Operator.Height = 480;
```

如果不需要全分辨率图像，可以减少采集区域来提升帧率。

## 十、注意事项

### 1. 相机初始化只需要一次

不要每次采集前都 Init() 一次。Init() 应该在程序启动时调用，采集循环中只调用 Run()。

### 2. 单次采集中不要多次 Run()

如果你想确保每次采集都是全新的图像，每次调 Run() 后立即使用结果，不要在上一次结果还没用完时再次 Run()。

### 3. 采集超时处理

生产环境中一定要设置超时并处理超时情况，否则相机信号丢失时程序会一直卡在 Run() 调用上。

### 4. 释放资源

程序退出时应该关闭相机连接：

```csharp
acqTool.Operator.Shutdown();
```

## 总结

1. CogAcqFifoTool 是 VisionPro 中负责图像采集的核心工具，封装了相机连接、触发和缓冲管理；
2. 三种触发模式：自由运行（调试用）、软件触发（程序控制）、硬件触发（产线自动化）；
3. 采集和检测建议分离到不同线程，避免相互阻塞；
4. 生产环境必须做采集超时和异常处理；
5. Init() 只做一次，Shutdown() 在程序退出时调用。
