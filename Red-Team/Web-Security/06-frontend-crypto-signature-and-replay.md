# 06 - 前端加密、签名与防重放逆向实战

## 一、 背景说明

在现代 Web 业务系统中，出于安全考虑，前端通常会引入多层防护来保护请求数据：

- **加密**（AES/RSA）保护数据的机密性
- **签名**（HMAC）保护数据的完整性
- **防重放**（时间戳 + nonce）保护请求的时效性和唯一性

这导致在渗透测试中，Burp 抓包看到的往往是密文或带签名的请求，使得后续的 SQL 注入、越权、密码爆破等测试难以开展。

本笔记记录一次完整的"前端 JS 逆向 + Python 脚本复现"闭环实战，覆盖以下典型场景：

| 类别 | 场景 |
| :--- | :--- |
| **对称加密** | AES 固定 Key、AES 服务端下发 Key、DES 动态 Key |
| **非对称加密** | 纯 RSA |
| **混合加密** | AES + RSA（工业级方案） |
| **签名** | 明文加签（HMAC-SHA256）、加签 key 在服务端 |
| **防重放** | RSA 加密时间戳 + 3 秒窗口 + requestID |

- **测试环境**：CentOS 7 Docker 部署 `encrypt-labs`
- **测试账号**：`admin` / `123456`
- **核心工具**：Burp Suite、Chrome DevTools、Python3 (pycryptodome)

## 二、 通用方法论

面对任何前端加密或签名的目标，遵循以下五步：

```
1. 抓包      → 看到密文或签名请求
2. 定位      → F12 Sources 面板找关键函数（全局搜索 encrypt / CryptoJS / setPublicKey / Hmac）
3. 逆向      → 分析算法、Key/IV、Mode、Padding、拼接规则、盐
4. 复现      → 用 Python 100% 复现前端逻辑，本地自检通过后再发出去
5. 验证      → 用 Burp Repeater 或 requests 直接发送，观察服务端响应
```

**关键原则**：

- **不要死磕密文**——密文本身没有信息，价值在于找到它的生成函数。
- **本地自检胜过盲目爆破**——先证明"我加密的，自己能解开"，再发出去验证。
- **代码混淆不是障碍**——变量名变成 `_0x...` 不影响逻辑，看调用栈和参数就能还原。
- **控制变量法是逆向利器**——改一个参数，看输出变化，反推拼接规则。

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

username = "admin"
password = "123456"
plaintext = json.dumps({"username": username, "password": password}, separators=(',', ':'))
print(f"[*] 明文 JSON: {plaintext}")

key = b'1234567890123456'
iv = b'1234567890123456'

cipher = AES.new(key, AES.MODE_CBC, iv)
padded = pad(plaintext.encode('utf-8'), AES.block_size)
encrypted_bytes = cipher.encrypt(padded)

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

key_str = username[:8].ljust(8, '6')
key = key_str.encode('utf-8')
iv_str = '9999' + username[:4].ljust(4, '9')
iv = iv_str.encode('utf-8')

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

pub = RSA.import_key(public_key_pem)
cipher_enc = PKCS1_v1_5.new(pub)
encrypted = cipher_enc.encrypt(plaintext.encode('utf-8'))
b64_cipher = base64.b64encode(encrypted).decode('utf-8')
print(f"[*] RSA 密文: {b64_cipher}")

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

## 八、 AES 服务端获取 Key 关卡

### 8.1 与 AES 固定 Key 的差异

| 维度 | AES固定Key | AES服务端获取Key |
| :--- | :--- | :--- |
| Key/IV 位置 | 写死在前端 JS | 服务端动态下发 |
| 逆向方式 | 搜代码找到 | 抓包分析两个请求 |
| Python 复现 | 一条请求 | 两步流程 + Session 保持 |
| Key/IV 变化 | 固定 | 每次请求都不同 |

### 8.2 完整流程

**两步请求**：

