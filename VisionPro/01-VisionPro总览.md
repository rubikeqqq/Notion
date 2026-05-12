# VisionPro 常用类型总览

这一组笔记主要围绕 Cognex VisionPro 中最常用的功能和工具展开。

VisionPro 是 Cognex 推出的机器视觉 SDK，提供了一套完整的视觉工具链，包括图像采集、定位、测量、识别、校准等功能。它和 OpenCV 这类通用视觉库最大的区别是：VisionPro 更聚焦于工业视觉场景，工具之间可以可视化串联，且提供了交互式的开发环境（Cognex Explorer）。

如果只看学习顺序，可以先这样理解：

1. [02-VisionPro与传统视觉开发对比](02-VisionPro与传统视觉开发对比.md)：先建立整体认知；
2. [03-CogToolBlock基础](03-CogToolBlock基础.md)：视觉工具容器，所有工具的载体；
3. [04-CogPMAlign定位工具](04-CogPMAlign定位工具.md)：图案定位，最常用的视觉工具；
4. [05-CogFindCircle与边缘检测](05-CogFindCircle与边缘检测.md)：边缘检测和测量工具；
5. [06-CogBlobTool斑点分析](06-CogBlobTool斑点分析.md)：区域分析和缺陷检测；
6. [07-CogFixtureTool坐标空间](07-CogFixtureTool坐标空间.md)：坐标变换和物理单位映射；
7. [08-CogCalibTool校准工具](08-CogCalibTool校准工具.md)：相机标定和畸变校正；
8. [09-CogAcqFifoTool图像采集](09-CogAcqFifoTool图像采集.md)：相机连接和图像获取；
9. [10-CogOCR与CogIDTool识别](10-CogOCR与CogIDTool识别.md)：字符识别和一维/二维码读取；
10. [11-多工具串联与CogToolBlock流程设计](11-多工具串联与CogToolBlock流程设计.md)：把多个工具串成一条检测流程；
11. [12-VisionPro与WPF集成](12-VisionPro与WPF集成.md)：在 WPF 中集成 VisionPro 控件和工具；
12. [13-完整检测案例](13-完整检测案例.md)：把定位、测量、识别串成一套完整案例；
13. [14-常见坑和排错清单](14-常见坑和排错清单.md)：最后用于快速定位"为什么不工作"。

## 一、各篇笔记分别讲什么

### 1. CogToolBlock

重点是：

* 什么是 CogToolBlock，它和直接使用单个工具的区别；
* 如何添加、连接和管理工具；
* 运行时如何访问工具输入输出；
* 什么时候该用 CogToolBlock，什么时候可以直接用单个工具。

适合在这里开始读：

[03-CogToolBlock基础](03-CogToolBlock基础.md)

### 2. CogPMAlign

重点是：

* 什么是图案定位，适用于哪些场景；
* 训练模板和运行时匹配的基本流程；
* 匹配结果如何解读（分数、位置、角度）；
* 和 CogFixtureTool 配合如何做到"一次定位，多次复用"。

[04-CogPMAlign定位工具](04-CogPMAlign定位工具.md)

### 3. CogFindCircle / CogFindLine

重点是：

* 边缘检测工具适合哪些场景；
* 卡尺（Caliper）参数如何影响检测结果；
* 圆弧和直线检测的基本流程；
* 测量结果如何转换为实际物理单位。

[05-CogFindCircle与边缘检测](05-CogFindCircle与边缘检测.md)

### 4. CogBlobTool

重点是：

* 斑点分析适合哪些场景（缺陷检测、面积测量）；
* 阈值分割和极性选择的区别；
* 如何根据斑点结果筛选和过滤；
* BlobTool 和 PMAlign 结合使用的常见模式。

[06-CogBlobTool斑点分析](06-CogBlobTool斑点分析.md)

### 5. CogFixtureTool

重点是：

* 为什么需要坐标空间；
* 如何将图像坐标映射到物理坐标；
* FixtureTool 如何接收定位结果并传递坐标系；
* 多工具串联时 Fixture 的正确用法。

[07-CogFixtureTool坐标空间](07-CogFixtureTool坐标空间.md)

### 6. CogCalibTool

重点是：

* 什么时候需要校准；
* 校准板的选择和标定流程；
* 校准结果如何应用到检测流程；
* 校准精度和常见问题。

[08-CogCalibTool校准工具](08-CogCalibTool校准工具.md)

### 7. CogAcqFifoTool

重点是：

* 相机如何连接到 VisionPro；
* 触发模式和采集模式的区别；
* 如何配置相机参数；
* 采集过程中常见的异常处理。

[09-CogAcqFifoTool图像采集](09-CogAcqFifoTool图像采集.md)

### 8. CogOCR / CogIDTool

重点是：

