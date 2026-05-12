# SQL 基础语法与常用查询

不管用 SQL Server、SQLite 还是 MySQL，SQL 的基础语法是通用的。

这篇把最常用的 SQL 操作整理出来，以工控检测项目中的数据场景为例。

## 一、CRUD 基础

### 插入数据

```sql
INSERT INTO InspectionResults (ProductName, BatchNo, IsPassed, InspectTime)
VALUES ('PhoneA', 'BATCH001', 1, GETDATE());
```

批量插入：

```sql
INSERT INTO MeasurementResults (InspectId, ItemName, Value, Upper, Lower)
VALUES 
    (1, 'Length', 50.12, 50.05, 50.15),
    (1, 'Width',  30.08, 30.00, 30.10),
    (1, 'Height', 8.02,  7.95,  8.05);
```

### 查询数据

```sql
-- 查询所有列
SELECT * FROM InspectionResults;

-- 查询指定列
SELECT ProductName, IsPassed, InspectTime FROM InspectionResults;

-- 去重
SELECT DISTINCT ProductName FROM InspectionResults;
```

### 更新数据

```sql
UPDATE InspectionResults
SET Operator = 'ZhangSan'
WHERE Id = 100;
```

### 删除数据

```sql
DELETE FROM InspectionResults WHERE InspectTime < '2024-01-01';
```

## 二、WHERE 条件筛选

```sql
-- 比较运算符
SELECT * FROM InspectionResults
WHERE IsPassed = 0
  AND ProductName = 'PhoneA';

-- 时间范围
SELECT * FROM InspectionResults
WHERE InspectTime BETWEEN '2024-01-01' AND '2024-01-31';

-- 模糊匹配
SELECT * FROM Products WHERE ProductName LIKE 'Phone%';

-- IN 条件
SELECT * FROM InspectionResults
WHERE ProductName IN ('PhoneA', 'PhoneB', 'PhoneC');

-- NULL 判断
SELECT * FROM InspectionResults WHERE Operator IS NULL;
```

## 三、排序与分页

### 排序

```sql
SELECT ProductName, InspectTime, IsPassed
FROM InspectionResults
WHERE IsPassed = 0
ORDER BY InspectTime DESC;   -- 降序，最新的排最前面
```

### 分页

SQL Server：

```sql
SELECT * FROM InspectionResults
ORDER BY Id
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

SQLite / MySQL：

```sql
SELECT * FROM InspectionResults
ORDER BY Id
LIMIT 10 OFFSET 20;
```

## 四、聚合查询

```sql
-- 统计总数
SELECT COUNT(*) AS TotalCount FROM InspectionResults;

-- 分组统计
SELECT 
    ProductName,
    COUNT(*)                    AS TotalCount,
    SUM(CASE WHEN IsPassed = 0 THEN 1 ELSE 0 END) AS DefectCount,
    AVG(CASE WHEN IsPassed = 0 THEN 1.0 ELSE 0 END) * 100 AS DefectRate
FROM InspectionResults
WHERE InspectTime > '2024-01-01'
GROUP BY ProductName
ORDER BY TotalCount DESC;
```

常用的聚合函数：

| 函数 | 作用 | 适用 |
| ------ | ------ | ------ |
| COUNT() | 计数 | 统计记录数 |
| SUM() | 求和 | 数值列总和 |
| AVG() | 求平均 | 数值列平均值 |
| MIN()/MAX() | 最小/最大值 | 找出极值 |

## 五、JOIN 多表关联

工控项目中数据通常分散在多个表中，需要用 JOIN 把它们关联起来。

### INNER JOIN（内连接，取交集）

```sql
SELECT 
    r.Id,
    r.ProductName,
    r.IsPassed,
    m.ItemName,
    m.Value,
    m.Tolerance
