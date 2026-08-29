# frp 内网穿透学习笔记

## 1. 什么是 frp

frp 是一款开源的**内网穿透工具**，用于将内网服务映射到公网，使外部主机能够访问内网服务。

## 2. frp 核心组件

| 组件 | 作用 | 部署位置 |
| ---- | ---- | -------- |
| frps | 服务端 | 公网服务器 / 攻击机 |
| frpc | 客户端 | 内网机器 |

## 3. 实验环境

| 主机 | IP | 角色 |
| ---- | ---- | ---- |
| Kali | 10.0.0.131 | frp 服务端 |
| CentOS | 10.0.0.132 | frp 客户端 |
| Windows | 10.0.0.129 | 目标内网主机 |

## 4. 下载与解压

```bash
wget https://github.com/fatedier/frp/releases/download/v0.61.1/frp_0.61.1_linux_amd64.tar.gz
tar -xzf frp_0.61.1_linux_amd64.tar.gz
cd frp_0.61.1_linux_amd64
```

## 5. 服务端配置（Kali）

编辑 `frps.toml`：

```toml
bindPort = 7000
```

启动服务端：

```bash
./frps -c frps.toml
```

## 6. 客户端配置（CentOS）

编辑 `frpc.toml`：

```toml
serverAddr = "10.0.0.131"
serverPort = 7000

[[proxies]]
name = "centos-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
```

启动客户端：

```bash
./frpc -c frpc.toml
```

## 7. 验证隧道

在 Kali 上执行：

```bash
ssh root@127.0.0.1 -p 6000
```

成功登录 CentOS，说明隧道生效。

## 8. 映射 Windows RDP

客户端增加配置：

```toml
[[proxies]]
name = "windows-rdp"
type = "tcp"
localIP = "10.0.0.129"
localPort = 3389
remotePort = 6001
```

在 Kali 上连接：

```bash
xfreerdp /u:test /p:Lana0423 /v:127.0.0.1:6001 /cert:ignore
```

## 9. 常见问题

| 问题 | 排查方向 |
| ---- | -------- |
| 服务端未启动 | 检查 7000 端口是否被占用 |
| 客户端无法连接 | 检查 serverAddr 和 serverPort |
| 代理不生效 | 检查 localIP 与 localPort |
| RDP 连接失败 | 检查内网到目标 3389 端口是否连通 |

## 10. 总结

frp 是红队内网穿透的核心工具，能够将内网 SSH、RDP、Web 服务映射到攻击机本地，配合后续横向移动使用。
