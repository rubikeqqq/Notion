# SQL Server 详解

SQL Server 是微软推出的关系型数据库管理系统，也是 .NET / WPF 工控项目中最常用的后端数据库之一。

它和 Windows 生态、.NET 平台的整合深度是其他数据库无法比拟的。如果你的项目运行在 Windows 上，需要高性能、高可靠性的数据存储，SQL Server 通常是最自然的选择。

## 一、安装与配置

### 版本选择

| 版本 | 说明 | 适用场景 |
|------|------|----------|
| SQL Server Express | 免费，数据库最大 10GB | 小项目、开发测试 |
| SQL Server Standard | 商业版，无大小限制 | 中大型项目 |
| SQL Server Developer | 免费，仅开发测试 | 开发环境 |
| SQL Server LocalDB | 免费轻量版 | 单机应用、嵌入式 |

在工控项目中，大多数场景用 Standard 或 Express。如果是单机工控机，Express 就够用。

### 安装要点

1. 选择"数据库引擎服务"即可，不需要装 Analysis Services、Reporting Services 等额外组件；
2. 身份验证模式建议用"混合模式"（Windows + SQL Server 验证）；
3. 设置 sa 密码时注意复杂度要求；
4. 默认端口 1433，需要在防火墙中开放。

### 连接测试

```csharp
// 连接字符串格式
"Server=localhost;Database=InspectorDB;Integrated Security=True;"
"Server=192.168.1.100,1433;Database=InspectorDB;User Id=sa;Password=xxx;"
```

## 二、SSMS（SQL Server Management Studio）

SSMS 是 SQL Server 的官方管理工具，免费提供。

### 常用功能

- **对象资源管理器**：查看数据库、表、视图、存储过程等所有对象；
- **查询编辑器**：写 SQL 并执行，带智能提示和语法高亮；
- **表设计器**：可视化创建和修改表结构；
- **执行计划**：查看查询的执行计划，分析性能瓶颈；
- **备份/还原**：数据库的备份和还原操作。

### 工控中常用 SSMS 的场景

- 开发阶段建表、修改表结构；
- 排查数据问题（直接查看数据内容）；
- 分析慢查询的执行计划；
- 手动备份数据库。

## 三、T-SQL 特色语法

SQL Server 使用 T-SQL（Transact-SQL），在标准 SQL 基础上增加了许多功能。

### 常用内置函数

```sql
-- 时间函数
GETDATE()              -- 当前时间
DATEADD(DAY, -7, GETDATE())  -- 7天前
DATEDIFF(HOUR, start, end)   -- 时间差
FORMAT(GETDATE(), 'yyyy-MM-dd')  -- 格式化

-- 字符串函数
LEN('abc')             -- 长度
CHARINDEX('abc', '123abc456')  -- 查找位置
SUBSTRING('abcdef', 2, 3)     -- 截取
REPLACE('abc', 'a', 'x')      -- 替换

-- 类型转换
CAST(value AS type)
CONVERT(type, value)
```

### 局部变量

```sql
DECLARE @startTime DATETIME2 = '2024-01-01';
DECLARE @count INT;

SELECT @count = COUNT(*) FROM InspectionResults 
WHERE InspectTime > @startTime;

PRINT @count;
```

### 临时表

```sql
-- 创建临时表（只存在于当前会话）
CREATE TABLE #TempStats (
    ProductName NVARCHAR(50),
    TotalCount  INT
);

INSERT INTO #TempStats
SELECT ProductName, COUNT(*) 
FROM InspectionResults
GROUP BY ProductName;

-- 使用临时表
SELECT * FROM #TempStats;

-- 会话结束后自动删除
```

### TRY...CATCH

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    
    INSERT INTO InspectionResults (...) VALUES (...);
    INSERT INTO MeasurementResults (...) VALUES (...);
    
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION;
    
    SELECT 
        ERROR_NUMBER() AS ErrorNumber,
        ERROR_MESSAGE() AS ErrorMessage;
END CATCH
```

## 四、用户与权限

### 创建用户

```sql
-- 创建登录名
CREATE LOGIN inspector_user WITH PASSWORD = 'StrongPassword123!';

-- 创建数据库用户
USE InspectorDB;
CREATE USER inspector_user FOR LOGIN inspector_user;

-- 赋予权限（只读或读写）
EXEC sp_addrolemember 'db_datareader', 'inspector_user';   -- 只读
EXEC sp_addrolemember 'db_datawriter', 'inspector_user';   -- 写入
```

### 工控场景权限建议

- 运行时的 WPF 程序使用一个专门的数据库账户，只赋予它需要使用的最小权限；
- 管理和维护操作使用管理员账户；
- 不要使用 sa 账户连接业务程序。

## 五、备份与恢复

### 备份命令

```sql
-- 完整备份
BACKUP DATABASE InspectorDB 
TO DISK = 'D:\Backup\InspectorDB_20240101.bak';

