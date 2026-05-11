# 单例模式 Singleton

单例模式是最简单、最常用，也是最容易被滥用的设计模式。

## 一、单例模式解决什么问题

单例模式确保一个类只有一个实例，并提供一个全局访问点。

在工控项目中，常见的适合单例的场景：

- 配置管理器（整个系统只需要一套配置）；
- 日志记录器（所有模块往同一个地方写日志）；
- 硬件资源管理器（如相机设备列表，一个进程只需要一个管理实例）；
- 许可证管理（授权验证只需要一个实例）。

## 二、C# 中的实现方式

### 1. 基础实现（非线程安全）

```csharp
public class ConfigManager
{
    private static ConfigManager _instance;
    
    private ConfigManager() { }
    
    public static ConfigManager Instance
    {
        get
        {
            if (_instance == null)
                _instance = new ConfigManager();
            return _instance;
        }
    }
}
```

问题：多线程环境下可能创建多个实例。

### 2. 线程安全实现（双重锁定）

```csharp
public class ConfigManager
{
    private static ConfigManager _instance;
    private static readonly object _lock = new object();
    
    private ConfigManager() { }
    
    public static ConfigManager Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                        _instance = new ConfigManager();
                }
            }
            return _instance;
        }
    }
}
```

双重锁定在 C# 中是安全的，也是工业项目中最常用的实现。

### 3. Lazy\<T> 实现（推荐）

```csharp
public class ConfigManager
{
    private static readonly Lazy<ConfigManager> _instance = 
        new Lazy<ConfigManager>(() => new ConfigManager());
    
    private ConfigManager() { }
    
    public static ConfigManager Instance => _instance.Value;
}
```

这是 C# 中最简洁的线程安全实现方式，推荐在项目中使用。

## 三、一个工控场景的实际例子

```csharp
public class CameraManager
{
    private static readonly Lazy<CameraManager> _instance = 
        new Lazy<CameraManager>(() => new CameraManager());
    
    private List<ICamera> _cameras;
    
    private CameraManager()
    {
        _cameras = new List<ICamera>();
        EnumerateCameras();
    }
    
    public static CameraManager Instance => _instance.Value;
    
    private void EnumerateCameras() { /* 枚举系统中所有相机 */ }
    
    public ICamera GetCamera(int id) { /* ... */ }
}
```

这样在整个项目中，相机列表只需要初始化一次，任何模块都能通过 CameraManager.Instance 访问。

## 四、单例模式什么时候不该用

单例模式最大的问题：它引入了一种全局状态。

### 1. 破坏了依赖的可见性

如果使用单例，类 A 依赖了 ConfigManager.Instance，但看 A 的构造函数或方法签名是看不出来这个依赖的。这会让代码的依赖关系不透明。

### 2. 单元测试困难

单例的全局状态在测试之间会互相影响。测试 A 改了 ConfigManager 的某个属性，测试 B 就跑不过了。

### 3. 更好的替代方案

大多数情况下，依赖注入（DI）比单例更合适。

```csharp
// DI 方式，依赖关系在构造函数中明确可见
public class Inspector
{
    private readonly IConfigManager _config;
    public Inspector(IConfigManager config) { _config = config; }
}
```

然后通过 DI 容器配置为单例生命周期（AddSingleton），既保证了实例唯一性，又保持了依赖的可见性。

## 五、什么时候用单例，什么时候用 DI

| 场景 | 推荐方式 |
|------|----------|
| 系统全局配置 | DI 单例生命周期 |
| 日志记录 | 静态类或 DI 单例 |
| 硬件资源管理 | DI 单例生命周期 |
| 工具类/辅助方法 | 静态类 |
| 快速原型 | 传统单例 |
| 正式工程项目 | DI 单例生命周期（优先） |

## 六、几个容易踩的坑

1. 多线程环境下忘记加锁，低概率出现多个实例；
2. 继承了 IDisposable 的单例，在进程退出时释放顺序不确定；
3. 单例内部持有相机连接等需要手动释放的资源，忘记释放会导致内存泄漏；
4. 误用单例导致循环依赖（A 单例依赖 B 单例，B 单例依赖 A 单例）。

## 总结

1. 单例模式确保一个类只有一个实例并提供一个全局访问点；
2. C# 中推荐使用 Lazy\<T> 实现线程安全的单例；
3. 单例模式容易引入隐藏的全局依赖，降低代码的可测试性；
4. 在正式工程项目中，优先用 DI 容器的单例生命周期替代传统单例模式；
5. 了解单例模式是好的，但不要在任何需要"唯一"的地方都套用单例。
