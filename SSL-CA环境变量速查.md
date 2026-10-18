# SSL / CA 信任相关环境变量速查

> 适用场景：HTTPS 抓包（mitmproxy / Charles）、企业内网 CA、自签名证书、代理中间人、容器/CI 中的证书注入。
>
> 核心结论：**不同 TLS 栈读不同的变量，互不相通。** 给 Node 设的变量，Python/curl/Java/Chromium 不会读。排查时先确认“这个进程用的是哪套 TLS 栈”。

---

## 目录

- [1. 核心概念](#1-核心概念)
- [2. 按生态分类](#2-按生态分类)
- [3. 优先级速查](#3-优先级速查)
- [4. 合并 CA Bundle](#4-合并-ca-bundle)
- [5. MITM 抓包对照表](#5-mitm-抓包对照表)
- [6. Windows 上如何设置](#6-windows-上如何设置)
- [7. 常见坑](#7-常见坑)
- [8. 验证方法](#8-验证方法)
- [9. 一页速记](#9-一页速记)

---

## 1. 核心概念

### 1.1 四种“信任库”

| 类型 | 例子 | 谁在用 |
|---|---|---|
| 程序内置根证书 | Node 的 Mozilla 根、Python `certifi`、Rust `webpki-roots`、Dart 内置根 | Node、requests、rustls、部分静态编译程序 |
| 操作系统证书库 | Windows Cert Store、macOS Keychain、Linux `/etc/ssl/certs` | Chromium/Electron 渲染进程、.NET、Schannel、Go(Windows)、系统 curl |
| 程序自带 truststore | Java `cacerts`、应用自己的证书文件 | JVM、部分企业软件 |
| 进程级环境变量 | `NODE_EXTRA_CA_CERTS`、`REQUESTS_CA_BUNDLE` 等 | 具体库在运行时读取 |

同一个 HTTPS 请求，可能经过不止一层 TLS：

- Electron = Chromium 网络栈（渲染进程）+ Node/OpenSSL（主进程与 Utility 进程）
- 代理场景 = 客户端 → 代理（一次 TLS）→ 目标站（另一次 TLS）
- Java 应用内嵌 HttpClient = JVM truststore
- Python 应用调用 `curl` 子进程 = 变成 curl 的信任库

**只在一个地方装 CA，通常不够。**

### 1.2 追加 vs 替换

| 行为 | 变量 | 说明 |
|---|---|---|
| **追加** | `NODE_EXTRA_CA_CERTS` | 在内置根证书基础上追加，公共 CA 仍受信任 |
| **追加系统库** | `NODE_USE_SYSTEM_CA=1` / `--use-system-ca` | 内置根 + 操作系统证书库 |
| **替换** | `REQUESTS_CA_BUNDLE`、`CURL_CA_BUNDLE`、`SSL_CERT_FILE`、`GIT_SSL_CAINFO`、`PIP_CERT`、`AWS_CA_BUNDLE`、`GRPC_DEFAULT_SSL_ROOTS_FILE_PATH` | 指定的文件通常成为**唯一**信任集；只放私有 CA 会导致公网站点验证失败 |
| **替换** | Java `javax.net.ssl.trustStore` | 替换默认 `cacerts`，新库必须包含所需公共根证书 |
| **关闭校验** | `NODE_TLS_REJECT_UNAUTHORIZED=0`、`PYTHONHTTPSVERIFY=0`、`GIT_SSL_NO_VERIFY=true` | 仅调试，不要长期使用 |

### 1.3 读取时机

| 变量 | 读取时机 | 运行中修改是否生效 |
|---|---|---|
| `NODE_EXTRA_CA_CERTS` | Node 进程启动 | 否，必须重启进程 |
| `NODE_USE_SYSTEM_CA` | 进程启动 | 否 |
| `SSL_CERT_FILE` / `SSL_CERT_DIR` | 创建 SSL context / 初始化 TLS 库时 | 通常否，取决于程序 |
| `REQUESTS_CA_BUNDLE` / `CURL_CA_BUNDLE` | requests 每次请求读取 | 是（同一进程内可生效） |
| `CURL_CA_BUNDLE` | 每次创建 curl easy handle/连接 | 是（新连接） |
| `GIT_SSL_CAINFO` | 每次 Git HTTP 请求 | 是（新连接） |
| Java `trustStore` | JVM 启动 | 否 |
| 系统证书库 | 连接建立时查询 | 通常可立即生效（缓存例外） |

### 1.4 文件、目录与格式

| 形式 | 含义 | 典型变量 |
|---|---|---|
| **单个 PEM 文件** | 一个或多个证书用 `-----BEGIN CERTIFICATE-----` 拼接 | 大多数 `*_CA_BUNDLE`、`*_CAINFO`、`NODE_EXTRA_CA_CERTS` |
| **目录** | OpenSSL 的 hashed 证书目录（文件名是证书哈希） | `SSL_CERT_DIR`、`GIT_SSL_CAPATH` |
| **JKS / PKCS12** | Java 二进制 truststore | `javax.net.ssl.trustStore` |
| **DER / `.cer`** | 二进制单证书，**不能**直接当 PEM 用 | 需要 `openssl x509 -inform DER -outform PEM` 转换 |

> `NODE_EXTRA_CA_CERTS` 明确要求 PEM；`REQUESTS_CA_BUNDLE` / `CURL_CA_BUNDLE` 也是文件路径，不是目录。

生成 hashed 目录：

```bash
openssl rehash /path/to/certs
# 或旧命令
c_rehash /path/to/certs
```

### 1.5 命名规律

| 后缀 | 含义 |
|---|---|
| `*_CA_BUNDLE` | CA bundle 文件（PEM） |
| `*_CAINFO` | CA 信息文件（PEM），libcurl 术语 |
| `*_CAPATH` | CA 目录（hashed） |
| `*_CERT_FILE` | 证书文件 |
| `*_CERT_DIR` | 证书目录 |
| `trustStore` | Java 二进制信任库 |
| `keyStore` | 客户端证书/私钥库（不是 CA） |

---

## 2. 按生态分类

### 2.1 Node.js / Electron

#### `NODE_EXTRA_CA_CERTS`

- **作用**：在 Node 内置根证书上**追加** PEM 证书。
- **读取**：进程启动时一次；必须是可读文件。
- **路径**：官方要求绝对路径（Electron 应用也可能自行校验）。
- **格式**：PEM，可拼接多张证书。
- **限制**：
  - 如果代码显式传了 `ca:` / `ca` 选项，`NODE_EXTRA_CA_CERTS` 会被忽略。
  - **不影响 Electron 渲染进程**（Chromium 网络栈在 Windows 上读系统证书库）。
  - 不影响使用 native addon / BoringSSL / libcronet 等自带 TLS 栈的模块。
- **适用**：Node、npm/pnpm/yarn、Electron 主进程、Electron Utility/Node 子进程、`undici`、`node:https`。

```bat
set "NODE_EXTRA_CA_CERTS=C:\certs\mitmproxy-ca-cert.pem"
```

```bash
export NODE_EXTRA_CA_CERTS="$HOME/.mitmproxy/mitmproxy-ca-cert.pem"
```

#### `NODE_USE_SYSTEM_CA=1` / `--use-system-ca`

- **作用**：在内置根证书基础上**追加操作系统证书库**。
- **平台行为**：
  - Windows：读取机器/用户证书库，包括 Root、Trusted People、Enterprise Trust。
  - macOS：Keychain。
  - Linux：OpenSSL 默认路径，受 `SSL_CERT_FILE` / `SSL_CERT_DIR` 影响。
- **版本**：较新的 Node（22.15 起，具体版本以官方文档为准）。
- **组合**：可与 `NODE_EXTRA_CA_CERTS` 同时使用，信任集 = 内置 + 系统 + 额外 PEM。

```bat
set "NODE_USE_SYSTEM_CA=1"
set "NODE_EXTRA_CA_CERTS=C:\certs\corp-ca.pem"
```

```bash
NODE_USE_SYSTEM_CA=1 node app.js
# 等价
node --use-system-ca app.js
```

#### `--use-openssl-ca` 与 `SSL_CERT_FILE` / `SSL_CERT_DIR`

- `node --use-openssl-ca`：改用 OpenSSL 默认证书路径。
- 只有在这种模式下，Node 才会读取 `SSL_CERT_FILE` / `SSL_CERT_DIR`。
- `NODE_OPTIONS=--use-openssl-ca` 也可以，但很多应用会清理 `NODE_OPTIONS`。

```bash
NODE_OPTIONS=--use-openssl-ca \
SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt \
node app.js
```

#### `NODE_TLS_REJECT_UNAUTHORIZED=0`

- **作用**：关闭 Node TLS 证书校验。
- **风险**：进程级、全局，任何 HTTPS 都不再校验；会打印警告。
- **仅用于临时调试**。

#### Electron 的两套网络栈

| 位置 | TLS 栈 | 信任库/变量 |
|---|---|---|
| 主进程（`main.js`） | Node/OpenSSL | `NODE_EXTRA_CA_CERTS`、`NODE_USE_SYSTEM_CA` |
| Utility/Node 子进程 | Node/OpenSSL | 同上（取决于启动环境是否传入） |
| 渲染进程（BrowserWindow 页面） | Chromium | Windows 系统证书库；不读 Node 变量 |
| `net` 模块 / `session` | Chromium 网络服务 | 系统证书库；可用 `session.setCertificateVerifyProc` 自定义 |

#### npm / pnpm / yarn

- 都基于 Node，`NODE_EXTRA_CA_CERTS` 通常有效。
- npm 还有配置项：

```bash
npm config set cafile /path/to/ca.pem
npm config get cafile
```

---

### 2.2 Python

#### `REQUESTS_CA_BUNDLE`

- **作用**：`requests` 库的 CA bundle 文件路径。
- **语义**：**替换**默认的 `certifi` bundle。
- **优先级**：`verify=` 参数 > `REQUESTS_CA_BUNDLE` > `CURL_CA_BUNDLE` > `certifi`。
- **读取**：每次请求时读取环境变量。
- **适用**：`requests`，以及大量基于 requests 的 SDK（pip、部分云 SDK、Azure CLI 等）。

```bash
export REQUESTS_CA_BUNDLE=/path/to/combined-ca.pem
```

#### `CURL_CA_BUNDLE`

- **作用**：curl/libcurl 的 CA bundle。
- **在 requests 中**：作为 `REQUESTS_CA_BUNDLE` 未设置时的兜底。
- **在 Git 中**：如果 Git 没有显式设置 `http.sslCAInfo`，libcurl 会读取它。

#### `SSL_CERT_FILE` / `SSL_CERT_DIR`

- **作用**：OpenSSL 的默认 CA 文件/目录。
- **谁读**：
  - Python `ssl.create_default_context()`（Linux/macOS，走 OpenSSL 默认路径）
  - `urllib`、`aiohttp`、许多基于 `ssl` 模块的库
  - Ruby、PHP、OpenSSL CLI、Linux 上的 Go
  - Node（仅 `--use-openssl-ca`）
- **Windows 注意**：Python 的 `ssl.create_default_context()` 在 Windows 上通常枚举系统证书库，`SSL_CERT_FILE` 不一定生效。
- **语义**：**替换**默认路径（不是追加）。

#### `PIP_CERT`

- **作用**：pip 使用的 CA bundle。
- **等价**：`pip --cert /path/to/ca.pem`。
- **优先级**：`--cert` > `PIP_CERT` > requests 环境变量 > certifi。

#### `AWS_CA_BUNDLE`

- **作用**：AWS CLI、boto3/botocore 使用的 CA bundle。
- **优先级**：`--ca-bundle` > `AWS_CA_BUNDLE` > SDK 默认。

#### `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH`

- **作用**：gRPC C-core 的根证书文件。
- **适用**：Python/C++/Ruby/PHP 的 gRPC，以及 Node 的 native `grpc` 包。
- **不适用**：Node 的 `@grpc/grpc-js`（它走 Node TLS，用 `NODE_EXTRA_CA_CERTS`）。

#### `PYTHONHTTPSVERIFY=0`

- **作用**：关闭 `urllib` / `http.client` 的默认 HTTPS 校验。
- **不影响** requests/httpx 的显式校验逻辑。
- **仅调试**。

---

### 2.3 curl / libcurl

| 变量/参数 | 作用 |
|---|---|
| `CURL_CA_BUNDLE` | CA bundle 文件 |
| `CURL_SSL_BACKEND` | 选择 TLS 后端：`openssl`、`schannel`、`secure-transport` 等（不是 CA 设置） |
| `SSL_CERT_FILE` / `SSL_CERT_DIR` | OpenSSL 后端的默认 CA 路径 |
| `--cacert` | 单次命令指定 CA 文件，优先级最高 |
| `--capath` | 指定 hashed CA 目录 |

**Windows 特别说明**：

- Windows 10/11 自带的 `curl.exe` 默认使用 **Schannel**，直接读 Windows 证书库。
- Git for Windows 的 curl 可能是 OpenSSL 或 Schannel 构建，可用 `curl -V` 查看。
- Schannel 构建下 `CURL_CA_BUNDLE` / `--cacert` 的行为与 OpenSSL 构建不完全一致；最稳的是把 CA 装进 Windows 证书库。

```bash
curl -V
curl -v --cacert /path/to/ca.pem https://example.com/
```

---

### 2.4 Git

| 变量 | 作用 |
|---|---|
| `GIT_SSL_CAINFO` | CA 文件路径 |
| `GIT_SSL_CAPATH` | hashed CA 目录 |
| `GIT_SSL_NO_VERIFY=true` | 关闭校验（仅调试） |

- Git 使用 libcurl；`GIT_SSL_CAINFO` 会设置 `CURLOPT_CAINFO`，优先级高于 `CURL_CA_BUNDLE`。
- 对应配置项：

```bash
git config --global http.sslCAInfo /path/to/ca.pem
git config --global http.sslCAPath /path/to/certs
git config --global --get http.sslCAInfo
```

---

### 2.5 Java / JVM

Java **没有**标准的“CA 环境变量”。它使用 JVM 系统属性，环境变量只是用来注入参数。

| 系统属性 | 作用 |
|---|---|
| `javax.net.ssl.trustStore` | 信任库路径 |
| `javax.net.ssl.trustStorePassword` | 信任库密码 |
| `javax.net.ssl.trustStoreType` | `JKS` 或 `PKCS12`（JDK 9+ 默认 PKCS12） |
| `javax.net.ssl.keyStore` | 客户端证书库（不是 CA） |
| `javax.net.ssl.keyStorePassword` | 客户端证书库密码 |

注入方式：

| 变量 | 说明 |
|---|---|
| `JAVA_TOOL_OPTIONS` | 所有 JVM 启动时读取，最通用 |
| `JDK_JAVA_OPTIONS` | 仅 `java` 启动器读取 |
| `_JAVA_OPTIONS` | 非标准，部分 JVM 支持，少用 |

```bash
export JAVA_TOOL_OPTIONS="-Djavax.net.ssl.trustStore=/path/to/truststore.p12 -Djavax.net.ssl.trustStorePassword=changeit -Djavax.net.ssl.trustStoreType=PKCS12"
```

导入证书：

```bash
keytool -importcert -alias mitmproxy \
  -file mitmproxy-ca-cert.pem \
  -keystore custom-truststore.p12 \
  -storetype PKCS12 \
  -storepass changeit \
  -noprompt
```

> 注意：`trustStore` 是**替换**默认 `cacerts`。新库必须包含公共根证书，否则应用访问其它 HTTPS 站点会失败。更稳妥的做法是复制默认 `cacerts` 后再导入。

查找默认 `cacerts`：

```bash
keytool -list -cacerts -storepass changeit | head
# JDK 8 及更早：$JAVA_HOME/jre/lib/security/cacerts
# JDK 9+：$JAVA_HOME/lib/security/cacerts
```

---

### 2.6 Go

- **Windows**：使用系统证书库，通常**没有**环境变量可注入自定义 CA。
- **Linux / macOS**：`crypto/x509` 会读取 `SSL_CERT_FILE` 和 `SSL_CERT_DIR`。
- **静态编译 / 交叉编译**：如果程序使用了内置根证书或 `webpki-roots`，环境变量可能无效。
- `GOINSECURE` 只影响 `go get` 的模块下载校验，不是通用 TLS 信任设置。

```bash
# Linux/macOS
export SSL_CERT_FILE=/path/to/ca.pem
```

---

### 2.7 Rust / Cargo

| 场景 | 行为 |
|---|---|
| `reqwest` + `native-tls`（Windows） | 走 Schannel / 系统证书库 |
| `reqwest` + `native-tls`（Linux/OpenSSL） | 走 OpenSSL 默认路径，`SSL_CERT_FILE` 可能生效 |
| `reqwest` + `rustls-tls`（默认 webpki-roots） | 根证书**编译进程序**，环境变量基本无效 |
| `reqwest` + `rustls-tls-native-roots` | 读取系统证书库 |
| `openssl` crate | 受 `SSL_CERT_FILE` / `SSL_CERT_DIR` 影响 |

Cargo 自身：

| 变量 | 作用 |
|---|---|
| `CARGO_HTTP_CAINFO` | CA bundle 文件 |
| `CARGO_HTTP_CHECK_REVOKE` | Windows 上是否检查吊销（不是 CA 路径） |
| `CARGO_HTTP_PROXY` | 代理（顺带一提） |

```bash
export CARGO_HTTP_CAINFO=/path/to/ca.pem
```

---

### 2.8 .NET / Windows Schannel / PowerShell

- **Windows**：.NET（Framework 与 Core）、PowerShell、Schannel 应用都使用 **Windows 证书库**，不读 `SSL_CERT_FILE`、`REQUESTS_CA_BUNDLE`、`NODE_EXTRA_CA_CERTS`。
- **Linux/macOS 上的 .NET**：通常使用 OpenSSL 默认路径，`SSL_CERT_FILE` / `SSL_CERT_DIR` 往往有效。
- PowerShell 7 基于 .NET；Windows PowerShell 5.1 基于 .NET Framework，二者都走系统库。

安装证书：

```cmd
:: 机器级（需要管理员）
certutil -addstore -f Root C:\certs\ca.cer

:: 当前用户
certutil -user -addstore -f Root C:\certs\ca.cer

:: 查看
certutil -store Root | findstr /i "ca 名称"
certutil -user -store Root | findstr /i "ca 名称"
```

图形界面：`certmgr.msc`（当前用户）、`certlm.msc`（本机）。

> `.cer` 若是 DER 格式也可以直接导入；`certutil` 会识别。

---

### 2.9 Chromium / Electron 渲染进程

- **Windows**：读 Windows 系统证书库，不读 Node/Python/OpenSSL 的 CA 环境变量。
- **macOS**：读 Keychain。
- **Linux**：读 NSS 数据库或系统证书库（取决于发行版/构建）。

相关开关（调试用）：

| 开关 | 作用 |
|---|---|
| `--ignore-certificate-errors` | 忽略所有证书错误（极不安全） |
| `--ignore-certificate-errors-spki-list=<base64>` | 只忽略指定 SPKI 的证书错误 |
| `--proxy-server=...` | 设置代理（不是 CA） |
| `--proxy-bypass-list=...` | 设置代理绕过 |

企业环境可用 Chromium 策略分发根证书（Windows 上通常直接装系统根证书即可）。

---

### 2.10 其它工具 / 服务

| 工具 | 变量 | 说明 |
|---|---|---|
| OpenSSL CLI | `SSL_CERT_FILE`、`SSL_CERT_DIR`、`OPENSSL_CONF` | `OPENSSL_CONF` 可在配置里设置 `CAfile`/`CApath` |
| PostgreSQL (libpq) | `PGSSLROOTCERT` | 根证书文件；`PGSSLCERT`/`PGSSLKEY` 是客户端证书 |
| Docker CLI | `DOCKER_CERT_PATH` + `DOCKER_TLS_VERIFY=1` | 目录内需有 `ca.pem`、`cert.pem`、`key.pem` |
| AWS CLI / boto3 | `AWS_CA_BUNDLE` | 也可用 `--ca-bundle` |
| pip | `PIP_CERT` | 也可用 `--cert` |
| gRPC C-core | `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH` | Python/C++/Ruby/PHP/Node native grpc |
| Cargo | `CARGO_HTTP_CAINFO` | Rust 包管理器 |
| npm | `npm_config_cafile` / `npm config set cafile` | Node 生态也可用 `NODE_EXTRA_CA_CERTS` |
| Wireshark 解密 | `SSLKEYLOGFILE` | 不是信任设置；Chromium/Firefox/部分 OpenSSL 程序支持 |

---

## 3. 优先级速查

> 一般规律：**显式 API 参数 > 专用环境变量 > 通用环境变量 > 默认库**。

### Node.js

1. 代码显式传入的 `ca:` 选项（会忽略 `NODE_EXTRA_CA_CERTS`）
2. `NODE_EXTRA_CA_CERTS` 追加到基础信任集
3. 基础信任集由 `--use-bundled-ca`（默认）/ `--use-system-ca` / `--use-openssl-ca` 决定
4. `NODE_TLS_REJECT_UNAUTHORIZED=0` 会直接关闭校验

### Python requests

1. `requests.get(..., verify=...)`
2. `REQUESTS_CA_BUNDLE`
3. `CURL_CA_BUNDLE`
4. `certifi.where()`

### Python ssl / urllib / aiohttp

1. 代码显式 `ssl.SSLContext`
2. `SSL_CERT_FILE` / `SSL_CERT_DIR`（OpenSSL 平台）
3. 系统证书库（Windows/macOS）或 OpenSSL 默认路径

### curl

1. `--cacert` / `--capath`
2. `CURL_CA_BUNDLE`
3. `SSL_CERT_FILE` / `SSL_CERT_DIR`（OpenSSL 后端）
4. 系统证书库（Schannel/Secure Transport 后端）

### Git

1. `http.sslCAInfo` / `http.sslCAPath` 配置
2. `GIT_SSL_CAINFO` / `GIT_SSL_CAPATH`
3. `CURL_CA_BUNDLE`
4. libcurl 默认

### pip

1. `--cert`
2. `PIP_CERT`
3. `REQUESTS_CA_BUNDLE` / `CURL_CA_BUNDLE`
4. certifi

### AWS CLI / boto3

1. `--ca-bundle` / 客户端参数
2. `AWS_CA_BUNDLE`
3. SDK 默认（certifi）

### Java

1. 命令行 `-Djavax.net.ssl.trustStore=...`
2. `JAVA_TOOL_OPTIONS` / `JDK_JAVA_OPTIONS` 注入的 `-D...`
3. 默认 `$JAVA_HOME/lib/security/cacerts`

### Go

1. 代码显式 `tls.Config.RootCAs`
2. `SSL_CERT_FILE` / `SSL_CERT_DIR`（非 Windows）
3. 系统证书库

---

## 4. 合并 CA Bundle

很多变量是“替换”语义。如果既要信任 mitmproxy，又要信任公网 CA，就把两个 PEM 拼起来。

### 找到 certifi bundle

```bash
python -c "import certifi; print(certifi.where())"
```

### 找到 Node 内置根证书（可选）

```bash
node -e "process.stdout.write(require('tls').rootCertificates.join('\n'))" > node-roots.pem
```

### Git Bash / Linux / macOS

```bash
cat "$(python -c 'import certifi; print(certifi.where())')" \
    "$HOME/.mitmproxy/mitmproxy-ca-cert.pem" \
    > combined-ca.pem
```

### PowerShell

```powershell
$certifi = python -c "import certifi; print(certifi.where())"
Get-Content $certifi, "$env:USERPROFILE\.mitmproxy\mitmproxy-ca-cert.pem" |
    Out-File -Encoding ascii "$env:USERPROFILE\combined-ca.pem"
```

### 验证合并结果

```bash
grep -c "BEGIN CERTIFICATE" combined-ca.pem
openssl crl2pkcs7 -nocrl -certfile combined-ca.pem | openssl pkcs7 -print_certs -noout | head
```

> PEM 顺序一般不重要；信任锚是自签根证书，中间证书也可一起放进去。

---

## 5. MITM 抓包对照表

假设 mitmproxy CA 位于：

- Windows：`%USERPROFILE%\.mitmproxy\mitmproxy-ca-cert.pem`
- Linux/macOS：`~/.mitmproxy/mitmproxy-ca-cert.pem`

| 目标进程 | 推荐做法 |
|---|---|
| Node.js / Electron 主进程 / undici / npm | `NODE_EXTRA_CA_CERTS=<mitmproxy-ca-cert.pem>` |
| 新版 Node，想直接用系统库 | `NODE_USE_SYSTEM_CA=1` + 系统库安装 CA |
| Electron 渲染进程 / Chrome / Edge | 把 CA 装进 **Windows 根证书库**（`certutil -addstore`） |
| Python requests / pip / AWS CLI | `REQUESTS_CA_BUNDLE=<combined-ca.pem>`（pip 可用 `PIP_CERT`，AWS 用 `AWS_CA_BUNDLE`） |
| Python urllib / aiohttp（Linux/macOS） | `SSL_CERT_FILE=<combined-ca.pem>` |
| Python（Windows） | 优先用 `REQUESTS_CA_BUNDLE`；或把 CA 装进 Windows 证书库 |
| curl（OpenSSL 构建） | `CURL_CA_BUNDLE=<combined-ca.pem>` 或 `--cacert` |
| Git | `GIT_SSL_CAINFO=<combined-ca.pem>` |
| Java 应用 | `keytool` 导入到自定义 truststore，再用 `JAVA_TOOL_OPTIONS=-Djavax.net.ssl.trustStore=...` |
| .NET / PowerShell / Go(Windows) | 安装到 Windows 根证书库 |
| gRPC C-core | `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH=<combined-ca.pem>` |
| Cargo | `CARGO_HTTP_CAINFO=<combined-ca.pem>` |
| Docker CLI | `DOCKER_CERT_PATH` 指向含 `ca.pem` 的目录 |
| Wireshark 解密（非证书信任） | `SSLKEYLOGFILE=<path>`，并让目标程序继承该环境变量 |

> 如果目标程序有证书固定（certificate pinning）或自带 TLS 栈（libcronet、BoringSSL、rustls-webpki），上述变量可能全部无效，需要改配置、打补丁或使用 Frida 等动态手段。

---

## 6. Windows 上如何设置

### 6.1 临时（只影响当前 cmd 窗口及其子进程）

```bat
set "NODE_EXTRA_CA_CERTS=C:\certs\ca.pem"
set "REQUESTS_CA_BUNDLE=C:\certs\combined-ca.pem"
start "" "C:\path\to\app.exe"
```

### 6.2 PowerShell 当前会话

```powershell
$env:NODE_EXTRA_CA_CERTS = "C:\certs\ca.pem"
$env:REQUESTS_CA_BUNDLE = "C:\certs\combined-ca.pem"
& "C:\path\to\app.exe"
```

### 6.3 永久用户变量

```powershell
[Environment]::SetEnvironmentVariable(
    "NODE_EXTRA_CA_CERTS",
    "C:\certs\ca.pem",
    "User"
)
```

或：

```cmd
setx NODE_EXTRA_CA_CERTS "C:\certs\ca.pem"
```

> `setx` 不影响当前已经运行的进程；需要重新启动应用（有时需要重新登录/重启 Explorer）。

### 6.4 启动脚本示例（Electron 应用）

```bat
@echo off
set "NODE_EXTRA_CA_CERTS=%USERPROFILE%\.mitmproxy\mitmproxy-ca-cert.pem"
set "TODAY_HTTP_PROXY=127.0.0.1:8888"
start "" "C:\Users\xuqiao\AppData\Local\Programs\Today CN\TodayCN.exe"
```

### 6.5 安装到 Windows 证书库

```cmd
certutil -addstore -f Root C:\certs\mitmproxy-ca-cert.cer
```

- `Root` 也可换成 `TrustedPeople`（部分应用会读）。
- 机器级需要管理员权限；当前用户可用 `certutil -user -addstore -f Root ...`。
- 安装后重启目标应用。

---

## 7. 常见坑

1. **给 Electron 设了 `NODE_EXTRA_CA_CERTS`，渲染进程还是报错**
   - 渲染进程是 Chromium，不读 Node 变量；把 CA 装进 Windows 证书库。

2. **`REQUESTS_CA_BUNDLE` 只放了 mitmproxy CA，结果访问公网全失败**
   - 它是替换语义；需要合并公共根证书（见第 4 节）。

3. **`SSL_CERT_FILE` 在 Windows 上没用**
   - Windows 上很多栈走 Schannel/系统库；OpenSSL 系工具才读它。先确认 `curl -V` 的后端。

4. **Java 设了 `SSL_CERT_FILE` 没反应**
   - JVM 不读 OpenSSL 变量；必须配置 `javax.net.ssl.trustStore`。

5. **Node 设了变量，应用还是用旧证书**
   - `NODE_EXTRA_CA_CERTS` 只在进程启动时读取；必须完全重启进程。
   - 应用可能清理了环境变量（例如某些 Electron 应用会删除 `NODE_OPTIONS`、`HTTP_PROXY` 等）。

6. **代码里写了 `ca:` 选项**
   - 显式 `ca` 会忽略 `NODE_EXTRA_CA_CERTS`。

7. **模块用了自带根证书**
   - Rust `rustls` + webpki-roots、Dart、部分静态 Go 程序、libcronet 等可能完全不看环境变量。

8. **PEM 与 DER 混淆**
   - `.cer` 可能是 DER；用 `openssl x509 -inform DER -outform PEM -in cert.cer -out cert.pem` 转换。

9. **目录不是 hashed 目录**
   - `SSL_CERT_DIR` / `GIT_SSL_CAPATH` 需要 OpenSSL hashed 目录；用 `openssl rehash` 生成。

10. **相对路径**
    - `NODE_EXTRA_CA_CERTS` 要求绝对路径；其它变量也建议用绝对路径。

11. **证书链不完整**
    - 信任锚正确但中间证书缺失也会失败；确保 bundle 包含所需中间证书，或服务器正确下发。

12. **证书固定（pinning）**
    - 环境变量无法绕过 pinning；需要改代码、改配置或动态插桩。

13. **代理只影响连接，不影响信任**
    - `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` 是代理设置，与 CA 信任是两件事；MITM 场景两者都要配。

14. **系统时间错误**
    - 证书有效期校验失败时先检查系统时间。

---

## 8. 验证方法

### 8.1 查看环境变量是否生效

```bash
# Linux/macOS
env | grep -iE 'CA|CERT|SSL'

# Windows cmd
set | findstr /i "CA CERT SSL"

# PowerShell
Get-ChildItem Env: | Where-Object Name -match 'CA|CERT|SSL'
```

### 8.2 curl

```bash
curl -v --cacert /path/to/ca.pem https://example.com/
curl -v https://example.com/          # 使用环境变量/默认库
```

### 8.3 OpenSSL

```bash
openssl s_client -connect example.com:443 -servername example.com -CAfile /path/to/ca.pem
```

### 8.4 Python

```python
import os
import requests

print("REQUESTS_CA_BUNDLE =", os.environ.get("REQUESTS_CA_BUNDLE"))
print("SSL_CERT_FILE      =", os.environ.get("SSL_CERT_FILE"))
print(requests.get("https://example.com", timeout=10).status_code)
```

### 8.5 Node

```js
console.log("NODE_EXTRA_CA_CERTS =", process.env.NODE_EXTRA_CA_CERTS);
fetch("https://example.com")
  .then(r => console.log("status:", r.status))
  .catch(e => console.error("error:", e.message));
```

### 8.6 Java

```bash
keytool -list -cacerts -storepass changeit | head
# 或检查自定义 truststore
keytool -list -keystore custom-truststore.p12 -storepass changeit
```

### 8.7 Windows 证书库

```cmd
certutil -store Root | findstr /i "mitmproxy"
certutil -user -store Root | findstr /i "mitmproxy"
```

---

## 9. 一页速记

- **Node/Electron 主进程**：`NODE_EXTRA_CA_CERTS`（追加，启动时读）。
- **Node 想用系统库**：`NODE_USE_SYSTEM_CA=1` 或 `--use-system-ca`。
- **Node 想用 OpenSSL 路径**：`--use-openssl-ca` + `SSL_CERT_FILE`/`SSL_CERT_DIR`。
- **Python requests**：`REQUESTS_CA_BUNDLE`（替换 certifi；`CURL_CA_BUNDLE` 兜底）。
- **Python ssl/urllib/aiohttp**：`SSL_CERT_FILE`/`SSL_CERT_DIR`（主要 Unix）。
- **pip**：`PIP_CERT`。
- **AWS**：`AWS_CA_BUNDLE`。
- **gRPC C-core**：`GRPC_DEFAULT_SSL_ROOTS_FILE_PATH`。
- **curl**：`CURL_CA_BUNDLE` 或 `--cacert`；Windows 自带 curl 多为 Schannel。
- **Git**：`GIT_SSL_CAINFO` / `GIT_SSL_CAPATH`。
- **Java**：不读环境变量；`-Djavax.net.ssl.trustStore=...`，用 `JAVA_TOOL_OPTIONS` 注入。
- **Go**：Windows 用系统库；Linux/macOS 可读 `SSL_CERT_FILE`/`SSL_CERT_DIR`。
- **Rust**：`native-tls` 跟随系统；`rustls + webpki-roots` 基本无视环境变量；Cargo 用 `CARGO_HTTP_CAINFO`。
- **.NET / Schannel / PowerShell / Chromium/Electron 渲染进程**：只认操作系统证书库。
- **PostgreSQL**：`PGSSLROOTCERT`。
- **Docker**：`DOCKER_CERT_PATH` + `DOCKER_TLS_VERIFY=1`。
- **Wireshark 解密**：`SSLKEYLOGFILE`（不是信任设置）。
- **替换语义的变量**要合并公共根证书；**追加语义**的 `NODE_EXTRA_CA_CERTS` 不用。
- **改完变量要重启目标进程**；很多应用还会主动清理环境变量。
- **证书固定或自带 TLS 栈**：环境变量可能全部无效。

---

## 参考

- Node.js CLI: `NODE_EXTRA_CA_CERTS`、`NODE_USE_SYSTEM_CA=1`、`--use-system-ca`
- Node.js Learn: Enterprise Network Configuration
- Python `ssl` 文档：`SSL_CERT_FILE`、`SSL_CERT_DIR`、`get_default_verify_paths()`
- `requests` 文档：`REQUESTS_CA_BUNDLE`、`CURL_CA_BUNDLE`
- curl 文档：`CURL_CA_BUNDLE`、`--cacert`、`CURL_SSL_BACKEND`
- Java JSSE Reference Guide：`javax.net.ssl.trustStore`
- Cargo 文档：`CARGO_HTTP_CAINFO`
- gRPC 文档：`GRPC_DEFAULT_SSL_ROOTS_FILE_PATH`
