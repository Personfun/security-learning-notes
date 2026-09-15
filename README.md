# security-learning-notes

个人安全学习笔记仓库。记录我从 0 开始的红队渗透、内网安全、域渗透、蓝队应急响应、代码审计等学习内容与实战复盘。

## 📁 核心文档
- ✅ [完整攻击链学习笔记](./complete-attack-chain.md)
- ✅ [蓝队应急响应完整流程链](./blue-team-response-chain.md)

---

## 🎯 靶场实战笔记

### 1. 面试与 CTF 靶场（CTF-Practice）
按日期和编号归档的面试实战环境。
- ✅ [Lab-01 靶场记录（国产CMS+Redis 提权场景）](./CTF-Practice/Lab-01/)
- ✅ [Lab-02 靶场记录（本地 Docker 环境搭建与逃逸）](./CTF-Practice/Lab-02/)
- 📝 *Lab-03 待更新...*

### 2. 系统化学习靶场（Labs）
按知识体系归档的各类经典靶场环境。
- ✅ [MrRobot 靶场（WordPress + SUID 提权）](./Labs/MrRobot/)
- ✅ [AD-Company-Lab 靶场（域渗透环境）](./Labs/AD-Company-Lab/)
- ✅ [DVWA 靶场（Web 漏洞代码审计与绕过）](./Labs/DVWA/)
- ✅ [CTFHub 靶场（CTF 技能树与解题思路）](./Labs/CTFHub/)

---

## 🔴 红队攻防笔记

### Web 安全（📝 待系统整理）
- 📝 SQL 注入、XSS、CSRF、SSRF、文件上传、命令注入
- 📝 WAF 绕过与免杀 Webshell
- 📝 代码审计（PHP 基础、YzmCMS 真实审计）

### 内网渗透与隧道（🚧 部分完成）
- ✅ [frp 内网穿透学习笔记](./frp-neiwang-chuantou.md)
- ✅ [nps 内网穿透学习笔记](./nps-neiwang-chuantou.md)
- ✅ [chisel 内网穿透学习笔记](./chisel-neiwang-chuantou.md)
- ✅ [PTH 与横向移动学习笔记](./pth-and-lateral-movement.md)

### 提权与权限维持（📝 待补充）
- 📝 Linux 提权 (SUID, Sudo, Crontab, 内核漏洞)
- 📝 Windows 提权 (Potato 系列, PrintSpoofer)
- 📝 Docker 容器逃逸 (特权容器, Docker Socket, 危险挂载)
- 📝 Windows 权限维持 (隐藏用户, 注册表, 计划任务, 服务替换, 粘滞键, WMI后门)

### 域渗透（📝 待补充）
- 📝 Kerberoasting、AS-REP Roasting、DCSync、黄金票据
- 📝 BloodHound 域内信息收集与攻击路径分析

### 免杀与 C2（📝 待补充）
- 📝 MSF 深度使用与 Meterpreter 后渗透
- 📝 Cobalt Strike 基础与实战
- 📝 免杀基础 (XOR/AES 加密 Shellcode, 加载器编写)

---

## 🔵 蓝队防御笔记

### 应急响应（✅ 部分完成）
- ✅ [Linux 应急响应学习笔记](./blue-linux-incident-response.md)
- ✅ [Windows 应急响应学习笔记](./blue-windows-incident-response.md)
- ✅ [应急响应报告编写指南](./security-incident-report-guide.md)

### 日志分析与威胁狩猎（📝 待补充）
- 📝 日志分析 (Linux secure, Windows 事件 ID, Sysmon)
- 📝 威胁狩猎基础与实战

### 安全设备与运营（📝 待补充）
- 📝 WAF / IDS / HIDS 部署与配置
- 📝 ELK / Splunk 日志集中收集与告警规则编写

---

## 🛠️ 拓展与进阶

- 📝 移动端安全（Android APP 漏洞分析、iOS 安全）
- 📝 云安全（云原生环境渗透、云元数据利用）
- 📝 等保测评（测评流程、风险配置项）
- 📝 灰黑产追踪与情报分析（暗网监控、虚拟货币追踪、诈骗链路分析）

---

## 🚀 学习路线与计划（2026）

- **阶段一：红队核心深化**（Windows/Linux 提权、内网穿透、MSF、CS、免杀）🚧
- **阶段二：蓝队核心深化**（安全设备部署、日志分析、应急响应、威胁狩猎）🚧
- **阶段三：前沿技术与多平台攻击**（钓鱼、macOS、Android、iOS）📝
- **阶段四：灰黑产追踪与等保合规**（暗网监控、等保测评、项目报告）📝

---

## 📌 说明

- 本仓库仅用于个人学习记录。
- 所有技术内容均用于**合法授权的安全测试**，严禁用于非法用途。
- 笔记持续更新中，欢迎交流指导。
