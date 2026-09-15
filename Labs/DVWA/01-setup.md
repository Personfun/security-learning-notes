# 01 靶场搭建

本笔记记录如何在本地快速搭建 DVWA 练习环境（基于 Docker）。

## 1. 环境准备

- **攻击机**：Kali Linux（用于抓包、爆破、代码审计）
- **靶机**：本地 Docker 或 XAMPP 环境
- **浏览器插件**：Burp Suite 代理、HackBar（可选）

## 2. Docker 快速搭建（推荐）

使用 Docker 是最快、最干净的方式，不用配置复杂的 PHP 和 MySQL 环境。

```bash
# 拉取 DVWA 镜像
docker pull vulnerables/web-dvwa

# 启动容器，映射到本机 8080 端口
docker run -d -p 8080:80 vulnerables/web-dvwa
```

## 3. 初始化配置

1. 浏览器访问 `http://127.0.0.1:8080`。
2. 默认账号：`admin`，默认密码：`password`。
3. 登录后，点击左侧菜单 `Setup / Reset DB`，拉到最下方点击 `Create / Reset Database` 初始化数据库。
4. 初始化完成后会自动跳转到登录页，再次登录即可。

## 4. 常见踩坑记录

- **数据库连接失败**：确认容器正常启动（`docker ps`），如果本机 8080 端口被占用，修改宿主机的映射端口（如改成 `8081:80`）。
- **登录提示 CSRF token 错误**：清除浏览器缓存或使用无痕模式重新登录。
- **容器重启后数据丢失**：Docker 容器重启后，DVWA 数据库可能重置。重新点击 `Create / Reset Database` 即可。
