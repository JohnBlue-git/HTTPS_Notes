# HTTP / HTTPS Notes

這份筆記快速介紹：

- HTTP 與 HTTPS 的差異
- HTTP/1.0、HTTP/2、HTTP/3 的核心特性
- PEM 是什麼，以及憑證/私鑰如何存放
- 以文字圖示說明握手（handshake）流程
- 以 [openbmc/bmcweb](https://github.com/openbmc/bmcweb) 為例，說明後端如何支援 HTTP/1.0 與 HTTP/2

---

## 1) HTTP vs HTTPS

### HTTP 是什麼

HTTP（HyperText Transfer Protocol）是瀏覽器與伺服器溝通的應用層協定。

- 預設埠號通常是 `80`
- 資料以明文傳輸（若未搭配其他機制）
- 容易被竊聽、竄改、中間人攻擊（MITM）

### HTTPS 是什麼

HTTPS = HTTP + TLS（舊稱 SSL）。

- 預設埠號通常是 `443`
- 傳輸內容經過加密，提升機密性
- 藉由憑證驗證伺服器身份，降低假站風險
- 提供完整性保護，避免封包被靜默修改

### 一句話差異

- HTTP：像寄「明信片」，沿路可能被看見
- HTTPS：像寄「密封加密信件」，沿路難以偷看或竄改

---

## 2) HTTP/1.0 vs HTTP/2 vs HTTP/3

### ALPN 是什麼（Application-Layer Protocol Negotiation）

ALPN 是 TLS 握手中的「應用層協定協商」機制。client 會在握手時告訴 server
自己支援哪些協定（例如 `http/1.1`, `h2`），server 選一個回覆，雙方就用該協定通訊。

為什麼重要：

- 同一個 `443` 連線可自動協商要跑 HTTP/1.1 還是 HTTP/2
- 避免先用一種協定、再額外升級造成延遲
- 對多版本相容部署很關鍵（舊 client 可走 HTTP/1.1，新 client 可走 HTTP/2）

和各版本的關係：

- HTTP/1.0：歷史上不靠 ALPN（通常也不搭配現代 TLS 協商流程）
- HTTP/2：在 HTTPS 場景通常透過 ALPN 協商為 `h2`
- HTTP/3：在 QUIC + TLS 1.3 中也有協商概念；常見協定識別為 `h3`

簡化文字圖：

```text
ClientHello (ALPN: [h2, http/1.1])
            |
            v
ServerHello (ALPN selected: h2)
            |
            v
Use HTTP/2 frames on this connection
```

### HTTP/1.0（1996）

特性：

- 一個 TCP 連線通常只處理一個請求/回應（常見情況）
- 做完就關閉連線，頻繁建立連線成本高
- Header 無壓縮，重複欄位浪費頻寬

影響：

- 多資源頁面載入慢（HTML + CSS + JS + 圖片）

### HTTP/2（2015）

特性：

- Binary framing（二進位分幀）
- Multiplexing：同一條連線可並行多個 stream
- Header 壓縮（HPACK）
- Server Push（實務上已逐漸少用）

影響：

- 大幅減少「多連線排隊」問題
- 高延遲網路下，體感通常明顯優於 HTTP/1.x

### HTTP/3（2022）

特性：

- 改跑 QUIC（基於 UDP），不再依賴 TCP
- 將 TLS 1.3 整合進傳輸層握手
- 改善 TCP 的 Head-of-Line Blocking（跨 stream 影響）
- 連線遷移（Connection Migration）：網路切換更平滑（例如 Wi-Fi 切 4G）

影響：

- 在高丟包或行動網路場景更有優勢
- 首次與重連延遲可更低

---

## 3) PEM 是什麼

PEM（Privacy-Enhanced Mail）是常見的憑證與金鑰文字封裝格式。

你看到的 PEM 檔通常像這樣：

```text
-----BEGIN CERTIFICATE-----
MIID....(Base64)...
-----END CERTIFICATE-----
```

或：

```text
-----BEGIN PRIVATE KEY-----
MIIE....(Base64)...
-----END PRIVATE KEY-----
```

### PEM 常見內容

- Server certificate：伺服器憑證（公開）
- Private key：私鑰（必須保密）
- CA chain：中繼/根憑證鏈

### 在 HTTPS 的角色

- TLS 握手時，server 送出 certificate 給 client 驗證
- server 用 private key 證明「我真的持有這張憑證對應私鑰」
- client 驗證憑證鏈與主機名稱，決定是否信任

### 常見部署檔案

- `server.crt`（或 `fullchain.pem`）
- `server.key`（或 `privkey.pem`）
- 有些系統會把 cert + key + chain 合併成單一 `pem`

### PEM 在 OpenBMC 上實際更新憑證（Redfish / 路徑 / 重啟）

以下做法對應 bmcweb 的實作（`redfish-core/lib/certificate_service.hpp`）：

- HTTPS 憑證集合：`/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/`
- 單張 HTTPS 憑證：`/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/{id}`
- 通用替換動作：`/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/`

bmcweb 對應的後端常數（同檔案）：

- service：`xyz.openbmc_project.Certs.Manager.Server.Https`
- object base path：`/xyz/openbmc_project/certs/server/https`

#### 1) Redfish 操作路徑（推薦）

1. 先查目前 HTTPS 憑證列表

```bash
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

2. 直接上傳一張新憑證到 HTTPS collection（POST）

```bash
curl -k -u root:0penBmc \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }' \
  https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

