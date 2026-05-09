# CogOCR 与 CogIDTool 识别

在工业视觉项目中，除了定位和测量，识别也是一类非常常见的需求。

VisionPro 提供了两个主要的识别工具：

* **CogOCRMaxTool** —— 字符识别（OCR），识别印刷或刻印的文字；
* **CogIDTool** —— 读取一维码和二维码。

## 一、OCR 和 ID 的区别

| 能力 | CogOCRMaxTool | CogIDTool |
|------|--------------|-----------|
| 印刷字符 | 支持 | 不支持 |
| 一维码（Code 128、Code 39 等） | 不支持 | 支持 |
| 二维码（DataMatrix、QR Code 等） | 不支持 | 支持 |
| DPM 码（直接零件标识） | 有限支持 | 支持 |
| 需要训练 | 需要（训练字体） | 不需要（标准解码） |
| 字符不规则 | 可能困难 | 不适用 |

简单判断：

* 要读的是**"人眼能看懂的文字"** → 用 CogOCRMaxTool；
* 要读的是**"条码或二维码"** → 用 CogIDTool。

## 二、CogOCRMaxTool —— 字符识别

### 基本使用流程

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.OCRMax;

CogOCRMaxTool ocr = new CogOCRMaxTool();
ocr.InputImage = image;

// 设置字符区域
ocr.Region = charRegion;

// 加载训练好的字体文件
ocr.Font = CogSerializer.LoadObjectFromFile("font.ocf") as CogOCRMaxFont;

// 执行识别
ocr.Run();

// 读取结果
if (ocr.Results != null)
{
    string recognizedText = ocr.Results.Text;
    double confidence = ocr.Results.Confidence;

    Console.WriteLine($"识别结果：{recognizedText}，置信度：{confidence:F2}%");

    // 逐个字符检查
    foreach (CogOCRMaxCharResult charResult in ocr.Results.Chars)
    {
        Console.WriteLine($"字符：{charResult.Char}，置信度：{charResult.Confidence:F2}%");
    }
}
```

### 训练字体

OCRMax 使用前需要先训练字体。训练过程包括：

1. 采集包含待识别字符的图像；
2. 标记每个字符的位置和实际字符值；
3. 让工具学习字符的特征。

```csharp
// 创建字体训练器
CogOCRMaxFont font = new CogOCRMaxFont();

// 添加训练样本
CogOCRMaxCharSample sample = new CogOCRMaxCharSample();
sample.Image = charImage;
sample.Region = charRegion;
sample.Characters = "A";
font.AddSample(sample);

// 训练字体
font.Train();

// 保存字体文件
CogSerializer.SaveObjectToFile(font, "font.ocf");
```

在实际项目中，字体训练通常是一次性的。训练完成后将 .ocf 文件保存，运行程序时直接加载即可。

### 参数配置

```csharp
// 设置字符类型
ocr.RunParams.CharacterSet = CogOCRMaxCharacterSetConstants.Alphanumeric; // 字母数字
// 或使用自定义字符集
// ocr.RunParams.CharacterSet = CogOCRMaxCharacterSetConstants.UserDefined;

// 设置预期字符数（可选）
ocr.RunParams.NumberOfCharacters = 8;

// 设置置信度阈值
ocr.RunParams.AcceptThreshold = 60; // 低于 60% 视为不可靠
```

## 三、CogIDTool —— 条码 / 二维码读取

### 基本使用流程

```csharp
using Cognex.VisionPro;
using Cognex.VisionPro.ID;

CogIDTool idTool = new CogIDTool();
idTool.InputImage = image;

// 设置需要解码的码制
idTool.RunParams.CodeTypes = CogIDCodeTypeConstants.DataMatrix
                           | CogIDCodeTypeConstants.QRCode
                           | CogIDCodeTypeConstants.Code128;

// 设置搜索区域（可选，不设置则在全图搜索）
idTool.RunParams.SearchRegion = searchRegion;

// 执行解码
idTool.Run();

// 读取结果
if (idTool.Results != null)
{
    foreach (CogIDResult result in idTool.Results)
    {
        string decodedData = result.DecodedData;
        CogIDCodeType codeType = result.CodeType;

        Console.WriteLine($"码制：{codeType}，内容：{decodedData}");
    }
}
```

### 码制选择

```csharp
// 只读 DataMatrix
idTool.RunParams.CodeTypes = CogIDCodeTypeConstants.DataMatrix;

// 只读 QR Code
idTool.RunParams.CodeTypes = CogIDCodeTypeConstants.QRCode;

// 读取多种码制
idTool.RunParams.CodeTypes = CogIDCodeTypeConstants.DataMatrix
                           | CogIDCodeTypeConstants.Code128
                           | CogIDCodeTypeConstants.Code39;
