# 06 - 前端加解密对抗实战：从 AES 到 RSA 混合加密的完整逆向链

## 一、 背景说明

在现代 Web 业务系统中，出于安全考虑，前端通常会引入加解密技术来保护传输数据（如登录密码）。这导致在渗透测试中，Burp 抓包看到的往往是密文，使得后续的 SQL 注入、越权、密码爆破等测试难以开展。

本笔记记录一次完整的"前端 JS 逆向 + Python 脚本复现"闭环实战，覆盖**对称加密（AES/DES）、非对称加密（RSA）、混合加密（AES+RSA）** 四大典型场景。

- **测试环境**：CentOS 7 Docker 部署 `encrypt-labs`
- **测试账号**：`admin` / `123456`
- **核心工具**：Burp Suite、Chrome DevTools、Python3 (pycryptodome)

## 二、 通用方法论

面对任何前端加密的目标，遵循以下四步：

```
1. 抓包      → 看到密文请求
2. 定位      → F12 Sources 面板找加密函数（全局搜索 encrypt / CryptoJS / setPublicKey）
3. 逆向      → 分析算法、Key/IV、Mode、Padding、输出格式
4. 复现      → 用 Python 100% 复现前端逻辑，本地自检通过后再发出去
```

**关键原则**：
- **不要死磕密文**——密文本身没有信息，价值在于找到它的生成函数。
- **本地自检胜过盲目爆破**——先证明"我加密的，自己能解开"，再发出去验证。
- **代码混淆不是障碍**——变量名变成 `_0x...` 不影响逻辑，看调用栈和参数就能还原。

## 三、 对称加密通用模型

无论是 AES 还是 DES，对称加密的流程完全一致，可以浓缩成一个"万能公式"：

```
密文 = 格式转换( 加密器.加密( 填充( 明文 ) ) )
       ↑                ↑             ↑
     Hex/Base64      Key + IV +      Pkcs7 等
                      Mode
```

**四个关键参数**：
| 参数 | 说明 | 常见值 |
| :--- | :--- | :--- |
| 算法 | 决定用哪个加密器 | AES、DES、3DES、SM4 |
| Key/IV | 密钥和初始向量 | 写死、动态生成、服务端下发 |
| Mode | 加密模式 | CBC、ECB、CFB、OFB、CTR |
| Padding | 填充方式 | Pkcs7、Pkcs5、ZeroPadding |

**两个易混淆概念**：
- **块大小（Block Size）**：决定填充基准。DES 固定 8 字节，AES 固定 16 字节。
- **密钥长度（Key Size）**：决定强度。DES 固定 8 字节，AES 可以是 16/24/32 字节。
- **IV 长度总是等于块大小**。

**输出格式与算法无关**：Hex 和 Base64 只是编码方式，同一个算法可能输出 Hex，也可能输出 Base64，取决于开发选择。

## 四、 AES 固定 Key 关卡实战

### 4.1 抓包分析

Burp 拦截到如下请求：

```http
POST /encrypt/aes.php HTTP/1.1
Host: 10.0.0.132:82
Content-Type: application/x-www-form-urlencoded

encryptedData=nArXfVdnoe67UzojAPP2X%2B6qSiznLMBAI3a5Bi%2BzlNzXaUb9%2BgTXusl67b%2BDS9Zw
```

密文是 Base64 编码后再经 URL 编码。

### 4.2 前端加密逻辑分析

通过 DevTools 定位 `sendDataAes` 函数，分析出：

- **算法**：AES-128-CBC
- **Key**：`1234567890123456`（写死）
- **IV**：`1234567890123456`（写死）
- **填充**：Pkcs7
- **加密范围**：整个 JSON（`{"username":"admin","password":"123456"}`）
- **输出格式**：Base64，再经 `encodeURIComponent` URL 编码

### 4.3 Python 复现

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
import base64
import json

# 1. 明文（和前端一样，用 JSON 格式，separators 保证无空格）
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

### 4.4 武器化爆破实战

