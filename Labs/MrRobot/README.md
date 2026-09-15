# MrRobot 靶场学习笔记

- **靶场来源**：VulnHub
- **难度**：中等
- **目标 IP**：`x.x.x.x`（本地虚拟机环境）
- **获取权限**：www-data -> robot -> root
- **Flag 数量**：3/3

## 靶场简介

MrRobot 是一个基于 WordPress 的经典渗透测试靶场，主要考察信息收集、WordPress 漏洞利用、密码破解、SUID 提权等技能。

## 攻击链总览

```text
信息收集 (robots.txt) → WordPress 扫描 → WP 后台爆破 → 模板 GetShell
→ 密码破解 (john) → SSH 登录 robot 用户 → SUID nmap 提权 → root
```

## Flag 获取记录

| 序号 | 位置 | 获取方式 |
|:---|:---|:---|
| 1 | Web 目录 | 通过 `robots.txt` 发现字典文件并获取 |
| 2 | `/home/robot/` | 破解 `robot` 用户密码后获取 |
| 3 | `/root/` | 利用 SUID `nmap` 提权后获取 |

## 环境拓扑

攻击机 Kali（`192.168.x.x`） → 靶机 MrRobot（`x.x.x.x`）
```

5. 点击 `Commit new file`，提交信息：`docs: add MrRobot overview`

---

### 第二步：创建靶场搭建文档

#### 在 GitHub 网页操作：
1. 确保在 `Labs/MrRobot/` 目录下。
2. 点击 `Add file` → `Create new file`。
3. 文件名输入：`01-setup.md`
4. 复制以下内容：

```markdown
# 01 靶场搭建

MrRobot 靶场以 OVA 虚拟机镜像形式提供，需要在本地虚拟机软件（VMware 或 VirtualBox）中导入并运行。

## 1. 环境准备

- **攻击机**：Kali Linux（NAT 或桥接模式）
- **靶机**：MrRobot OVA 文件（从 VulnHub 官网下载）
- **虚拟机软件**：VMware Workstation 或 VirtualBox

## 2. 导入靶机

1. 打开 VMware/VirtualBox，选择「文件」→「打开」或「导入 OVA」。
2. 选择下载好的 `MrRobot.ova` 文件，按照向导完成导入。
3. 导入完成后，设置网络模式为 **NAT 模式**或 **桥接模式**（确保和 Kali 在同一网段）。

## 3. 发现靶机 IP

启动靶机后，在 Kali 中使用 `netdiscover` 或 `arp-scan` 扫描局域网内活跃主机：

```bash
# 扫描同网段存活主机
sudo netdiscover -r 192.168.x.0/24

# 或者使用 arp-scan
sudo arp-scan -l
```

找到靶机 IP 后，验证连通性：
```bash
ping x.x.x.x
```

## 4. 验证靶场环境

```bash
# 扫描靶机开放端口
nmap -sV -Pn x.x.x.x
```

预期结果：
- 80/tcp open http (Apache/WordPress)
- 22/tcp open ssh

## 5. 常见踩坑记录

- **虚拟机网络不通**：检查 VMware 的虚拟网络编辑器，确保 NAT 或桥接设置正确。
- **靶机 IP 扫不到**：有些靶机启动后需要等待 1-2 分钟网络初始化，多扫几次或重启靶机。
