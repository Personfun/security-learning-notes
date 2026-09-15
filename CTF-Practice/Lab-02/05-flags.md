# 05 Flag 获取记录

本靶场为本地搭建的 Docker 环境，Flag 由我们自己在初始化时插入到数据库或文件系统中，用于验证漏洞利用是否成功。

## Flag 列表

| 序号 | Flag 内容 | 获取方式 | 权限要求 |
|:---|:---|:---|:---|
| 1 | `flag{****}` | 通过 [NoSQL 注入 / 弱口令] 登录后台后获取 | 普通用户 / 管理员 |
| 2 | `flag{****}` | 通过 [命令执行 / 文件读取] 读取 `/flag` | www-data / 容器内权限 |
| 3 | `flag{****}` | 通过 [容器逃逸 / 提权] 读取宿主机 `/root/flag` | root / 宿主机权限 |

## 获取命令示例

```bash
# 读取容器内 flag
curl -s "http://127.0.0.1:8080/shell.php?cmd=cat /flag"

# 通过容器逃逸后读取宿主机 flag
cat /tmp/host/root/flag
```

## 环境清理

通关后，可以随时重置靶场环境：

```bash
docker-compose down
docker-compose up -d
```