1. `GET /encrypt/server_generate_key.php` → 返回 `{"aes_key": "...", "aes_iv": "..."}`
2. `POST /encrypt/aesserver.php` → 发送 `{"encryptedData": "..."}`

**服务端响应**：

```json
{
    "aes_key": "6NWKZA8LG/pk1071c/z9xw==",
    "aes_iv": "sfFYXckb/AoYNXbTTc+5gw=="
}
```

两个都是 Base64 编码的 16 字节。

### 8.3 Python 复现

```python
import requests
import json
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad

session = requests.Session()
base_url = "http://10.0.0.132:82"

# 第一步：请求 Key/IV
resp1 = session.get(f"{base_url}/encrypt/server_generate_key.php")
data = resp1.json()
aes_key = base64.b64decode(data["aes_key"])
aes_iv  = base64.b64decode(data["aes_iv"])

# 第二步：AES 加密
plaintext = json.dumps({"username": "admin", "password": "123456"}, separators=(',', ':'))
cipher = AES.new(aes_key, AES.MODE_CBC, aes_iv)
encrypted = base64.b64encode(cipher.encrypt(pad(plaintext.encode(), 16))).decode()

# 第三步：发送
resp2 = session.post(
    f"{base_url}/encrypt/aesserver.php",
    json={"encryptedData": encrypted},
    headers={"Content-Type": "application/json"}
)
print(resp2.text)
```

### 8.4 关键坑点

1. **必须用 `requests.Session()`**：Cookie（PHPSESSID）要跨两个请求保持不变，否则服务端找不到对应的 Key/IV。
2. **Key/IV 要 Base64 解码**：服务端下发的是 Base64 字符串，对应前端 JS 里的 `CryptoJS.enc.Base64.parse()`。
3. **服务端每次都生成新 Key/IV**：即使同一账号连续请求，Key/IV 也不同。

### 8.5 核心认知

- 这个关卡模拟的是真实业务里的**"会话级密钥"**机制——Key/IV 每次登录时动态生成，绑定到当前 Session。
- 攻击者即使抓包看到了密文，也无法解密，因为没有 Key（Key 只在服务端内存里短暂存在）。
- 但**仍然可以被绕过**：攻击者可以自己去要一份 Key/IV，然后用它加密任意数据发出去——这正是我们 Python 脚本做的事。

## 九、 HMAC-SHA256 签名 + 防重放（明文加签）

### 9.1 前端加密逻辑

定位 `sendDataWithNonce` 函数：

- **哈希函数**：HMAC-SHA256
- **盐（Secret）**：`be56e057f20f883e`
- **拼接规则**：`username + password + nonce + timestamp`（无分隔符）
- **nonce 生成**：`Math.random().toString(36).substring(2)`
- **timestamp**：`Math.floor(Date.now() / 1000)`（秒级）
- **输出**：Hex

### 9.2 Python 复现

```python
import hmac
import hashlib
import json
import time
import random
import string

def gen_nonce():
    chars = string.ascii_lowercase + string.digits
    return ''.join(random.choice(chars) for _ in range(12))

username = "admin"
password = "123456"
nonce = gen_nonce()
timestamp = int(time.time())
secret = "be56e057f20f883e"

message = username + password + nonce + str(timestamp)
print(f"[*] nonce:     {nonce}")
print(f"[*] timestamp: {timestamp}")
print(f"[*] 待签名原文: {message}")

signature = hmac.new(
    secret.encode('utf-8'),
    message.encode('utf-8'),
    hashlib.sha256
).hexdigest()
print(f"[*] signature: {signature}")

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

### 9.3 验证结果

- Python 生成新请求 → Burp 发送 → 服务端返回 `{"success":true}`

### 9.4 关键认知

- HMAC-SHA256 比纯 MD5/SHA256 更安全，盐作为算法参数传入。
- 拼接规则没有分隔符时要注意顺序，控制变量法可以推导。
- 时间戳和 nonce 是防重放的核心机制。

## 十、 禁止重放关卡实战

### 10.1 前端加密逻辑

定位 `sendLoginRequest` + `generateRequestData` 函数：

- **加密对象**：毫秒级时间戳 `Date.now()`
- **加密方式**：RSA 公钥加密
- **发送字段**：`username` + `password` + `random`（加密后的时间戳）
- **请求格式**：JSON

### 10.2 服务端防重放机制（从源码读出）

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

### 10.3 Python 复现

```python
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
import base64
import json
import time
import requests

