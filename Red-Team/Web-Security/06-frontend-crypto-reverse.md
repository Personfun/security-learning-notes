# 06 - 前端加解密对抗实战：DES 动态 Key 逆向与 Python 复现

## 一、 背景说明
在现代 Web 业务系统中，出于安全考虑，前端通常会引入加解密技术来保护传输数据（如登录密码）。这导致在渗透测试中，Burp 抓包看到的往往是密文，使得后续的 SQL 注入、越权、密码爆破等测试难以开展。

本笔记记录一次完整的“前端 JS 逆向 + Python 脚本复现”的闭环实战，旨在解决测试过程中遇到前端加密无法直接测试的痛点。

- **测试环境**：CentOS 7 Docker 部署 `encrypt-labs`
- **测试目标**：`http://10.0.0.132:82/encrypt/des.php`
- **测试账号**：`admin` / `123456`
- **核心工具**：Burp Suite、Chrome DevTools、Python3 (pycryptodome)

## 二、 抓包分析：认清困境
在浏览器输入账号密码并提交，Burp 拦截到如下请求：

```http
POST /encrypt/des.php HTTP/1.1
Host: 10.0.0.132:82
Content-Type: application/json

{
  "username": "admin",
  "password": "3981f487c5afde43"
}
```

**分析发现**：
- `username` 为明文传输，服务端用于识别用户身份。
- `password` 变成了 `3981f487c5afde43`（长度为 16 位的十六进制字符串），无法直接读写出明文 `123456`。

## 三、 JS 逆向：定位并分析加密逻辑
在 Burp 抓包后，按 `F12` 打开开发者工具，切换至 `Sources`（源代码）面板。
全局搜索 `encrypt`，定位到关键函数 `encryptAndSendDataDES`。

### 1. 算法与参数
代码虽然经过了混淆处理（变量名为 `_0x...`），但通过分析核心逻辑，可以反推出以下参数：
- **算法**：DES（对称加密）
- **模式**：CBC
- **填充**：Pkcs7
- **输出格式**：Hex

### 2. Key 与 IV 的生成规则（动态生成）
这是该场景的难点，Key 和 IV 并非写死在前端，而是基于用户名动态计算得出：

- **Key 生成逻辑**：
  `username.slice(0, 8).padEnd(8, '6')`
  *逻辑*：截取用户名前8位，不足8位用字符'6'补齐。
  *示例*：`admin` -> `admin` -> `admin666`

- **IV 生成逻辑**：
  `'9999' + username.slice(0, 4).padEnd(4, '9')`
  *逻辑*：固定前缀'9999'拼接用户名前4位，不足4位用'9'补齐。
  *示例*：`admin` -> `admi` -> `9999admi`

## 四、 Python 脚本复现
为了验证我们的分析是否正确，使用 Python 完全模拟前端逻辑：

```python
from Crypto.Cipher import DES
from Crypto.Util.Padding import pad

# 1. 输入与前端一致
username = "admin"
password = "123456"

# 2. 生成 Key
key_str = username[:8].ljust(8, '6')
key = key_str.encode('utf-8')

# 3. 生成 IV
iv_str = '9999' + username[:4].ljust(4, '9')
iv = iv_str.encode('utf-8')

# 4. DES 加密（CBC 模式，Pkcs7 填充）
cipher = DES.new(key, DES.MODE_CBC, iv)
padded_password = pad(password.encode('utf-8'), DES.block_size)
encrypted_bytes = cipher.encrypt(padded_password)

# 5. 输出 Hex
print(f"[*] 生成的 Key: {key_str}")
print(f"[*] 生成的 IV : {iv_str}")
print(f"[*] 前端密文 (Hex): {encrypted_bytes.hex()}")
```

**执行结果**：
```text
[*] 生成的 Key: admin666
[*] 生成的 IV : 9999admi
[*] 前端密文 (Hex): 3981f487c5afde43
```

## 五、 验证结论
- **Burp 抓包密文**：`3981f487c5afde43`
- **Python 计算密文**：`3981f487c5afde43`
- **结论**：字符完全一致，逆向成功。证明了前端加密算法、模式、填充以及动态密钥生成逻辑已完全被掌控。

