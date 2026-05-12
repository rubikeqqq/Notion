# ORM 框架对比

ORM（Object-Relational Mapping）解决的核心问题是：把数据库中的表和 C# 中的对象映射起来，让开发者可以用操作对象的方式来操作数据库。

在 .NET 生态中，最常用的三个选择是 EF Core、Dapper 和纯 ADO.NET。

## 一、三者的基本区别

| 特性 | EF Core | Dapper | ADO.NET |
| ------ | --------- | -------- | --------- |
| 类型 | 重量级 ORM | 轻量级 Micro ORM | 原生数据访问 |
| 学习成本 | 较高 | 低 | 低 |
| 代码量 | 最少 | 少 | 最多 |
| 性能 | 中等 | 接近原生 | 最高 |
| 对象跟踪 | 支持 | 不支持 | 不支持 |
| 自动迁移 | 支持 | 不支持 | 不支持 |
| 复杂查询 | LINQ 强类型 | 手写 SQL | 手写 SQL |
| Lazy Loading | 支持 | 不支持 | 不支持 |

## 二、EF Core

EF Core 是微软官方的 ORM 框架，也是 .NET 生态中最主流的数据库访问技术。

### EF Core 优点

1. **LINQ 查询**——可以用 C# 写强类型查询，编译期检查错误

    ```csharp
    var defects = await context.InspectionResults
        .Where(r => r.IsPassed == false 
                 && r.InspectTime > startDate)
        .OrderByDescending(r => r.InspectTime)
        .Select(r => new { r.ProductName, r.DefectType })
        .ToListAsync();
    ```

2. **自动迁移**——模型变化时自动更新数据库结构

    ```bash
    dotnet ef migrations add AddMeasurementTable
    dotnet ef database update
    ```

3. **Change Tracking**——跟踪对象的修改状态，SaveChanges 时自动生成 SQL

    ```csharp
    var result = await context.InspectionResults.FindAsync(id);
    result.Operator = "ZhangSan";  // 自动跟踪修改
    await context.SaveChangesAsync();  // 自动生成 UPDATE
    ```

4. **导航属性**——对象之间直接通过属性导航

    ```csharp
    public class InspectionResult
    {
        public int Id { get; set; }
        public ICollection<MeasurementResult> Measurements { get; set; }
    }

    // 使用
    var results = context.InspectionResults
        .Include(r => r.Measurements)
        .ToList();
    ```

### EF Core 缺点

1. 性能开销——EF Core 生成的 SQL 有时候不是最优的；
1. N+1 查询问题——不注意 Include 会产生大量查询；
1. 学习成本——需要理解延迟加载、导航属性、生命周期等概念。

### EF Core 适用场景

- 复杂业务逻辑、多表关联的项目；
- 需要频繁修改数据库结构的开发阶段；
- 团队对 SQL 不熟悉，更习惯 C# 思维。

## 三、Dapper

Dapper 是 Stack Overflow 团队开发的轻量级 Micro ORM。它本质上是对 ADO.NET 的扩展方法封装。

### Dapper 优点

1. **性能极高**——和手写 ADO.NET 几乎一样

    ```csharp
    // Dapper 查询
    var results = connection.Query<InspectionResult>(
        "SELECT * FROM InspectionResults WHERE InspectTime > @time",
        new { time = startDate }).ToList();
    ```

2. **简单直接**——就是写 SQL，没有迁移、跟踪、代理等概念

    ```csharp
    // 插入
    var id = connection.Execute(
        "INSERT INTO InspectionResults (ProductName, IsPassed) VALUES (@Name, @Passed)",
        new { Name = "PhoneA", Passed = true });

    // 查询单个对象
    var result = connection.QueryFirstOrDefault<InspectionResult>(
        "SELECT * FROM InspectionResults WHERE Id = @id", new { id = 1 });

    // 查询集合
    var list = connection.Query<InspectionResult>(sql);
    ```