public_key_pem = b"""-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDRvA7giwinEkaTYllDYCkzujvi
NH+up0XAKXQot8RixKGpB7nr8AdidEvuo+wVCxZwDK3hlcRGrrqt0Gxqwc11btlM
DSj92Mr3xSaJcshZU8kfj325L8DRh9jpruphHBfh955ihvbednGAvOHOrz3Qy3Cb
ocDbsNeCwNpRxwjIdQIDAQAB
-----END PUBLIC KEY-----"""

timestamp_ms = int(time.time() * 1000)
print(f"[*] 时间戳（毫秒）: {timestamp_ms}")

pub = RSA.import_key(public_key_pem)
cipher = PKCS1_v1_5.new(pub)
encrypted = cipher.encrypt(str(timestamp_ms).encode())
random_field = base64.b64encode(encrypted).decode()
print(f"[*] random 字段: {random_field[:60]}...")

body = {
    "username": "admin",
    "password": "123456",
    "random": random_field
}

url = "http://10.0.0.132:82/encrypt/norepeater.php"
response = requests.post(url, json=body)
print(f"\n[*] 响应: {response.text}")
```

### 10.4 关键认知

- **3 秒窗口太短**，必须用脚本直接发请求，不能手动复制到 Burp。
- RSA 在这里不是加密密码，而是**加密时间戳防篡改**。
- 密码在请求里是**明文**——服务端用 `md5($password)` 比对，不需要解密。
- 用 `requests.post(url, json=body)` 直接发送，可加 `proxies` 参数走 Burp 观察流量。

## 十一、 加签 key 在服务端关卡

### 11.1 与"明文加签"的差异

| 维度 | 明文加签 | 加签key在服务端 |
| :--- | :--- | :--- |
| 盐的位置 | 写死在前端 JS（`be56e057f20f883e`） | **服务端持有，前端拿不到** |
| 签名生成方 | 前端自己算 | **先去服务端要** |
| 逆向方式 | 直接读代码 | 抓包分析两步请求 |
| 关键约束 | 无 | **timestamp 两步必须一致** |

### 11.2 完整流程

**两步请求**：

1. `POST /encrypt/get-signature.php` → 发 `{username, password, timestamp}` → 返回 `{signature}`
2. `POST /encrypt/signdataserver.php` → 发 `{username, password, timestamp, signature}`

**服务端响应**：

```json
{
    "signature": "93bf23c414cf65195e8aa67f1fde1072e86038f5ad76cd88ce77fa70976bd569"
}
```

### 11.3 Python 复现

```python
import requests
import json
import time

session = requests.Session()
base_url = "http://10.0.0.132:82"

timestamp = int(time.time())

# 第一步：向服务端请求签名
payload1 = {
    "username": "admin",
    "password": "123456",
    "timestamp": timestamp
}
resp1 = session.post(f"{base_url}/encrypt/get-signature.php", json=payload1)
signature = resp1.json()["signature"]

