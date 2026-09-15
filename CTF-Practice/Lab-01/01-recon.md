# 01 信息收集

## 端口扫描

```bash
nmap -sV -Pn -p 222,800,3306,6379 x.x.x.x
```

结果：
- 222/tcp open ssh OpenSSH 7.9p1 Debian 10
- 800/tcp open http Apache 2.4.59
- 3306/tcp closed（公网 closed，内部可连）
- 6379/tcp open redis（后通过 netstat 确认）

## 目录扫描

```bash
dirsearch -u http://x.x.x.x:800/ -e php,html,js
```

关键发现：
- `/.git/` → 源码泄露
- `/admin_login.php` → 后台入口
- `/phpMyAdmin/` → 数据库面板
- `/upload.php` → 401 Basic 认证
- `/uploads/` → 上传目录

## 踩坑记录

- 最初 nmap 没扫到 6379 端口，后来通过 WebShell 执行 `netstat -tlnp` 才发现 Redis 在本机监听。
