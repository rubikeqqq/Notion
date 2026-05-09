# VisionPro 与 WPF 集成

VisionPro 的检测工具运行在后台，但工业视觉应用通常需要一个 UI 来显示图像、展示结果、接收操作指令。

WPF 是 VisionPro 最常见的桌面 UI 框架之一。

这篇讲的是如何把 VisionPro 的视觉能力和 WPF 的 UI 能力结合起来。

## 一、CogDisplay 控件

VisionPro 提供了一个专门的 WPF 显示控件：CogDisplay。

它的作用是：在 WPF 界面中显示图像、叠加检测结果（十字线、圆、矩形、文本等）、支持交互操作（缩放、平移、测量）。

### 1. 在 XAML 中引入 CogDisplay

```xml
<Window x:Class="DemoApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:cog="clr-namespace:Cognex.VisionPro.Display;assembly=Cognex.VisionPro.Display">

    <Grid>
        <!-- CogDisplay 控件 -->
        <cog:CogDisplay x:Name="Display"
                        Width="800" Height="600"
                        AutoSize="False"
                        MouseMode="Pan"/>
    </Grid>
</Window>
```

### 2. 在代码中显示图像

```csharp
using Cognex.VisionPro;

// 从采集工具获取图像
CogImage8Grey image = acqTool.OutputImage as CogImage8Grey;

// 在 CogDisplay 中显示
Display.Image = image;

// 自动适应显示区域
Display.Fit(true);
```

### 3. 叠加图形

```csharp
// 绘制十字线
CogCrossline cross = new CogCrossline();
cross.CenterX = 300;
cross.CenterY = 250;
cross.Color = CogColorConstants.Red;
cross.LineWidthInScreenPixels = 1;
Display.StaticGraphics.Add(cross, "Cross");

// 绘制矩形
CogRectangle rect = new CogRectangle();
rect.SetCenterWidthHeight(300, 250, 100, 80);
rect.Color = CogColorConstants.Green;
Display.StaticGraphics.Add(rect, "ROI");

// 绘制文本
CogGraphicLabel label = new CogGraphicLabel();
label.SetXYText(10, 10, "检测结果：OK");
label.Color = CogColorConstants.Green;
Display.StaticGraphics.Add(label, "Result");

// 叠加检测结果的圆
CogCircle circle = new CogCircle();
circle.CenterX = findCircle.Results.CircleCenterX;
circle.CenterY = findCircle.Results.CircleCenterY;
circle.Radius = findCircle.Results.CircleRadius;
circle.Color = CogColorConstants.Yellow;
Display.InteractiveGraphics.Add(circle, "MeasuredCircle");
```

## 二、在 WPF 中使用 WindowsFormsHost

如果你的 WPF 项目中无法直接引用 CogDisplay，或者版本兼容性问题，也可以通过 WindowsFormsHost 使用 WinForms 版本的 CogDisplay。

```xml
<Window x:Class="DemoApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:wf="clr-namespace:System.Windows.Forms;assembly=System.Windows.Forms"
        xmlns:winforms="clr-namespace:System.Windows.Forms.Integration;assembly=WindowsFormsIntegration">

    <winforms:WindowsFormsHost x:Name="FormsHost" Width="800" Height="600">
        <wf:Panel x:Name="DisplayPanel" />
    </winforms:WindowsFormsHost>
</Window>
```

```csharp
using Cognex.VisionPro.Display;

private CogDisplay _display;

public MainWindow()
{
    InitializeComponent();
    Loaded += (s, e) =>
    {
        _display = new CogDisplay();
        _display.Dock = DockStyle.Fill;
        DisplayPanel.Controls.Add(_display);
    };
}
```

## 三、整体 UI 结构

一个典型的 WPF + VisionPro 视觉检测界面通常包括：