3. 或使用 ReplaceCertificate action（指定要替換哪張）

```bash
curl -k -u root:0penBmc \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "CertificateType": "PEM",
    "CertificateUri": {
      "@odata.id": "/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/1"
    }
  }' \
  https://<bmc>/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/
```

補充：bmcweb 在這個 action 內只接受 `CertificateType = PEM`。

#### 2) 檔案/物件路徑怎麼對應

邏輯上可理解為：

- Redfish `.../HTTPS/Certificates/{id}`
- 對應到 D-Bus object path：`/xyz/openbmc_project/certs/server/https/{id}`
- 由 `xyz.openbmc_project.Certs.Manager.Server.Https` 這個 service 處理 Install/Replace

文字流程圖：

```text
Admin uploads PEM (Redfish)
   |
   +-- POST /Managers/bmc/NetworkProtocol/HTTPS/Certificates
   |          or
   +-- POST /CertificateService/Actions/CertificateService.ReplaceCertificate
   |
bmcweb parses certificate body
   |
calls D-Bus service:
  xyz.openbmc_project.Certs.Manager.Server.Https
   |
updates object path:
  /xyz/openbmc_project/certs/server/https/<id>
   |
HTTPS endpoint starts using new cert (implementation/platform dependent)
```

#### 3) 是否需要重啟服務

- 多數情況下，透過憑證管理 service 安裝/替換後會自動生效。
- 若你測試時仍看到舊憑證，可在 BMC shell 手動重啟 HTTPS 服務（bmcweb）：

```bash
systemctl restart bmcweb.service
```

- 驗證新憑證是否生效：

```bash
openssl s_client -connect <bmc>:443 -servername <bmc> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

## 4) Handshake 文字圖示

下面用簡化流程描述「建立安全連線」時的主要步驟。

### A. HTTPS over TCP（常見於 HTTP/1.1、HTTP/2）

先 TCP 三向握手，再 TLS 握手。

```text
Client                                Server
  | -------- SYN ----------------------> |
  | <----- SYN-ACK --------------------- |
  | -------- ACK ----------------------> |   (TCP connected)
  | -------- ClientHello -------------> |
  | <------- ServerHello + Cert ------- |
  | -------- Key Exchange/Finished ---> |
  | <------- Finished ------------------ |   (TLS established)
  | ======== HTTP Request (encrypted) =>|
  | <= HTTP Response (encrypted) =======|