1. 用 Python 批量生成 20 个常见密码的 AES 密文，输出到 `aes_payloads.txt`
2. Burp Intruder 配置 Positions（只标记 `encryptedData=` 后的密文段）
3. Payloads 加载 `aes_payloads.txt`，Start Attack
4. **结果**：第 1 条（`123456`）返回 Length=16，Response=`{"success":true}`，命中！

### 4.5 关键经验

- **Length 是判断依据**：成功响应 vs 失败响应的 Body 长度差异明显。
- **URL 编码要处理**：前端用了 `encodeURIComponent`，Python 里用 `urllib.parse.quote` 对齐。
- **JSON 分隔符要严格对齐**：`json.dumps(..., separators=(',', ':'))` 保证和 `JSON.stringify` 一致。

## 五、 DES 动态 Key 关卡（原理学习）

### 5.1 前端加密逻辑

定位 `encryptAndSendDataDES` 函数：

- **算法**：DES-CBC
- **填充**：Pkcs7
- **输出格式**：Hex
- **Key 生成**：`username.slice(0, 8).padEnd(8, '6')` → `admin666`
- **IV 生成**：`'9999' + username.slice(0, 4).padEnd(4, '9')` → `9999admi`

### 5.2 Python 复现

```python
from Crypto.Cipher import DES
from Crypto.Util.Padding import pad

username = "admin"
password = "123456"

# 动态生成 Key 和 IV
key_str = username[:8].ljust(8, '6')
key = key_str.encode('utf-8')
iv_str = '9999' + username[:4].ljust(4, '9')
iv = iv_str.encode('utf-8')

# DES 加密
cipher = DES.new(key, DES.MODE_CBC, iv)
padded_password = pad(password.encode('utf-8'), DES.block_size)
encrypted_bytes = cipher.encrypt(padded_password)

print(f"[*] 生成的 Key: {key_str}")
print(f"[*] 生成的 IV : {iv_str}")
print(f"[*] 前端密文 (Hex): {encrypted_bytes.hex()}")
```

**验证结果**：
- Python 生成：`3981f487c5afde43`
- Burp 抓包：`3981f487c5afde43`
- **完全一致，逆向成功**。

### 5.3 踩坑记录：DES 在 OpenSSL 3.x 下不可用

服务端解密始终报错 `error:0308010C:digital envelope routines::unsupported`。

**原因**：PHP 8.2 底层的 OpenSSL 3.x 出于安全考虑，将 DES、3DES、RC4 等弱算法移到了 legacy provider，默认不加载。

**结论**：属于靶场自身的兼容性问题，非逆向错误。**实战中 DES 已淘汰，重点掌握 AES 和 RSA。**

## 六、 纯 RSA 关卡实战

### 6.1 前端加密逻辑

定位 `sendEncryptedDataRSA` 函数：

- **算法**：RSA（非对称加密）
- **公钥**：明文写死在 JS 中（`MIGfMA0GCSq...`）
- **填充**：PKCS#1 v1.5
- **加密对象**：整个 JSON
- **输出**：Base64，再经 `URLSearchParams` 编码
- **请求体**：`data=<密文>`

### 6.2 Python 复现

```python
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
import base64
import json
import urllib.parse

public_key_pem = b"""-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDRvA7giwinEkaTYllDYCkzujvi
NH+up0XAKXQot8RixKGpB7nr8AdidEvuo+wVCxZwDK3hlcRGrrqt0Gxqwc11btlM
DSj92Mr3xSaJcshZU8kfj325L8DRh9jpruphHBfh955ihvbednGAvOHOrz3Qy3Cb
ocDbsNeCwNpRxwjIdQIDAQAB
-----END PUBLIC KEY-----"""

plaintext = json.dumps({"username": "admin", "password": "123456"}, separators=(',', ':'))
print(f"[*] 明文 JSON: {plaintext}")

# RSA 加密
pub = RSA.import_key(public_key_pem)
cipher_enc = PKCS1_v1_5.new(pub)
encrypted = cipher_enc.encrypt(plaintext.encode('utf-8'))
b64_cipher = base64.b64encode(encrypted).decode('utf-8')
print(f"[*] RSA 密文: {b64_cipher}")

# URL 编码（用于 Burp）
url_encoded = urllib.parse.quote(b64_cipher, safe='')
print(f"[*] 用于 Burp 的 URL 编码密文:\n{url_encoded}")
```

