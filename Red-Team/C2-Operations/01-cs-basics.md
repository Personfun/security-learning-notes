# 01 - Cobalt Strike 基础

## 一、 CS 是什么

Cobalt Strike 是一款商业化的红队 C2（Command and Control）平台，2012 年由 Raphael Mudge 开发，是目前攻防演练和渗透测试中最主流的团队协作工具。

**核心能力**：
- 多人在线协作，共享 Beacon 会话
- 支持 HTTP / HTTPS / SMB / TCP 多种通信协议
- 灵活的后渗透模块（提权、抓密码、横向移动）
- 强大的插件生态（Bof、Artifact Kit、Malleable C2）

**常见应用场景**：
- 护网行动中作为红队主力 C2
- 内网渗透的长期权限维持
- 团队协作式的渗透测试

## 二、 架构

CS 由三部分组成：

```
┌──────────────────────┐
│  CS Client（Java GUI）│  ← 攻击者操作的图形界面（Windows）
├──────────────────────┤
│  Team Server         │  ← 服务端，中转所有流量（Linux）
├──────────────────────┤
│  Beacon Payload      │  ← 注入到目标机器的木马
└──────────────────────┘
```

- **Team Server**：跑在 Linux 上，负责监听、调度、转发
- **Client**：跑在 Windows 上，图形化操作界面（Java 程序）
- **Beacon**：跑在目标机器上，真正被控制的"傀儡"

**启动流程**：
1. Kali 上启动 Team Server：`sudo ./teamserver 10.0.0.131 123456`
2. Windows 上双击客户端连接：Host=10.0.0.131, Port=50050, Password=123456

## 三、 Listener（监听器）

Listener 是告诉 CS "Beacon 应该回连到哪、用什么协议"。

### 常见类型

| 类型 | 说明 | 适用场景 |
| :--- | :--- | :--- |
| **HTTP** | 明文 HTTP，流量大 | 内网穿透 |
| **HTTPS** | 加密 HTTPS，隐蔽性强 | 外网渗透（默认推荐） |
| **SMB** | 通过命名管道通信 | 内网横向，目标不出网 |
| **TCP** | 原始 TCP 连接 | 内网直连 |
| **Foreign HTTP/HTTPS** | 外部 C2（如 MSF）转发 | 集成其他 C2 框架 |

### 创建步骤

1. `Cobalt Strike → Listeners → Add`
2. 填写配置：
   - **Name**：`test-http`（自定义）
   - **Payload**：`windows/beacon_http/reverse_http`
   - **Host**：`10.0.0.131`（Team Server IP）
   - **Port**：`8080`
3. Save

**关键点**：Host 必须填 Team Server 的 IP，Beacon 会回连到这个地址。

## 四、 Payload（木马）

### 生成方式

| 路径 | 说明 |
| :--- | :--- |
| `Payloads → Windows Stager Payload` | 小体积，先连回再下载完整 Beacon |
| `Payloads → Windows Stageless Payload` | 完整 payload，体积大但更稳 |
| `Attacks → Packages → Windows Executable` | 标准 EXE 生成方式 |
| `Attacks → Scripted Web Delivery` | 生成 PowerShell / bitsadmin 一行命令 |

### Stager vs Stageless

| 维度 | Stager | Stageless |
| :--- | :--- | :--- |
| **体积** | 小（~19KB） | 大（~200KB+） |
| **机制** | 先回连 CS，下载完整 Beacon | 包含完整功能，一次加载 |
| **优势** | 隐蔽、穿透性强 | 稳定、不依赖二次下载 |
| **劣势** | 依赖网络稳定 | 体积大，易被检测 |

### 常见输出格式

- **Windows EXE**：最常见的可执行文件
- **Windows Service EXE**：服务型木马
- **DLL**：动态链接库，可配合 DLL 劫持
- **PowerShell**：无文件落地
- **VBA**：Office 宏，钓鱼附件
- **HTA**：HTML Application，钓鱼链接

## 五、 Beacon（会话）

Beacon 是 CS 上线后的"会话"，等价于 MSF 的 Meterpreter。

### Beacon 类型

| 类型 | 协议 | 场景 |
| :--- | :--- | :--- |
| **HTTP Beacon** | HTTP | 最通用，穿透防火墙 |
| **HTTPS Beacon** | HTTPS | 加密，隐蔽性最高（默认推荐） |
| **SMB Beacon** | SMB 命名管道 | 内网横向，目标不出网 |
| **TCP Beacon** | TCP | 内网直连 |
| **Foreign Beacon** | 外部 C2 转发 | 从 MSF 等其他 C2 转到 CS |

