# Linux 日志分析与应急响应排查

Linux 服务器通常扮演着 Web 服务器、数据库服务器、跳板机的角色。掌握 Linux 日志分析，是蓝队发现入侵、溯源攻击者的关键技能。

## 1. 核心日志文件概览

| 日志路径 | 适用发行版 | 核心记录内容 |
|:---|:---|:---|
| `/var/log/secure` | CentOS/RHEL | 认证、授权、SSH 登录记录 |
| `/var/log/auth.log` | Ubuntu/Debian | 认证、授权、SSH 登录记录 |
| `/var/log/messages` | CentOS/RHEL | 系统全局日志（内核、服务） |
| `/var/log/syslog` | Ubuntu/Debian | 系统全局日志 |
| `/var/log/cron` | 通用 | 计划任务执行日志 |
| `/var/log/apache2/access.log` | Debian 系 Web | Web 访问日志 |
| `/var/log/nginx/access.log` | Nginx | Web 访问日志 |
| `~/.bash_history` | 通用 | 用户命令历史记录 |

## 2. SSH 登录与爆破排查

### 关键日志分析
```bash
# 查看失败的 SSH 登录尝试
grep "Failed password" /var/log/secure
grep "Failed password" /var/log/auth.log

# 查看成功的 SSH 登录记录
grep "Accepted password" /var/log/secure
grep "Accepted password" /var/log/auth.log

# 统计爆破源 IP（按出现次数排序）
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
```

### 防御建议
- 部署 `fail2ban` 自动封禁频繁失败的 IP。
- 修改 SSH 默认端口（如改成 222），关闭 root 密码登录，仅允许密钥登录。

## 3. Web 访问日志分析与 WebShell 排查

### 常见攻击特征
```bash
# 查找 SQL 注入尝试
grep -E "UNION|SELECT|OR.*=.*=" /var/log/apache2/access.log

# 查找目录扫描（大量 404）
awk '{print $9}' /var/log/apache2/access.log | sort | uniq -c | sort -nr

# 查找文件上传尝试
grep -E "POST.*\.php" /var/log/apache2/access.log | grep -E "upload|file"
```

### WebShell 排查思路
```bash
# 查找最近 24 小时内被修改的 PHP 文件
find /var/www/html -name "*.php" -mtime -1

# 查找包含危险函数的 PHP 文件
grep -rn "eval\|system\|shell_exec\|assert" /var/www/html --include="*.php"

# 查找异常文件名（如随机字符串的 PHP 文件）
find /var/www/html -name "*.php" | grep -E "[a-zA-Z0-9]{8,}\.php"
```

## 4. 挖矿木马与进程伪装排查

### 挖矿木马特征
- CPU 占用异常高，进程名为随机字符串或伪装成系统进程。
- 发起大量对外连接，连接矿池 IP。
- 存在于 `/tmp`、`/dev/shm` 等临时目录。

### 排查命令
```bash
# 查看 CPU 占用最高的进程
top -c
ps aux --sort=-%cpu | head -10

# 查看进程的真实可执行文件路径（识别伪装进程）
ls -la /proc/<PID>/exe
cat /proc/<PID>/cmdline

# 查看网络连接与对应进程
netstat -antp | grep ESTABLISHED
ss -antp | grep ESTABLISHED

# 查找临时目录中的可疑文件
ls -la /tmp /var/tmp /dev/shm
```

### 清除思路
1. 结束恶意进程：`kill -9 <PID>`。
2. 删除恶意文件：`rm -rf /tmp/<恶意文件名>`。
3. 清除定时任务：`crontab -l`，删除恶意任务。
4. 清除启动项：检查 `/etc/rc.local`、`/etc/cron*`、`systemctl list-units`。
5. 检查 SSH 公钥：`cat ~/.ssh/authorized_keys`，清除攻击者留下的公钥。

## 5. 历史命令与痕迹清理

攻击者通常会清理 `~/.bash_history` 以掩盖痕迹。排查时需注意：
```bash
# 查看历史命令
cat ~/.bash_history

# 检查历史记录是否被清空或篡改
ls -la ~/.bash_history
stat ~/.bash_history
```
如果 `.bash_history` 文件大小为 0，且没有正常的历史命令，说明可能被攻击者清理过。

## 6. 面试高频问题

**Q1：如何排查 Linux 服务器是否被 SSH 爆破？**
A：查看 `/var/log/secure` 或 `/var/log/auth.log`，使用 `grep "Failed password"` 统计失败次数，找出爆破源 IP。如果看到大量失败后出现 `Accepted password`，说明爆破成功。

**Q2：如何发现服务器上的挖矿木马？**
A：通过 `top` 或 `ps aux --sort=-%cpu` 查看 CPU 占用高的进程。使用 `ls -la /proc/<PID>/exe` 查看进程的真实路径，确认是否伪装。同时检查 `/tmp`、`/dev/shm` 等目录是否有异常文件，使用 `netstat -antp` 查看是否连接矿池。

**Q3：如果攻击者清除了 `/var/log/secure`，你怎么办？**
A：如果本地日志被清除，需要依靠集中日志收集系统（如 ELK、Splunk）中留存的日志进行溯源。同时检查是否有其他日志（如 `.bash_history`、Web 日志、审计日志）能辅助还原攻击时间线。

**Q4：如何排查 WebShell？**
A：1. 检查 Web 目录中最近被修改的 PHP 文件（`find -mtime -1`）。2. 使用 `grep` 搜索包含 `eval`、`system` 等危险函数的文件。3. 检查是否存在异常文件名（随机字符串）。4. 结合 Web 访问日志，查看攻击者的访问路径。