# 第二步：携带签名发送登录请求
payload2 = {
    "username": "admin",
    "password": "123456",
    "timestamp": timestamp,
    "signature": signature
}
resp2 = session.post(f"{base_url}/encrypt/signdataserver.php", json=payload2)
print(resp2.text)
```

### 11.4 关键坑点

1. **两步的 timestamp 必须一致**：服务端会用同样的 timestamp 重新算一遍签名。
2. **必须用 `requests.Session()`**：两步请求的 PHPSESSID 相同，否则服务端找不到对应的签名。
3. **签名是服务端下发的**：前端无法自己计算，因为盐不在前端。

### 11.5 核心认知

- 这个关卡模拟真实业务中"签名服务"的架构：签名逻辑集中在服务端，前端只负责"转发"。
- **优势**：签名规则更新时不需要改前端；盐不暴露在客户端。
- **局限**：仍然需要请求服务端才能拿到签名，攻击者可以通过脚本模拟两步请求。
- **真实业务里**，签名服务会附加更多前置条件：登录验证、短信验证码、限频、审计。

## 十二、 总结与能力清单

经过以上所有关卡的实战，完整掌握了前端加密、签名与防重放对抗的核心能力：

| 能力 | 状态 |
| :--- | :--- |
| 前端 JS 逆向（含混淆代码） | ✅ |
| 识别 AES / DES / RSA / HMAC 算法特征 | ✅ |
| 提取写死、动态生成、服务端下发的 Key/IV | ✅ |
| Python 复现 AES / DES 对称加密 | ✅ |
| Python 复现 RSA 非对称加密 | ✅ |
| Python 复现 AES+RSA 混合加密 | ✅ |
| Python 复现 HMAC-SHA256 签名 | ✅ |
| Burp Intruder 自动化爆破 | ✅ |
| 处理两步请求与 Session 保持 | ✅ |
| URL 编码 / Base64 / Hex 三种格式处理 | ✅ |
| 服务端源码审计对齐 | ✅ |
| 本地自检验证思路 | ✅ |
| 时间窗口与防重放机制处理 | ✅ |

**下一步进阶方向**：

1. **autoDecoder 插件**：把 Python 脚本包装成 HTTP 服务，让 Burp 自动加解密。Burp 里看到的是明文，发出去的自动加密，实现"透明代理"效果。
2. **mitmproxy 脚本**：跨工具复用加解密逻辑，支持 Burp + SQLMap + 自定义脚本协同。
3. **实战拓展**：找真实网站进行完整的"抓包 → 定位 → 逆向 → 复现 → 自动化"闭环。

## 十三、 附录：环境准备

在 CentOS 7 或 Kali Linux 下，需安装依赖库：

```bash
pip3 install pycryptodome requests
```

> 注：安装包名是 `pycryptodome`，但导入时用 `from Crypto.xxx import xxx`，是历史兼容原因。

**常用命令备忘**：

```bash
# Base64 编解码
echo -n "hello" | base64
echo "aGVsbG8=" | base64 -d

# Hex 转换
echo -n "hello" | xxd -p
echo "68656c6c6f" | xxd -r -p

# URL 编码
python3 -c "import urllib.parse; print(urllib.parse.quote('a+b/c='))"
```

## 十四、autoDecoder 透明代理配置

### 14.1 目标
让 Burp 里永远显示明文，发出去的自动加密，收到的自动解密。
之后 Intruder 爆破、SQLMap 注入、越权测试，都可以像未加密网站一样操作。

### 14.2 架构
```
┌─────────┐  明文   ┌──────────────┐  密文  ┌──────────┐
│  Burp   │ ─────→ │ autoDecoder  │ ────→ │  服务器  │
│  Repeater│        │  + Flask 服务│        │          │
│         │ ←───── │              │ ←──── │          │
└─────────┘  明文   └──────────────┘  密文  └──────────┘
```

### 14.3 三步配置

**第一步：写 Flask 加解密服务**

```python
from flask import Flask, request
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad
import base64
import urllib.parse

app = Flask(__name__)

KEY = b'1234567890123456'
IV  = b'1234567890123456'
PREFIX = 'encryptedData='

def aes_encrypt(plaintext):
    cipher = AES.new(KEY, AES.MODE_CBC, IV)
    padded = pad(plaintext.encode('utf-8'), 16)
    b64 = base64.b64encode(cipher.encrypt(padded)).decode('utf-8')
    return urllib.parse.quote(b64, safe='')  # 关键：URL 编码

def aes_decrypt(ciphertext_urlencoded):
    ciphertext = urllib.parse.unquote(ciphertext_urlencoded)
    cipher = AES.new(KEY, AES.MODE_CBC, IV)
    decrypted = unpad(cipher.decrypt(base64.b64decode(ciphertext)), 16)
    return decrypted.decode('utf-8')

