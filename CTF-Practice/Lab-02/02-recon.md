# 02 信息收集

本次靶场为本地 Docker 环境，所有操作均在本地回环地址（`127.0.0.1`）或 Docker 内部网段进行。

## 1. 端口扫描

使用 Nmap 扫描本地常用端口，确认 Docker 容器映射到宿主机的端口。

```bash
nmap -sV -Pn -p 80,3000,3306,6379,8080,27017 127.0.0.1
```

**预期结果**（根据实际环境填写，以下为示例）：
- 3000/tcp open http → 某开源 API 平台（如 YApi）
- 6379/tcp open redis → Redis 服务
- 27017/tcp open mongodb → MongoDB 服务
- 80/8080 端口映射到 Web 服务

## 2. 目录扫描

针对 Web 服务进行目录枚举，寻找敏感路径。

```bash
dirsearch -u http://127.0.0.1:3000/ -e php,html,js,json
```

**关键发现**（以 YApi 为例）：
- `/api/` → API 接口路径
- `/login` → 登录入口
- `/project/` → 项目列表
- 可能暴露的配置文件或备份文件

## 3. 服务识别与指纹识别

```bash
# 查看 HTTP 响应头
curl -I http://127.0.0.1:3000/

# 查看 Docker 容器信息
docker ps
docker inspect <容器ID>
```

通过响应头（Server、X-Powered-By 等）和 Docker 容器信息，确认 Web 应用的具体框架和版本。

## 4. 信息收集成果

- **Web 服务端口**：3000
- **技术栈**：Node.js + MongoDB（或根据实际填写）
- **已知账号信息**：尝试弱口令（admin/admin、root/root）
- **Redis 服务**：6379 端口开放，尝试无密码连接或弱口令

## 踩坑记录

- 如果 Nmap 扫不到端口，检查 Docker 容器的端口映射是否正确（`docker ps` 查看 `0.0.0.0:xxxx->xxxx/tcp`）。
- 如果本地 80 端口被占用，Docker 可能映射到 8080 或 8000，需根据实际输出调整扫描端口。
