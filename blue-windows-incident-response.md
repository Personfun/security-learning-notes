# Windows 应急响应学习笔记

## 1. 排查项目

| 项目 | 命令 |
| ---- | ---- |
| 用户账户 | `net user` |
| 管理员组 | `net localgroup administrators` |
| 计划任务 | `schtasks /query` |
| 服务 | `sc query` |
| 启动项 | `reg query HKLM\...\Run` |
| 网络连接 | `netstat -ano` |
| 进程 | `tasklist` |

## 2. 常见后门位置

| 后门类型 | 位置 |
| ---- | ---- |
| 隐藏账户 | `net user` 查看 `$` 结尾用户 |
| 计划任务 | `schtasks` 中非系统任务 |
| 注册表启动项 | `Run` 键 |
| 服务后门 | 非系统服务 |

## 3. 实操案例

### 3.1 发现隐藏账户

```bash
net user
```

发现可疑账户 `wmihacker`。

### 3.2 分析账户

```bash
net user wmihacker
```

确认在管理员组，且从未登录。

### 3.3 删除账户

```bash
net user wmihacker /delete
```

### 3.4 发现计划任务后门

```bash
schtasks /query /fo LIST /v
```

发现任务 `SystemUpdateTask` 执行恶意命令。

### 3.5 删除计划任务

```bash
schtasks /delete /tn "SystemUpdateTask" /f
```

### 3.6 发现注册表启动项后门

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

发现 `Update` 键执行恶意命令。

### 3.7 删除启动项

```bash
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "Update" /f
```

## 4. 关键经验

- 重点看 `$` 结尾的隐藏账户
- 计划任务要看“要运行的任务”
- 启动项要看命令内容
- 网络连接关注非正常 IP