3. **多结果集**

    ```csharp
    using var multi = connection.QueryMultiple(
        "SELECT * FROM InspectionResults WHERE Id = @id; " +
        "SELECT * FROM MeasurementResults WHERE InspectId = @id",
        new { id = 1 });

    var result = multi.Read<InspectionResult>().First();
    var measurements = multi.Read<MeasurementResult>().ToList();
    ```

### Dapper 缺点

1. 手写 SQL——没有 LINQ 的编译期检查，表名拼错了要运行时才知道；
1. 没有自动迁移——表结构变更需要手动处理；
1. 没有 Change Tracking——修改了对象需要自己写 UPDATE。

### Dapper 适用场景

- 对性能要求高的场景（高并发写入、大量数据查询）；
- 开发团队对 SQL 很熟悉；
- 简单的数据访问层，不需要复杂的对象关系映射。

## 四、Pure ADO.NET

直接使用 SqlConnection、SqlCommand、SqlDataReader。

### Pure ADO.NET 优点

- 完全控制 SQL 执行过程；
- 性能最高；
- 没有任何黑盒行为。

### Pure ADO.NET 缺点

- 代码量大，每个查询都要写 Connection → Command → Reader → 手动映射；
- 容易出错（忘记 using、忘记 Close）。

### Pure ADO.NET 适用场景

- 极高性能要求的场景；
- 简单的工具类或者批处理任务。

## 五、工控场景选型建议

### 推荐组合

在工控 WPF 项目中，常见的做法是**EF Core + Dapper 混合使用**：

- 用 EF Core 做标准的 CRUD 和维护开发效率（迁移、建表、简单查询）；
- 在性能关键的查询和批量写入路径上用 Dapper。

### 具体建议

| 场景 | 推荐方案 |
| ------ | ---------- |
| 标准 CRUD、页面列表 | EF Core |
| 检测结果批量写入（高频率） | Dapper 或 SqlBulkCopy |
| 报表查询、大量数据聚合 | Dapper |
| 开发和测试阶段（表结构频繁改） | EF Core 迁移 |
| 定时任务、后台批量操作 | Dapper |

### 混合使用示例

```csharp
public class InspectionRepository
{
    // EF Core 部分
    public async Task<List<InspectionResult>> GetTodayResults()
    {
        using var ctx = new AppDbContext();
        return await ctx.InspectionResults
            .Where(r => r.InspectTime > DateTime.Today)
            .ToListAsync();
    }
    
    // Dapper 部分（批量写入）
    public async Task BatchInsertMeasurements(List<MeasurementResult> items)
    {
        using var conn = new SqlConnection(_connString);
        await conn.ExecuteAsync(
            "INSERT INTO MeasurementResults (InspectId, ItemName, Value) " +
            "VALUES (@InspectId, @ItemName, @Value)",
            items);  // Dapper 支持批量参数
    }
}
```

## 六、性能对比速查

| 操作 | ADO.NET | Dapper | EF Core |
| ------ | --------- | -------- | --------- |
| 单条查询 | 最快 | ≈ ADO.NET | 慢 10-20% |
| 批量查询（1000 条） | 最快 | ≈ ADO.NET | 慢 20-30% |
| 批量插入（1000 条） | SqlBulkCopy 最快 | 慢于 BulkCopy | 最慢 |
| 复杂 JOIN 查询 | 手写最优 | 手写最优 | 可能生成低效 SQL |

在工控项目中，检测结果批量写入的性能通常是瓶颈，这部分推荐用 Dapper 或 SqlBulkCopy。

## 总结

1. EF Core 开发效率最高，适合标准 CRUD 和快速开发；
2. Dapper 性能接近 ADO.NET，适合高频率写入和复杂查询；
3. ADO.NET 原生提供了最大的控制权，但代码量最大；
4. 工控项目中推荐 EF Core + Dapper 混搭——各取所长；
5. 检测结果批量写入和高频采集场景不要用 EF Core 逐条插入，用 Dapper 或者 SqlBulkCopy。
