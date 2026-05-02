# ControlTemplate 控件模板

WPF包含数据模板和控件模板，其中控件模板又包括ControlTemplate和ItemsPanelTemplate，这里讨论一下ControlTemplate。

这里讲一下控件模板与Style的异同，都是为了实现控件的个性化。Style只能改变控件已有的属性（比如字体颜色等），但控件模板可以改变空间的内部结构（VisualTree，视觉树）来完成更为复杂的定制，比如我们可以定制一个带图片的按钮。

要替换控件的模板，我们只需要声明一个ControlTemplate对象，并对该ControlTemplate对象做相应的配置，然后将该ControlTemplate对象赋值给控件的Template属性就可以了。

ControlTemplate包含两个重要的属性：
1，VisualTree，该模板的视觉树，其实我们就是使用这个属性来描述控件的外观的
2，Triggers，触发器列表，里面包含一些触发器Trigger，我们可以定制这个触发器列表来使控件对外界的刺激发生反应，比如鼠标经过时文本变成粗体等。

``` xml
<Button>
  <Button.Template>
    <ControlTemplate>
      <!--定义视觉树-->
      <Grid>
        <EllipseName="faceEllipse"
          Width="{TemplateBinding Button.Width}"
          Height="{TemplateBinding Control.Height}"
          Fill="{TemplateBinding Button.Background}"/>
          <TextBlockName="txtBlock"
            Margin="{TemplateBinding Button.Padding}"
            VerticalAlignment="Center"
            HorizontalAlignment="Center"
            Text="{TemplateBinding Button.Content}" />
      </Grid>
      <!--定义视觉树_end-->
    </ControlTemplate>
  </Button.Template>
</Button>
```

在上面的代码中，我们修改了Button的Template属性，我们定义了一个ControlTemplate，在 < ControlTemplate > ... </ControlTemplate>之间包含的是模板的视觉树，也就是如何显示控件的外观，我们这里使用了一个Ellipse（椭圆）和一个TextBlock（文本块）来定义控件的外观。
很容易联想到一个问题：控件（Button）的一些属性，比如高度、宽度、文本等如何在新定义的外观中表现出来呢？
我们使用TemplateBinding 将控件的属性与新外观中的元素的属性关联起来Width="{TemplateBinding Button.Width}" ，这样我们就使得椭圆的宽度与按钮的宽度绑定在一起而保持一致，同理我们使用Text="{TemplateBinding Button.Content}"将TextBlock的文本与按钮的Content属性绑定在一起。

除了定义控件的默认外观外，也许我们想还定义当外界刺激我们的控件时，控件外观做出相应的变化，这是我们需要触发器。参考以下代码：

``` xml
<Button Content="test btn"
  Grid.Column="1"
  Grid.ColumnSpan="1"
  Grid.Row="1"
  Grid.RowSpan="1"  >
        <Button.Template>
          <ControlTemplate>
            <!--定义视觉树-->
            <Grid>
              <Ellipse Name="faceEllipse"
                Width="{TemplateBinding Button.Width}"
                Height="{TemplateBinding Control.Height}"  
                Fill="{TemplateBinding Button.Background}"/>
              <TextBlock Name="txtBlock"
                Margin="{TemplateBinding Button.Padding}"
                VerticalAlignment="Center"  
                HorizontalAlignment="Center"  
                Text="{TemplateBinding Button.Content}" />
            </Grid>
            <!--定义视觉树_end-->
            <!--定义触发器-->
            <ControlTemplate.Triggers>
              <Trigger  Property="Button.IsMouseOver"  Value="True">
                <Setter Property="Button.Foreground" Value="Red" />
              </Trigger>
            </ControlTemplate.Triggers>
            <!--定义触发器_End-->
          </ControlTemplate>
        </Button.Template>
      </Button>
```