```

重點：

- TCP 與 TLS 是兩段握手
- HTTP/2 常見跑在 TLS 之上（透過 ALPN 協商 `h2`）

### B. HTTP/3 over QUIC（UDP）

QUIC 內含傳輸握手 + TLS 1.3，整體更精簡。

```text
Client                                Server
  | ---- Initial (ClientHello) -------> |
  | <- Initial (ServerHello, Cert, ...) |
  | ---- Handshake Finished ----------> |   (QUIC + TLS ready)
  | ===== HTTP/3 Request (stream) =====>|
  | <==== HTTP/3 Response (stream) =====|
```

重點：

- 少了傳統「先 TCP 再 TLS」的拆段成本
- 在特定網路條件下能減少延遲與阻塞

---

## 5) 後端範例：bmcweb 真實程式碼怎麼處理 HTTP/1.x、HTTP/2、TLS

專案：<https://github.com/openbmc/bmcweb>

以下片段來自 bmcweb（主線），用來回答「它到底怎麼處理？」。

### A. bmcweb 如何用 C++ / Boost.Asio 實作 TLS

重點先說：`http/http_connection.hpp` 不直接讀 PEM。

- 這個檔案負責「連線層 TLS 流程」（detect SSL、handshake、ALPN 分流）
- PEM 載入在 SSL context 初始化（`src/ssl_key_handler.cpp`）

#### A-1) 連線層：`http/http_connection.hpp` 做 TLS 偵測與握手

來源：`http/http_connection.hpp`

```cpp
void start()
{
  ...
  readClientIp();
  boost::beast::async_detect_ssl(
      adaptor.next_layer(), buffer,
      std::bind_front(&self_type::afterDetectSsl, this,
                      shared_from_this()));
}
```

```cpp
void afterDetectSsl(const std::shared_ptr<self_type>& /*self*/,
                    boost::beast::error_code ec, bool isTls)
{
  ...
  if (isTls)
  {
    httpType = HttpType::HTTPS;
    adaptor.async_handshake(
        boost::asio::ssl::stream_base::server, buffer.data(),
        std::bind_front(&self_type::afterSslHandshake, this,
                        shared_from_this()));
  }
  else
  {
    httpType = HttpType::HTTP;
    doReadHeaders();
  }
}
```

```cpp
void afterSslHandshake(const std::shared_ptr<self_type>& /*self*/,
                       const boost::system::error_code& ec,
                       size_t bytesParsed)
{
  buffer.consume(bytesParsed);
  if (ec)
  {
    BMCWEB_LOG_WARNING("{} SSL handshake failed", logPtr(this));
    return;
  }
  BMCWEB_LOG_DEBUG("{} SSL handshake succeeded", logPtr(this));
  ...
}
```

解讀：

- 連線進來先 `async_detect_ssl`，判斷是 TLS 還是明文 HTTP
- 若是 TLS，呼叫 `async_handshake` 做非阻塞握手
- 握手成功後才進入後續 HTTP/1.x 或 HTTP/2 分流

#### A-2) PEM 載入：`src/ssl_key_handler.cpp` 建立 SSL context

來源：`http/http_server.hpp`

```cpp
void loadCertificate()
{
  if constexpr (BMCWEB_INSECURE_DISABLE_SSL)
  {
    return;
  }

  adaptorCtx = ensuressl::getSslServerContext();
}
```

來源：`src/ssl_key_handler.cpp`

```cpp
std::shared_ptr<boost::asio::ssl::context> getSslServerContext()
{
  boost::asio::ssl::context sslCtx(boost::asio::ssl::context::tls_server);

  auto certFile = ensureCertificate();
  if (!getSslContext(sslCtx, certFile))
  {
    BMCWEB_LOG_CRITICAL("Couldn't get server context");
    return nullptr;
  }
  ...
}
```

```cpp
static bool getSslContext(boost::asio::ssl::context& mSslContext,
              const std::string& sslPemFile)
{
  ...
  if (!sslPemFile.empty())
  {
    boost::asio::const_buffer buf(sslPemFile.data(), sslPemFile.size());
    mSslContext.use_certificate_chain(buf, ec);
    ...
    mSslContext.use_private_key(buf, boost::asio::ssl::context::pem, ec);
    ...
  }
  ...
}
```

```cpp
static std::string ensureCertificate()
{
  ...
  fs::path certFile = certPath / "server.pem";
  ...
  std::string sslPemFile(certFile);
  return ensuressl::ensureOpensslKeyPresentAndValid(sslPemFile);
}
```

解讀：

- bmcweb 伺服器啟動時先 `loadCertificate()`
- `getSslServerContext()` 會準備/驗證 `/etc/ssl/certs/https/server.pem`
- 然後把 PEM 內容以 `use_certificate_chain` + `use_private_key(..., pem)` 載入
- 最後 `http/http_connection.hpp` 才使用這個已就緒的 TLS context 接連線

TLS 實作流程文字圖：

```text
Server::run()
   |