```xml
<Window x:Class="DemoApp.MainWindow">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>   <!-- 工具栏 -->
            <RowDefinition Height="*"/>       <!-- 图像显示区 -->
            <RowDefinition Height="Auto"/>   <!-- 状态栏 -->
        </Grid.RowDefinitions>

        <!-- 工具栏 -->
        <ToolBar Grid.Row="0">
            <Button Content="开始采集" Command="{Binding StartAcqCommand}"/>
            <Button Content="执行检测" Command="{Binding InspectCommand}"/>
            <Separator/>
            <Button Content="保存图像" Command="{Binding SaveImageCommand}"/>
        </ToolBar>

        <!-- 图像显示区 -->
        <ContentControl Grid.Row="1" x:Name="DisplayHost"/>

        <!-- 状态栏 -->
        <StatusBar Grid.Row="2">
            <TextBlock Text="{Binding StatusMessage}"/>
            <Separator/>
            <TextBlock Text="{Binding InspectCount}"/>
        </StatusBar>
    </Grid>
</Window>
```

## 四、和 CommunityToolkit.Mvvm 配合

VisionPro 的 UI 层和 WPF MVVM 模式可以很好地配合。

### 1. ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Cognex.VisionPro;
using DemoApp.Services;

namespace DemoApp.ViewModels;

public partial class MainViewModel : ObservableObject
{
    private readonly IVisionService _visionService;

    [ObservableProperty]
    private string statusMessage = "就绪";

    [ObservableProperty]
    private int inspectCount;

    [ObservableProperty]
    private int passCount;

    [ObservableProperty]
    private int failCount;

    // 图像数据（供 View 显示用）
    [ObservableProperty]
    private CogImage8Grey? currentImage;

    public MainViewModel(IVisionService visionService)
    {
        _visionService = visionService;
    }

    [RelayCommand]
    private async Task InspectAsync()
    {
        try
        {
            StatusMessage = "检测中...";
            CurrentImage = _visionService.AcquireImage();

            if (CurrentImage == null)
            {
                StatusMessage = "采集失败";
                return;
            }

            var result = await Task.Run(() => _visionService.RunInspection(CurrentImage));

            InspectCount++;

            if (result.IsOK)
            {
                PassCount++;
                StatusMessage = "检测结果：OK";
            }
            else
            {
                FailCount++;
                StatusMessage = $"检测结果：NG - {result.Message}";
            }
        }
        catch (Exception ex)
        {
            StatusMessage = $"异常：{ex.Message}";
        }
    }

    [RelayCommand]
    private void SaveImage()
    {
        if (CurrentImage != null)
        {
            string fileName = $"Capture_{DateTime.Now:yyyyMMdd_HHmmss}.bmp";
            CurrentImage.Save(fileName);
            StatusMessage = $"图像已保存：{fileName}";
        }
    }
}
```

### 2. Code-Behind（负责 CogDisplay 的绑定）

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.Display;
using DemoApp.ViewModels;

namespace DemoApp.Views;

public partial class MainWindow : Window
{
    private readonly CogDisplay _display;

    public MainWindow(MainViewModel viewModel)
    {
        InitializeComponent();
        DataContext = viewModel;

        // 创建 CogDisplay
        _display = new CogDisplay();
        _display.Dock = DockStyle.Fill;
        DisplayPanel.Controls.Add(_display);

        // 监听图像变化，更新显示
        viewModel.PropertyChanged += (s, e) =>
        {
            if (e.PropertyName == nameof(MainViewModel.CurrentImage))
            {
                _display.Image = viewModel.CurrentImage;
                _display.Fit(true);
            }
        };

        // 显示检测结果图形
        viewModel.InspectionCompleted += OnInspectionCompleted;
    }

    private void OnInspectionCompleted(object? sender, InspectionResultEventArgs e)
    {
        // 在主线程中更新显示图形
        Dispatcher.Invoke(() =>
        {
            _display.StaticGraphics.Clear();

            if (e.Result.IsOK)
            {
                var label = new CogGraphicLabel();
                label.SetXYText(10, 10, "OK");
                label.Color = CogColorConstants.Green;
                _display.StaticGraphics.Add(label, "Status");
            }
            else
            {
                var label = new CogGraphicLabel();
                label.SetXYText(10, 10, "NG");
                label.Color = CogColorConstants.Red;
                _display.StaticGraphics.Add(label, "Status");
            }
        });
    }
}
```