### 6.3 关键坑点：密文里的 `+` 号

RSA 密文是 Base64，包含 `+`、`/`、`=`。直接放到 URL 参数里会被破坏。

- **`+` 会被当作空格**
- **`/` 会被路径解析**
- **`=` 会被参数解析**

**正确做法**：Python 里用 `urllib.parse.quote(cipher, safe='')` 做 URL 编码。
**错误示范**：直接复制密文进 Burp，返回 `Missing username or password`。
**正确验证**：URL 编码后发送，返回 `{"success":true}`。

### 6.4 核心认知

- **RSA 公钥是公开的**，逆向目标是"提取公钥并复现加密"，不是"破解"。
- **RSA 每次加密结果都不同**（内含随机填充），不能用"密文对比"验证，正确方式是用私钥自解密，比对明文是否一致。
- **RSA 加密长度有限**（≤ 密钥长度 - 11 字节），这就是为什么真实业务要用 AES+RSA 混合加密。

## 七、 AES+RSA 混合加密（工业级方案）

### 7.1 业务场景

真实业务（如 App 登录、运营商网厅）的工业级方案：
- **AES 加密数据主体**（快）
- **RSA 加密 AES 的 Key/IV**（安全传输密钥）
- **每次登录随机生成 Key/IV**（防重放、防分析）

### 7.2 前端加密流程

定位 `sendDataAesRsa` 函数：

1. `CryptoJS.lib.WordArray.random(16)` 生成随机 AES Key 和 IV
2. 用 AES-128-CBC-Pkcs7 加密整个 JSON，输出 Base64
3. 用 RSA 公钥分别加密 **AES Key 的 Base64 字符串** 和 **IV 的 Base64 字符串**
4. 组装三个字段：`encryptedData`、`encryptedKey`、`encryptedIv`

### 7.3 服务端解密流程（从源码读出）

```php
$aesKey = decryptRSA($encryptedKey, $privateKey);  // RSA 解密 → 得到 Base64 字符串
$aesIv  = decryptRSA($encryptedIv,  $privateKey);

$decryptedData = openssl_decrypt(
    base64_decode($encryptedData),
    'aes-128-cbc',
    base64_decode($aesKey),   // 再 base64_decode 才得到 16 字节
    OPENSSL_RAW_DATA,
    base64_decode($aesIv)
);
```

### 7.4 Python 复现

```python
from Crypto.Cipher import AES, PKCS1_v1_5
from Crypto.PublicKey import RSA
from Crypto.Util.Padding import pad
import base64
import json
import os

public_key_pem = b"""-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDRvA7giwinEkaTYllDYCkzujvi
NH+up0XAKXQot8RixKGpB7nr8AdidEvuo+wVCxZwDK3hlcRGrrqt0Gxqwc11btlM
DSj92Mr3xSaJcshZU8kfj325L8DRh9jpruphHBfh955ihvbednGAvOHOrz3Qy3Cb
ocDbsNeCwNpRxwjIdQIDAQAB
-----END PUBLIC KEY-----"""

# ===== 加密流程 =====

# 1. 随机生成 16 字节 AES Key 和 IV
aes_key = os.urandom(16)
aes_iv  = os.urandom(16)

# 2. AES-128-CBC 加密 JSON，输出 Base64
plaintext = json.dumps({"username": "admin", "password": "123456"}, separators=(',', ':'))
cipher_aes = AES.new(aes_key, AES.MODE_CBC, aes_iv)
encrypted_data = base64.b64encode(cipher_aes.encrypt(pad(plaintext.encode(), 16))).decode()

# 3. 用 RSA 公钥加密 AES Key/IV 的 Base64 字符串
aes_key_b64 = base64.b64encode(aes_key).decode()
aes_iv_b64  = base64.b64encode(aes_iv).decode()

rsa_pub = RSA.import_key(public_key_pem)
cipher_rsa = PKCS1_v1_5.new(rsa_pub)
encrypted_key = base64.b64encode(cipher_rsa.encrypt(aes_key_b64.encode())).decode()
encrypted_iv  = base64.b64encode(cipher_rsa.encrypt(aes_iv_b64.encode())).decode()

# 4. 组装 JSON
body = {
    "encryptedData": encrypted_data,
    "encryptedKey": encrypted_key,
    "encryptedIv": encrypted_iv
}
print("[*] 请求体 JSON:")
print(json.dumps(body, indent=2))
```