在上面的代码中注意到 <ControlTemplate.Triggers>...<ControlTemplate.Triggers> 之间的部分，我们定义了触发器 <Trigger Property="Button.IsMouseOver" Value="True">，其表示当 Button 的 IsMouseOver 属性变成 True 时，将使用设置器 <Setter Property="Button.Foreground" Value="Red" > 来将 Button 的 Foreground 属性设置为 Red。这里有一个隐含的意思是：当 Button 的 IsMouseOver 属性变成 False 时，设置器中设置的属性将恢复原值。

如果我要重用我的模板，应该怎么办呢？
我相信你已经想到了，你只要把它定义为资源

``` xml
<Window.Resources>
    <ControlTemplate TargetType="Button" x:Key="ButtonTemplate">
        <!--定义视觉树-->
        <Grid>
          <Ellipse Name="faceEllipse"
            Width="{TemplateBinding Button.Width}"
            Height="{TemplateBinding Control.Height}"  
            Fill="{TemplateBinding Button.Background}"/>
          <TextBlock Name="txtBlock"
            Margin="{TemplateBinding Button.Padding}"
            VerticalAlignment="Center"  
            HorizontalAlignment="Center"  
            Text="{TemplateBinding Button.Content}" />
        </Grid>
        <!--定义视觉树_end-->
        <!--定义触发器-->
        <ControlTemplate.Triggers>
          <Trigger  Property="Button.IsMouseOver"  Value="True">
            <Setter Property="Button.Foreground" Value="Red" />
          </Trigger>
        </ControlTemplate.Triggers>
        <!--定义触发器_End-->
    </ControlTemplate>
  </Window.Resources>
  ```

上面的代码将我们原来的模板定义为窗体范围内的资源，其中 TargetType ="Button"指示我们的模板作用对象为Button，这样在整个窗体范围内的按钮都可以使用这个模板了，模板的使用方法也很简单

``` xml
<ButtonContent="test btn"Template="{StaticResource ButtonTemplate}" />
<Window
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    Title="ControlTemplateTest" Height="300" Width="300"
    >

  <Window.Resources>
    <ControlTemplate TargetType="Button" x:Key="ButtonTemplate">
        <!--定义视觉树-->
        <Grid>
          <Ellipse Name="faceEllipse"
            Width="{TemplateBinding Button.Width}"
            Height="{TemplateBinding Control.Height}"  
            Fill="{TemplateBinding Button.Background}"/>
          <TextBlock Name="txtBlock"
            Margin="{TemplateBinding Button.Padding}"
            VerticalAlignment="Center"  
            HorizontalAlignment="Center"  
            Text="{TemplateBinding Button.Content}" />
        </Grid>
        <!--定义视觉树_end-->
        <!--定义触发器-->
        <ControlTemplate.Triggers>
          <Trigger  Property="Button.IsMouseOver"  Value="True">
            <Setter Property="Button.Foreground" Value="Red" />
          </Trigger>
        </ControlTemplate.Triggers>
        <!--定义触发器_End-->
    </ControlTemplate>
  </Window.Resources>
  
    <Grid ShowGridLines="True">
      
      <Grid.ColumnDefinitions>
        <ColumnDefinition Width="0.2*"/>
        <ColumnDefinition Width="0.6*"/>
        <ColumnDefinition Width="0.2*"/>
      </Grid.ColumnDefinitions>
      <Grid.RowDefinitions>
        <RowDefinition Height="0.3*"/>
        <RowDefinition Height="0.3*"/>
        <RowDefinition Height="0.4*"/>
      </Grid.RowDefinitions>
 
      <Button Content="test btn1" 
        Grid.Column="0" 
        Grid.ColumnSpan="1" 
        Grid.Row="0" 
        Grid.RowSpan="1"  />
      <Button Content="test btn2"
        Grid.Column="1" 
        Grid.ColumnSpan="1" 
        Grid.Row="1" 
        Grid.RowSpan="1"  
        Template="{StaticResource ButtonTemplate}" />
      <Button Content="test btn2"
        Grid.Column="2" 
        Grid.ColumnSpan="1" 
        Grid.Row="2" 
        Grid.RowSpan="1"  
        Template="{StaticResource ButtonTemplate}" />
        
        
    </Grid>
</Window>
```