* 字符识别和条码读取的适用场景；
* OCR 训练和运行时使用的区别；
* 一维码和二维码读取的参数配置；
* 识别失败时的排错思路。

[10-CogOCR与CogIDTool识别](10-CogOCR与CogIDTool识别.md)

### 9. 多工具串联

重点是：

* 什么时候需要把多个工具串联起来；
* CogToolBlock 中工具的连接顺序和执行依赖；
* 如何设计工具链以支持不同产品型号；
* 工具链的性能考虑。

[11-多工具串联与CogToolBlock流程设计](11-多工具串联与CogToolBlock流程设计.md)

### 10. VisionPro 与 WPF 集成

重点是：

* CogDisplay 控件在 WPF 中的使用；
* 如何在代码中加载和执行工具；
* 采集、显示、检测三种模式的 UI 组织；
* 和 CommunityToolkit.Mvvm 如何配合。

[12-VisionPro与WPF集成](12-VisionPro与WPF集成.md)

### 11. 完整检测案例

重点是：

* 如何把定位、Fixture、测量、识别串成一条完整流程；
* 检测结果如何汇总和展示；
* NG/OK 判定逻辑如何组织；
* 检测日志和图像保存怎么处理。

[13-完整检测案例](13-完整检测案例.md)

## 二、如果按项目实践理解，可以这样分层

### 第一层：图像来源

* CogAcqFifoTool
* CogImageFileTool

这一层解决的是：
"图像从哪里来——相机采集还是读取文件。"

### 第二层：定位与坐标系

* CogPMAlign
* CogFixtureTool
* CogCalibTool

这一层解决的是：
"要检测的目标在哪里，如何将图像坐标对应到物理世界。"

### 第三层：检测与测量

* CogFindCircle / CogFindLine
* CogBlobTool
* CogOCR / CogIDTool

这一层解决的是：
"要检测什么——尺寸、缺陷、字符还是条码。"

### 第四层：工具容器与流程

* CogToolBlock

这一层解决的是：
"如何把检测流程组织成可复用、可配置的工具链。"

## 三、一个最实用的学习顺序

如果你是第一次接触 Cognex VisionPro，建议按下面顺序看：

1. [02-VisionPro与传统视觉开发对比](02-VisionPro与传统视觉开发对比.md)
2. [03-CogToolBlock基础](03-CogToolBlock基础.md)
3. [04-CogPMAlign定位工具](04-CogPMAlign定位工具.md)
4. [05-CogFindCircle与边缘检测](05-CogFindCircle与边缘检测.md)
5. [06-CogBlobTool斑点分析](06-CogBlobTool斑点分析.md)
6. [07-CogFixtureTool坐标空间](07-CogFixtureTool坐标空间.md)
7. [08-CogCalibTool校准工具](08-CogCalibTool校准工具.md)
8. [09-CogAcqFifoTool图像采集](09-CogAcqFifoTool图像采集.md)
9. [10-CogOCR与CogIDTool识别](10-CogOCR与CogIDTool识别.md)
10. [11-多工具串联与CogToolBlock流程设计](11-多工具串联与CogToolBlock流程设计.md)
11. [12-VisionPro与WPF集成](12-VisionPro与WPF集成.md)
12. [13-完整检测案例](13-完整检测案例.md)
13. [14-常见坑和排错清单](14-常见坑和排错清单.md)

这样顺序更自然，因为它基本符合从工具到流程、从单项功能到完整项目的学习路径。

## 四、常见组合方式

在实际项目里，最常见的组合通常是：

### 1. 单一定位 + 测量

* CogPMAlign
* CogFindCircle
* CogFixtureTool

例如：工件定位后测量孔径和边缘距离。

### 2. 图案定位 + 缺陷检测

* CogPMAlign
* CogFixtureTool
* CogBlobTool

例如：PCB 定位后检测焊点缺陷。

### 3. 定位 + 识别

* CogPMAlign
* CogFixtureTool
* CogOCR / CogIDTool

例如：产品定位后读取 DPM 码或字符。

### 4. 完整检测流程

* CogAcqFifoTool
* CogPMAlign
* CogFixtureTool
* CogFindCircle / CogBlobTool / CogOCR
* CogToolBlock

例如：采集图像 → 定位 → 坐标映射 → 多检测项 → 结果汇总。

## 总结

1. VisionPro 的常用工具可以分成四层：图像来源、定位与坐标系、检测与测量、工具容器与流程；
2. 先理解 PMAlign 和 CogToolBlock，再学其他工具会更顺畅；
3. 坐标变换（Fixture）是串联多个工具的关键，不要等到出问题才回头看；
4. 加上 WPF 集成和完整案例后，这一组内容已经覆盖了 VisionPro 在工业视觉项目中最常见的基础能力、工具串接、界面集成和排错理解。
