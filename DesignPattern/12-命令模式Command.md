# 命令模式 Command

命令模式解决的核心问题是：把请求封装成对象，从而支持对请求进行参数化、排队、记录和撤销。

说人话就是：不是直接调一个方法，而是创建一个"命令对象"来描述你要做的事，然后由执行器去执行这个命令。

## 一、命令模式解决什么问题

设想一个产线的"一键校准"功能，它包含多个步骤：

1. 放置标定板；
2. 相机采集标定图像；
3. 执行标定算法；
4. 保存标定结果；
5. 验证标定精度。

直接写的话：

```csharp
public void Calibrate()
{
    Step1_PlaceCalibrationPlate();
    Step2_AcquireImages();
    Step3_RunCalibrationAlgorithm();
    Step4_SaveCalibrationResult();
    Step5_ValidateAccuracy();
}
```

这段代码的问题是：

- 没法撤销——如果某一步出错了，很难回退；
- 没法排队——需要把多个操作放进队列中按顺序执行；
- 没法记录——不知道哪些操作被做了什么；
- 没法做延迟执行——想"预设定时执行"很难。

## 二、命令模式的实现

```csharp
// 抽象命令
public interface ICommand
{
    void Execute();
    void Undo();
}

// 具体命令
public class AcquireImageCommand : ICommand
{
    private readonly ICamera _camera;
    private ImageData _capturedImage;
    
    public AcquireImageCommand(ICamera camera) { _camera = camera; }
    
    public void Execute()
    {
        _capturedImage = _camera.Acquire();
    }
    
    public void Undo()
    {
        // 采集的"撤销"是清除缓存
        _capturedImage = null;
    }
}

public class RunCalibrationCommand : ICommand
{
    private readonly ICalibrationService _service;
    private readonly ImageData[] _images;
    private CalibrationResult previousResult; // 用于撤销
    
    public RunCalibrationCommand(ICalibrationService service, ImageData[] images)
    {
        _service = service;
        _images = images;
    }
    
    public void Execute()
    {
        previousResult = _service.CurrentResult;
        _service.Calibrate(_images);
    }
    
    public void Undo()
    {
        // 恢复到之前的标定结果
        _service.Restore(previousResult);
    }
}

// 命令执行器（Invoker）
public class CommandExecutor
{
    private readonly Stack<ICommand> _history = new();
    
    public void Execute(ICommand command)
    {
        command.Execute();
        _history.Push(command);
    }
    
    public void UndoLast()
    {
        if (_history.TryPop(out var command))
            command.Undo();
    }
    
    public void ExecuteAll(IEnumerable<ICommand> commands)
    {
        foreach (var cmd in commands)
            Execute(cmd);
    }
}
```

使用方式：

```csharp
var executor = new CommandExecutor();

var calibrationSteps = new ICommand[]
{
    new AcquireImageCommand(camera),
    new AcquireImageCommand(camera),
    new AcquireImageCommand(camera),
    new RunCalibrationCommand(calibration, images),
    new SaveCalibrationCommand(calibration, config),
    new ValidateCalibrationCommand(calibration)
};

executor.ExecuteAll(calibrationSteps);

// 如果验证不通过，一键撤销所有步骤
executor.UndoLast();  // 撤销验证
executor.UndoLast();  // 撤销保存
// ...
```

## 三、命令模式的高级功能

### 命令队列

```csharp
public class CommandQueue
{
    private readonly Queue<ICommand> _queue = new();
    
    public void Enqueue(ICommand command) => _queue.Enqueue(command);
    
    public void ExecuteAll()
    {
        while (_queue.TryDequeue(out var cmd))
            cmd.Execute();
    }
    
    public int PendingCount => _queue.Count;
}
```

### 宏命令（复合命令）

```csharp
public class MacroCommand : ICommand
{
    private readonly List<ICommand> _commands = new();
    
    public void Add(ICommand command) => _commands.Add(command);
    
    public void Execute()
    {
        foreach (var cmd in _commands)
            cmd.Execute();
    }
    
    public void Undo()
    {
        // 反向撤销
        for (int i = _commands.Count - 1; i >= 0; i--)
            _commands[i].Undo();
    }
}
```

## 四、工控场景的实际运用

### 1. 产线动作序列管理

在产线自动化中，经常有一系列标准动作：

```csharp
var sequence = new MacroCommand();
sequence.Add(new MoveToLoadPositionCommand(robot));
sequence.Add(new TriggerCameraCommand(camera));
sequence.Add(new InspectProductCommand(inspector));
sequence.Add(new SortProductCommand(sorter, result));
sequence.Add(new MoveToHomeCommand(robot));

executor.Execute(sequence);
```

### 2. 参数调整的撤销

```csharp
// 在调参界面中支持 Ctrl+Z
public class ChangeExposureCommand : ICommand
{
    private readonly ICamera _camera;
    private readonly double _newValue;
    private double _oldValue;
    
    public void Execute()
    {
        _oldValue = _camera.Exposure;
        _camera.SetExposure(_newValue);
    }
    
    public void Undo() => _camera.SetExposure(_oldValue);
}
```

### 3. 多产品配方的延迟下发

在产线切换产品时，可以把整个切换过程封装成命令，在指定时间自动执行。

## 五、命令模式 vs 策略模式

| 模式 | 关注点 | 举例 |
|------|--------|------|
| 命令模式 | 封装一个请求/操作 | "执行标定"是一个命令 |
| 策略模式 | 封装一组可互换的算法 | "使用 Blob 算法"是一个策略 |

命令模式关注的是"做什么"以及"怎么管理这个操作（撤销、排队、记录）"，策略模式关注的是"怎么做"。

## 六、C# 中的函数式命令

如果命令逻辑很简单，C# 的委托也可以作为轻量级命令：

```csharp
public class SimpleCommand : ICommand
{
    private readonly Action _execute;
    private readonly Action _undo;
    
    public SimpleCommand(Action execute, Action undo)
    {
        _execute = execute;
        _undo = undo;
    }
    
    public void Execute() => _execute();
    public void Undo() => _undo();
}

// 使用
var cmd = new SimpleCommand(
    execute: () => camera.SetExposure(2000),
    undo: () => camera.SetExposure(oldValue)
);
```

## 总结

1. 命令模式把请求封装成对象，支持排队、撤销、记录和延迟执行；
2. 命令模式包含命令接口、具体命令、调用者、接收者四个角色；
3. 在工控项目中常用于产线动作序列管理、参数调整撤销、配方延迟下发；
4. 宏命令（MacroCommand）可以把多个命令组合成一个复合命令；
5. 命令模式和策略模式的区别在于：命令关注"操作的管理"，策略关注"算法的选择"。
