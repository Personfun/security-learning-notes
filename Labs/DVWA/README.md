# DVWA 靶场学习笔记

- **靶场来源**：本地 Docker / XAMPP 环境
- **难度**：简单 - 中等
- **目标**：`http://127.0.0.1:8080`（本地回环地址）
- **获取权限**：Web 权限（代码审计与漏洞利用）
- **Flag 数量**：无特定 Flag，以掌握漏洞原理与绕过技术为目标

## 靶场简介

DVWA (Damn Vulnerable Web Application) 是一个经典的 PHP/MySQL Web 漏洞练习环境。它包含了 SQL 注入、XSS、文件包含、文件上传、命令注入等常见漏洞，并且提供 Low、Medium、High、Impossible 四个难度级别，非常适合用来理解漏洞的成因及防御方式。

## 学习路线

```text
环境搭建 → 基础漏洞练习 (Low) → 绕过技巧 (Medium/High)
→ 代码审计 (Impossible) → 总结防御与面试点
