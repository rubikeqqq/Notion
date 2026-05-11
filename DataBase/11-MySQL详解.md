# MySQL 详解

MySQL 是当前最流行的开源关系型数据库管理系统之一，由 Oracle 公司维护。

在工控项目中，MySQL 的使用场景通常是：作为远程服务器数据库，接收多个工控机的数据上报，或者作为 MES 系统的数据库。在 Windows 工控生态中它不如 SQL Server 常见，但在跨平台或开源的场景中它是首选。

## 一、安装与配置

### Windows 安装

MySQL Installer 是最方便的安装方式（从官网下载）：

- 选择"Server only"或"Developer Default"；
- 安装过程中会要求设置 root 密码；
- 默认端口 3306；
- 建议选择"作为 Windows 服务运行"，开机自启。

```sql
-- 安装完成后检查服务状态
-- Windows: net start MySQL80
```

### Linux 安装

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install mysql-server
sudo mysql_secure_installation

# 启动服务
sudo systemctl start mysql
sudo systemctl enable mysql
```

### 连接测试

```bash
mysql -u root -p -h localhost
```

```csharp
// C# 连接字符串
"Server=192.168.1.100;Port=3306;Database=InspectorDB;User=root;Password=xxx;"

// 推荐使用 MySqlConnector（性能优于 Oracle 官方包）
dotnet add package MySqlConnector
```

## 二、存储引擎

MySQL 的一个重要特性是支持多种存储引擎。最常用的是 InnoDB 和 MyISAM。

### InnoDB（推荐）

从 MySQL 5.5 开始，InnoDB 是默认存储引擎。

特点：

- 支持事务（ACID）；
- 支持外键约束；
- 支持行级锁（高并发场景表现好）；
- 崩溃恢复能力强。

在工控项目中，绝大多数场景应该用 InnoDB。

### MyISAM

早期 MySQL 的默认引擎。

特点：

- 不支持事务；
- 只支持表级锁（并发写入时性能差）；
- 读取速度快（某些场景比 InnoDB 快）；
- 崩溃后需要修复表。

现在基本只在只读的历史数据归档场景中使用。

### 查看和指定存储引擎

```sql
-- 查看默认引擎
SHOW ENGINES;

-- 建表时指定引擎
CREATE TABLE InspectionResults (
    Id INT PRIMARY KEY AUTO_INCREMENT,
    ProductName VARCHAR(50) NOT NULL
) ENGINE=InnoDB;

-- 修改已有表的引擎
ALTER TABLE InspectionResults ENGINE=InnoDB;
```

**在工控项目中，始终使用 InnoDB。**

## 三、T-SQL vs MySQL SQL 语法差异

MySQL 的 SQL 语法和 SQL Server 有一些差异，需要注意：

### 自增列

```sql
-- SQL Server
Id INT IDENTITY(1,1)

-- MySQL
Id INT AUTO_INCREMENT
```

### 分页

```sql
-- SQL Server
SELECT * FROM InspectionResults
ORDER BY Id
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- MySQL
SELECT * FROM InspectionResults
ORDER BY Id
LIMIT 10 OFFSET 20;
```

### 时间函数

```sql
-- SQL Server
GETDATE()
DATEADD(DAY, -7, GETDATE())

-- MySQL
NOW()
DATE_SUB(NOW(), INTERVAL 7 DAY)
```

### 字符串拼接

```sql
-- SQL Server
SELECT 'A' + 'B';

-- MySQL
SELECT CONCAT('A', 'B');
```

### TOP vs LIMIT

```sql
-- SQL Server
SELECT TOP 10 * FROM Products;

-- MySQL
SELECT * FROM Products LIMIT 10;
```

在 EF Core 或 Dapper 中，这些差异通常被 ORM 屏蔽了。但如果手写 SQL 或者做数据库迁移，需要注意。

## 四、用户与权限管理

### 创建用户

```sql
-- 创建用户
CREATE USER 'inspector_app'@'192.168.1.%' IDENTIFIED BY 'StrongPassword123!';

-- 授权（只给必要的最小权限）
GRANT SELECT, INSERT, UPDATE ON InspectorDB.* TO 'inspector_app'@'192.168.1.%';

-- 刷新权限
FLUSH PRIVILEGES;
```

### 赋予远程访问

```sql
-- 允许远程访问（注意安全风险）
CREATE USER 'inspector_app'@'%' IDENTIFIED BY 'StrongPassword123!';
GRANT SELECT, INSERT ON InspectorDB.* TO 'inspector_app'@'%';
FLUSH PRIVILEGES;
```

安全建议：

- 限制来源 IP（`@'192.168.1.%'` 只允许内网）；
- 不要给业务程序 root 权限；
- 只赋予业务需要的最小权限（通常只需要 SELECT、INSERT、UPDATE）。

## 五、主从复制

MySQL 支持主从复制（Master-Slave Replication），在工控项目中可以用于：

- 读写分离（主库写，从库读）；
- 数据备份（从库实时备份）；
- 远程同步。

### 基本配置

主库（my.cnf）：

```ini
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_do_db = InspectorDB
```

从库（my.cnf）：

```ini
[mysqld]
server-id = 2
relay_log = mysql-relay-bin
replicate_do_db = InspectorDB
```

### 在从库上执行

```sql
CHANGE MASTER TO
    MASTER_HOST='192.168.1.100',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=0;

