# security-learning-notes

个人安全学习笔记仓库。本仓库记录了我从零开始学习红队渗透、内网安全、域渗透、蓝队应急响应、代码审计以及安全运营等内容的过程与实战复盘。

## 一、靶场实战记录

记录各类靶场的完整攻击链、命令、踩坑记录和复盘总结。按实战场景和知识体系分为两大类。

### 面试与CTF靶场（CTF-Practice）
按日期和编号归档的面试实战环境，侧重于真实攻击链的完整复现与面试考点提炼。

- [Lab-01 靶场记录（某国产CMS + Redis 提权场景）](./CTF-Practice/Lab-01/)
- [Lab-02 靶场记录（本地 Docker 环境搭建与容器逃逸）](./CTF-Practice/Lab-02/)

### 系统化学习靶场（Labs）
按知识体系归档的各类经典靶场环境，侧重于漏洞原理、代码审计与防御视角的对照学习。

- [MrRobot 靶场（WordPress渗透与SUID提权）](./Labs/MrRobot/)
- [AD-Company-Lab 靶场（Active Directory 域渗透环境）](./Labs/AD-Company-Lab/)
- [DVWA 靶场（Web漏洞代码审计与四难度绕过）](./Labs/DVWA/)
- [CTFHub 靶场（CTF技能树与解题思路）](./Labs/CTFHub/)

## 二、红队攻防笔记

记录红队渗透过程中的信息收集、漏洞利用、权限提升与横向移动等技术细节。

### Web安全
涵盖常见Web漏洞的原理、利用方式、代码审计方法及防御手段。
- [SQL注入学习笔记](./Red-Team/Web-Security/01-sql-injection.md)
- [XSS跨站脚本学习笔记](./Red-Team/Web-Security/02-xss.md)
- [CSRF与SSRF学习笔记](./Red-Team/Web-Security/03-csrf-ssrf.md)
- [文件上传与命令注入学习笔记](./Red-Team/Web-Security/04-file-upload-and-command-injection.md)
- [代码审计学习笔记](./Red-Team/Web-Security/05-code-audit.md)

### 内网渗透与隧道
记录内网穿透工具的使用、代理搭建以及横向移动技术。
- [frp内网穿透学习笔记](./Red-Team/Internal-Pentest/01-frp.md)
- [nps内网穿透学习笔记](./Red-Team/Internal-Pentest/02-nps.md)
- [chisel内网穿透学习笔记](./Red-Team/Internal-Pentest/03-chisel.md)
- [PTH与横向移动学习笔记](./Red-Team/Internal-Pentest/04-pth-and-lateral-movement.md)

### 提权与权限维持
记录Linux与Windows环境下的提权思路、漏洞利用以及后门持久化技术。
- [Linux提权学习笔记](./Red-Team/Privilege-Escalation/01-linux-privesc.md)
- [Windows提权与权限维持学习笔记](./Red-Team/Privilege-Escalation/02-windows-privesc.md)

### 域渗透
记录Active Directory环境下的信息收集、Kerberos协议攻击与域控权限获取。
- [域渗透（Active Directory）学习笔记](./Red-Team/AD-Pentest/01-domain-pentest.md)

## 三、蓝队防御笔记

记录蓝队视角下的日志分析、应急响应、安全设备原理以及合规建设。

### 应急响应
涵盖Linux/Windows环境下的入侵排查、木马清除、痕迹分析与报告编写。
- [Linux应急响应学习笔记](./Blue-Team/Incident-Response/01-linux-incident-response.md)
- [Windows应急响应学习笔记](./Blue-Team/Incident-Response/02-windows-incident-response.md)
- [蓝队应急响应完整流程链](./Blue-Team/Incident-Response/03-response-chain.md)
- [应急响应报告编写指南](./Blue-Team/Incident-Response/04-report-guide.md)

### 日志分析与威胁狩猎
涵盖Windows核心事件ID、Linux日志分析、Sysmon部署及SIEM关联告警。
- [Windows日志分析与核心事件ID](./Blue-Team/Log-Analysis/01-windows-event-ids.md)
- [Linux日志分析与应急响应排查](./Blue-Team/Log-Analysis/02-linux-logs.md)
- [Sysmon与SIEM基础](./Blue-Team/Log-Analysis/03-sysmon-and-siem.md)

### 安全设备与运营
记录常见安全防护设备的原理、部署模式及红蓝对抗中的绕过与检测。
- [WAF、IDS与HIDS学习笔记](./Blue-Team/Security-Devices/01-waf-ids-hids.md)

### 等保合规
记录国内网络安全等级保护（等保2.0）的定级、备案、建设整改与测评流程。
- [等保2.0基础与流程](./Blue-Team/Compliance/01-classified-protection.md)

## 四、进阶与拓展

记录安全领域的前沿技术和主动防御思维。
- [威胁狩猎基础](./Advanced-Topics/01-threat-hunting.md)
- [灰黑产追踪与威胁情报分析](./Advanced-Topics/02-cybercrime-tracking.md)
- [移动端安全基础（Android / iOS）](./Advanced-Topics/03-mobile-security.md)

## 五、学习路线

- Web渗透测试与代码审计
- Linux / Windows 提权与权限维持
- Docker容器逃逸与内网横向移动
- 内网穿透与域渗透
- 免杀基础与C2工具使用
- 蓝队应急响应与日志分析
- 安全设备部署与威胁狩猎
- 等保合规与灰黑产追踪

## 六、说明

本仓库仅用于个人学习记录。所有技术内容均用于合法授权的安全测试，严禁用于非法用途。
