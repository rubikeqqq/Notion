# SQLite 详解

SQLite 是一个嵌入式关系型数据库引擎。它不需要独立的数据库服务，数据库就是一个文件。

在工控 WPF 项目中，SQLite 是最常用的本地数据存储方案。因为它零配置、文件型、不需要安装任何数据库软件。

## 一、SQLite 的核心特点

### 嵌入式、零配置

SQLite 不是一个 C/S 架构的数据库。它以库文件的形式嵌入到应用程序中：

- 不需要安装数据库服务；
- 不需要配置数据库管理员；
- 不需要开放端口；
- 引用一个 NuGet 包就能开始用。

```bash
dotnet add package Microsoft.Data.Sqlite
```

### 数据库就是一个文件

SQLite 的整个数据库存储在单个文件中（通常后缀为 .db 或 .sqlite）：

- 备份就是复制文件；
- 迁移就是复制文件到新机器；
- 没有日志文件、没有控制文件、没有配置文件。

```csharp
// 连接 SQLite 数据库（文件不存在时会自动创建）
"Data Source=C:\InspectorData\InspectorDB.db"
```

### 服务器less

SQLite 直接读写磁盘文件，没有中间的服务进程。这意味着：

- 没有网络开销（比远程 SQL Server 快很多）；
- 没有服务管理（不用管服务启停）；
- 但同时也意味着：不能通过网络访问，只能由本地进程读写。

## 二、SQLite 在工控项目中的定位

SQLite 在工控软件中的最常见用法就是**本地数据存储**。

### 适合的场景

- 单机检测设备的本地数据库；
- 离线缓存（网络断开时的临时存储）；
- 配置文件、配方数据、检测日志的本地持久化；
- WPF 桌面应用的数据库。

### 不适合的场景

- 多台工控机需要实时共享数据（应该用 SQL Server）；
- 高并发写入（多个进程同时写同一个 SQLite 文件）；
- 大量数据需要复杂的用户权限管理。

## 三、与 .NET 配合

### NuGet 包

```bash
dotnet add package Microsoft.Data.Sqlite
```

（如果同时使用 EF Core，还需要 `Microsoft.EntityFrameworkCore.Sqlite`）

### ADO.NET 基本操作

```csharp
using var conn = new SqliteConnection("Data Source=InspectorDB.db");
conn.Open();

// 建表
using var cmd = conn.CreateCommand();
cmd.CommandText = @"
    CREATE TABLE IF NOT EXISTS InspectionResults (
        Id          INTEGER PRIMARY KEY AUTOINCREMENT,
        ProductName TEXT    NOT NULL,
        BatchNo     TEXT,
        IsPassed    INTEGER NOT NULL,
        InspectTime TEXT    NOT NULL
    )";
cmd.ExecuteNonQuery();

// 插入
cmd.CommandText = "INSERT INTO InspectionResults (ProductName, IsPassed, InspectTime) " +
                  "VALUES (@name, @passed, @time)";
cmd.Parameters.AddWithValue("@name", "PhoneA");
cmd.Parameters.AddWithValue("@passed", 1);
cmd.Parameters.AddWithValue("@time", DateTime.UtcNow.ToString("O"));
cmd.ExecuteNonQuery();

// 查询
cmd.CommandText = "SELECT * FROM InspectionResults";
using var reader = cmd.ExecuteReader();
while (reader.Read())
{
    var name = reader.GetString(1);
    var passed = reader.GetBoolean(2);
}
```

### EF Core 配置

```csharp
public class AppDbContext : DbContext
{
    public DbSet<InspectionResult> InspectionResults { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlite("Data Source=InspectorDB.db");
    }
}

// 迁移命令
// dotnet ef migrations add InitialCreate
// dotnet ef database update
```

### Dapper 查询

```csharp
using var conn = new SqliteConnection("Data Source=InspectorDB.db");

var results = conn.Query<InspectionResult>(
    "SELECT * FROM InspectionResults WHERE InspectTime > @time",
    new { time = startTime });
```