如果你想加入一些动画效果的话，可以在触发器中，调用一个故事板来达到对事件响应时的动画效果

``` xml
<!--定义动画资源-->
      <ControlTemplate.Resources>
        <Storyboard x:Key="MouseClickButtonStoryboard">
          <DoubleAnimationUsingKeyFrames Storyboard.TargetName="faceEllipse" Storyboard.TargetProperty="Width" BeginTime="00:00:00">
            <SplineDoubleKeyFrame KeyTime="00:00:00" Value="50"/>
            <SplineDoubleKeyFrame KeyTime="00:00:00.3" Value="100"/>
          </DoubleAnimationUsingKeyFrames>
        </Storyboard>
      </ControlTemplate.Resources>
      ```
我们为模板定义了一个动画资源，此后在模板的触发器中我们就可以调用该资源来实现一个动画效果了：
``` xml
<EventTrigger RoutedEvent="Mouse.MouseDown" SourceName="faceEllipse">
          <EventTrigger.Actions>
            <BeginStoryboard Storyboard="{StaticResource MouseClickButtonStoryboard}"/>
          </EventTrigger.Actions>
        </EventTrigger>

<Window
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    Title="ControlTemplateTest" Height="300" Width="300"
    >

  <Window.Resources>
    <ControlTemplate TargetType="Button" x:Key="ButtonTemplate">
      <!--定义视觉树-->
      <Grid>
        <Ellipse Name="faceEllipse" Width="{TemplateBinding Button.Width}" Height="{TemplateBinding Control.Height}"  Fill="{TemplateBinding Button.Background}"/>
        <TextBlock Name="txtBlock" Margin="{TemplateBinding Button.Padding}" VerticalAlignment="Center"  HorizontalAlignment="Center"  Text="{TemplateBinding Button.Content}" />
      </Grid>
      <!--定义视觉树_end-->

      <!--定义动画资源-->
      <ControlTemplate.Resources>
        <Storyboard x:Key="MouseClickButtonStoryboard">
          <DoubleAnimationUsingKeyFrames Storyboard.TargetName="faceEllipse" Storyboard.TargetProperty="Width" BeginTime="00:00:00">
            <SplineDoubleKeyFrame KeyTime="00:00:00" Value="50"/>
            <SplineDoubleKeyFrame KeyTime="00:00:00.3" Value="100"/>
          </DoubleAnimationUsingKeyFrames>
        </Storyboard>
      </ControlTemplate.Resources>
      <!--定义动画资源_end-->
 
      <!--定义触发器-->
      <ControlTemplate.Triggers>
        <Trigger  Property="Button.IsMouseOver"  Value="True">
          <Setter Property="Button.Foreground" Value="Red" />
        </Trigger>
        <EventTrigger RoutedEvent="Mouse.MouseDown" SourceName="faceEllipse">
          <EventTrigger.Actions>
            <BeginStoryboard Storyboard="{StaticResource MouseClickButtonStoryboard}"/>
          </EventTrigger.Actions>    
        </EventTrigger>
        <EventTrigger RoutedEvent="Mouse.MouseDown" SourceName="txtBlock">
          <EventTrigger.Actions>
            <BeginStoryboard Storyboard="{StaticResource MouseClickButtonStoryboard}"/>
          </EventTrigger.Actions>
        </EventTrigger>
      </ControlTemplate.Triggers>
      <!--定义触发器_End-->
      
    </ControlTemplate>

  </Window.Resources>
  
    <Grid ShowGridLines="True">
      
      <Grid.ColumnDefinitions>
        <ColumnDefinition Width="0.2*"/>
        <ColumnDefinition Width="0.6*"/>
        <ColumnDefinition Width="0.2*"/>
      </Grid.ColumnDefinitions>
      <Grid.RowDefinitions>
        <RowDefinition Height="0.3*"/>
        <RowDefinition Height="0.3*"/>
        <RowDefinition Height="0.4*"/>
      </Grid.RowDefinitions>
 
      <Button Content="test btn1" Grid.Column="0" Grid.ColumnSpan="1" Grid.Row="0" Grid.RowSpan="1"  />
      <Button Content="test btn2" Grid.Column="1" Grid.ColumnSpan="1" Grid.Row="1" Grid.RowSpan="1"  Template="{StaticResource ButtonTemplate}" />
      <Button Content="test btn2" Grid.Column="2" Grid.ColumnSpan="1" Grid.Row="2" Grid.RowSpan="1"  Template="{StaticResource ButtonTemplate}" />
        
    </Grid>
</Window>
```