@app.route('/encode', methods=['POST'])
def encode():
    body = request.form.get('dataBody', '').strip('\n')
    if body.startswith(PREFIX):
        value = body[len(PREFIX):]
        result = PREFIX + aes_encrypt(value)
    else:
        result = aes_encrypt(body)
    return result

@app.route('/decode', methods=['POST'])
def decode():
    body = request.form.get('dataBody', '').strip('\n')
    if body.startswith(PREFIX):
        value = body[len(PREFIX):]
        try:
            result = PREFIX + aes_decrypt(value)
        except Exception:
            result = body
    else:
        result = body
    return result

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8888, threaded=True)
```

**关键点**：
- 参数名必须是 **`dataBody`**（不是 `data`），插件按这个发数据。
- 加密后返回时要做 **URL 编码**（`urllib.parse.quote`），否则 Base64 里的 `+` 会被服务端当成空格。

**第二步：下载并加载 autoDecoder 插件**

- 下载地址：`https://github.com/f0ng/autoDecoder/releases`
- 选 `jdk14` 版本（对应 Burp 2024+）
- Burp → Extensions → Add → 选 jar → 加载成功

**第三步：配置 autoDecoder**

| 标签页 | 配置 |
| :--- | :--- |
| **Options** | 勾选 `接口加解密`，域名填靶场地址（如 `10.0.0.132`） |
| **接口加解密** | Encode URL: `http://127.0.0.1:8888/encode`，Decode URL: `http://127.0.0.1:8888/decode` |
| 保存配置 | 会弹出文件对话框，选默认路径即可 |

### 14.4 使用方式
- **Repeater 的 `autoDecoder` 子标签**：显示明文（可编辑），在这里改请求。
- **原始 `Pretty` / `Raw` 标签**：显示密文（只读），用来查看真实发送内容。
- **发送后**：插件自动加密 → 服务端返回密文 → 插件自动解密 → Burp 显示明文。

### 14.5 踩坑记录
1. **Flask 参数名**：必须是 `dataBody`，不是 `data`。
2. **Base64 里的 `+`**：必须 URL 编码，否则服务端解析时被当空格，导致 `Invalid input`。
3. **原始编辑区被锁**：这是插件设计，改明文要去 `autoDecoder` 子标签。
4. **域名匹配**：只填域名，不带端口（如 `10.0.0.132`）。

### 14.6 实战验证：Intruder 明文字典爆破

**测试目标**：AES 固定 Key 关卡的登录密码。

**配置**：
- Positions：只标记密码值部分 `§123456§`
- Payloads：8 个明文密码（123456、admin、admin888、password 等）
- Resource pool：默认 10 并发
- autoDecoder：接口加解密已启用，域名 `10.0.0.132`

**攻击结果**：

| Payload | Length | 判定 |
| :--- | :--- | :--- |
| **123456** | **378** | ✅ 命中（Response: `{"success":true}`） |
| admin | 379 | ❌ |
| admin888 | 379 | ❌ |
| password | 379 | ❌ |
| ... | ... | ... |

**关键收获**：
- 整个爆破过程中，Payload 是**明文**，加密由 autoDecoder 自动完成。
- 不需要写 Python 脚本批量生成密文，不需要手动复制粘贴。
- 检测出正确密码后，Intruder 的 Response 面板直接显示解密后的明文响应。
- **透明代理让 Burp 对加密网站的测试体验和普通网站完全一致**。

**应用场景扩展**：
- **SQL 注入**：直接在明文参数里测 `' OR 1=1--`，插件自动加密。
- **越权测试**：修改 `userId=1` 到 `userId=2`，插件自动加密发送。
- **重放测试**：时间戳过期？插件自动生成新时间戳。
- **自动化扫描**：Burp Scanner 能像扫描普通网站一样扫描加密目标。

## 十五、mitmproxy：可编程代理工具