loadCertificate()
   |
getSslServerContext()
   |
read / verify PEM (server.pem)
   |
use_certificate_chain + use_private_key
   |
accept socket
   |
async_detect_ssl(...)
   |
   +-- isTls = false -> HttpType::HTTP  -> doReadHeaders()
   |
   +-- isTls = true  -> HttpType::HTTPS -> async_handshake(server)
                                      |
                                      +-- fail -> close/return
                                      |
                                      +-- ok   -> afterSslHandshake()
                                                  -> ALPN / HTTP parser
```

### B. TLS ALPN：協商到 `h2` 就切進 HTTP/2

來源：`src/ssl_key_handler.cpp`

```cpp
static int alpnSelectProtoCallback(
  SSL* /*unused*/, const unsigned char** out, unsigned char* outlen,
  const unsigned char* in, unsigned int inlen, void* /*unused*/)
{
  int rv = nghttp2_select_alpn(out, outlen, in, inlen);
  if (rv == -1)
  {
    return SSL_TLSEXT_ERR_NOACK;
  }
  if (rv == 1)
  {
    BMCWEB_LOG_DEBUG("Selected HTTP2");
  }
  return SSL_TLSEXT_ERR_OK;
}
```

解讀：

- TLS 握手時透過 ALPN 從 client 提供的協定中選擇
- 選到 `h2` 即表示後續以 HTTP/2 通訊

### C. 握手後判斷 ALPN，若是 `h2` 直接 `upgradeToHttp2()`

來源：`http/http_connection.hpp`

```cpp
if constexpr (BMCWEB_HTTP2)
{
  const unsigned char* alpn = nullptr;
  unsigned int alpnlen = 0;
  SSL_get0_alpn_selected(adaptor.native_handle(), &alpn, &alpnlen);
  if (alpn != nullptr)
  {
    std::string_view selectedProtocol(
      std::bit_cast<const char*>(alpn), alpnlen);
    BMCWEB_LOG_DEBUG("ALPN selected protocol \"{}\" len: {}",
             selectedProtocol, alpnlen);
    if (selectedProtocol == "h2")
    {
      upgradeToHttp2();
      return;
    }
  }
}

doReadHeaders();
```

解讀：

- 這段就是 HTTPS 連線分流點
- `h2` -> HTTP2Connection
- 其他（如 `http/1.1`）-> 繼續走 HTTP/1.x header parser

### D. HTTP/1.x 請求處理：版本檢查 + keep-alive 決策

來源：`http/http_connection.hpp`

```cpp
// Check for HTTP version 1.1.
if (req->version() == 11)
{
  if (req->getHeaderValue(field::host).empty())
  {
    ...
  }
}