最后留一个ImageButton自定义控件以供参考

``` csharp
ImageButton.cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;

namespace SoftRGB.Controls
{
    /// <summary>
    /// 按照步骤 1a 或 1b 操作，然后执行步骤 2 以在 XAML 文件中使用此自定义控件。
    ///
    /// 步骤 1a) 在当前项目中存在的 XAML 文件中使用该自定义控件。
    /// 将此 XmlNamespace 特性添加到要使用该特性的标记文件的根
    /// 元素中:
    ///
    ///     xmlns:MyNamespace="clr-namespace:SoftRGB.Controls"
    ///
    ///
    /// 步骤 1b) 在其他项目中存在的 XAML 文件中使用该自定义控件。
    /// 将此 XmlNamespace 特性添加到要使用该特性的标记文件的根
    /// 元素中:
    ///
    ///     xmlns:MyNamespace="clr-namespace:SoftRGB.Controls;assembly=SoftRGB.Controls"
    ///
    /// 您还需要添加一个从 XAML 文件所在的项目到此项目的项目引用，
    /// 并重新生成以避免编译错误:
    ///
    ///     在解决方案资源管理器中右击目标项目，然后依次单击
    ///     “添加引用”->“项目”->[浏览查找并选择此项目]
    ///
    ///
    /// 步骤 2)
    /// 继续操作并在 XAML 文件中使用控件。
    ///
    ///     <MyNamespace:ImageButton/>
    ///
    /// </summary>
    public class ImageButton : Button
    {
        static ImageButton()
        {
            DefaultStyleKeyProperty.OverrideMetadata(typeof(ImageButton), new FrameworkPropertyMetadata(typeof(ImageButton)));
        }

        #region 依赖属性
 
        public static readonly DependencyProperty NormalImageProperty =
            DependencyProperty.Register("NormalImage", typeof(string), typeof(ImageButton), null);
        public static readonly DependencyProperty OverImageProperty =
           DependencyProperty.Register("OverImage", typeof(string), typeof(ImageButton), null);
        public static readonly DependencyProperty PressedImageProperty =
           DependencyProperty.Register("PressedImage", typeof(string), typeof(ImageButton), null);
        public static readonly DependencyProperty DisableImageProperty =
           DependencyProperty.Register("DisableImage", typeof(string), typeof(ImageButton), null);
 
        #endregion
 
        #region 属性
 
        /// <summary>
        /// 正常状态下显示的图片
        /// </summary>
        public string NormalImage
        {
            get { return (string)GetValue(NormalImageProperty); }
            set { SetValue(NormalImageProperty, value); }
        }
 
        /// <summary>
        /// 鼠标滑过状态下显示的图片
        /// </summary>
        public string OverImage
        {
            get { return (string)GetValue(OverImageProperty); }
            set { SetValue(OverImageProperty, value); }
        }
 
        /// <summary>
        /// 鼠标点击状态下显示的图片
        /// </summary>
        public string PressedImage
        {
            get { return (string)GetValue(PressedImageProperty); }
            set { SetValue(PressedImageProperty, value); }
        }
 
        /// <summary>
        /// 禁用状态下显示的图片
        /// </summary>
        public string DisableImage
        {
            get { return (string)GetValue(DisableImageProperty); }
            set { SetValue(DisableImageProperty, value); }
        }
 
        #endregion
 
    }
}
```

