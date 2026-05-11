# ADO.NET 与数据库连接

ADO.NET 是 .NET 中访问数据库的基础技术。不管是直接使用 ADO.NET 还是用 ORM（EF Core / Dapper），底层都是通过 ADO.NET 来连接和操作数据库的。

理解 ADO.NET 的核心对象，对于排查数据库连接问题、优化性能很有帮助。

## 一、ADO.NET 的核心对象

### 1. Connection（连接）

Connection 代表与数据库之间的物理连接。

```csharp
// SQL Server
using var conn = new SqlConnection(
    "Server=localhost;Database=InspectorDB;Integrated Security=True;");

// SQLite
using var conn = new SqliteConnection(
    "Data Source=InspectorDB.db");

// MySQL
using var conn = new MySqlConnection(
    "Server=localhost;Database=InspectorDB;User=root;Password=123456;");
```

### 2. Command（命令）

Command 代表要执行的 SQL 语句或存储过程。

```csharp
using var cmd = new SqlCommand(
    "SELECT COUNT(*) FROM InspectionResults WHERE IsPassed = 0",
    connection);
var count = (int)cmd.ExecuteScalar();
```

### 3. DataReader（数据读取器）

DataReader 是只进、只读的数据流，适合逐行读取大量数据。

```csharp
using var reader = cmd.ExecuteReader();
while (reader.Read())
{
    var id = reader.GetInt32(0);
    var name = reader.GetString(1);
    var passed = reader.GetBoolean(2);
    // 处理每一行
}
```

DataReader 的特点是：

- 按需读取，不一次性加载所有数据到内存；
- 读取期间连接是占用状态，不能做其他操作；
- 适合大量数据的读取，但读取期间会占用连接。

### 4. DataAdapter + DataSet（离线数据）

DataAdapter 把数据填充到 DataSet 中，填充后连接即可关闭，数据在内存中操作。

```csharp
var adapter = new SqlDataAdapter("SELECT * FROM Products", connection);
var dataSet = new DataSet();
adapter.Fill(dataSet, "Products");

// 此时连接已经释放，可以在内存中操作数据
var table = dataSet.Tables["Products"];
```

适合场景：数据量不大、需要离线操作的场景。

## 二、连接字符串

连接字符串是数据库连接的核心配置。

### SQL Server

```csharp
// Windows 身份验证
"Server=localhost;Database=InspectorDB;Integrated Security=True;"

// SQL Server 身份验证
"Server=192.168.1.100;Database=InspectorDB;User Id=sa;Password=xxx;"

// 连接池配置
"Server=localhost;Database=InspectorDB;Integrated Security=True;"
+ "Max Pool Size=100;Min Pool Size=10;Connection Timeout=15;"
```

### SQLite

```csharp
// 基本配置
"Data Source=InspectorDB.db"

// 启用 WAL 模式（提升并发性能）
"Data Source=InspectorDB.db;Pooling=True;"
```

### MySQL

```csharp
"Server=localhost;Database=InspectorDB;User=root;Password=xxx;"
```

## 三、连接池

连接池是 ADO.NET 中一个非常重要的机制。

### 连接池的工作原理

- 每次打开连接时，.NET 不会立即创建物理连接，而是先从连接池中查找可用的连接；
- 每次关闭连接时，连接不会真正销毁，而是回到连接池中等待复用；
- 这样可以避免频繁创建和销毁物理连接的开销。

### 连接池的关键参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| Max Pool Size | 池中最大连接数 | 100 |
| Min Pool Size | 池中最小保持的连接数 | 0 |
| Connection Timeout | 等待连接超时时间（秒） | 15 |
| Pooling | 是否启用连接池 | True |

### 连接池失效的常见原因

- 没有 using 释放连接，连接泄漏，池被耗尽；
- 连接字符串不同（比如拼了不同的参数），每个不同的连接字符串都会创建独立的池；
- 分布式事务或未完成的本地事务。

## 四、事务管理

### ADO.NET 本地事务

