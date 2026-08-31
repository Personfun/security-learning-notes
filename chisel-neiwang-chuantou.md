# chisel 内网穿透学习笔记

## 1. 什么是 chisel

chisel 是一款单二进制的内网穿透工具，支持反向隧道和 SOCKS 代理。

## 2. 环境

| 主机 | IP | 角色 |
| ---- | ---- | ---- |
| Kali | 10.0.0.131 | chisel 服务端 |
| CentOS | 10.0.0.132 | chisel 客户端 |

## 3. 服务端启动

```bash
./chisel server -p 7001 --reverse
```

## 4. 客户端启动

```bash
./chisel client 10.0.0.131:7001 R:6002:127.0.0.1:22
```

## 5. 验证

```bash
ssh root@127.0.0.1 -p 6002
```

## 6. 总结

chisel 和 frp、nps 原理类似，但更轻量，适合快速建立反向隧道。