-- 差异备份
BACKUP DATABASE InspectorDB 
TO DISK = 'D:\Backup\InspectorDB_Diff.bak'
WITH DIFFERENTIAL;
```

### 恢复命令

```sql
RESTORE DATABASE InspectorDB
FROM DISK = 'D:\Backup\InspectorDB_20240101.bak'
WITH REPLACE;
```

### 工控场景的备份策略

- 每天做一次完整备份（可以用 SQL Server Agent 定时任务）；
- 重要的检测数据建议备份到不同硬盘或网络位置；
- 定期验证备份文件的可恢复性。

## 六、与 .NET 配合

### SqlClient NuGet 包

```bash
dotnet add package Microsoft.Data.SqlClient
```

（注意：System.Data.SqlClient 已标记为过时，新项目使用 Microsoft.Data.SqlClient）

### EF Core 配置

```csharp
public class AppDbContext : DbContext
{
    public DbSet<InspectionResult> InspectionResults { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer(
            "Server=localhost;Database=InspectorDB;Integrated Security=True;");
    }
}
```

### Dapper 查询

```csharp
using var conn = new SqlConnection(connectionString);

// 参数化查询
var results = await conn.QueryAsync<InspectionResult>(@"
    SELECT Id, ProductName, IsPassed, InspectTime
    FROM InspectionResults
    WHERE InspectTime BETWEEN @from AND @to
    ORDER BY InspectTime DESC",
    new { from = startTime, to = endTime });
```

### SqlBulkCopy 批量写入

```csharp
// 高频写入场景使用
using var bulkCopy = new SqlBulkCopy(connectionString);
bulkCopy.DestinationTableName = "MeasurementResults";
bulkCopy.BatchSize = 1000;

var table = new DataTable();
table.Columns.Add("InspectId", typeof(long));
table.Columns.Add("ItemName", typeof(string));
table.Columns.Add("Value", typeof(double));

foreach (var item in measurements)
    table.Rows.Add(item.InspectId, item.ItemName, item.Value);

bulkCopy.WriteToServer(table);
```

## 七、性能调优思路

### 慢查询排查

```sql
-- 查看当前正在运行的查询
SELECT 
    r.session_id,
    r.status,
    r.command,
    t.text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.status = 'running';
```

### 索引建议

```sql
-- 查看缺失索引建议
SELECT 
    migs.avg_user_impact,
    mid.statement,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs 
    ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid 
    ON mig.index_handle = mid.index_handle;
```

### 常见优化手段

1. 给 WHERE 条件中的列加索引；
2. 避免在 WHERE 中使用函数（`WHERE DATE(Col) = ...` 会让索引失效）；
3. 用 `WITH (NOLOCK)` 在允许脏读的场景减少锁竞争；
4. 大表分页用 `OFFSET...FETCH` 代替 `ROW_NUMBER()`；
5. 批量写入用 SqlBulkCopy 而不是逐条 INSERT。

## 八、工控场景典型用法

### 连接池配置

```csharp
// 工控项目建议适当增加连接池大小
"Server=localhost;Database=InspectorDB;Integrated Security=True;" +
"Max Pool Size=200;Min Pool Size=10;Connection Timeout=15;"
```

### 断线重连

```csharp
public async Task ExecuteWithRetryAsync(Func<SqlConnection, Task> action)
{
    int maxRetries = 3;
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            using var conn = new SqlConnection(_connStr);
            await conn.OpenAsync();
            await action(conn);
            return;
        }
        catch (SqlException ex) when (i < maxRetries - 1)
        {
            await Task.Delay(1000 * (i + 1));  // 指数退避
        }
    }
}
```

### Windows 服务监视

SQL Server 作为 Windows 服务运行。如果服务停止，工控软件应该能检测到并给出提示：

```csharp
// 检查 SQL Server 服务状态
using var sc = new ServiceController("MSSQLSERVER");
bool isRunning = sc.Status == ServiceControllerStatus.Running;
```

## 总结

1. SQL Server 在 Windows / .NET 生态中集成度最高，是工控中大型项目的首选；
2. 使用 T-SQL 的 TRY...CATCH 和事务确保数据一致性；
3. 工控场景中高频写入使用 SqlBulkCopy，避免逐条 INSERT；
4. 新项目使用 Microsoft.Data.SqlClient，不要用已过时的 System.Data.SqlClient；
5. 定期备份 + 验证备份的可恢复性，是生产环境的基本要求；
6. 连接池配置和断线重连是工控软件健壮性的重要保障。