...
keepAlive = req->keepAlive();
```

解讀：

- bmcweb 不會把 HTTP/1.0 和 HTTP/1.1 寫成兩套 handler
- 它讀 `req->version()` 與 `req->keepAlive()`，交由 Beast 的語意處理
- 因此 HTTP/1.0 請求也能被接住；是否長連線由請求語意決定

### E. 支援 h2c：HTTP/1.1 Upgrade 到 HTTP/2

來源：`http/http_connection.hpp`

```cpp
if (BMCWEB_HTTP2 && isH2c)
{
  std::string_view base64settings = req->req["HTTP2-Settings"];
  if (utility::base64Decode<true>(base64settings, http2settings))
  {
    res.result(boost::beast::http::status::switching_protocols);
    res.addHeader(boost::beast::http::field::connection, "Upgrade");
    res.addHeader(boost::beast::http::field::upgrade, "h2c");
  }
}
```

以及：

```cpp
if (res.result() == boost::beast::http::status::switching_protocols)
{
  upgradeToHttp2();
  return;
}
```

解讀：

- 明文 HTTP 連線也能用 Upgrade 走 h2c
- server 回 101 後切換為 HTTP2Connection

### F. HTTP/2 內部：frame callback -> 組 request -> submit response

來源：`http/http2_connection.hpp`

```cpp
int onFrameRecvCallback(const nghttp2_frame& frame)
{
  BMCWEB_LOG_DEBUG("frame type {}", static_cast<int>(frame.hd.type));
  switch (frame.hd.type)
  {
    case NGHTTP2_DATA:
    case NGHTTP2_HEADERS:
      if ((frame.hd.flags & NGHTTP2_FLAG_END_STREAM) != 0)
      {
        return onRequestRecv(frame.hd.stream_id);
      }
      break;
    default:
      break;
  }
  return 0;
}
```

```cpp
int rv = ngSession.submitResponse(streamId, hdr, &dataPrd);
if (rv != 0)
{
  BMCWEB_LOG_ERROR("Fatal error: {}", nghttp2_strerror(rv));
  close();
  return -1;
}
```

解讀：

- nghttp2 收到 HEADERS/DATA 並在 END_STREAM 判定請求完成
- 交給既有應用層 handler 產生回應
- 透過 nghttp2 再編碼成 HTTP/2 frame 回送

### G. 文字流程圖（你可以直接放在筆記）

#### 1) HTTPS + ALPN（HTTP/1.x 或 HTTP/2）

```text
TCP accept
   |
detect SSL?
   |
   +-- no  -> HTTP (明文) 路徑
   |
   +-- yes -> TLS handshake
         |
         +-- ALPN == h2 ?
           |
           +-- yes -> upgradeToHttp2() -> HTTP2Connection
           |
           +-- no  -> doReadHeaders() -> HTTP/1.x parser
```

#### 2) HTTP/1.1 h2c Upgrade 路徑

```text
HTTP/1.1 request
   |
check Connection: Upgrade + Upgrade: h2c
   |
decode HTTP2-Settings
   |
set status 101 Switching Protocols
   |
after write response
   |
upgradeToHttp2() -> startFromSettings(...) -> HTTP2Connection
```

#### 3) HTTP/2 請求生命週期

```text
nghttp2 receives frame
   |
HEADERS/DATA callbacks
   |
END_STREAM?
   |
   +-- no  -> continue receiving
   |
   +-- yes -> onRequestRecv(streamId)
         |
         +-- auth / route / handler->handle(...)
         |
         +-- submitResponse(streamId, ...)
```

### H. 實測指令

```bash
# 測試 HTTP/1.0
curl -v --http1.0 https://<bmc-host>/redfish/v1

# 測試 HTTP/1.1
curl -v --http1.1 https://<bmc-host>/redfish/v1

# 測試 HTTP/2 (TLS ALPN)
curl -v --http2 https://<bmc-host>/redfish/v1
```

看重點：

- `ALPN, server accepted to use h2`（HTTP/2）
- 回應第一行是 `HTTP/1.1 ...` 或 `HTTP/2 ...`

---

## 6) 快速結論

- 要安全傳輸，優先使用 HTTPS（TLS）
- HTTP/2 主要解決 HTTP/1.x 在連線與併發上的效能瓶頸
- HTTP/3 透過 QUIC 在延遲、丟包場景通常更有優勢
- 後端是否「支援 HTTP/2」常牽涉應用程式、TLS 設定、反向代理三者共同配置


