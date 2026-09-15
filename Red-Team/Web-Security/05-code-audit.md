# 代码审计学习笔记

代码审计（白盒测试）是通过阅读源码，从代码层面发现安全漏洞的过程。相比黑盒渗透，代码审计能更精准地定位漏洞成因，也是安全工程师的核心竞争力之一。

## 1. 核心思想

代码审计的本质是**追踪数据流**：找到用户可控的输入（Source），顺着代码逻辑追踪，看它最终有没有安全地到达危险函数（Sink）。

```text
用户输入 (Source) → 变量传递 (Filter/Logic) → 危险函数 (Sink)
```

## 2. 审计方法论（通用流程）

1. **找入口（Source）**：搜索全局变量 `$_GET`、`$_POST`、`$_REQUEST`、`$_COOKIE`，这些是用户可控的输入点。
2. **追踪变量**：顺着变量在代码里的赋值、传递过程，看是否经过了过滤函数（如 `htmlspecialchars`、`is_numeric`、`addslashes`）。
3. **定位危险函数（Sink）**：
   - SQL注入：`mysql_query`、`mysqli_query`、`->query`（未预编译）
   - 命令执行：`system`、`exec`、`shell_exec`、`passthru`、`popen`
   - 代码执行：`eval`、`assert`、`create_function`、`preg_replace` /e
   - 文件包含：`include`、`require`、`include_once`
   - 文件操作：`file_get_contents`、`fopen`、`unlink`、`move_uploaded_file`
4. **判断过滤机制**：分析过滤函数是黑名单还是白名单，是否能被绕过。

## 3. 常见漏洞的审计重点

### SQL 注入审计
- 查找直接拼接 SQL 字符串的代码（如 `"SELECT * FROM users WHERE id = $id"`）。
- 检查是否使用参数化查询（Prepared Statements）。如果使用 PDO 且 `bindParam`，则相对安全。
- 注意二次注入：数据入库时被转义，取出时未转义又拼接到新 SQL 中。

### 文件包含审计
- 查找 `include($_GET['file'])` 这类用户可控的包含点。
- 检查是否使用了白名单（`in_array`）、`open_basedir` 限制。

### 命令执行审计
- 查找 `system($cmd)` 且 `$cmd` 来自用户输入。
- 检查是否使用了 `escapeshellarg` 和 `escapeshellcmd`。

### 文件上传审计
- 查看 `move_uploaded_file` 逻辑，检查后缀白名单、文件重命名、上传目录执行权限限制。

## 4. 常用工具与命令

### grep 命令行审计
```bash
# 搜索危险函数
grep -rn "system\|exec\|shell_exec\|eval" ./src/

# 搜索用户输入
grep -rn "\$_GET\|\$_POST\|\$_REQUEST" ./src/

# 组合搜索（先找输入，再看是否流向危险函数）
grep -rn "include\|require" ./src/ | grep "\$_GET"
```

### 辅助工具
- **Seay 源代码审计系统**：图形化，自动扫描常见漏洞。
- **RIPS**：专业的 PHP 代码审计工具。
- **CodeQL**：语义代码分析引擎，适合大型项目深度审计。

## 5. 实战复盘（结合 DVWA 与真实 CMS 审计）

- **DVWA**：通过对比 Low、Medium、High、Impossible 四难度源码，理解黑名单与白名单的区别。
  - 命令注入 High 过滤了 `&&` 和 `;`，但漏了 `%0a`（换行符）。
  - 文件包含 Medium 用 `str_replace` 删除 `../`，可通过双写绕过。
- **真实 CMS 审计**：通过 `grep` 批量搜索危险函数，发现某个二次注入点（`UPDATE ... WHERE id='$id'` 未预编译）。结论是主流 CMS 核心功能已做防护，0day 往往藏在业务逻辑中。

## 6. 面试高频问题

**Q1：代码审计的通用流程是什么？**
A：核心是追踪数据流：找入口（`$_GET`、`$_POST`）→ 追踪变量传递 → 定位危险函数（`system`、`eval`、`query`）→ 判断过滤机制（黑/白名单）。可以使用 `grep` 命令行快速筛选，也可以用 Seay、RIPS、CodeQL 等工具辅助。

**Q2：黑名单和白名单在代码审计中怎么区分？**
A：黑名单是“过滤已知的坏字符”（如 `str_replace` 删除 `../`），白名单是“只允许已知的好字符”（如 `in_array` 精确匹配）。黑名单容易绕过（双写、编码、大小写），白名单更安全。

**Q3：二次注入的原理是什么？怎么审计？**
A：二次注入是指恶意数据在第一次入库时被转义（安全），但在后续从数据库取出并再次拼接进 SQL 语句时未做处理，导致注入。审计时重点关注“从数据库读数据 → 拼接到新 SQL”的代码逻辑。
4. **域渗透**（Kerberoasting、DCSync、黄金票据）

我们下一篇写 **`Red-Team/Privilege-Escalation/01-linux-privesc.md`**（Linux 提权），把你之前学过的 SUID、Sudo、Cron、内核漏洞等内容系统化。准备好了吗？回复「继续」，我们开始攻坚提权模块！