### Beacon 的异步机制（核心特点）

**核心区别**：Beacon 默认不是长连接，而是每隔一段时间回连一次。

**两个关键参数**：
- **Sleep**：回连间隔（默认 60 秒）
- **Jitter**：在 Sleep 基础上加随机偏移（默认 0%）

**举例**：`sleep 60 20` = 每 60±20%（48~72 秒）回连一次。

**为什么这么设计**：

| 原因 | 说明 |
| :--- | :--- |
| **隐蔽** | 长连接容易被防火墙/IDS 发现 |
| **抗溯源** | 流量稀疏、不规则，难以被识别为 C2 |
| **持久** | 即使网络抖动，Beacon 也会重连 |
| **降低带宽消耗** | 命令排队，下次回连时统一发送 |

**实战中的权衡**：
- **隐蔽优先**：`sleep 300 30`（5 分钟心跳，30% 抖动），适合长期潜伏
- **效率优先**：`sleep 5`（5 秒心跳），适合快速渗透

## 六、 CS 典型工作流

```
1. 启动 Team Server（Kali）
    ↓
2. 客户端连接（Windows）
    ↓
3. 创建 Listener（HTTP/HTTPS）
    ↓
4. 生成 Payload（EXE/PowerShell/HTA）
    ↓
5. 目标机器执行 Payload
    ↓
6. Beacon 上线，进入 Sessions 面板
    ↓
7. 交互：shell / upload / download / inject
    ↓
8. Socks 代理 + 端口转发，横向内网
    ↓
9. 提权、抓密码、域渗透
```

**上线成功标志**：
```
[*] initial beacon from test@10.0.0.129 (DESKTOP-6O9K55F)
```

底部状态栏：`[TeamServer IP: 10.0.0.131 | Beacons: 1 | lag: 00]`

## 七、 实战中的关键操作

### 7.1 上线后第一步：调整心跳

```
beacon> sleep 5
```

让 Beacon 快速响应，方便调试。长期潜伏时改回 `sleep 300 30`。

### 7.2 会话交互

```
beacon> shell whoami
[+] received output: desktop-6o9k55f\test
```

### 7.3 Socks 代理（批量访问内网）

```
beacon> socks 1080
[+] started SOCKS4a server on: 1080
```

Kali 上配置 proxychains，然后扫内网：
```bash
proxychains nmap -sT -Pn 10.0.0.132
```

### 7.4 端口转发（单个服务映射）

```
beacon> rportfwd 8080 10.0.0.132 22
[+] started reverse port forward on 8080 to 10.0.0.132:22
```

本地验证：
```bash
nc -v 127.0.0.1 8080
```

## 八、 常见坑点

### 8.1 Payload 被杀软秒杀
- **原因**：CS 默认 Payload 特征明显
- **解决**：关闭杀软实时保护，或使用 Artifact Kit 定制免杀

### 8.2 Beacon 上线后很快断开
- **原因**：目标重启、杀软清理、网络抖动
- **解决**：执行 `inject` 迁移到 `explorer.exe` 或 `svchost.exe`

### 8.3 会话迁移失败
- **报错**：`Cannot migrate into this process (insufficient privileges)`
- **原因**：目标进程是 SYSTEM 权限，当前会话是普通用户
- **解决**：先 `getsystem` 提权，或迁移到同权限进程（explorer.exe）

### 8.4 Team Server 启动报错
- **报错**：`Superuser privileges are required`
- **解决**：用 `sudo` 启动
- **报错**：`Port 50050 already in use`
- **解决**：换端口：`./teamserver 10.0.0.131 123456 50051`

## 九、 学习重点

| 模块 | 优先级 |
| :--- | :--- |
| Listener / Payload / Beacon 概念 | 高 |
| Beacon 异步机制（Sleep/Jitter） | 高 |
| Beacon 常用命令 | 高 |
| Socks 代理 / 端口转发 | 高 |
| Bof 开发 / Artifact Kit / Malleable C2 | 进阶 |

## 十、 参考资料

- Cobalt Strike 官方文档
- 《Cobalt Strike 权威指南》
- CS 4.9.1 用户手册（随工具包附带）
