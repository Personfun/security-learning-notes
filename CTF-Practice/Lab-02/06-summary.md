# 06 靶场复盘与总结

## 1. 攻击链回顾

```text
环境搭建 → 信息收集 → 漏洞发现与利用 → Getshell
→ 容器内提权 / 容器逃逸 → 获取 Flag
```

## 2. 技能点梳理

| 阶段 | 涉及技能 | 对应知识库笔记 |
|:---|:---|:---|
| 环境搭建 | Docker 基础、Vulhub 使用 | 本目录 `01-environment-setup.md` |
| 信息收集 | Nmap、Dirsearch、指纹识别 | 本目录 `02-recon.md` |
| 漏洞利用 | NoSQL 注入、RCE、越权 | 本目录 `03-vuln-exploit.md` |
| 提权 | SUID、Sudo、Docker 逃逸 | 本目录 `04-privesc.md` |

## 3. 踩坑记录

- **端口冲突**：宿主机 80 端口被占用时，需修改 Docker 端口映射。
- **Docker Socket 未挂载**：如果容器内没有 `/var/run/docker.sock`，无法用该方式逃逸，需寻找其他路径。
- **NoSQL 注入格式**：Content-Type 必须是 `application/json`，否则后端不解析。

## 4. 面试话术准备

**如果在面试中被问到“你打过哪些本地靶场”：**
> “我本地用 Docker 搭建了一套靶场环境，主要是为了复现 Web 漏洞和容器逃逸。从信息收集到 Getshell，再到提权逃逸，整个链路我都走通过一遍。比如在某 API 平台上，我通过 NoSQL 注入绕过登录，拿到后台 RCE，然后利用 Docker Socket 逃逸到宿主机，最终拿到 root 权限。”

## 5. 后续计划

- 尝试在同一靶场中练习不同的漏洞路径（如 SQL 注入替代 NoSQL 注入）。
- 在靶场中加入 WAF 或安全监控，练习绕过和蓝队检测。
- 继续更新知识库，把实战心得整理成面试题。
```