### 15.1 mitmproxy 是什么

mitmproxy 是一个**独立的、可编程的中间人代理工具**，用 Python 编写。它和 Burp 一样工作在浏览器和服务器之间，抓取 HTTP/HTTPS 流量。

**它包含三个命令**：

| 命令 | 形态 | 用途 |
| :--- | :--- | :--- |
| `mitmproxy` | 命令行交互界面 | 手动拦截、查看、修改流量 |
| `mitmweb` | 浏览器图形界面 | 类似 Burp Web UI，适合初学者 |
| `mitmdump` | 无界面命令行 | 脚本自动化、批量处理 |

### 15.2 和 Burp 的核心区别

| 维度 | Burp | mitmproxy |
| :--- | :--- | :--- |
| **形态** | 桌面应用 | 命令行 / Web UI |
| **加解密** | 需要装插件（autoDecoder） | **原生支持 Python 脚本** |
| **界面** | 完善的图形界面 | mitmweb 界面较基础 |
| **Repeater** | 有 | 有（点击请求 → Replay） |
| **Intruder** | 有 | 无，需用脚本实现 |
| **Scanner** | 有 | 无 |
| **脚本机制** | 插件形式，受 API 限制 | 直接写 Python，无限制 |
| **定位** | 综合渗透测试平台 | 可编程代理工具 |

**两者看到的流量都是"浏览器发出的真实密文"**——因为都在代理层拦截。区别在于：

- **Burp + autoDecoder**：在 Burp Repeater 里显示"你手写的明文"，插件自动加密后发送
- **mitmproxy**：脚本拦截真实流量，需要在脚本里判断是明文还是密文

### 15.3 和 autoDecoder 的对比

| 维度 | autoDecoder | mitmproxy |
| :--- | :--- | :--- |
| **工作层次** | Burp 内部（Repeater/Intruder） | 真实代理层 |
| **加解密逻辑** | 外部 Flask 服务 | 脚本直接内置 |
| **交互方式** | 你在 Burp 里发明文 | 浏览器/工具直接发包 |
| **适用范围** | 只在 Burp 内 | 所有走代理的工具 |
| **部署** | Burp + Flask 两个组件 | 一个 mitmproxy 进程 |

**核心差异**：
- **autoDecoder**：让你在 Burp 里透明操作（改明文）
- **mitmproxy**：让所有经过代理的工具（curl、SQLMap、ffuf 等）自动获得加解密能力

### 15.4 mitmproxy 的真正价值

**给不支持加密的第三方工具加透明加密。**

典型场景：

| 工具 | 场景 | 效果 |
| :--- | :--- | :--- |
| **SQLMap** | 测试加密网站的 SQL 注入 | SQLMap 发明文，mitmproxy 自动加密 |
| **ffuf** | 目录扫描（参数需要加密） | ffuf 用明文，mitmproxy 自动加密 |
| **curl** | 手动调试 | curl 发明文，mitmproxy 自动加密 |
| **Burp** | 和 autoDecoder 互补 | Burp → mitmproxy → 目标 |

**典型命令**：
```bash
curl -x http://127.0.0.1:8889 \
  -X POST http://10.0.0.132:82/encrypt/aes.php \
  -d 'encryptedData={"username":"admin","password":"123456"}'
```
curl 发的是**明文**，但经过 mitmproxy 后，靶场收到的是**密文**。

### 15.5 快速上手

**1. 启动 mitmweb**：
```bash
mitmweb --listen-port 8889
```

- **8889**：代理端口（浏览器要连这个）
- **8081**：Web UI 端口（浏览器访问这个看流量）

**2. 配置浏览器代理**：`127.0.0.1:8889`

**3. 访问 `http://127.0.0.1:8081`** 查看流量

**4. 加载 Python 脚本**：
```bash
mitmweb --listen-port 8889 -s ~/aes_mitm.py
```

### 15.6 mitmproxy 脚本示例

**核心机制**：脚本定义几个回调函数，mitmproxy 在流量经过时自动调用。