START SLAVE;

-- 查看复制状态
SHOW SLAVE STATUS\G
```

### 工控场景建议

主从复制在需要数据高可用和远程灾备的工控场景中很有用，但配置和维护有一定复杂度。如果是单服务器的小型项目，不需要主从复制。

## 六、与 .NET 配合

### MySqlConnector（推荐）

```bash
dotnet add package MySqlConnector
```

使用：

```csharp
using var conn = new MySqlConnection(
    "Server=192.168.1.100;Database=InspectorDB;User=inspector_app;Password=xxx;");
await conn.OpenAsync();

// 查询
using var cmd = new MySqlCommand(
    "SELECT COUNT(*) FROM InspectionResults WHERE InspectTime > @time", conn);
cmd.Parameters.AddWithValue("@time", startTime);
var count = (int)await cmd.ExecuteScalarAsync();
```

### Oracle官方包

```bash
dotnet add package MySql.Data
```

MySqlConnector 的性能通常优于 Oracle 官方包，推荐使用。

### EF Core 配置

```bash
dotnet add package Pomelo.EntityFrameworkCore.MySql
```

```csharp
public class AppDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseMySql(
            "Server=localhost;Database=InspectorDB;User=root;Password=xxx;",
            ServerVersion.AutoDetect("Server=localhost;Database=InspectorDB;User=root;Password=xxx;"));
    }
}
```

### Dapper

```csharp
using var conn = new MySqlConnection(connectionString);

var results = await conn.QueryAsync<InspectionResult>(
    "SELECT * FROM InspectionResults WHERE InspectTime > @time",
    new { time = startTime });
```

## 七、备份与恢复

### 命令行备份

```bash
# 备份（mysqldump）
mysqldump -u root -p InspectorDB > InspectorDB_20240101.sql

# 恢复
mysql -u root -p InspectorDB < InspectorDB_20240101.sql
```

### 二进制日志备份（用于增量恢复）

```bash
# 查看二进制日志列表
mysqlbinlog mysql-bin.000001 > binlog1.sql
```

### 备份策略

- 每天一次全量备份（mysqldump）；
- 可选：开启二进制日志用于增量恢复；
- 备份文件压缩存储（mysqldump 生成的 SQL 文件通常可以压缩到 10-20% 的大小）。

## 八、管理工具

### MySQL Workbench

官方提供的免费 GUI 工具。

常用功能：

- 数据库设计（ER 图）；
- SQL 编辑和执行；
- 服务器状态监控；
- 备份和恢复；
- 用户管理。

### CLI 命令行

```bash
mysql -u root -p

-- 查看数据库列表
SHOW DATABASES;

-- 选择数据库
USE InspectorDB;

-- 查看表列表
SHOW TABLES;

-- 查看表结构
DESCRIBE InspectionResults;

-- 查看慢查询
SHOW FULL PROCESSLIST;
```

## 九、工控场景中的定位

### 什么时候用 MySQL

- 项目需要跨平台支持（Windows + Linux 混合部署）；
- 项目是开源或低成本方案，不想支付 SQL Server 授权费；
- 后端有 Web 服务需要访问数据库（PHP/Java/Node.js 生态 MySQL 支持最好）；
- 工控软件 + Web 报表系统的组合。

### 什么时候不用 MySQL

- 项目完全在 Windows / .NET 生态内，没有跨平台需求 → SQL Server 更省心；
- 单机设备不需要远程数据库 → SQLite 更简单；
- 需要深度整合 Windows 域验证、Windows 服务等微软生态特性 → SQL Server。

### 工控混搭架构示例

```
工控机 A（SQLite 本地） ──→   
工控机 B（SQLite 本地） ──→   MySQL 服务器（数据汇总） ←→ Web 报表系统
工控机 C（SQLite 本地） ──→
```

## 十、常见问题

### 字符编码

MySQL 的字符集配置不当时容易出现乱码问题。

```sql
-- 建库时指定 utf8mb4（支持 emoji 和所有 Unicode 字符）
CREATE DATABASE InspectorDB 
DEFAULT CHARACTER SET utf8mb4 
DEFAULT COLLATE utf8mb4_unicode_ci;

-- 连接字符串中指定编码
"Server=localhost;Database=InspectorDB;User=root;Password=xxx;CharSet=utf8mb4"
```

### 连接超时

MySQL 默认的 wait_timeout 是 8 小时。连接空闲超过这个时间会被服务器断开。

工控软件长时间空闲后操作数据库可能报连接错误，建议：

1. 使用连接池（自动维护连接有效性）；
2. 在连接字符串中设置 `Connection Lifetime` 或 `Connection Idle Timeout`。

## 总结

1. MySQL 是开源的关系型数据库，适合跨平台和开源场景；
2. 工控场景中始终使用 InnoDB 存储引擎，支持事务和行级锁；
3. MySQL 的 SQL 语法和 SQL Server 在一些细节上有差异（分页、时间函数、自增列等），迁移时注意；
4. C# 中推荐使用 MySqlConnector + EF Core (Pomelo) 或 Dapper；
5. 字符集使用 utf8mb4 避免乱码问题；
6. MySQL 在工控项目中通常扮演远程汇总数据库的角色，和本地 SQLite 配合使用。
