# 04 面试问题点与复盘

针对域渗透靶场，整理高频面试题及标准回答思路。

## Q1：AS-REP Roasting 和 Kerberoasting 的区别是什么？
**A**：
- **AS-REP Roasting** 针对的是**未开启 Kerberos 预认证**的域用户。攻击者可以直接请求 AS-REP 响应，拿到加密的 TGT 哈希进行离线破解。
- **Kerberoasting** 针对的是**注册了 SPN 的服务账户**。攻击者通过请求 TGS 服务票据，拿到加密的 TGS 哈希进行离线破解。
- 两者请求的票据阶段不同，hashcat 破解模式也不同（AS-REP 是 18200，Kerberoasting 是 13100）。

## Q2：DCSync 的原理是什么？
**A**：DCSync 利用了域控的“目录复制服务 (DRS)”。正常情况下只有域控之间会互相同步数据。如果攻击者获取了具有“复制目录更改”权限的账户（如域管或特定服务账户），就可以伪装成一台域控，向真实域控请求同步数据（包括所有用户的 NTLM 哈希和 KRBTGT 哈希）。常用工具是 `impacket-secretsdump`。

## Q3：什么是黄金票据（Golden Ticket）？
**A**：黄金票据是一种权限维持技术。当攻击者拿到 KRBTGT 账户的 NTLM 哈希后，可以使用 `impacket-ticketer` 伪造任意用户（如 Administrator）的 TGT 票据。因为 TGT 是由 KRBTGT 签名的，伪造的票据会被域控认为是合法的，从而获得域管权限。它不依赖账户密码，隐蔽性很强。

## Q4：你如何实现横向移动？PTH 的原理是什么？
**A**：横向移动常用工具有 PsExec、WMI、SMB 等。核心原理是 Pass The Hash (PTH)，即当攻击者拿到用户 NTLM 哈希后，不需要明文密码，直接利用哈希进行认证。在 Windows 中，NTLM 认证只使用哈希，所以只要哈希不变，密码更改前都能成功。通过 `impacket-psexec -hashes` 即可实现。

## Q5：BloodHound 在域渗透中的作用是什么？
**A**：BloodHound 是一个域环境分析工具。它通过收集 LDAP 数据（用户、组、计算机、会话、ACL 等），将域内的信任关系和攻击路径可视化。攻击者可以快速找到从普通域用户到域管的最短路径，比如“用户 A 对计算机 B 有管理权限，计算机 B 上有域管会话”等。

## Q6：从蓝队角度，你觉得域渗透哪一步最容易被发现？
**A**：从蓝队视角，最容易被发现的是：
1. **Kerberoasting**：事件 ID 4769 中大量 RC4 加密的 TGS 请求，非常显眼。
2. **DCSync**：事件 ID 4662 中非域控机器发起目录复制，这是高危告警。
3. **PsExec 横向**：会在目标机器上创建 `PSEXESVC.exe` 服务，事件 ID 4688 和 7045 都会记录。
