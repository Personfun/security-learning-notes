# nps 内网穿透学习笔记

## 1. 什么是 nps

nps 是一款带有 Web 管理面板的内网穿透工具，用于将内网服务映射到外网，使外部主机能够访问内网服务。

## 2. nps 和 frp 的区别

| 对比项 | frp | nps |
| ---- | ---- | ---- |
| 配置方式 | 配置文件 | Web 管理面板 |
| 服务端 | frps | nps |
| 客户端 | frpc | npc |
| 可视化 | 无 | 有 Web 管理界面 |
| 核心原理 | 内网客户端主动连接服务端 | 相同 |

## 3. 为什么能绕过防火墙

防火墙通常允许内网主动访问外网，但禁止外网主动访问内网。

nps 利用“内网可以出网”的规则：

1. 内网客户端 npc 主动连接外网服务端 nps
2. 连接建立后，外网访问服务端端口
3. 服务端把流量转发给内网客户端

## 4. 实验环境

| 主机 | IP | 角色 |
| ---- | ---- | ---- |
| Kali | 10.0.0.131 | nps 服务端 |
| CentOS | 10.0.0.132 | npc 客户端 |

## 5. 下载 nps

服务端：

```bash
wget https://github.com/ehang-io/nps/releases/download/v0.26.10/linux_amd64_server.tar.gz
```

客户端：

```bash
wget https://github.com/ehang-io/nps/releases/download/v0.26.10/linux_amd64_client.tar.gz
```

## 6. 启动 nps 服务端

```bash
tar -xzf linux_amd64_server.tar.gz
./nps
```

默认 Web 管理地址：

```text
http://127.0.0.1:8080
用户名：admin
密码：123
```

## 7. 在 Web 面板添加客户端

1. 登录 Web 面板
2. 点击“客户端”
3. 新增客户端，填写备注
4. 获取客户端 ID 和验证密钥

## 8. 启动 npc 客户端

```bash
tar -xzf linux_amd64_client.tar.gz
./npc -server=服务端IP:8024 -vkey=验证密钥 -type=tcp
```

## 9. 创建 TCP 隧道

1. 在 Web 面板点击“TCP隧道”
2. 新增隧道
3. 填写客户端 ID
4. 服务端端口：6000
5. 目标地址：127.0.0.1:22

## 10. 验证隧道

在 Kali 上执行：

```bash
ssh root@127.0.0.1 -p 6000
```

成功登录 CentOS，说明 nps 隧道生效。

## 11. 映射 Windows RDP

在 Web 面板新增 TCP 隧道：

| 字段 | 值 |
| ---- | ---- |
| 客户端 ID | CentOS |
| 服务端端口 | 6001 |
| 目标地址 | 10.0.0.129:3389 |

在 Kali 上连接：

```bash
xfreerdp /u:test /p:Lana0423 /v:127.0.0.1:6001 /cert:ignore
```

成功弹出 Windows 桌面，说明 nps 映射 RDP 成功。

## 12. 总结

nps 和 frp 原理相似，但 nps 提供 Web 管理面板，配置更直观。真实红队中常用于把内网 SSH、RDP 等端口映射出来，绕过防火墙限制。
