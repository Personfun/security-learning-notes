# 07 面试高频问题与标准答案

## Q1：简单介绍一下这个靶场的整体攻击链？
**A**：整体思路是：信息收集 → 源码泄露 → 获取数据库凭证 → 修改后台密码 → 登录后台 → 文件上传绕过 → 获取 WebShell → 提权 → 拿到三个 Flag。具体为：nmap 扫端口 → dirsearch 扫目录发现 `.git` 泄露 → 使用 git-dumper 下载源码拿到数据库密码 → 登录 phpMyAdmin 修改 admin 密码 → 登录 CMS 后台 → 在 upload.php 处利用 `.phar` 绕过上传 WebShell → 执行 `/readflag` 拿 Flag1 → 读取 `/flag-user` 拿 Flag2 → 利用 Redis 写 SSH 公钥提权到 root → 读取 `/root/flag` 拿 Flag3。

## Q2：源码泄露是怎么发现的？拿到了什么？
**A**：通过 dirsearch 目录扫描发现 `/.git/` 路径可访问。使用 `git-dumper` 工具将源码下载到本地。在 `application/database.php` 文件中，直接拿到了 MySQL 数据库的账号和密码，为后续登录 phpMyAdmin 打下基础。

## Q3：后台密码的加密方式是什么？加盐 MD5 的原理？
**A**：该 CMS 采用的是加盐 MD5，算法是 `md5(salt + password)`。
**核心逻辑**：先将盐值和密码拼接在一起，再对整体计算 MD5。而不是先 MD5 再加密。
**利用过程**：通过源码找到 `func_encrypt` 函数，确认算法；从数据库 `ey_config` 表查出盐值 `auth_code`；用 SQL 语句 `SELECT MD5(CONCAT((SELECT value FROM ey_config WHERE name='system_auth_code'), '123456'));` 算出新密码的哈希，并 UPDATE 到管理员表中。
**为什么要加盐**：防止彩虹表破解，同样的密码在不同系统会产生不同的哈希。

## Q4：文件上传是怎么绕过的？为什么用 `.phar`？
**A**：目标上传点只做了黑名单校验，拦截了 `.php` 和 `.phtml`。
尝试 `.php.jpg` 上传成功，但被强制改名为 `.jpg`，无法被 PHP 解析执行。
`.phar` 是 PHP 的归档格式（PHP Archive），默认情况下 PHP 引擎也会将其解析为 PHP 代码，由于黑名单没有包含它，所以绕过成功，成功拿到 WebShell。

## Q5：你怎么确认 Redis 是 root 权限运行的？
**A**：有两种方式：
1. `ps aux | grep redis` 查看进程运行用户，显示 root 就是 root 权限。
2. 直接测试：执行 `redis-cli config set dir /root/.ssh/`，如果能成功写入并返回 OK，说明 Redis 有 `/root/` 目录的写权限，也说明它是 root 运行的。我们在靶场中用的是方法2。

## Q6：Redis 提权的原理是什么？
**A**：Redis 提供了 `config set dir`（改写入目录）、`config set dbfilename`（改写入文件名）和 `save`（保存到磁盘）命令。
攻击者将目录改到 `/root/.ssh/`，文件名改成 `authorized_keys`，把 SSH 公钥写入 Redis，执行 save 后，公钥就被写到了 `/root/.ssh/authorized_keys`。随后用私钥 SSH 登录，直接拿到 root 权限。
**细节**：写入时公钥前后要加 `\n\n`，防止 RDB 二进制头影响 SSH 解析。

## Q7：如果 Redis 不是 root 运行，你还能提权吗？
**A**：可以尝试三种方式：
1. 写 WebShell 到 Web 目录（如果 Redis 用户对 Web 目录有写权限）。
2. 写 crontab 反弹 shell（写 `/var/spool/cron/crontabs/` 下对应用户的文件）。
3. 如果 Redis 是 4.x/5.x 版本，尝试主从复制 RCE：伪造主节点，让目标 Redis 同步并加载恶意 `.so` 模块，执行系统命令（Redis 6.x 已修复此漏洞）。

## Q8：如果源码被加密了，你还能找到密码算法吗？
**A**：可以尝试替代思路：
1. 动态测试：创建一个已知密码的账户，对比数据库哈希，反推算法。
2. 下载同版本源码，对比分析。
3. 搜索公开的漏洞分析文章。
4. 尝试通过其他漏洞（如 SQL 注入）直接改数据库密码。