```python
from mitmproxy import http
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
import base64
import urllib.parse

KEY = b'1234567890123456'
IV  = b'1234567890123456'
PREFIX = 'encryptedData='

def aes_encrypt(plaintext):
    cipher = AES.new(KEY, AES.MODE_CBC, IV)
    padded = pad(plaintext.encode('utf-8'), 16)
    b64 = base64.b64encode(cipher.encrypt(padded)).decode('utf-8')
    return urllib.parse.quote(b64, safe='')

def is_plaintext(value):
    """判断是否为明文（JSON 以 { 开头，包含 username）"""
    return value.startswith('{') and 'username' in value

def request(flow: http.HTTPFlow):
    """每个请求经过时触发"""
    if '/encrypt/aes.php' not in flow.request.path:
        return

    body = flow.request.get_text()
    if body.startswith(PREFIX):
        value = body[len(PREFIX):]
        if is_plaintext(value):
            # 是明文 → 加密
            encrypted = aes_encrypt(value)
            flow.request.set_text(f'{PREFIX}{encrypted}')
            print(f"[请求] 明文 → 已加密")
        else:
            print(f"[请求] 已是密文，跳过")

def response(flow: http.HTTPFlow):
    """每个响应回来时触发"""
    if '/encrypt/aes.php' in flow.request.path:
        body = flow.response.get_text()
        print(f"[响应] {body[:80]}")
```

**加载方式**：
```bash
mitmweb --listen-port 8889 -s ~/aes_mitm.py
```

### 15.7 踩坑记录：mitmproxy 不能替代 autoDecoder

**问题**：在浏览器里点登录时，mitmproxy 脚本收到的是**浏览器已经加密的密文**，不是明文。

**原因**：
- Burp + autoDecoder 在 **Repeater 层**工作，你手写的内容就是明文，插件负责加密
- mitmproxy 在**代理层**工作，浏览器发出的流量已经是密文

**结论**：
- 想"在 Burp 里发明文" → 用 autoDecoder
- 想"让第三方工具自动加密" → 用 mitmproxy
- 两者不冲突，可以互补

### 15.8 实战场景总结

| 场景 | 推荐工具 |
| :--- | :--- |
| 在 Burp 里测试加密网站（Intruder 爆破、SQL 注入） | autoDecoder |
| 让 SQLMap 测加密网站 | mitmproxy |
| 让 ffuf 扫加密 API | mitmproxy |
| 让 curl 调加密接口 | mitmproxy |
| 需要复杂流量处理逻辑 | mitmproxy |
| 习惯用 Burp 图形界面 | autoDecoder |
| 无图形界面环境（SSH 远程） | mitmproxy |

## 十六、Chrome DevTools 断点与 Hook 技术

### 16.1 三种核心断点

#### XHR/fetch 断点（最常用）
**用途**：请求发出时自动断下，从调用栈回溯加密函数。

**操作**：
1. F12 → Sources → 右侧 XHR/fetch Breakpoints → 点 `+`
2. 输入 URL 关键词（如 `aes.php`）
3. 触发请求，页面会停在发送前

**典型用途**：加密函数名被混淆，无法用全局搜索定位时。

#### DOM 事件断点
**用途**：加密发生在点击/提交事件时。

**操作**：
1. Sources → Event Listener Breakpoints
2. 展开 Mouse → 勾 `click`，或 Control → 勾 `submit`
3. 点击按钮，断点触发

#### 条件断点
**用途**：函数被调用多次，只在特定条件下断。

**操作**：
1. 在源码中找到关键行
2. 右键行号 → Add conditional breakpoint
3. 输入条件（如 `_0x54dcc5 === 'admin'`）

### 16.2 断点触发后的三个动作

**1. 看 Call Stack（调用栈）**
从下往上看调用链，找到加密函数所在层：
```
sendDataAes          ← 目标函数
onclick              ← 事件处理
dispatchEvent        ← 浏览器机制
```

