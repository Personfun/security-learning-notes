# AD-Company-Lab 域渗透靶场学习笔记

- **靶场来源**：本地搭建（VMware/VirtualBox 环境）
- **难度**：中高难度
- **目标**：`company.local`（模拟企业内网域名）
- **获取权限**：普通域用户 -> 域管理员
- **Flag 数量**：3/3

## 靶场简介

本靶场是一个模拟企业内网架构的 Active Directory 域环境，主要考察信息收集、域内枚举、Kerberos 协议漏洞利用（Kerberoasting、AS-REP Roasting）、横向移动（PTH、PsExec）、DCSync 以及黄金票据等域渗透核心技能。

## 攻击链总览

```text
信息收集 (端口扫描、域信息枚举) → 获得普通域用户凭据
→ 域内信息收集 (SPN、AS-REP) → Kerberoasting / AS-REP Roasting
→ 破解服务账户密码 → 横向移动 (PTH / PsExec / WMI)
→ DCSync 导出 KRBTGT 哈希 → 伪造黄金票据 / 登录域控 → 获取 Flag
```

## Flag 获取记录

| 序号 | 位置 | 获取方式 |
|:---|:---|:---|
| 1 | 成员机桌面 | 通过普通域用户登录获取 |
| 2 | 文件服务器 | 利用服务账户凭据横向移动获取 |
| 3 | 域控 | 通过 DCSync / 黄金票据获取域管权限后读取 |

## 环境拓扑（参考）

| 主机角色 | 主机名 | IP地址 | 操作系统 |
|:---|:---|:---|:---|
| 攻击机 | Kali | `10.0.0.x` | Kali Linux |
| 域控制器 | DC | `10.0.0.x` | Windows Server 2019 |
| 成员机 | Win10-A | `10.0.0.x` | Windows 10 |
| 成员机 | Win10-B | `10.0.0.x` | Windows 10 |
