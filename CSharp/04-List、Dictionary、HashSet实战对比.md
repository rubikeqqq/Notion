# List、Dictionary、HashSet 实战对比

很多人第一次系统用集合时，最容易犯的错不是不会写 API，而是选错集合：

* 明明是按键查值，却一直用 `List<T>` 遍历；
* 明明只想去重，却还在手写循环判断；
* 明明数据天然有唯一键，却没想到 `Dictionary<TKey, TValue>`；
* 看见都是泛型集合，就不知道 `List<T>`、`Dictionary<TKey, TValue>`、`HashSet<T>` 到底该怎么分。

如果先记一句结论，可以先记这个：

**`List<T>` 适合顺序存放，`Dictionary<TKey, TValue>` 适合键值查找，`HashSet<T>` 适合去重和存在性判断。**

这篇笔记主要解决“实战里该怎么选”的问题，不追求讲底层源码，而是先把使用场景和判断逻辑理顺。

如果你想先看更高层的类型与集合地图，可以先看：

* [CSharp/02-常见基础类型与集合总览.md](02-%E5%B8%B8%E8%A7%81%E5%9F%BA%E7%A1%80%E7%B1%BB%E5%9E%8B%E4%B8%8E%E9%9B%86%E5%90%88%E6%80%BB%E8%A7%88.md)

---

## 一、先看三者最核心的区别

### 1. `List<T>`

`List<T>` 更像“可变长度数组”。

它最适合：

* 保持元素顺序；
* 按索引访问；
* 动态增加或删除元素。

例如：

```csharp
List<string> names = new List<string>();
names.Add("Tom");
names.Add("Jerry");

Console.WriteLine(names[0]);
```

### 2. `Dictionary<TKey, TValue>`

字典更像“键和值的映射表”。

它最适合：

* 通过键快速找值；
* 用唯一键标识对象；
* 组织配置、缓存、索引数据。

例如：

```csharp
Dictionary<int, string> users = new Dictionary<int, string>();
users[1] = "Alice";
users[2] = "Bob";

Console.WriteLine(users[1]);
```

### 3. `HashSet<T>`

`HashSet<T>` 更像“只关心元素是否存在的一组不重复值”。

它最适合：

* 去重；
* 判断某个值是否已经存在；
* 集合运算，例如交集、并集、差集。

例如：

```csharp
HashSet<string> tags = new HashSet<string>();
tags.Add("C#");
tags.Add("C#");

Console.WriteLine(tags.Count); // 1
```

---

## 二、List 最适合什么场景

### 1. 需要保留顺序

例如商品列表、消息列表、日志列表、表格行数据。

```csharp
List<string> messages = new List<string>
{
 "第一条",
 "第二条",
 "第三条"
};
```

你可以稳定地按顺序遍历它。

### 2. 需要按索引访问

例如：

```csharp
string first = messages[0];
```

如果你的数据访问逻辑天然是“第 0 个、第 1 个、第 2 个”，那 `List<T>` 很自然。

### 3. 元素数量不固定

数组长度固定，而 `List<T>` 可以动态增删。

```csharp
List<int> numbers = new List<int>();
numbers.Add(1);
numbers.Add(2);
numbers.Remove(1);
```

### 4. List 不适合什么

如果你总是写这种代码：

```csharp
User? user = users.FirstOrDefault(x => x.Id == id);
```

而且查找频率很高，那就要开始警惕：

**你也许该用字典，而不是一直在列表里扫。**

---

## 三、Dictionary 最适合什么场景

### 1. 通过键查对象

这是字典最典型的场景。

```csharp
Dictionary<int, User> userMap = new Dictionary<int, User>();
userMap[1001] = new User { Id = 1001, Name = "Alice" };

User user = userMap[1001];
```

如果你有明确的唯一键，比如：

* 用户 ID；
* 订单号；
* 配置名；
* 设备编码；

通常就该先想到 `Dictionary<TKey, TValue>`。

### 2. 做索引和缓存

例如把一批对象按 ID 建索引：

```csharp
Dictionary<int, Product> productMap = products.ToDictionary(x => x.Id);
```