**2. 看 Scope（作用域变量）**
右侧 Scope 面板显示当前函数所有变量，能看到：
- `_0x807d91` = 明文 JSON
- `_0x67b862` = AES Key（WordArray）
- `_0x2d9cd5` = AES IV（WordArray）
- `_0x1375d7` = 密文

**3. 用 Console 执行表达式**
在断点暂停时，Console 里可以直接求值：
```javascript
CryptoJS.enc.Utf8.stringify(_0x67b862)  // → "1234567890123456"
CryptoJS.enc.Utf8.stringify(_0x2d9cd5)  // → "1234567890123456"
_0x807d91  // → {"username":"admin","password":"123456"}
```

### 16.3 Hook 技术

**Hook = 给函数装监听器**，函数被调用时自动打印入参出参。

#### 方式一：Override（最基础）
```javascript
var _originalEncrypt = CryptoJS.AES.encrypt;
CryptoJS.AES.encrypt = function(data, key, options) {
    console.log("=== AES 加密被调用 ===");
    console.log("明文:", data.toString());
    console.log("Key:", key.toString(CryptoJS.enc.Utf8));
    if (options && options.iv) {
        console.log("IV:", options.iv.toString(CryptoJS.enc.Utf8));
    }
    var result = _originalEncrypt.apply(this, arguments);
    console.log("密文:", result.toString());
    return result;
};
```

**执行效果**（在 Console 里粘贴后点登录）：
```
=== AES 加密被调用 ===
明文: {"username":"admin","password":"123456"}
Key: 1234567890123456
IV: 1234567890123456
密文: nArXfVdnoe67UzojAPP2X+6qSiznLMBAI3a5Bi+zlNzXaUb9+gTXusl67b+DS9Zw
```

#### 方式二：Object.defineProperty（劫持属性）
```javascript
var _cookie = document.cookie;
Object.defineProperty(document, 'cookie', {
    get: function() {
        console.log("读取 Cookie:", _cookie);
        return _cookie;
    },
    set: function(val) {
        console.log("设置 Cookie:", val);
        _cookie = val;
    }
});
```

**用途**：追踪 `document.cookie` 的读写，定位 cookie 生成逻辑。

#### 方式三：Proxy（拦截整个对象）
```javascript
var handler = {
    get: function(obj, prop) {
        console.log("读取属性:", prop);
        return obj[prop];
    },
    apply: function(target, thisArg, args) {
        console.log("函数调用入参:", args);
        var result = target.apply(thisArg, args);
        console.log("函数返回:", result);
        return result;
    }
};
window.targetFunction = new Proxy(window.targetFunction, handler);
```

**用途**：拦截对象属性读取和函数调用，不易被反调试检测。

### 16.4 断点 vs Hook 对比

| 维度 | 断点 | Hook |
| :--- | :--- | :--- |
| **操作** | F12 → Sources → 添加断点 | F12 → Console → 粘贴脚本 |
| **触发** | 请求发出时自动断下 | 函数被调用时自动打印 |
| **能拿到** | 调用链 + 所有中间变量 | 函数的入参出参 |
| **是否暂停页面** | ✅ 要按 F8 释放 | ❌ 无感运行 |
| **学习曲线** | 中 | 低 |
| **适合场景** | 深入分析调用链、找混淆变量 | 快速定位密钥、批量监控 |

**实战组合**：
1. 先用 Hook 快速拿到密钥 → 1 分钟
2. 再用断点深入分析调用链 → 5 分钟
3. 最后用 Python 复现 + autoDecoder 自动化

### 16.5 关键认知
- **断点不是"暂停"，是"透明观察"**——你可以看到函数执行时的所有内部状态。
- **Hook 不是"修改代码"，是"包装函数"**——原函数逻辑不变，只是多了日志。
- **WordArray 要转成字符串**——CryptoJS 的 Key/IV 是 WordArray 对象，必须用 `CryptoJS.enc.Utf8.stringify()` 才看得懂。
- **断点用 F8 释放**：忘了释放页面会一直卡住。