FROM InspectionResults r
INNER JOIN MeasurementResults m ON r.Id = m.InspectId
WHERE r.InspectTime > '2024-01-01';
```

### LEFT JOIN（左连接，左表全保留）

```sql
-- 查出所有产品信息，即使有些产品没有检测记录
SELECT p.ProductName, r.InspectTime, r.IsPassed
FROM Products p
LEFT JOIN InspectionResults r ON p.Id = r.ProductId;
```

### 实际场景：检测结果汇总

```sql
SELECT 
    r.Id,
    p.ProductName,
    r.BatchNo,
    r.InspectTime,
    r.IsPassed,
    m.ItemName,
    m.Value,
    m.Tolerance,
    CASE WHEN m.Value BETWEEN m.Lower AND m.Upper 
         THEN 'OK' ELSE 'NG' END AS ItemStatus
FROM InspectionResults r
JOIN Products p ON r.ProductId = p.Id
JOIN MeasurementResults m ON r.Id = m.InspectId
ORDER BY r.InspectTime DESC;
```

## 六、子查询

```sql
-- 查出有不合格记录的批次
SELECT DISTINCT BatchNo
FROM InspectionResults
WHERE ProductName = 'PhoneA'
  AND IsPassed = 0;

-- 查出超过平均值的测量结果
SELECT * FROM MeasurementResults
WHERE Value > (SELECT AVG(Value) FROM MeasurementResults);
```

在工控项目中，子查询通常不如 JOIN 高效，大量数据下需要测试执行计划。

## 七、窗口函数（SQL Server / MySQL 8+）

窗口函数可以在不改变行数的情况下做聚合计算。

```sql
-- 对每个产品按时间排序，标记序号
SELECT 
    ProductName,
    InspectTime,
    IsPassed,
    ROW_NUMBER() OVER (
        PARTITION BY ProductName 
        ORDER BY InspectTime DESC
    ) AS RowNum
FROM InspectionResults;
```

窗口函数在"取每个产品最近一条检测记录"这类场景中非常有用。

## 八、C# 参数化查询

在 C# 中执行 SQL 时，不要拼接 SQL 字符串，使用参数化查询防止 SQL 注入。

### 好的写法（参数化）

```csharp
using var cmd = new SqlCommand(
    "SELECT * FROM InspectionResults WHERE ProductName = @name AND InspectTime > @time", 
    connection);
cmd.Parameters.AddWithValue("@name", productName);
cmd.Parameters.AddWithValue("@time", startTime);
```

### 不好的写法（字符串拼接）

```csharp
// 不要这样写！有 SQL 注入风险
var sql = $"SELECT * FROM InspectionResults WHERE ProductName = '{productName}'";
```

### 批量写入

工控场景中经常需要批量写入检测数据，使用表值参数可以大幅提升性能：

```csharp
// 使用 SqlBulkCopy 批量写入
using var bulkCopy = new SqlBulkCopy(connection);
bulkCopy.DestinationTableName = "MeasurementResults";
bulkCopy.WriteToServer(dataTable);
```

## 九、常用 SQL 速查

| 场景 | SQL |
| ------ | ----- |
| 查最新 N 条记录 | `SELECT TOP 10 * FROM table ORDER BY time DESC` |
| 按日期分组统计 | `GROUP BY CAST(time AS DATE)` |
| 去掉重复数据 | `SELECT DISTINCT col FROM table` |
| 条件计数 | `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` |
| 分页取数据 | `OFFSET x ROWS FETCH NEXT y ROWS ONLY` |
| 查找重复值 | `GROUP BY col HAVING COUNT(*) > 1` |

## 总结

1. CRUD 是 SQL 的基础，重点掌握 SELECT 的各种变体；
2. WHERE 条件、JOIN、GROUP BY 是工控查询中最常用的三个操作；
3. 参数化查询是 C# 操作数据库的底线要求，永远不要拼接 SQL；
4. 批量写入是工控场景的性能关键点，单条 INSERT 在大量数据下效率很低；
5. 窗口函数在复杂统计分析时非常有用，但 SQLite 不支持。
