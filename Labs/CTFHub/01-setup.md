# 01 平台使用与本地环境

CTFHub 是一个在线 CTF 练习平台，提供技能树模式。本笔记记录平台的基本使用方法和本地替代方案。

## 1. 在线平台使用

1. 访问 CTFHub 官网并注册账号（使用邮箱注册即可）。
2. 进入「技能树」板块，选择对应关卡（如信息泄露、SQL注入、命令注入等）。
3. 点击「启动题目环境」，平台会分配一个临时的靶机地址（形如 `http://challenge-xxxx.ctfhub.com`）。
4. 在本地 Kali 或浏览器中访问该地址，进行渗透测试。
5. 拿到 Flag 后提交，环境会自动销毁。

## 2. 本地替代方案（若无法访问外网）

如果网络受限或想离线练习，可以使用 Vulhub 或其他本地靶场模拟 CTFHub 中的常见题型。

```bash
# 例如：本地搭建 SQL 注入环境
git clone https://github.com/vulhub/vulhub.git
cd vulhub/sqli
docker-compose up -d
```

## 3. 常用工具准备

在 Kali 上确保以下工具可用：

```bash
# 安装常用工具
sudo apt update
sudo apt install -y dirsearch sqlmap gobuster git-dumper
```

## 4. 常见踩坑记录

- **环境启动失败**：在线平台有时会有并发限制，稍等几分钟再试。
- **Flag 提交错误**：注意 Flag 格式通常为 `ctfhub{xxxxxxxx}`，区分大小写，不要有多余空格。
- **本地靶场端口冲突**：如果 80 端口被占用，修改 `docker-compose.yml` 的映射端口（如改为 `8080:80`）。