## 六、 实战经验总结与扩展思考
1. **不要死磕密文**：看到密文的第一反应应该是寻找加密函数，而不是尝试直接去解密（除非是弱算法）。
2. **动态 Key 的应对思路**：遇到非写死的 Key，不要急于去爆破。先观察它基于什么参数生成（常见的有用户名、时间戳、随机数）。可以通过**控制变量法**（例如修改用户名）观察生成的密文变化，反推其生成逻辑，最后再读混淆代码验证。
3. **后续自动化测试方向**：
   - 在掌握纯 Python 复现脚本后，可以考虑编写脚本生成密码字典，对目标进行自动化测试验证。
   - 进阶方向是配置 Burp 插件（如 `autoDecoder`）或使用 `mitmproxy` 编写自定义脚本。将复现的 Python 逻辑部署到中间人代理中，实现**浏览器/工具端输入明文，代理层自动加密并发送**的透明代理效果，从而让常规的漏洞扫描工具正常工作。

## 七、 附：脚本环境准备
在 CentOS 7 或 Kali Linux 下，需安装依赖库：
```bash
pip3 install pycryptodome
```
## 踩坑记录
在 DES 关卡中，服务端解密始终报错 `error:0308010C:digital envelope routines::unsupported`。
排查发现，PHP 8.2 底层的 OpenSSL 3.x 默认禁用了 DES 算法（移到了 legacy provider），
属于靶场自身的兼容性问题，并非前端逆向出错。
解决方案：跳过 DES 关卡，直接使用 AES 关卡进行学习（AES 在 OpenSSL 3.x 中正常支持）。

## 八、实战验证：AES 固定 Key 关卡

### 8.1 前端加密逻辑分析
通过 DevTools 定位 `sendDataAes` 函数，分析出：
- **算法**：AES-128-CBC
- **Key**：`1234567890123456`（写死）
- **IV**：`1234567890123456`（写死）
- **填充**：Pkcs7
- **加密范围**：整个 JSON（`{"username":"admin","password":"123456"}`）
- **输出格式**：Base64，再经 URL 编码

### 8.2 Python 复现验证
```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
import base64
import json

# 1. 明文（和前端一样，用 JSON 格式）
username = "admin"
password = "123456"
plaintext = json.dumps({"username": username, "password": password}, separators=(',', ':'))
print(f"[*] 明文 JSON: {plaintext}")

# 2. Key 和 IV（写死的）
key = b'1234567890123456'
iv = b'1234567890123456'

# 3. AES-CBC-Pkcs7 加密
cipher = AES.new(key, AES.MODE_CBC, iv)
padded = pad(plaintext.encode('utf-8'), AES.block_size)
encrypted_bytes = cipher.encrypt(padded)

# 4. 转 Base64（前端 toString() 默认就是这个）
base64_result = base64.b64encode(encrypted_bytes).decode('utf-8')
print(f"[*] AES 密文 (Base64): {base64_result}")
```

**验证结果**：
- Python 生成：`nArXfVdnoe67UzojAPP2X+6qSiznLMBAI3a5Bi+zlNzXaUb9+gTXusl67b+DS9Zw`
- Burp 抓包（URL 解码后）：完全一致

### 8.3 武器化爆破实战
1. 用 Python 批量生成 20 个常见密码的 AES 密文，输出到 `aes_payloads.txt`
2. Burp Intruder 配置 Positions（只标记 `encryptedData=` 后的密文段）
3. Payloads 加载 `aes_payloads.txt`
4. Start Attack
5. **结果**：第 1 条（`123456`）返回 Length=16，Response=`{"success":true}`，命中！

### 8.4 关键经验
- **Length 是判断依据**：成功响应 vs 失败响应的 Body 长度差异明显
- **URL 编码要处理**：前端用了 `encodeURIComponent`，Python 里用 `urllib.parse.quote` 对齐
- **JSON 分隔符要严格对齐**：`json.dumps(..., separators=(',', ':'))` 保证和 `JSON.stringify` 一致（无空格）

### 8.5 踩坑记录（DES 关卡）
DES 关卡的服务端解密报错 `error:0308010C:digital envelope routines::unsupported`。
原因是 PHP 8.2 底层的 OpenSSL 3.x 默认禁用了 DES 算法（移到了 legacy provider）。
属于靶场自身兼容性问题，非逆向错误。**实战中 DES 已淘汰，重点掌握 AES 和 RSA。**
