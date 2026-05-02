# NavigateUri

这个属性主要用于 Hyperlink 控件，用来指定点击超链接时要导航到的目标地址，是实现 WPF 界面中超链接功能的核心属性。

## 核心用法

### 基础使用场景（导航到外部 URL）

NavigateUri 最常用的场景是在 WPF 界面中创建可点击的超链接，跳转到浏览器的外部网页。需要注意：Hyperlink 必须嵌套在 TextBlock 中使用（因为它是内联元素，不能直接放在布局容器里）。

```xml
<Window x:Class="WpfHyperlinkDemo.MainWindow"
xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
Title="Hyperlink 示例" Height="300" Width="500">
    <Grid VerticalAlignment="Center" HorizontalAlignment="Center">
    <!-- Hyperlink 必须嵌套在 TextBlock 中 -->
        <TextBlock FontSize="16">
            <Hyperlink NavigateUri="https://www.baidu.com" Click="Hyperlink_Click">
            </Hyperlink>
        </TextBlock>
    </Grid>
</Window>
```

后台代码

```c#
using System;
using System.Diagnostics;
using System.Windows;
using System.Windows.Documents;

namespace WpfHyperlinkDemo
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        // Hyperlink 点击事件处理
        private void Hyperlink_Click(object sender, RoutedEventArgs e)
        {
            // 获取点击的 Hyperlink 控件
            Hyperlink hyperlink = sender as Hyperlink;
            if (hyperlink != null && hyperlink.NavigateUri != null)
            {
                try
                {
                    // 调用系统默认浏览器打开指定 URL
                    Process.Start(new ProcessStartInfo
                    {
                        FileName = hyperlink.NavigateUri.ToString(),
                        UseShellExecute = true // 关键：允许启动外部程序（浏览器）
                    });
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"打开链接失败：{ex.Message}");
                }
            }
        }
    }
}
```

### 进阶用法：导航到本地资源 / 内部页面

NavigateUri 也可以指向本地文件（如本地 HTML、文档）或 WPF 应用内的页面（配合 Frame 控件）。

* 打开本地文件

```xml
<TextBlock>
    <Hyperlink NavigateUri="file:///C:/Users/xxx/Desktop/测试文档.pdf" Click="Hyperlink_Click">
        打开本地PDF文档
    </Hyperlink>
</TextBlock>
```

* 在 Frame 中导航到内部页面

如果你的 WPF 应用使用 Page 做页面跳转，可以通过 NavigateUri 配合 Frame 实现内部导航：

```xml
<Window>
    <Grid>
        <!-- 用于显示内部页面的 Frame -->
        <Frame x:Name="mainFrame" Height="300" Width="500"/>
        
        <!-- 内部导航的超链接 -->
        <TextBlock Margin="0,320,0,0">
            <Hyperlink NavigateUri="Pages/AboutPage.xaml" Click="InternalHyperlink_Click">
                查看关于页面
            </Hyperlink>
        </TextBlock>
    </Grid>
</Window>
```

后台处理内部导航

```c#
private void InternalHyperlink_Click(object sender, RoutedEventArgs e)
{
    Hyperlink hyperlink = sender as Hyperlink;
    if (hyperlink != null && hyperlink.NavigateUri != null)
    {
        // 让 Frame 导航到指定的内部页面
        mainFrame.Navigate(new Uri(hyperlink.NavigateUri.ToString(), UriKind.Relative));
    }
}
```

## 关键注意事项

1. Uri 格式规范：
    * 外部 URL 必须带协议（http:///https://），否则会被识别为本地路径；
    * 本地文件需用 file:/// 前缀（注意三个斜杠），路径中的反斜杠 \ 要换成正斜杠 /，或转义为 \\。
2. 权限问题：
    * 调用 Process.Start 打开外部链接时，.NET Core/.NET 5+ 需设置 UseShellExecute = true（如示例中），否则会报错；
    * 打开本地文件时要确保文件路径正确，且应用有访问权限。
3. 样式美化：
    * 默认 Hyperlink 点击后会有下划线，可通过样式移除：

```xml
<Hyperlink NavigateUri="https://www.baidu.com" Click="Hyperlink_Click">
    <Hyperlink.Style>
        <Style TargetType="Hyperlink">
            <Setter Property="TextDecorations" Value="None"/>
            <Style.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter Property="TextDecorations" Value="Underline"/>
                </Trigger>
            </Style.Triggers>
        </Style>
    </Hyperlink.Style>
    百度首页
</Hyperlink>
```

## 总结

1. NavigateUri 是 Hyperlink 控件的核心属性，用于指定导航目标地址（外部 URL、本地文件、内部页面）；
2. 使用时需将 Hyperlink 嵌套在 TextBlock 中，点击事件需手动处理（打开外部链接用 Process.Start，内部导航用 Frame.Navigate）；
3. 注意 Uri 格式规范（协议前缀、路径分隔符）和权限配置（UseShellExecute = true），避免导航失败。
