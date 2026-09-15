# Linux 应急响应学习笔记

## 1. 应急响应标准流程

```
确认事件 → 隔离主机 → 排查后门 → 分析来源 → 恢复系统 → 写报告
```

## 2. 常见安全事件

| 事件 | 特征 |
| ---- | ---- |
| 挖矿木马 | CPU 飙高、异常进程 |
| 勒索病毒 | 文件被加密 |
| 反弹 shell | 异常外连 |
| Webshell | Web 目录出现可疑文件 |

## 3. 排查命令

| 项目 | 命令 |
| ---- | ---- |
| 查看进程 | `ps -aux` |
| 查看 CPU | `top` |
| 查看网络 | `ss -antp` |
| 查看启动项 | `systemctl list-unit-files --state=enabled` |
| 查看计划任务 | `crontab -l` |
| 查看登录日志 | `last` |
| 查看历史命令 | `history` |

## 4. 挖矿木马排查实操

### 4.1 发现异常

```bash
top -b -n 1 | head -20
```

发现高 CPU 进程。

### 4.2 锁定进程

```bash
ps -aux --sort=-%cpu
```

### 4.3 查看进程命令

```bash
cat /proc/PID/cmdline
```

### 4.4 识破伪装

```bash
ls -la /proc/PID/exe
```

如果指向 `/usr/bin/bash`，而进程名伪装成内核线程，则高度可疑。

### 4.5 查看网络连接

```bash
ss -antp | grep PID
```

### 4.6 固定证据

```bash
ps -aux > /tmp/evidence.txt
ss -antp >> /tmp/evidence.txt
```

### 4.7 处置

```bash
kill -9 PID
```

## 5. 关键经验

- 不要只看进程名
- 一定要看 `/proc/PID/exe`
- 内核线程没有 `/usr/bin/bash` 这样的路径
- 先取证再处置
