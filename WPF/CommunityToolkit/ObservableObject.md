# ObservableObject

在 CommunityToolkit.Mvvm 中，ObservableObject 是最基础、最核心的类型之一。

它的主要职责只有一个：

为 ViewModel 提供属性变更通知能力。

如果把它说得更具体一点，它主要帮我们解决的是：

1. 属性值变了，UI 要自动刷新；
2. 不想每个属性都手写一遍 PropertyChanged；
3. 希望 ViewModel 有一个统一的通知基类。

很多其他特性和类型，其实都是建立在它的能力之上的，比如：

* ObservableProperty
* RelayCommand 相关特性
* 通知其他属性刷新
* 通知命令 CanExecute 重新计算

所以如果说 ObservableProperty 是高频特性，那么 ObservableObject 就是它背后的基础设施。

## 一、为什么需要 ObservableObject

WPF 的数据绑定要想在属性变化后自动刷新界面，通常需要对象实现 INotifyPropertyChanged。

如果完全手写，代码大概是这样：

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace Demo.ViewModels;

public class MainViewModel : INotifyPropertyChanged
{
 public event PropertyChangedEventHandler? PropertyChanged;

 protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
 {
  PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
 }

 private string title = string.Empty;

 public string Title
 {
  get => title;
  set
  {
   if (title != value)
   {
    title = value;
    OnPropertyChanged();
   }
  }
 }
}
```

这套写法没有问题，但明显比较重复。

ObservableObject 的意义就是把这些基础能力封装好，让你不用每个 ViewModel 都从零开始写一遍。

## 二、基础写法

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 private string title = string.Empty;

 public string Title
 {
  get => title;
  set => SetProperty(ref title, value);
 }
}
```

这里最关键的点是：

* 类继承了 ObservableObject；
* 属性 setter 中使用 SetProperty；
* 属性值变化时会自动触发通知。

对应 XAML：

```xml
<TextBox Text="{Binding Title, UpdateSourceTrigger=PropertyChanged}"/>
<TextBlock Text="{Binding Title}"/>
```

当 TextBox 修改 Title 后，TextBlock 会自动刷新显示。

## 三、ObservableObject 提供了什么能力

ObservableObject 最常用的能力主要有两类：

1. PropertyChanged 事件；
2. SetProperty 这一套辅助方法。

你可以把它理解成：

* PropertyChanged 是通知出口；
* SetProperty 是最常用的更新入口。

在实际项目里，很多属性通知都通过 SetProperty 完成。

## 四、SetProperty 是干什么的

SetProperty 是 ObservableObject 中最常用的方法。

典型写法：

```csharp
private bool isBusy;

public bool IsBusy
{
 get => isBusy;
 set => SetProperty(ref isBusy, value);
}
```

它实际上帮你做了几件事：

1. 比较旧值和新值是否相同；
2. 如果不同，就更新字段；
3. 触发 PropertyChanging 和 PropertyChanged；
4. 返回是否真的发生了变化。

所以它比手写：

```csharp
isBusy = value;
OnPropertyChanged();
```

更稳妥，因为它不会在值没变化时反复通知。

## 五、一个最常见的 ViewModel 示例

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class UserViewModel : ObservableObject
{
 private string name = string.Empty;
 private int age;
 private bool isSelected;

 public string Name
 {
  get => name;
  set => SetProperty(ref name, value);
 }

 public int Age
 {
  get => age;
  set => SetProperty(ref age, value);
 }

 public bool IsSelected
 {
  get => isSelected;
  set => SetProperty(ref isSelected, value);
 }
}
```

这就是最基础的 ObservableObject 用法。

如果不使用 ObservableProperty 特性，那么很多 ViewModel 就会保持这种“继承 ObservableObject + 手写属性 + SetProperty”的结构。

## 六、手写属性和 ObservableProperty 的关系

这两个不是对立关系，而是上下层关系：

* ObservableObject 提供通知能力；
* ObservableProperty 负责自动生成属性代码。

例如：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 [ObservableProperty]
 private string userName = string.Empty;
}
```

这里虽然你没有手写 UserName 属性，但生成后的属性本质上仍然是建立在 ObservableObject 的能力之上。

所以可以这样理解：

* 只用 ObservableObject：你自己写属性；
* ObservableObject + ObservableProperty：交给生成器帮你写属性。

## 七、什么时候只用 ObservableObject 就够了

虽然现在很多场景都喜欢直接用 ObservableProperty，但有些时候手写属性反而更清楚。

比如：

* 属性 setter 里逻辑比较复杂；
* 属性不是普通字段映射，而是包装外部模型；
* 需要更细粒度地控制赋值流程；
* 不想引入源生成器特性风格。

