# Halcon + WPF 示例工程骨架建议

这份附加文档的目标不是继续补概念，而是把前面那 26 篇笔记里最适合先落地成示例工程的部分挑出来，整理成一套可直接开工的代码骨架建议。

如果你下一步要从笔记走向工程，建议不要一上来就把全部能力都塞进项目里，而是先做一个最小但完整的页面级案例。

## 一、最适合先落成示例工程的 5 篇

如果只能先挑 3 到 5 篇作为示例工程的核心参考，我建议优先看下面这 5 篇：

1. [17-多步骤检测流程设计](17-多步骤检测流程设计.md)
2. [20-HalconDotNet与工程引用](20-HalconDotNet与工程引用.md)
3. [21-HSmartWindowControlWPF显示控件](21-HSmartWindowControlWPF显示控件.md)
4. [24-Halcon检测服务与WPF页面集成](24-Halcon检测服务与WPF页面集成.md)
5. [25-Halcon+WPF完整案例](25-Halcon+WPF完整案例.md)

如果想补一个第 6 篇作为结构稳定器，优先加：

* [23-Halcon与MVVM数据绑定配合](23-Halcon与MVVM数据绑定配合.md)

## 二、为什么先选这 5 篇

### 1. 17 负责把算法流程拆清楚

它决定你的检测流程在工程里是不是按“输入 -> 预处理 -> 定位 -> 分支检测 -> 汇总”来组织。

### 2. 20 负责把 Halcon 真正接进 .NET 工程

它解决运行环境、位数、引用和最小调用入口这些前提问题。

### 3. 21 负责把图像显示入口搭起来

没有显示控件，后面的页面调试和结果叠加都无从谈起。

### 4. 24 负责把算法从页面代码里抽到服务层

这一步决定你的项目后续能不能维护。

### 5. 25 负责把整条页面闭环打通

它把加载图像、执行检测、显示结果、保存结果这些动作串成一个完整示例。

## 三、第一版示例工程建议只做什么

不要一开始就做多页面、多工位、多相机。

第一版建议只做一个页面、一个检测流程、一个结果模型，功能范围控制在下面这些：

1. 加载本地图像；
2. 显示原图；
3. 执行一次 Halcon 检测；
4. 叠加一个或两个检测结果；
5. 页面显示 OK / NG、一个测量值、一个识别值；
6. 保存带叠加结果图。

这个范围已经足够把整条工程链跑通。

## 四、建议的示例项目功能主题

如果要先选一个最适合落地的示例主题，我建议用下面这个：

### 零件定位 + 孔径测量 + 字符读取

原因很简单：

* 有主定位；
* 有坐标跟随；
* 有一个测量项；
* 有一个识别项；
* 有图像叠加显示；
* 能覆盖 Halcon + WPF 工程中最关键的结构。

相比纯缺陷检测或纯读码，这个主题更适合当第一个综合示例。

## 五、建议的最小项目结构

```text
HalconDemoApp/
  App.xaml
  App.xaml.cs
  Models/
    InspectionResult.cs
    OverlayResult.cs
  Services/
    IHalconInspectionService.cs
    HalconInspectionService.cs
    IImageFileService.cs
    ImageFileService.cs
  ViewModels/
    MainWindowViewModel.cs
  Views/
    MainWindow.xaml
    MainWindow.xaml.cs
  Display/
    HalconDisplayAdapter.cs
  Resources/
    template.png
    sample-images/
```

这套结构的核心思想是：

* `Services` 放 Halcon 流程；
* `ViewModels` 放页面状态和命令；
* `Views` 放页面与显示控件；
* `Display` 单独放显示适配层，避免 ViewModel 直接碰控件；
* `Models` 放检测结果和叠加结果模型。

## 六、每个核心文件建议负责什么

### 1. App.xaml.cs

负责：

* 应用启动；
* 简单依赖注入装配；
* 创建主窗口和主 ViewModel。

### 2. MainWindow.xaml

负责：

* 页面布局；
* HSmartWindowControlWPF 放置；
* 按钮、状态区、结果区。

### 3. MainWindowViewModel.cs

负责：

* 加载图像命令；
* 执行检测命令；
* 保存结果命令；
* 当前状态文字；
* 当前检测结果；
* 当前图像路径。

### 4. HalconInspectionService.cs

负责：

* 读取或接收图像；
* 运行预处理；
* 运行模板匹配；
* 做 ROI 跟随；
* 测量孔径；
* 读取字符；
* 汇总结果对象。

### 5. HalconDisplayAdapter.cs

负责：

* 显示原图；
* 清空与重绘；
* 叠加轮廓、区域、文本；
* 隔离 HSmartWindowControlWPF 的具体调用细节。

### 6. InspectionResult.cs

建议至少包含：

* `IsOk`
* `Message`
* `HoleDiameter`
* `CodeText`
* `ElapsedMilliseconds`
* `OverlayContours`
* `OverlayRegions`
* `OverlayTexts`

## 七、第一版检测服务建议怎么拆

第一版不需要把服务拆得太细，建议先按下面结构：

### 1. Initialize

