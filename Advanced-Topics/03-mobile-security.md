# 移动端安全基础（Android / iOS）

移动端安全是当前红队和安服业务的重要方向。本文记录 Android 和 iOS 渗透测试的基础环境、常用工具、常见漏洞与绕过思路。

## 一、Android 安全基础

### 1. 环境准备
- **测试机**：一台 Root 过的 Android 手机，或者使用 Android Studio 自带的模拟器（AVD）、Genymotion。
- **攻击机**：Kali Linux（用于抓包和运行 Frida）。
- **常用工具**：
  - `adb`：Android Debug Bridge，用于设备连接、安装包、文件传输。
  - `Apktool`：反编译和重打包 APK。
  - `Jadx`：将 APK 反编译为 Java 代码，便于源码审计。
  - `Frida`：动态插桩工具（Hook 神器）。
  - `Burp Suite`：抓取 HTTP/HTTPS 流量。

### 2. 抓包与 SSL Pinning 绕过

Android 7.0 以后，系统不再信任用户安装的 CA 证书，导致 Burp 抓包失败。同时，很多 APP 开启了 SSL Pinning（证书绑定），即使安装了系统级证书也无法抓包。

**绕过方式：**
1. **将 Burp 证书安装为系统证书**（需要 Root）：
   ```bash
   # 将证书转换为 PEM 格式，计算 Hash，重命名并推入系统证书目录
   openssl x509 -inform DER -in cacert.der -out cacert.pem
   openssl x509 -inform PEM -subject_hash_old -in cacert.pem | head -1
   # 重命名文件为 <hash>.0，推入 /system/etc/security/cacerts/
   ```
2. **使用 Frida 绕过 SSL Pinning**：
   ```bash
   # 启动 frida-server
   adb shell su -c "/data/local/tmp/frida-server &"
   # 在 Kali 上运行绕过脚本
   frida -U -f com.example.app -l bypass_ssl_pinning.js --no-pause
   ```
3. **使用 LSPosed + TrustMeAlready**（Xposed 模块）：一键绕过 SSL Pinning。

### 3. 反编译与源码审计

```bash
# 使用 Jadx 反编译 APK
jadx -d output_dir app.apk

# 使用 Apktool 反编译资源文件
apktool d app.apk -o output_dir
```
重点关注：
- 硬编码的密码、API Key、数据库地址。
- 组件暴露（`AndroidManifest.xml` 中的 `android:exported="true"`）。
- WebView 漏洞（`setJavaScriptEnabled(true)` 配合 `addJavascriptInterface`）。
- 不安全的存储（SharedPreferences、SQLite 中的敏感信息）。

### 4. 常见漏洞
- **组件暴露**：Activity、Service、Broadcast Receiver、Content Provider 未做权限控制，可被其他 APP 调用。
- **WebView 漏洞**：JS 接口泄露、文件读取、跨站脚本。
- **数据库与文件泄露**：`/data/data/<包名>/` 目录下的敏感文件。
- **弱加密**：使用 MD5、Base64 存储敏感数据。

## 二、iOS 安全基础

### 1. 环境准备
- **测试机**：越狱的 iPhone 或 iPad（推荐 Checkra1n 或 Unc0ver）。
- **攻击机**：macOS 或 Linux（需要安装 `libimobiledevice`）。
- **常用工具**：
  - `Cydia`：越狱后的应用商店，安装 Frida、SSH 等工具。
  - `frida-ios-dump`：对 APP 进行脱壳，提取 IPA。
  - `Clutch` / `bagbak`：脱壳工具。
  - `IDA Pro` / `Hopper`：逆向分析二进制文件。

### 2. 抓包与绕过
iOS 的抓包和 Android 类似，通过设置代理和安装证书。但 iOS 也有证书绑定（SSL Pinning）。
- **绕过方式**：
  - 使用 `SSL Kill Switch 2`（Cydia 插件）。
  - 使用 Frida 脚本 Hook 证书验证函数。
  - 使用 `objection` 工具：`objection -g com.example.app explore`，然后执行 `ios sslpinning disable`。

### 3. 脱壳与逆向
App Store 下载的 APP 被加密（加壳），无法直接反编译。
```bash
# 使用 frida-ios-dump 脱壳
python3 dump.py -u root -p 22 com.example.app -o /tmp/app.ipa
```
脱壳后，使用 `class-dump` 导出头文件，用 IDA/Hopper 分析汇编代码，查找敏感逻辑。

## 三、面试高频问题

**Q1：Android 如何绕过 SSL Pinning？**
A：1. 将 Burp 证书安装为系统证书（需 Root）。2. 使用 Frida 编写 Hook 脚本，拦截并绕过证书校验函数。3. 使用 LSPosed 模块（如 TrustMeAlready）。4. 反编译 APK，修改源码（NOP 掉证书校验），重新打包签名。

**Q2：移动端常见漏洞有哪些？**
A：1. 组件暴露（Android 的四大组件）。2. WebView 漏洞（JS 接口、文件读取）。3. 不安全的数据存储（明文存储密码、Token）。4. 弱加密（MD5、Base64）。5. SSL Pinning 绕过导致的流量泄露。6. 服务端 API 未做严格的鉴权。

**Q3：如何对一个加壳的 APP 进行脱壳？**
A：Android 可以使用 FART、DumpDex 等脱壳机，或者使用 Frida 动态脱壳。iOS 可以使用 `frida-ios-dump` 或 `Clutch`。脱壳后，使用 Jadx（Android）或 class-dump + IDA（iOS）进行静态分析。