例如：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 private double price;
 private int quantity;

 public double Price
 {
  get => price;
  set
  {
   if (SetProperty(ref price, value))
   {
    OnPropertyChanged(nameof(TotalAmount));
   }
  }
 }

 public int Quantity
 {
  get => quantity;
  set
  {
   if (SetProperty(ref quantity, value))
   {
    OnPropertyChanged(nameof(TotalAmount));
   }
  }
 }

 public double TotalAmount => Price * Quantity;
}
```

这类场景里，手写属性往往比全交给特性更直观。

## 八、如何手动通知属性变化

除了 SetProperty，ObservableObject 也支持你手动调用通知方法。

### 1. OnPropertyChanged

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace Demo.ViewModels;

public partial class MainViewModel : ObservableObject
{
 public string FullName => $"{FirstName} {LastName}";

 private string firstName = string.Empty;
 public string FirstName
 {
  get => firstName;
  set
  {
   if (SetProperty(ref firstName, value))
   {
    OnPropertyChanged(nameof(FullName));
   }
  }
 }

 private string lastName = string.Empty;
 public string LastName
 {
  get => lastName;
  set
  {
   if (SetProperty(ref lastName, value))
   {
    OnPropertyChanged(nameof(FullName));
   }
  }
 }
}
```

这里的 FullName 没有字段，也没有 setter，但只要 FirstName 或 LastName 变化，就手动通知它刷新。

### 2. OnPropertyChanging

在少数场景下，你也可能会在属性变化前做处理。

```csharp
protected override void OnPropertyChanging(System.ComponentModel.PropertyChangingEventArgs e)
{
 base.OnPropertyChanging(e);
}
```

这类写法通常不如 PropertyChanged 常见，但在需要观察“变化前状态”的场景里是可用的。

## 九、一个更贴近项目的示例

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace Demo.ViewModels;

public partial class ProductEditorViewModel : ObservableObject
{
 private string productName = string.Empty;
 private decimal price;
 private int stock;
 private string statusMessage = "未保存";

 public string ProductName
 {
  get => productName;
  set
  {
   if (SetProperty(ref productName, value))
   {
    SaveCommand.NotifyCanExecuteChanged();
   }
  }
 }

 public decimal Price
 {
  get => price;
  set
  {
   if (SetProperty(ref price, value))
   {
    OnPropertyChanged(nameof(TotalText));
    SaveCommand.NotifyCanExecuteChanged();
   }
  }
 }

 public int Stock
 {
  get => stock;
  set
  {
   if (SetProperty(ref stock, value))
   {
    OnPropertyChanged(nameof(TotalText));
    SaveCommand.NotifyCanExecuteChanged();
   }
  }
 }

 public string StatusMessage
 {
  get => statusMessage;
  set => SetProperty(ref statusMessage, value);
 }

 public string TotalText => $"库存总值：{Price * Stock:C2}";

 public IRelayCommand SaveCommand { get; }

 public ProductEditorViewModel()
 {
  SaveCommand = new RelayCommand(Save, CanSave);
 }

 private void Save()
 {
  StatusMessage = "保存成功";
 }

 private bool CanSave()
 {
  return !string.IsNullOrWhiteSpace(ProductName)
   && Price > 0
   && Stock >= 0;
 }
}
```

这个例子说明了 ObservableObject 的几个核心使用点：

1. 手写属性时用 SetProperty 触发通知；
2. 计算属性通过 OnPropertyChanged 手动刷新；
3. 属性变化后可以顺带刷新命令状态；
4. 即使不用 ObservableProperty，也能完整支持 MVVM。

## 十、常见注意点

### 1. 不要赋值后忘记通知

下面这种写法在 ViewModel 中很常见，但它不会刷新 UI：

```csharp
private string title = string.Empty;

public string Title
{
 get => title;
 set => title = value;
}
```

因为这里只是改了字段，没有通知界面。

### 2. 优先使用 SetProperty，不要每次都自己写 if

虽然手写也可以，但 SetProperty 已经把最常见的比较和通知逻辑封装好了。

### 3. 计算属性要记得联动通知

如果某个属性依赖别的属性计算出来，原属性变化后，要记得调用 OnPropertyChanged(nameof(计算属性))。

### 4. 不是所有类都应该继承 ObservableObject

通常只有 ViewModel、界面状态模型、少量专门给 UI 绑定的数据对象适合继承它。

普通领域模型、纯业务实体、数据库实体，不一定适合直接继承 ObservableObject。

### 5. 手写属性多了以后，可以再考虑迁移到 ObservableProperty

如果一个类里大多数属性都只是“字段 + SetProperty”，那后续改成 ObservableProperty 往往会更简洁。

## 十一、ObservableObject 和 ObservableRecipient 的区别

很多人刚接触 Toolkit 时，会同时看到 ObservableObject 和 ObservableRecipient。

可以先简单理解为：

* ObservableObject：只负责属性通知；
* ObservableRecipient：在属性通知基础上，又整合了消息机制相关能力。

如果你当前只是写普通 WPF ViewModel，大多数时候先用 ObservableObject 就够了。

## 总结

1. ObservableObject 是 CommunityToolkit.Mvvm 中最基础的通知基类；
2. 它封装了 INotifyPropertyChanged 和常用的 SetProperty 逻辑；
3. 即使不使用 ObservableProperty，你也可以只靠 ObservableObject 手写完整的可绑定属性；
4. 它特别适合 ViewModel、界面状态对象这类需要和 UI 同步的数据结构；
5. 理解了 ObservableObject，再去理解 ObservableProperty、RelayCommand 等特性会更顺畅。