``` xml  
 <Style TargetType="{x:Type local:ImageButton}">
        <Setter Property="Background" Value="Transparent"/>
        <Setter Property="FocusVisualStyle" Value="{x:Null}"/>
        <Setter Property="VerticalAlignment" Value="Center"/>
        <Setter Property="HorizontalAlignment" Value="Center"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="{x:Type local:ImageButton}">
                    <Grid x:Name="Root" Background="Transparent">
                        <VisualStateManager.VisualStateGroups>
                            <VisualStateGroup x:Name="CommonStates">
                                <VisualState x:Name="Normal">
                                    <Storyboard>
                                        <ObjectAnimationUsingKeyFrames Storyboard.TargetName="NormalUI" Storyboard.TargetProperty="Visibility">
                                            <DiscreteObjectKeyFrame KeyTime="0" Value="{StaticResource Visible}"/>
                                        </ObjectAnimationUsingKeyFrames>
                                    </Storyboard>
                                </VisualState>
                                <VisualState x:Name="MouseOver">
                                    <Storyboard>
                                        <ObjectAnimationUsingKeyFrames Storyboard.TargetName="OverUI" Storyboard.TargetProperty="Visibility">
                                            <DiscreteObjectKeyFrame KeyTime="0" Value="{StaticResource Visible}"/>
                                        </ObjectAnimationUsingKeyFrames>
                                    </Storyboard>
                                </VisualState>
                                <VisualState x:Name="Pressed">
                                    <Storyboard>
                                        <ObjectAnimationUsingKeyFrames Storyboard.TargetName="PressedUI" Storyboard.TargetProperty="Visibility">
                                            <DiscreteObjectKeyFrame KeyTime="0" Value="{StaticResource Visible}"/>
                                        </ObjectAnimationUsingKeyFrames>
                                    </Storyboard>
                                </VisualState>
                                <VisualState x:Name="Disabled">
                                    <Storyboard>
                                        <ObjectAnimationUsingKeyFrames Storyboard.TargetName="DisableUI" Storyboard.TargetProperty="Visibility">
                                            <DiscreteObjectKeyFrame KeyTime="0" Value="{StaticResource Visible}"/>
                                        </ObjectAnimationUsingKeyFrames>
                                    </Storyboard>
                                </VisualState>
                            </VisualStateGroup>
                        </VisualStateManager.VisualStateGroups>
                        <Border Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}">
                            <Grid>
                                <Image x:Name="NormalUI" Source="{Binding NormalImage, RelativeSource={RelativeSource TemplatedParent}}" Visibility="Collapsed"/>
                                <Image x:Name="OverUI" Source="{Binding OverImage, RelativeSource={RelativeSource TemplatedParent}}" Visibility="Collapsed"/>
                                <Image x:Name="PressedUI" Source="{Binding PressedImage, RelativeSource={RelativeSource TemplatedParent}}" Visibility="Collapsed"/>
                                <Image x:Name="DisableUI" Source="{Binding DisableImage, RelativeSource={RelativeSource TemplatedParent}}" Visibility="Collapsed"/>
                                <ContentPresenter x:Name="ContentPresenter" Margin="{TemplateBinding Padding}" ContentTemplate="{TemplateBinding ContentTemplate}"
                                                  VerticalAlignment="{TemplateBinding VerticalContentAlignment}"
                                                  HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}"/>
                            </Grid>
                        </Border>
                    </Grid>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
    ```
<ControlTemplate TargetType="{x:Type local:ImageButton}">... </ControlTemplate>为控件模板，这里需说明一下在xaml文件中要引用ImageButton类要先引用ImageButton所在的命名空间。。。。
