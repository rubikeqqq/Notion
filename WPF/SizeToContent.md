# SizeToContent
核心来说，它是 Window（窗口）、Popup 等控件的核心属性，作用是让控件根据内部内容的实际大小自动调整自身尺寸，无需手动设置固定的 Width/Height，避免内容被截断或窗口留白过多。

## 核心作用与枚举值

SizeToContent 的类型是 SizeToContent 枚举，包含 4 个可选值，覆盖不同的自动适配策略，用表格能快速理解：

| 枚举值         | 含义                                                       | 典型使用场景                           |
| -------------- | ---------------------------------------------------------- | -------------------------------------- |
| Manual         | 默认值，不自动适配，尺寸由 Width/Height 或用户拖动窗口决定 | 固定尺寸的编辑器、主窗口               |
| Width          | 仅自动调整宽度适配内容，高度保持手动设置 / 用户拖动的大小  | 宽度不固定、高度固定的提示条           |
| Height         | 仅自动调整高度适配内容，宽度保持手动设置 / 用户拖动的大小  | 表单窗口（宽度固定，高度随表单项变化） |
| WidthAndHeight | 宽度和高度都自动适配内容，窗口刚好 “包裹” 内部所有内容     | 弹窗、提示框、确认框（无多余留白）     |

## 代码示例

### 示例 1：最常用 - 宽高都自动适配（弹窗场景）

适合做消息提示、确认弹窗，窗口尺寸刚好包裹内容，无多余留白：
``` xml
<Window x:Class="SizeToContentDemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="自动适配弹窗"
        SizeToContent="WidthAndHeight"  <!-- 核心：宽高自动适配 -->
        ResizeMode="NoResize"> <!-- 可选：禁止用户拖动调整尺寸，保持适配效果 -->

    <!-- 内部内容：窗口会刚好包裹这个StackPanel -->
    <StackPanel Margin="20" Spacing="10">
        <TextBlock Text="操作成功！" FontSize="18" FontWeight="Bold"/>
        <TextBlock Text="数据已保存到本地" FontSize="12" Foreground="Gray"/>
        <Button Content="确定" Width="80" Height="30" Margin="0 10 0 0"/>
    </StackPanel>
</Window>
```

运行效果：窗口宽度刚好容纳文字 + 按钮，高度刚好包裹所有元素，无需手动设 Width/Height。

### 示例 2：仅适配高度（宽度固定）

适合表单类窗口，宽度固定，高度随表单项数量自动变化：

```xml
<Window Title="仅适配高度的表单"
        Width="300"  <!-- 宽度固定300 -->
        SizeToContent="Height"> <!-- 高度自动适配内容 -->
    <StackPanel Margin="20" Spacing="10">
        <TextBlock Text="用户信息表单"/>
        <TextBox PlaceholderText="请输入姓名"/>
        <TextBox PlaceholderText="请输入手机号"/>
        <TextBox PlaceholderText="请输入地址"/>
        <!-- 新增/删除表单项时，窗口高度会自动增加/减少 -->
    </StackPanel>
</Window>
```

## 关键注意事项

1. 优先级规则：当设置 SizeToContent=Width/Height/WidthAndHeight 时，手动设置的对应维度 Width/Height 会被覆盖。比如 SizeToContent=Width 时，写 Width="500" 无效，宽度由内容决定。
2. 适用范围：主要用于 Window 和 Popup 控件，普通容器（如 StackPanel/Grid）没有这个属性（它们靠自身布局逻辑自动适配）。
3. 与 ResizeMode 配合： 
    * 若想保持自动适配效果，设 ResizeMode="NoResize" 禁止用户拖动调整；  
    * 若允许用户调整，手动拖动窗口后，SizeToContent 的自动适配效果会失效（手动调整的尺寸优先）。

## 总结

1. SizeToContent 核心作用是让 Window/Popup 自动适配内部内容的尺寸，避免固定宽高的弊端；
2. 常用值：WidthAndHeight（弹窗）、Height（表单），Manual 是默认的手动控制；
3. 手动设置的 Width/Height 会被对应维度的自动适配覆盖，可配合 ResizeMode 控制窗口是否可拖动。