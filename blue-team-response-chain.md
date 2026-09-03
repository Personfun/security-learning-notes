# 蓝队应急响应完整流程链

本文档记录蓝队从发现安全事件到完成响应的完整流程。

## 一、蓝队核心流程

```
发现事件 → 确认事件 → 隔离主机 → 排查后门 → 分析来源 → 恢复系统 → 写报告
```

## 二、Linux 应急响应流程

### 1. 发现异常

- CPU 飙高
- 异常进程
- 异常网络连接

### 2. 排查顺序

```
查看进程 → 查看CPU → 查看网络 → 查看启动项 → 查看计划任务 → 查看日志
```

### 3. 常用命令

```bash
ps -aux
top
ss -antp
systemctl list-unit-files
crontab -l
last
history
```

### 4. 处置

- 杀死恶意进程
- 删除后门文件
- 清除计划任务
- 清除 SSH 密钥
- 修改密码

## 三、Windows 应急响应流程

### 1. 发现异常

- 账户异常
- 计划任务异常
- 启动项异常
- 服务异常
- 网络连接异常

### 2. 排查顺序

```
用户账户 → 管理员组 → 计划任务 → 服务 → 启动项 → 网络连接 → 进程
```

### 3. 常用命令

```bash
net user
net localgroup administrators
schtasks /query
sc query
reg query HKLM\...\Run
netstat -ano
tasklist
```

### 4. 处置

- 删除隐藏账户
- 删除恶意计划任务
- 删除恶意启动项
- 停止恶意服务
- 封禁异常 IP

## 四、应急响应报告结构

```
事件概述
排查过程
发现的后门
处置措施
修复建议
```

## 五、技术笔记索引

- [Linux 应急响应](./blue-linux-incident-response.md)
- [Windows 应急响应](./blue-windows-incident-response.md)
