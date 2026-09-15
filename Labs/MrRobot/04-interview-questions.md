# 04 面试问题点与复盘

针对 MrRobot 靶场，整理出面试中高频出现的问题及标准回答。

## Q1：你是如何发现靶机上的 WordPress 站点的？
**A**：在信息收集阶段，我先通过 `nmap` 扫描发现 80 端口开放 HTTP 服务。接着使用 `dirb` 进行目录扫描，发现 `/wp-login.php` 和 `/wp-admin/` 等特征路径，确认这是一个 WordPress 站点。此外，`robots.txt` 文件中也泄露了网站的基础信息。

## Q2：WordPress 后台密码是怎么爆破的？用了什么工具？
**A**：我使用 `wpscan` 扫描出 WordPress 的用户名（如 `elliot`）。然后在 `robots.txt` 中发现了一个字典文件 `fsocity.dic`。利用该字典，通过 `wpscan` 的密码爆破模块，成功破解出后台密码 `ER28-0652`。

## Q3：你是怎么拿到 WebShell 的？
**A**：登录 WordPress 后台后，在「外观 → 主题编辑器」中，选择了一个不常被访问的模板文件（如 `404.php`），将一段 PHP 反弹 Shell 代码写入其中。然后在 Kali 上监听端口，通过访问不存在的页面触发 404.php，成功获得 `www-data` 权限的 Shell。

## Q4：你是在哪里找到第二个 Flag 的？密码怎么破解的？
**A**：拿到 `www-data` 权限后，我在 `/home/robot/` 目录下发现了一个 `password.raw-md5` 文件，里面存放着 `robot` 用户的密码哈希。我把哈希放入 Kali 中，使用 `john` 配合 `rockyou.txt` 字典进行破解，拿到了明文密码。随后通过 SSH 登录 `robot` 用户，读取第二个 Flag。

## Q5：SUID 提权是什么？你是怎么利用 nmap 提权的？
**A**：SUID（Set Owner User ID）是一种特殊权限，允许普通用户以文件所有者（如 root）的权限执行该程序。我使用 `find / -perm -4000` 发现 `/usr/local/bin/nmap` 具有 SUID 权限。老版本的 nmap 存在 `--interactive` 交互模式，进入该模式后可以执行 `!sh` 调起一个 Shell。由于 nmap 以 root 权限运行，因此这个 Shell 直接就是 root 权限。

## Q6：如果目标没有 `robots.txt` 泄露字典，你还会怎么破解 WordPress 密码？
**A**：如果字典泄露不存在，我会尝试使用通用字典（如 `rockyou.txt`）进行爆破，或者使用 `wpscan` 的 `--api-token` 功能尝试漏洞利用。另外也可以尝试 SQL 注入等其他方式获取凭据，或者寻找其他入口点。

## Q7：从蓝队角度，你如何检测这种攻击？
**A**：从蓝队角度，我会重点监控：
1. Web 日志中针对 `/wp-login.php` 的高频失败请求（检测暴力破解）。
2. 出站流量中异常端口的连接（检测反弹 Shell）。
3. 服务器敏感文件（如 `404.php`）被修改的告警（检测 Webshell 上传）。
4. `/home/` 目录下敏感文件的异常读取（检测权限提升）。