```csharp
using var connection = new SqlConnection(connectionString);
connection.Open();

using var transaction = connection.BeginTransaction();
try
{
    using var cmd = connection.CreateCommand();
    cmd.Transaction = transaction;
    
    cmd.CommandText = "INSERT INTO InspectionResults (...) VALUES (...)";
    cmd.ExecuteNonQuery();
    
    cmd.CommandText = "UPDATE ProductionCount SET Total = Total + 1";
    cmd.ExecuteNonQuery();
    
    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```

### 事务隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|------------|------|
| Read Uncommitted | 可能 | 可能 | 可能 |
| Read Committed（默认） | 避免 | 可能 | 可能 |
| Repeatable Read | 避免 | 避免 | 可能 |
| Serializable | 避免 | 避免 | 避免 |

在工控项目中，大多数场景使用默认的 Read Committed 就够了。只在需要严格控制时使用 Serializable。

## 五、异步操作

在 WPF 工控软件中，数据库操作不应该阻塞 UI 线程。ADO.NET 提供了异步方法：

```csharp
public async Task<List<InspectionResult>> GetResultsAsync(DateTime from, DateTime to)
{
    var results = new List<InspectionResult>();
    
    using var conn = new SqlConnection(connectionString);
    await conn.OpenAsync();
    
    using var cmd = new SqlCommand(
        "SELECT Id, ProductName, IsPassed, InspectTime FROM InspectionResults " +
        "WHERE InspectTime BETWEEN @from AND @to", conn);
    cmd.Parameters.AddWithValue("@from", from);
    cmd.Parameters.AddWithValue("@to", to);
    
    using var reader = await cmd.ExecuteReaderAsync();
    while (await reader.ReadAsync())
    {
        results.Add(new InspectionResult
        {
            Id = reader.GetInt32(0),
            ProductName = reader.GetString(1),
            IsPassed = reader.GetBoolean(2),
            InspectTime = reader.GetDateTime(3)
        });
    }
    
    return results;
}
```

异步操作的关键点：

- OpenAsync / ExecuteReaderAsync / ReadAsync 是异步版本；
- 在 WPF 中，配合 await 使用不会阻塞 UI 线程；
- 异步操作仍然占用连接池的连接。

## 六、using 与资源释放

ADO.NET 的所有核心对象都实现了 IDisposable，必须用 using 确保释放。

```csharp
// 好的写法：using 自动释放
using var conn = new SqlConnection(connectionString);
using var cmd = new SqlCommand(sql, conn);
using var reader = cmd.ExecuteReader();

// 不好的写法：没有用 using，可能导致连接泄漏
var conn = new SqlConnection(connectionString);
conn.Open();
// 忘记 Close / Dispose
```

连接泄漏是工控项目中最常见的数据库问题之一——长期运行后连接池被耗尽，程序报"超时时间已到"的错误。

## 七、选择 ADO.NET 还是 ORM

| 场景 | 推荐 |
|------|------|
| 需要大量手写 SQL 控制 | ADO.NET 原生 |
| 复杂查询、多表关联 | Dapper |
| 增量更新、对象跟踪、迁移 | EF Core |
| 快速原型 | EF Core |

在工控场景中，Dapper 是一个很常用的折中方案——既有接近 ADO.NET 的性能，又提供了对象映射。

## 八、连接测试示例

```csharp
public bool TestConnection()
{
    try
    {
        using var conn = new SqlConnection(connectionString);
        conn.Open();
        return true;
    }
    catch (Exception ex)
    {
        Logger.LogError($"数据库连接失败: {ex.Message}");
        return false;
    }
}
```

这个测试方法可以用在工控软件启动时的自检流程中。

## 总结

1. ADO.NET 的核心对象是 Connection、Command、DataReader 和 DataAdapter；
2. 连接池是提升性能的关键机制，但连接泄漏会让池失效；
3. 事务保证一组操作的一致性，WPF 中推荐使用异步操作避免阻塞 UI；
4. 所有 ADO.NET 对象都应该用 using 确保释放，这是避免连接泄漏的根本方法；
5. 在 ADO.NET、Dapper、EF Core 之间的选择取决于项目的复杂度和性能要求。