这样后面通过 ID 找对象就不用反复遍历列表。

### 3. 配置和映射关系

例如：

```csharp
Dictionary<string, string> config = new Dictionary<string, string>
{
 ["Theme"] = "Dark",
 ["Language"] = "zh-CN"
};
```

### 4. Dictionary 使用时要注意什么

#### 第一，不要默认认为 key 一定存在

直接这样写：

```csharp
var user = userMap[id];
```

如果键不存在，会抛异常。

更稳的写法通常是：

```csharp
if (userMap.TryGetValue(id, out User? user))
{
 Console.WriteLine(user.Name);
}
```

#### 第二，键应该有明确唯一性

如果键本身不稳定、不唯一，字典就会用得很别扭。

---

## 四、HashSet 最适合什么场景

### 1. 去重

最经典的用途就是去重。

```csharp
List<int> numbers = new List<int> { 1, 2, 2, 3, 3, 3 };
HashSet<int> uniqueNumbers = new HashSet<int>(numbers);
```

### 2. 判断一个值是否已出现

例如：

```csharp
HashSet<string> visited = new HashSet<string>();

if (!visited.Contains(url))
{
 visited.Add(url);
}
```

这种“出现过没有”的问题，`HashSet<T>` 比 `List<T>` 更自然。

### 3. 集合运算

例如交集、并集：

```csharp
HashSet<int> set1 = new HashSet<int> { 1, 2, 3 };
HashSet<int> set2 = new HashSet<int> { 3, 4, 5 };

set1.IntersectWith(set2);
```

### 4. HashSet 不适合什么

如果你需要：

* 保持明确顺序；
* 通过索引取第几个元素；
* 一对一键值映射；

那 `HashSet<T>` 就不是最合适的选择。

---

## 五、一个最常见的实战对比

### 1. 用户列表展示

如果你只是要在界面上显示一组用户，并按顺序遍历：

* 更适合 `List<User>`

### 2. 根据用户 ID 快速找用户

如果你频繁通过用户 ID 查用户对象：

* 更适合 `Dictionary<int, User>`

### 3. 判断某个用户名是否重复

如果你只想判断用户名有没有出现过：

* 更适合 `HashSet<string>`

这三个场景本质上是在回答三个完全不同的问题：

* 顺序存放？
* 按键查找？
* 是否存在且不重复？

---

## 六、什么时候会从一个集合切换到另一个

实际项目里，经常不是“永远只用一种集合”，而是一个主数据结构加一个辅助索引结构。

例如：

```csharp
List<User> users = GetUsers();
Dictionary<int, User> userMap = users.ToDictionary(x => x.Id);
HashSet<string> userNames = users.Select(x => x.Name).ToHashSet();
```

这里：

* `List<User>` 负责顺序展示；
* `Dictionary<int, User>` 负责按 ID 查找；
* `HashSet<string>` 负责快速判断用户名是否已存在。

这才是更贴近实战的思路。

---

## 七、几个高频误区

### 1. 不要把 List 当万能集合

`List<T>` 最常用，但不是所有问题都该落到它上面。

### 2. 不要为了“高级”而乱用 Dictionary

如果你根本没有明确的键，字典反而会让结构变怪。

### 3. 不要忽略 HashSet

很多“去重”和“存在性判断”的代码还在手写循环，其实这正是 `HashSet<T>` 最擅长的地方。

### 4. 集合选择是由访问模式决定的

不要问“哪个集合最好”，要问：

**我的数据主要怎么用。**

---

## 八、总结

最后把这篇压成几句最重要的话。

### 1. 三种集合怎么分工

* `List<T>`：顺序存放、按索引访问；
* `Dictionary<TKey, TValue>`：按键查值；
* `HashSet<T>`：去重和存在性判断。

### 2. 最后记一套判断框架

你可以把集合选择记成下面三问：

* 我是不是要保留顺序？
* 我是不是要通过键查值？
* 我是不是主要关心是否存在和去重？

如果这三问先想清楚，`List<T>`、`Dictionary<TKey, TValue>`、`HashSet<T>` 基本就不容易选错。