负责：

* 加载模板图；
* 创建 shape model；
* 准备固定参考 ROI。

### 2. Inspect

负责：

* 输入图像；
* 执行完整检测流程；
* 返回单次 `InspectionResult`。

### 3. Dispose 或资源释放

负责：

* 释放模型句柄；
* 释放长期持有的 Halcon 资源。

这样就已经比“页面按钮里直接写一堆 Halcon 算子”稳很多。

## 八、页面层建议保留哪些最小交互

第一版建议只保留下面几个按钮：

1. 加载图像
2. 开始检测
3. 清空结果
4. 保存结果图

不要一开始就加入：

* 大量参数编辑；
* ROI 手动画框；
* 多模板切换；
* 多页面导航；
* 批量图像队列。

这些都可以第二版再加。

## 九、建议的最小数据流

```text
用户点击“加载图像”
  -> ViewModel 记录当前图像路径
  -> DisplayAdapter 显示原图

用户点击“开始检测”
  -> ViewModel 调用 HalconInspectionService.Inspect
  -> Service 返回 InspectionResult
  -> ViewModel 更新状态文字和结果值
  -> DisplayAdapter 显示图像叠加结果

用户点击“保存结果图”
  -> ViewModel 使用当前 InspectionResult 驱动保存
```

这条链路已经足够形成一个完整、可展示、可继续扩展的桌面示例。

## 十、第一版项目不要急着做什么

下面这些内容都建议放到第二阶段：

* 多页面或导航框架
* 实时相机采集
* 参数持久化编辑器
* 多工位支持
* 多线程复杂调度
* 多种检测配方切换

第一版最重要的是先把“结构对不对”跑通，而不是功能做满。

## 十一、一个推荐的开发顺序

建议按下面顺序落代码：

1. 先建最小 WPF 工程并确认 HalconDotNet 引用正常；
2. 先把 HSmartWindowControlWPF 显示原图跑通；
3. 再加一个最小 `HalconInspectionService`，先只返回假结果或简单读图结果；
4. 再接入模板匹配和单个测量项；
5. 再补字符读取；
6. 最后补结果叠加和保存结果图。

这个顺序能最大限度减少同时调试太多问题。

## 十二、这套骨架为什么适合当第一版

因为它满足几个关键条件：

* 页面简单；
* 结构清楚；
* 算法和显示边界明确；
* 能覆盖 Halcon + WPF 最常见的集成问题；
* 后面加功能时不需要推翻项目骨架。

## 总结

1. 第一版 Halcon + WPF 示例工程，最适合围绕“定位 + 测量 + 识别”做一个单页面闭环；
2. 最值得先反推工程骨架的 5 篇是 17、20、21、24、25，必要时补 23；
3. 项目结构建议以 `View + ViewModel + Detection Service + Display Adapter + Result Model` 为核心；
4. 第一版先把加载图像、执行检测、叠加显示、保存结果跑通，比一开始做大而全更稳。

* 图像显示入口；
* 结果叠加的宿主控件。

没有显示层，示例工程很难形成可见闭环。

### 3. 23 号文档解决“MVVM 边界怎么划”

它对应的是：

* View、ViewModel、算法服务怎么拆；
* 页面状态和算法结果如何衔接；
* 不让页面后置和算法逻辑缠在一起。

### 4. 24 号文档解决“Halcon 流程到底放哪层”

它对应的是：

* Detection Service 封装；
* 初始化和单次检测分离；
* 结构化结果回传。

这是示例工程能不能长期扩展的关键。

### 5. 25 号文档解决“如何形成页面级闭环”

它对应的是：

* 加载图像；
* 执行检测；
* 图像叠加；
* 结果显示；
* 保存日志或结果图。

它最适合当作示例工程的验收目标。

## 三、建议先做什么，不建议一开始做什么

### 建议先做

* 单窗口 WPF 页面；
* 本地图像加载；
* 一条最小 Halcon 检测流程；
* 一个检测服务；
* 一个结果对象；
* 一个主显示控件；
* 一组基本按钮命令。

### 不建议一开始就做

* 多页面导航；
* 多相机接入；
* 大量参数编辑器；
* 复杂权限系统；
* 完整数据库或 MES 对接；
* 一次把所有 26 篇内容都做进一个工程。

第一版最重要的是：

把“输入 -> 检测 -> 显示 -> 结果”先跑通。

## 四、最小示例工程建议结构

建议先做成下面这种结构：

```text
HalconWpfDemo/
  App.xaml
  App.xaml.cs
  Views/
    MainWindow.xaml
    MainWindow.xaml.cs
  ViewModels/
    MainWindowViewModel.cs
  Services/
    IHalconInspectionService.cs
    HalconInspectionService.cs
    IHalconDisplayBridge.cs
    HalconDisplayBridge.cs
  Models/
    InspectionResult.cs
    OverlayItem.cs
  Resources/
    Images/
      demo-part.png
  README.md
```

这个结构的重点不是“唯一正确”，而是足够小、职责也够清楚。

## 五、每一层建议承担什么职责

### 1. MainWindow.xaml

负责：

