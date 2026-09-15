# security-learning-notes

个人安全学习笔记仓库。记录我从 0 开始的红队渗透、内网安全、域渗透、蓝队应急响应、代码审计等学习内容与实战复盘。

## 📁 核心文档
- ✅ [完整攻击链学习笔记](./Red-Team/Attack-Chain/complete-attack-chain.md)
- ✅ [蓝队应急响应完整流程链](./Blue-Team/Incident-Response/03-response-chain.md)

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

### Web 安全
- ✅ [SQL 注入学习笔记](./Red-Team/Web-Security/01-sql-injection.md)
- 📝 XSS 跨站脚本（待补充）
- 📝 CSRF 与 SSRF（待补充）
- 📝 文件上传与命令注入（待补充）
- 📝 代码审计（PHP 基础、真实 CMS 审计，待补充）

### 内网渗透与隧道
- ✅ [frp 内网穿透学习笔记](./Red-Team/Internal-Pentest/01-frp.md)
- ✅ [nps 内网穿透学习笔记](./Red-Team/Internal-Pentest/02-nps.md)
- ✅ [chisel 内网穿透学习笔记](./Red-Team/Internal-Pentest/03-chisel.md)
- ✅ [PTH 与横向移动学习笔记](./Red-Team/Internal-Pentest/04-pth-and-lateral-movement.md)

### 提权与权限维持（📝 待补充）
- 📝 Linux 提权 (SUID, Sudo, Crontab, 内核漏洞)
- 📝 Windows 提权 (Potato 系列, PrintSpoofer)
- 📝 Docker 容器逃逸
- 📝 Windows 权限维持 (隐藏用户, 注册表, 计划任务, 粘滞键, WMI后门)

### 域渗透（📝 待补充）
- 📝 Kerberoasting、AS-REP Roasting、DCSync、黄金票据

### 免杀与 C2（📝 待补充）
- 📝 MSF 深度使用、Cobalt Strike 基础、免杀基础

---

## 🔵 蓝队防御笔记

### 应急响应
- ✅ [Linux 应急响应学习笔记](./Blue-Team/Incident-Response/01-linux-incident-response.md)
- ✅ [Windows 应急响应学习笔记](./Blue-Team/Incident-Response/02-windows-incident-response.md)
- ✅ [应急响应报告编写指南](./Blue-Team/Incident-Response/04-report-guide.md)

### 日志分析与威胁狩猎（📝 待补充）
- 📝 日志分析 (Linux secure, Windows 事件 ID, Sysmon)
- 📝 威胁狩猎基础与实战

---

## 🚀 学习路线

- Web 渗透测试
- Linux / Windows 提权
- Docker 容器逃逸
- 内网穿透与横向移动
- 免杀基础
- 蓝队应急响应
- 代码审计
- 灰黑产追踪与情报分析

## 📌 说明

本仓库仅用于个人学习记录。所有技术内容均用于合法授权的安全测试，严禁用于非法用途。
