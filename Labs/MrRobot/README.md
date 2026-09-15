# MrRobot 靶场学习笔记

- **靶场来源**：VulnHub
- **难度**：中等
- **目标 IP**：`x.x.x.x`（本地虚拟机环境）
- **获取权限**：www-data -> robot -> root
- **Flag 数量**：3/3

## 靶场简介

MrRobot 是一个基于 WordPress 的经典渗透测试靶场，主要考察信息收集、WordPress 漏洞利用、密码破解、SUID 提权等技能。

## 攻击链总览

```text
信息收集 (robots.txt) → WordPress 扫描 → WP 后台爆破 → 模板 GetShell
→ 密码破解 (john) → SSH 登录 robot 用户 → SUID nmap 提权 → root
```

## Flag 获取记录

| 序号 | 位置 | 获取方式 |
|:---|:---|:---|
| 1 | Web 目录 | 通过 `robots.txt` 发现字典文件并获取 |
| 2 | `/home/robot/` | 破解 `robot` 用户密码后获取 |
| 3 | `/root/` | 利用 SUID `nmap` 提权后获取 |

## 环境拓扑

攻击机 Kali（`192.168.x.x`） → 靶机 MrRobot（`x.x.x.x`）

## 5. 常见踩坑记录

- **虚拟机网络不通**：检查 VMware 的虚拟网络编辑器，确保 NAT 或桥接设置正确。
- **靶机 IP 扫不到**：有些靶机启动后需要等待 1-2 分钟网络初始化，多扫几次或重启靶机。