## 四、数据类型映射

SQLite 的类型系统和 SQL Server / MySQL 不太一样。

SQLite 只有 5 种存储类型（被称为存储类）：

| 存储类 | 对应 .NET 类型 | 说明 |
|--------|---------------|------|
| INTEGER | int, long | 整型 |
| REAL | float, double | 浮点型 |
| TEXT | string | 字符串 |
| BLOB | byte[] | 二进制 |
| NULL | null | 空值 |

### 类型亲和性

SQLite 在声明列类型时很灵活：

```sql
CREATE TABLE Test (
    Id    INTEGER,  -- 整数亲和
    Name  TEXT,     -- 文本亲和 
    Price REAL,     -- 实数亲和
    Data  BLOB,     -- 二进制亲和
    Any   ANYTYPE   -- 无亲和，什么都能存
);
```

在实际使用中，建议给列声明类型让代码更清晰。SQLite 会按声明的类型做自动转换。

### 关于布尔值和日期

SQLite 没有专门的 BOOL 和 DATETIME 类型：

- 布尔值：用 INTEGER，0 表示 false，1 表示 true；
- 日期时间：用 TEXT，存储 ISO 8601 格式字符串（如 "2024-01-01T12:00:00Z"）。

在 EF Core 中这些映射是自动处理的，直接使用 C# 的 bool 和 DateTime 即可。

## 五、WAL 模式（提升并发性能）

SQLite 的默认日志模式是 DELETE，每次写入时：

1. 把原始数据复制到回滚日志；
2. 修改数据库文件；
3. 删除回滚日志。

这种模式下写入时会锁住数据库文件（锁定整个文件），其他进程的读操作会被阻塞。

### WAL（Write-Ahead Logging）模式

WAL 模式改变了写入策略：

1. 修改写入到 WAL 文件，而不是直接写数据库；
2. 读取时可以同时读数据库和 WAL（读写不互斥）；
3. 定期将 WAL 文件合并回主数据库（Checkpoint）。

```sql
-- 启用 WAL 模式
PRAGMA journal_mode=WAL;

-- 在连接字符串中启用
"Data Source=InspectorDB.db;Pooling=True"
```

### 在 WAL 模式下验证

```csharp
using var conn = new SqliteConnection("Data Source=InspectorDB.db");
conn.Open();
using var cmd = conn.CreateCommand();
cmd.CommandText = "PRAGMA journal_mode=WAL";
cmd.ExecuteNonQuery();  // 返回 "wal"
```

### WAL 模式的注意事项

- 会额外生成一个 .wal 文件和一个 .shm 文件；
- 长时间不 checkpoint 会导致 WAL 文件过大；
- 建议定期执行 `PRAGMA wal_checkpoint(TRUNCATE)`。

## 六、并发限制与解决方案

### SQLite 的并发限制

SQLite 允许多个进程同时读取，但同一时间只允许一个进程写入。

在 WAL 模式下：

- 读读不冲突：多个连接可以同时读取；
- 读写不冲突：一个连接在写，其他连接可以读（但不能写）；
- 写写冲突：多个连接同时写会报 `SQLITE_BUSY`。

### 解决策略

```csharp
// 1. 重试机制
private void ExecuteWithRetry(SqliteConnection conn, string sql)
{
    const int maxRetries = 5;
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            using var cmd = conn.CreateCommand();
            cmd.CommandText = sql;
            cmd.ExecuteNonQuery();
            return;
        }
        catch (SqliteException ex) when (ex.SqliteErrorCode == 5 && i < maxRetries - 1)
        {
            // SQLITE_BUSY，等待后重试
            Thread.Sleep(100 * (i + 1));
        }
    }
}

// 2. 用锁确保同一时间只有一个写入操作
private readonly object _writeLock = new();

public void WriteData(Action<SqliteConnection> action)
{
    lock (_writeLock)
    {
        using var conn = new SqliteConnection(_connStr);
        conn.Open();
        action(conn);
    }
}
```

### 实际建议

