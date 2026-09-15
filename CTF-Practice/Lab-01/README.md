# Lab-01 靶场记录

- **日期**：2026-09-12
- **目标**：`x.x.x.x`
- **开放端口**：222 (SSH)、800 (HTTP)、3306 (MySQL)、6379 (Redis)
- **Web 系统**：某国产 CMS
- **Flag 数量**：3/3
- **最终权限**：root

## 攻击链总览

```text
nmap 扫端口 → dirsearch 扫目录 → .git 源码泄露
→ database.php 拿 MySQL 密码 → phpMyAdmin 改 admin 密码
→ 登录 CMS 后台 → upload.php Basic 认证爆破
→ .phar 绕过上传 WebShell → /readflag 拿 Flag1
→ /flag-user 拿 Flag2 → Redis 写 SSH 公钥提权
→ root → /root/flag 拿 Flag3
```

## 三个 Flag（已脱敏）

| Flag | 获取方式 |
|:---|:---|
| `flag{****}` | `/readflag`（SUID 程序） |
| `flag{****}` | `/flag-user`（www-data 可读） |
| `flag{****}` | `/root/flag`（Redis 提权到 root） |

## 环境拓扑

攻击机 Kali → 靶机（Debian 10，Apache + PHP + MySQL + Redis）