**验证结果**：
- 用服务端私钥本地自检：`Key 是否一致: True`、`IV 是否一致: True`、`是否与原始明文一致: True`
- Burp Repeater 发送 JSON，返回 `{"success":true}`

### 7.5 关键坑点

1. **RSA 加密的对象是 Base64 字符串**，不是原始 16 字节。前端是 `aesKey.toString(Base64)`，服务端要再 `base64_decode` 一次。
2. **AES 模式是 CBC，填充是 Pkcs7，块大小是 16**。
3. **本地自检方法**：用服务端源码里的私钥，在 Python 里解密自己生成的密文，验证明文是否一致。

### 7.6 核心认知

- **RSA 只加密密钥，不加密数据**——这是混合加密的精髓。
- **AES Key 每次随机**——即使同一个密码，每次请求密文都不同。
- **前后端对齐检查**：先本地自检通过，再发出去，这样即使失败也能快速定位是前端逻辑错还是服务端拒收。

## 八、 总结与能力清单

经过这四个关卡的实战，完整掌握了前端加解密对抗的核心能力：

| 能力 | 状态 |
| :--- | :--- |
| 前端 JS 逆向（含混淆代码） | ✅ |
| 识别 AES / DES / RSA 算法特征 | ✅ |
| 提取写死或动态生成的 Key/IV | ✅ |
| Python 复现 AES 对称加密 | ✅ |
| Python 复现 RSA 非对称加密 | ✅ |
| Python 复现 AES+RSA 混合加密 | ✅ |
| Burp Intruder 自动化爆破 | ✅ |
| URL 编码 / Base64 / Hex 三种格式处理 | ✅ |
| 服务端源码审计对齐 | ✅ |
| 本地自检验证思路 | ✅ |

**下一步进阶方向**：

1. **autoDecoder 插件**：把 Python 脚本包装成 HTTP 服务，让 Burp 自动加解密。Burp 里看到的是明文，发出去的自动加密，实现"透明代理"效果。
2. **签名（Sign）逆向**：处理 `sign = MD5(参数排序 + 盐 + timestamp)` 类防护。
3. **防重放（Nonce + Timestamp）**：理解时间窗口、一次性令牌等机制。

## 九、 附：脚本环境准备

在 CentOS 7 或 Kali Linux 下，需安装依赖库：

```bash
pip3 install pycryptodome
```

> 注：安装包名是 `pycryptodome`，但导入时用 `from Crypto.xxx import xxx`，是历史兼容原因。

## 十、HMAC-SHA256 签名 + 防重放关卡

### 10.1 前端加密逻辑
定位 `sendDataWithNonce` 函数：
- **哈希函数**：HMAC-SHA256
- **盐（Secret）**：`be56e057f20f883e`
- **拼接规则**：`username + password + nonce + timestamp`（无分隔符）
- **nonce 生成**：`Math.random().toString(36).substring(2)`
- **timestamp**：`Math.floor(Date.now() / 1000)`（秒级）
- **输出**：Hex

