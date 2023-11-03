### # XML注释

```csharp
<see cref="member"/>
<!-- or -->
<see cref="member">Link text</see>
<!-- or -->
<see href="link">Link Text</see>
<!-- or -->
<see langword="keyword"/>
```

- cref="member"：对可从当前编译环境调用的成员或字段的引用。 编译器检查是否存在给定的码位元素，并将 member 传递到输出 XML 中的元素名称。 将成员置于双引号 (" ") 内。 可以使用单独的结束标记为“cref”提供不同的链接文本。
- href="link"：指向给定 URL 的可单击链接。 例如，<see href="https://github.com">GitHub</see> 生成一个可单击的链接，其中包含文本 GitHub，该文本链接到 https://github.com。
- langword="keyword"：语言关键字，例如 true 或其他有效[关键字](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/)之一。

<see> 标记可用于从文本内指定链接。 使用 [](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/xmldoc/recommended-tags#seealso) 指示文本应该放在“另请参阅”部分中。 使用 [cref 属性](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/xmldoc/recommended-tags#cref-attribute)创建指向代码元素的文档页的内部超链接。 包含类型参数以指定对泛型类型或方法（如 cref="IDictionary{T, U}"）的引用。 此外，href 还是一个有效属性，将用作超链接。

| 标记           | 说明                                             |
| -------------- | ------------------------------------------------ |
| <c>            | 把行中的文本标记为代码，例如<c> int i = 10;</c>  |
| <code>         | 把多行标记为代码                                 |
| <example>      | 标记为一个代码示例                               |
| <include>      | 包含其他文档说明文件的注释（编译器要验证其语法） |
| <list>         | 把列表插入到文档中                               |
| <para>         | 建立文本的结构                                   |
| <param>        | 标记方法的参数（编译器要验证其语法）             |
| <paramref>     | 表明一个单词是方法的参数（编译器要验证其语法）   |
| <permission>   | 说明对成员的访问（编译器要验证其语法）           |
| <remarks>      | 给成员添加描述                                   |
| <returns>      | 说明方法的返回值                                 |
| <see>          | 提供对另一个参数的交叉引用（编译器要验证其语法） |
| <seealso>      | 提供描述中的“参见”部分（编译器要验证其语法）     |
| <summary>      | 提供类型或成员的简洁小结                         |
| <typeparam>    | 用在泛型类型的注释中，以说明一个参数类型         |
| <typeparamref> | 类型参数的名称                                   |
| <value>        | 描述属性                                         |