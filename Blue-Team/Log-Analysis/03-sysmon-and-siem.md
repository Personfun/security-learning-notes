# Sysmon 与 SIEM 基础

企业级安全运营不能只靠 Windows 自带日志和单机排查，需要借助 Sysmon 增强日志采集，结合 SIEM 进行集中存储、关联分析和告警。

## 一、Sysmon 基础

Sysmon（System Monitor）是微软官方提供的高级系统监控驱动。它比 Windows 自带事件日志记录得更细，是蓝队威胁狩猎的利器。

### 1. Sysmon 核心事件 ID

| 事件 ID | 含义 | 安全价值 |
|:---|:---|:---|
| **1** | 进程创建 | 记录**完整命令行**和父进程（自带日志 4688 往往没有命令行） |
| **3** | 网络连接 | 记录进程的 IP、端口、域名，用于检测 C2 通信 |
| **7** | 镜像加载 | 检测 DLL 劫持、反射加载 |
| **8** | 远程线程创建 | 检测进程注入（如 Cobalt Strike 的 Inject） |
| **11** | 文件创建 | 检测 Webshell 落地、临时文件释放 |
| **12/13/14** | 注册表操作 | 检测持久化后门（如 Run 键、IFEO） |
| **22** | DNS 查询 | 检测 DNS 隧道、恶意域名请求 |

### 2. Sysmon 安装与配置

```cmd
:: 下载 Sysmon 和配置文件（如 SwiftOnSecurity 的 sysmon-config）
:: 安装并应用配置
sysmon64.exe -accepteula -i sysmonconfig.xml

:: 更新配置
sysmon64.exe -c sysmonconfig.xml
```

## 二、SIEM 基础（以 ELK 为例）

SIEM（安全信息和事件管理）负责将分散在各终端、服务器的日志集中收集、存储、分析和可视化。ELK 是开源 SIEM 的主流方案，包括 Elasticsearch（存储与搜索）、Logstash（收集与解析）、Kibana（可视化）。

### 1. ELK 基础架构

```text
终端（Winlogbeat / Filebeat） → Logstash（解析） → Elasticsearch（存储） → Kibana（展示与告警）
```

### 2. 常用日志收集工具
- **Winlogbeat**：收集 Windows 事件日志和 Sysmon 日志。
- **Filebeat**：收集 Linux 系统日志、Web 日志、文本日志。
- **Packetbeat**：收集网络流量数据。

### 3. Kibana 常用查询示例（KQL）

```kql
# 查询登录失败
event.code: "4625"

# 查询执行了 whoami 的进程创建（Sysmon 事件 ID 1）
event.code: "1" and process.command_line: *whoami*

# 查询可疑的计划任务创建
event.code: "4698" or event.code: "1" and process.command_line: *schtasks*
```

## 三、SIEM 告警规则编写（关联分析）

单一事件往往不能说明问题，真正的威胁往往藏在“事件关联”里。

### 典型关联告警示例

**告警 1：SSH 爆破成功**
- 规则：同一个源 IP，在 5 分钟内触发 20 次 `4625`（登录失败），随后出现一次 `4624`（登录成功）。
- 意义：检测 SSH 或 RDP 爆破。

**告警 2：WebShell 上传与执行**
- 规则：Sysmon 事件 ID 11（在 Web 目录创建 `.php` 文件）后，紧接着事件 ID 1（由 Web 进程 `w3wp.exe` 或 `php-fpm` 启动 `cmd.exe`）。
- 意义：检测 Webshell 落地并执行命令。

**告警 3：PsExec 横向移动**
- 规则：事件 ID 7045（安装了新服务 `PSEXESVC`），紧接着事件 ID 4624 类型 3（网络登录）。
- 意义：检测内网横向移动。

## 四、面试高频问题

**Q1：Sysmon 和 Windows 自带事件日志有什么区别？**
A：Sysmon 记录更细粒度的事件，特别是**进程创建时的完整命令行参数**（自带日志 4688 需要额外配置才能记录）、网络连接、DNS 查询、文件创建等。Sysmon 是蓝队威胁狩猎的核心数据源。

**Q2：企业内网如何构建日志集中收集体系？**
A：终端通过 Winlogbeat/Filebeat 采集日志，发送到 Logstash 或 Kafka 进行缓冲和解析，最终存储到 Elasticsearch 或 Splunk。在 Kibana 或 Splunk 上配置关联规则，实现集中监控和告警。

**Q3：如何检测无文件攻击？**
A：无文件攻击通常依赖 PowerShell、WMI、注册表等。检测重点包括：Sysmon 事件 ID 1（进程创建，关注 `powershell -enc` 等编码命令）、事件 ID 8（远程线程注入）、事件 ID 13（注册表修改），以及 PowerShell 4104 脚本块日志。

**Q4：如何检测 C2 通信？**
A：关注 Sysmon 事件 ID 3（网络连接）和事件 ID 22（DNS 查询）。结合威胁情报，匹配恶意 IP 或域名。同时，关注异常时间间隔的心跳连接（如每 60 秒一次），这是 C2 的典型特征。
