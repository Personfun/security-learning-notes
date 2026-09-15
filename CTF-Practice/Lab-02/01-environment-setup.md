# Lab-02 靶场环境搭建笔记

本笔记记录如何从零开始搭建一套用于本地练习的漏洞靶场环境（基于 Docker）。本地环境使用 `127.0.0.1` 或 `localhost`，不影响真实网络。

## 1. 环境依赖与前置准备

- **操作系统**：Kali Linux 或 Ubuntu（推荐使用虚拟机快照，方便随时重置）
- **必备工具**：Docker、Docker Compose、Git
- **硬件要求**：建议分配 4GB 以上内存，20GB 以上磁盘空间

### 安装 Docker 与 Docker Compose

```bash
# 更新源
sudo apt update && sudo apt upgrade -y

# 安装 Docker
sudo apt install docker.io docker-compose -y

# 启动 Docker 并设置开机自启
sudo systemctl start docker
sudo systemctl enable docker

# 验证安装
docker -v
docker-compose -v
```

## 2. 获取靶场源码或镜像

通常靶场环境可以通过拉取 GitHub 仓库或 Docker 镜像来获取。

### 方式一：使用 Vulhub 开源靶场（推荐）

Vulhub 是一个开源的漏洞靶场集合，包含了大量的 Web 漏洞环境。

```bash
# 拉取 Vulhub 仓库
git clone https://github.com/vulhub/vulhub.git

# 进入你要练习的漏洞目录（例如某 CMS 或 API 平台）
cd vulhub/<具体的漏洞目录>
```

### 方式二：使用 Docker Compose 一键启动

大部分靶场都提供了 `docker-compose.yml` 文件，直接在目录下执行：

```bash
# 启动容器（-d 表示后台运行）
docker-compose up -d
```

## 3. 查看运行状态与访问靶场

```bash
# 查看当前运行的容器
docker ps

# 查看容器占用的端口
netstat -tlnp
```

根据 `docker ps` 输出的端口映射（例如 `0.0.0.0:8080->80/tcp`），在浏览器中访问对应的本地地址：

- 如果映射到 80 端口：`http://127.0.0.1/`
- 如果映射到 8080 端口：`http://127.0.0.1:8080/`

## 4. 初始化配置

部分靶场启动后需要进行初始化：
- 访问安装向导页面进行数据库配置。
- 默认账号密码可能为 `admin/admin`、`root/root`，具体见对应靶场的 `README.md` 或 Docker 配置文件。
- 若需要插入 Flag，可以进入容器内部或挂载卷进行操作：

```bash
# 进入容器内部
docker exec -it <容器ID或名称> /bin/bash
```

## 5. 常见踩坑记录

- **端口冲突**：如果本机 80 或 8080 端口已被占用，需要修改 `docker-compose.yml` 文件中的端口映射（例如改成 `8081:80`）。
- **内存不足**：如果容器启动后自动退出，可能是内存不够。使用 `docker logs <容器ID>` 查看日志。
- **镜像拉取慢**：可以配置 Docker 国内加速镜像源。

## 6. 环境重置与销毁

练习完成后，或者环境被破坏，可以随时重置：

```bash
# 停止并删除当前容器
docker-compose down

# 重新启动（恢复到初始状态）
docker-compose up -d
```

如果需要彻底删除镜像：
```bash
docker rmi <镜像ID>
```
