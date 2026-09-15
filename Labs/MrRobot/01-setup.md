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