## 五、三种界面模式的 UI 组织

视觉检测应用通常有三种界面模式：

### 1. 运行模式（Operator）

* 显示实时图像；
* 显示检测结果（OK/NG 指示灯、数据）；
* 基本的开始/停止控制；
* 界面简洁，操作员只操作必要按钮。

### 2. 调试模式（Engineer）

* 同样的图像显示，但可以看到更多信息（工具 ROI、边缘点、匹配结果）；
* 参数调整面板（曝光、检测参数、阈值）；
* 保存/加载配置。

### 3. 维护模式（Maintenance）

* 历史检测记录查询；
* 统计数据（良率、CPK、趋势图）；
* 日志查看。

这三种模式可以在同一个窗口通过 Tab 或切换布局来实现，也可以做成不同的 View 通过导航切换。

## 六、性能考虑

### 1. 图像更新频率

CogDisplay 的图像更新是耗时的。如果采集帧率很高（如 30fps），不要每帧都更新 Display，否则 UI 线程会扛不住。

```csharp
// 限流：每 200ms 更新一次显示
private DateTime _lastDisplayUpdate = DateTime.MinValue;

if ((DateTime.Now - _lastDisplayUpdate).TotalMilliseconds > 200)
{
    _display.Image = image;
    _display.Fit(true);
    _lastDisplayUpdate = DateTime.Now;
}
```

### 2. 检测和 UI 分离

检测逻辑应该放在后台线程，不要在 UI 线程中执行检测：

```csharp
// 正确：使用 Task.Run 将检测放到后台线程
var result = await Task.Run(() => _visionService.RunInspection(image));

// 错误：直接在 UI 线程执行
// var result = _visionService.RunInspection(image);
```

### 3. 图形叠加不要过多

CogDisplay 上叠加的图形越多，刷新越慢。动态图形只保留当前检测需要显示的，不需要的一律清除。

## 七、常见问题

### 1. CogDisplay 在 WPF 中无法获取焦点

这是因为 CogDisplay 是 WinForms 控件，在 WPF 的 WindowsFormsHost 中的焦点行为不同。可以通过设置 WindowsFormsHost 的焦点行为来改善。

### 2. 跨线程访问 CogDisplay

CogDisplay 必须在创建它的线程（通常是 UI 线程）上访问。任何后台线程的操作都要通过 Dispatcher.Invoke 调度到 UI 线程。

### 3. VisionPro 和 WPF 的引用问题

确保项目中引用了正确的 VisionPro 程序集：
* Cognex.VisionPro.dll
* Cognex.VisionPro.Display.dll
* 以及其他你使用的工具程序集（Cognex.VisionPro.PMAlign.dll 等）

### 4. 许可证

VisionPro 运行需要许可证。在开发机上安装 VisionPro 时会自动配置许可证。部署到产线时，需要确保目标机器有合法的许可证。

## 总结

1. CogDisplay 是 VisionPro 在 WPF 中显示图像和检测结果的核心控件；
2. 通过 WindowsFormsHost 可以在 WPF 中使用 WinForms 版本的 CogDisplay；
3. VisionPro 的检测逻辑和 CommunityToolkit.Mvvm 可以很好地配合，ViewModel 负责业务逻辑，View 负责图像显示；
4. 检测操作应该在后台线程执行，不要在 UI 线程中阻塞；
5. CogDisplay 的图像更新需要限流，高帧率场景不要每帧都刷新显示。