### 10.2 Python 复现
```python
import hmac
import hashlib
import json
import time
import random
import string

# 生成 nonce（12 位小写字母+数字）
def gen_nonce():
    chars = string.ascii_lowercase + string.digits
    return ''.join(random.choice(chars) for _ in range(12))

# 输入
username = "admin"
password = "123456"
nonce = gen_nonce()             # ← 每次随机
timestamp = int(time.time())    # ← 每次当前时间
secret = "be56e057f20f883e"

# 拼接
message = username + password + nonce + str(timestamp)
print(f"[*] nonce:     {nonce}")
print(f"[*] timestamp: {timestamp}")
print(f"[*] 待签名原文: {message}")

# HMAC-SHA256
signature = hmac.new(
    secret.encode('utf-8'),
    message.encode('utf-8'),
    hashlib.sha256
).hexdigest()
print(f"[*] signature: {signature}")

# 组装 JSON
body = {
    "username": username,
    "password": password,
    "nonce": nonce,
    "timestamp": timestamp,
    "signature": signature
}
print("\n[*] 请求体 JSON:")
print(json.dumps(body, indent=2))
```

### 10.3 验证结果
- Python 生成新请求 → Burp 发送 → 服务端返回 `{"success":true}`

### 10.4 关键认知
- HMAC-SHA256 比纯 MD5/SHA256 更安全，盐作为算法参数传入。
- 拼接规则没有分隔符时要注意顺序，控制变量法可以推导。
- 时间戳和 nonce 是防重放的核心机制。

## 十一、禁止重放关卡实战

### 11.1 前端加密逻辑
定位 `sendLoginRequest` + `generateRequestData` 函数：
- **加密对象**：毫秒级时间戳 `Date.now()`
- **加密方式**：RSA 公钥加密
- **发送字段**：`username` + `password` + `random`（加密后的时间戳）
- **请求格式**：JSON

### 11.2 服务端防重放机制（从源码读出）
```php
$timestamp = rsaDecrypt($data['random'], $privateKey);
$currentTimestamp = time() * 1000;
$timeWindow = 3000;  // 3秒窗口！

if (abs($currentTimestamp - $timestamp) > $timeWindow) {
    echo json_encode(['success' => false, 'error' => 'No Repeater']);
    exit;
}

$requestID = hash('sha256', $username . $password . $timestamp . $currentTimestamp);
// 检查 requestID 是否已存在（防重放第二层）
```

**双重防护**：
1. **时间窗口**：3 秒内的时间戳才有效。
2. **requestID 唯一性**：同一个 requestID 只能用一次。

### 12.3 Python 复现
```python
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
import base64
import json
import time
import requests

# 1. 前端 JS 里的公钥
public_key_pem = b"""-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDRvA7giwinEkaTYllDYCkzujvi
NH+up0XAKXQot8RixKGpB7nr8AdidEvuo+wVCxZwDK3hlcRGrrqt0Gxqwc11btlM
DSj92Mr3xSaJcshZU8kfj325L8DRh9jpruphHBfh955ihvbednGAvOHOrz3Qy3Cb
ocDbsNeCwNpRxwjIdQIDAQAB
-----END PUBLIC KEY-----"""

# 2. 生成毫秒级时间戳（和前端 Date.now() 一致）
timestamp_ms = int(time.time() * 1000)
print(f"[*] 时间戳（毫秒）: {timestamp_ms}")

# 3. RSA 加密时间戳
pub = RSA.import_key(public_key_pem)
cipher = PKCS1_v1_5.new(pub)
encrypted = cipher.encrypt(str(timestamp_ms).encode())
random_field = base64.b64encode(encrypted).decode()
print(f"[*] random 字段: {random_field[:60]}...")

# 4. 组装 JSON
body = {
    "username": "admin",
    "password": "123456",
    "random": random_field
}

# 5. 立刻发送（不能超过 3 秒！）
url = "http://10.0.0.132:82/encrypt/norepeater.php"

# 方式 A：不走 Burp 代理，直接发
response = requests.post(url, json=body)
print(f"\n[*] 响应: {response.text}")
```

### 12.4 关键认知
- **3 秒窗口太短**，必须用脚本直接发请求，不能手动复制到 Burp。
- RSA 在这里不是加密密码，而是**加密时间戳防篡改**。
- 密码在请求里是**明文**——服务端用 `md5($password)` 比对，不需要解密。
- 用 `requests.post(url, json=body)` 直接发送，可加 `proxies` 参数走 Burp 观察流量。