```

注意：启用的码制越多，解码时间越长。如果事先知道码制，尽量只启用一种。

### DPM（直接零件标识）配置

```csharp
// 启用 DPM 模式（适配刻印、点阵等低对比度标识）
idTool.RunParams.DPM = true;

// 配置 DPM 参数
idTool.RunParams.DPMPreprocessingMode = CogIDDPMPreprocessingModeConstants.Auto;
```

金属上的激光刻印、塑料上的点阵标识等 DPM 码的对比度通常较低，开启 DPM 模式可以提高解码率。

## 四、实际识别案例

### 案例：读取 PCB 上的 DataMatrix 码

```csharp
public class DPMReader
{
    private CogIDTool _idTool = new CogIDTool();

    public DPMReader()
    {
        // 配置只读 DataMatrix
        _idTool.RunParams.CodeTypes = CogIDCodeTypeConstants.DataMatrix;
        _idTool.RunParams.DPM = true;
        _idTool.RunParams.Timeout = 3000; // 3 秒超时
    }

    public string ReadCode(CogImage8Grey image)
    {
        _idTool.InputImage = image;
        _idTool.Run();

        if (_idTool.Results != null && _idTool.Results.Count > 0)
        {
            return _idTool.Results[0].DecodedData;
        }

        return string.Empty;
    }
}
```

### 案例：读取产品序列号（OCR）

```csharp
public class SerialNumberReader
{
    private CogOCRMaxTool _ocr = new CogOCRMaxTool();

    public SerialNumberReader(string fontFile)
    {
        _ocr.Font = CogSerializer.LoadObjectFromFile(fontFile) as CogOCRMaxFont;
        _ocr.RunParams.CharacterSet = CogOCRMaxCharacterSetConstants.Alphanumeric;
        _ocr.RunParams.AcceptThreshold = 70;
    }

    public string ReadSN(CogImage8Grey image, CogRegion region)
    {
        _ocr.InputImage = image;
        _ocr.Region = region;
        _ocr.Run();

        if (_ocr.Results != null && _ocr.Results.Confidence >= 70)
        {
            return _ocr.Results.Text.Trim();
        }

        return string.Empty;
    }
}
```

## 五、识别和定位的配合

和测量工具一样，识别通常也跟在 FixtureTool 之后：

```csharp
// 1. 定位 + 坐标对齐
pmAlign.Run();
fixture.Fixture = pmAlign.Results[0].GetPose();
fixture.Run();

CogImage8Grey alignedImage = fixture.OutputImage as CogImage8Grey;

// 2. 在摆正后的图像上读取条码
idTool.InputImage = alignedImage;
idTool.Run();
```

这样做的优点是：

* 搜索区域可以在标准坐标位置配置；
* 不需要根据产品位置动态调整识别区域。

## 六、常见问题排查

### OCR 识别率低

* 字体训练样本是否充足（建议每个字符至少 5~10 个样本）；
* 光照是否稳定；
* 字符是否清晰、有无变形；
* 字符区域是否正确定位。

### ID 工具读不出码

* 码是否在图像中足够清晰；
* 码的尺寸是否合适（太大或太小都会影响解码）；
* 码制是否已启用；
* DPM 码是否开启了 DPM 模式；
* 是否有反光、阴影等干扰。

### 识别速度慢

* 是否启用了过多码制；
* 搜索区域是否过大；
* 图像分辨率是否过高；
* 对于 ID 工具，可以缩短超时时间。

## 七、注意事项

### 1. OCR 不是 100% 准确的

即使置信度很高，OCR 也有可能读错。关键场景建议：

* 加校验位或校验和；
* 结合光源和字符质量确保源头清晰；
* 对关键字符做二次确认。

### 2. ID 工具的超时设置

```csharp
// 设置超时，防止卡在难以解码的图像上
idTool.RunParams.Timeout = 2000; // 2 秒后超时返回
```

### 3. 条形码的朝向

一维码对朝向不敏感，但二维码（特别是 DataMatrix）需要清晰的对位图案。如果二维码有方向性要求，需要在图像中保证其可读性。

## 总结

1. CogOCRMaxTool 用于字符识别，需要先训练字体，适合印刷或刻印文字的读取；
2. CogIDTool 用于一维码和二维码解码，不需要训练，开箱即用；
3. 识别工具通常也跟在 PMAlign + FixtureTool 之后，保证识别区域跟随产品位置；
4. OCR 不是 100% 准确的，关键场景要考虑校验机制；
5. ID 工具要注意码制选择和超时设置，读 DPM 码时要开启 DPM 模式。