对于工控单机软件，单进程操作 SQLite，通常不会遇到并发写入问题。

如果确实有多个进程需要同时写入本地数据，建议考虑用 SQL Server LocalDB 或改用 C/S 架构。

## 七、备份与维护

### 备份

SQLite 备份就是复制文件：

```csharp
// 热备份（在线备份）
public void BackupDatabase(string sourcePath, string backupPath)
{
    using var source = new SqliteConnection($"Data Source={sourcePath}");
    source.Open();
    
    // 使用 SQLite 的备份 API
    source.BackupDatabase(backupPath);
}
```

备份时需要注意：

- 使用 WAL 模式时，备份前最好执行一次 Checkpoint；
- 备份期间写入操作会被阻塞；
- 建议在系统空闲时做备份。

### 数据库维护

```sql
-- 重建索引（提升查询性能）
REINDEX;

-- 清理空闲空间
VACUUM;

-- 查看数据库完整性
PRAGMA integrity_check;
```

### 数据库大小监控

```sql
-- 查看数据库大小
SELECT page_count * page_size AS size_bytes
FROM pragma_page_count(), pragma_page_size();
```

## 八、工控场景典型用法

### WPF 本地数据存储

在单机检测设备中，SQLite 是最常用的本地存储方案：

```csharp
// 数据库文件放在应用数据目录
var dbPath = Path.Combine(
    AppDomain.CurrentDomain.BaseDirectory, 
    "Data", 
    "InspectorDB.db");

// 确保目录存在
Directory.CreateDirectory(Path.GetDirectoryName(dbPath));

// 连接字符串
var connStr = $"Data Source={dbPath};Pooling=True";
```

### 读写缓存

SQLite 作为本地缓存，定时上传到服务器：

```csharp
public class LocalCache
{
    private readonly string _connStr;
    
    public LocalCache(string dbPath)
    {
        _connStr = $"Data Source={dbPath};Pooling=True";
        InitializeDatabase();
    }
    
    private void InitializeDatabase()
    {
        using var conn = new SqliteConnection(_connStr);
        conn.Open();
        
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            CREATE TABLE IF NOT EXISTS PendingUploads (
                Id          INTEGER PRIMARY KEY AUTOINCREMENT,
                Data        TEXT    NOT NULL,
                CreatedAt   TEXT    NOT NULL,
                IsUploaded  INTEGER DEFAULT 0
            )";
        cmd.ExecuteNonQuery();
    }
    
    public void SaveForUpload(string jsonData)
    {
        using var conn = new SqliteConnection(_connStr);
        conn.Open();
        
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            INSERT INTO PendingUploads (Data, CreatedAt)
            VALUES (@data, @time)";
        cmd.Parameters.AddWithValue("@data", jsonData);
        cmd.Parameters.AddWithValue("@time", DateTime.UtcNow.ToString("O"));
        cmd.ExecuteNonQuery();
    }
}
```

## 九、SQLite 限制

1. 不支持存储过程；
2. 不支持用户权限管理（靠文件权限）；
3. 不支持 ALTER TABLE 的很多操作（如 DROP COLUMN 需要 SQLite 3.35+）；
4. 并发写入能力有限（WAL 模式下有改善，但仍是单写入者）；
5. 没有网络协议（不能远程连接，只能本地访问）；
6. 窗口函数在 SQLite 3.25+ 才支持。

## 总结

1. SQLite 是嵌入式关系型数据库，零配置、文件型存储，最适合工控单机设备的本地数据存储；
2. 在 WPF 项目中通过 Microsoft.Data.Sqlite 或 EF Core 使用，开发和部署都非常简单；
3. WAL 模式可以显著提升并发性能，建议在工控场景中启用；
4. SQLite 的并发写入能力有限，多进程同时写同一个数据库文件可能导致 SQLITE_BUSY；
5. 数据库就是一个文件，备份即复制，这既是优势（简单）也是局限（不能远程访问）；
6. 工控项目中典型的架构是 SQLite 本地缓存 + SQL Server 远程汇总。