* 页面布局；
* HSmartWindowControlWPF 宿主；
* 基础按钮区；
* 基础结果显示区。

建议先放这几个按钮：

* 加载图像
* 开始检测
* 清空显示
* 保存结果图

### 2. MainWindowViewModel.cs

负责：

* LoadImageCommand
* InspectCommand
* ClearCommand
* SaveResultCommand
* 当前状态文本
* 当前结果对象
* 当前图像路径
* 当前是否正在检测

它不建议直接写大量 Halcon 算子。

### 3. HalconInspectionService.cs

负责：

* 接收图像路径或图像对象；
* 执行 Halcon 流程；
* 返回 InspectionResult；
* 管理模型和句柄初始化。

第一版建议流程尽量简单，例如：

* 读图
* 转灰度
* 做基础预处理
* 做一个基础区域提取或简单匹配
* 生成一组叠加结果
* 返回 OK / NG 和说明文字

### 4. HalconDisplayBridge.cs

负责：

* 把图像显示到 HSmartWindowControlWPF；
* 把 OverlayItem 转成叠加图层；
* 统一管理清空、刷新和重绘。

这层的意义是：

不让 ViewModel 直接变成控件操作器。

### 5. InspectionResult.cs

建议至少包含：

* IsOk
* Message
* ElapsedMilliseconds
* MeasureValue
* CodeText
* DefectCount
* ImagePath
* OverlayItems

这样后续页面展示和日志保存都会顺很多。

## 六、第一版示例工程建议实现什么检测流程

不要一开始就上最复杂的完整案例。

第一版建议只做下面两种之一：

### 方案 A：区域检测最小闭环

流程：

1. 加载图像；
2. 转灰度；
3. 阈值分割；
4. 连通域分析；
5. 筛选一个主区域；
6. 在图像上叠加区域轮廓；
7. 显示面积或中心坐标；
8. 输出 OK / NG。

优点是：

* 算法轻；
* 依赖少；
* 最容易先跑通界面闭环。

### 方案 B：匹配 + 跟随 ROI 最小闭环

流程：

1. 加载图像；
2. 模板匹配；
3. 输出位置和角度；
4. 根据姿态生成一个跟随 ROI；
5. 在 ROI 中做一个简单测量或区域判断；
6. 叠加匹配结果和 ROI；
7. 输出 OK / NG。

优点是：

* 更贴近正式项目；
* 能尽早验证“定位 + 跟随”的主线。

如果是第一次落工程，建议先做方案 A，再升级到方案 B。

## 七、从笔记到代码的映射关系

可以按下面方式理解：

### 20 -> 项目启动前置

先解决引用、位数、许可证和最小 Halcon 调用。

### 21 -> 主窗口显示区

先让主窗口能稳定显示图像。

### 23 -> View / ViewModel / Service 分层

先确定项目结构边界，不要写着写着回到后置代码堆砌模式。

### 24 -> 把检测流程塞进服务层

先让页面通过 ViewModel 调用服务，而不是自己跑流程。

### 25 -> 作为最终页面闭环验收

先按“能加载图、能检测、能显示结果、能保存”来验收第一版。

## 八、第一版页面建议长什么样

可以直接做成一个非常务实的布局：

```text
+------------------------------------------------------+
| 加载图像 | 开始检测 | 清空显示 | 保存结果图            |
+--------------------------+---------------------------+
|                          | 状态：就绪                |
|   HSmartWindowControl    | 结果：OK / NG             |
|                          | 面积 / 位置 / 耗时        |
|                          | 缺陷数 / 识别值           |
+--------------------------+---------------------------+
```

重点不是炫 UI，而是快速验证整条链路。

## 九、第一版最重要的验收标准

建议第一轮示例工程至少满足下面这些：

1. 程序能稳定启动并正确加载 Halcon 运行环境；
2. 图像能显示在 WPF 页面中；
3. 点击按钮后能执行一条 Halcon 检测流程；
4. 页面能同时显示文本结果和图像叠加结果；
5. 代码结构中 View、ViewModel、Service 边界基本清楚；
6. 至少能保存一张带结果的输出图或一条结果记录。

## 十、建议的下一步扩展顺序

第一版跑通后，再按下面顺序升级更稳：

1. 把方案 A 升级成方案 B
2. 引入 [17-多步骤检测流程设计](17-多步骤检测流程设计.md) 的流程拆层
3. 把 [22-Halcon图像显示与交互封装](22-Halcon图像显示与交互封装.md) 加进显示桥接层
4. 再把 [26-Halcon+WPF常见坑和排错](26-Halcon+WPF常见坑和排错.md) 作为项目排错清单使用

## 总结

1. 如果要从这套笔记反推第一个 Halcon + WPF 示例工程，最值得优先抓的是 20、21、23、24、25 这 5 篇；
2. 第一版项目目标应该是最小闭环，而不是一次做成大而全系统；
3. 推荐先用单窗口、单服务、单结果模型的方式把链路打通；
4. 只要“输入 -> 检测 -> 显示 -> 保存”跑通，这个示例工程就已经具备继续扩展的基础。
