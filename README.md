# HTTP / HTTPS Notes

This note gives a quick introduction to:

- The difference between HTTP and HTTPS
- Core features of HTTP/1.0, HTTP/2, and HTTP/3
- What PEM is and how certificates/private keys are stored
- Handshake process explained with text diagrams
- How backend support works in [openbmc/bmcweb](https://github.com/openbmc/bmcweb) for HTTP/1.0 and HTTP/2

---

## 1) HTTP vs HTTPS

### What HTTP Is

HTTP (HyperText Transfer Protocol) is an application-layer protocol used by browsers and servers to communicate.

- Default port is usually `80`
- Data is transmitted in plaintext (unless combined with other protections)
- Vulnerable to eavesdropping, tampering, and MITM attacks

### What HTTPS Is

HTTPS = HTTP + TLS (formerly called SSL).

- Default port is usually `443`
- Traffic is encrypted for confidentiality
- Certificates verify server identity to reduce fake-site risk
- Integrity protection helps prevent silent payload tampering

### One-Line Difference

- HTTP: like sending a postcard, visible along the route
- HTTPS: like sending a sealed encrypted letter, much harder to read or alter

---

## 2) HTTP/1.0 vs HTTP/2 vs HTTP/3

### What ALPN Is (Application-Layer Protocol Negotiation)

ALPN is an application protocol negotiation mechanism inside the TLS handshake. The client tells the server which protocols it supports (for example `http/1.1`, `h2`), and the server chooses one.

Why it matters:

- One `443` connection can negotiate HTTP/1.1 or HTTP/2 automatically
- Avoids extra upgrade round-trips and reduces latency
- Critical for mixed-version compatibility deployments

Relationship with protocol versions:

- HTTP/1.0: historically does not rely on ALPN
- HTTP/2: in HTTPS deployments, it is typically negotiated as `h2` via ALPN
- HTTP/3: over QUIC + TLS 1.3, negotiation still exists; common identifier is `h3`

Simplified text diagram:

```text
ClientHello (ALPN: [h2, http/1.1])
            |
            v
ServerHello (ALPN selected: h2)
            |
            v
Use HTTP/2 frames on this connection
```

### HTTP/1.0 (1996)

Features:

- One TCP connection usually handles one request/response
- Connection is often closed after response, causing frequent reconnect costs
- No header compression, so repeated headers waste bandwidth

Impact:

- Multi-resource pages load slowly (HTML + CSS + JS + images)

### HTTP/2 (2015)

Features:

- Binary framing
- Multiplexing: multiple streams over one connection
- Header compression (HPACK)
- Server Push (now less common in real-world usage)

Impact:

- Greatly reduces multi-connection queueing problems
- Usually better perceived performance than HTTP/1.x on high-latency networks

### HTTP/3 (2022)

Features:

- Uses QUIC (based on UDP), no longer depends on TCP
- Integrates TLS 1.3 into transport handshake
- Improves TCP-style head-of-line blocking across streams
- Connection migration for smoother network switching (for example Wi-Fi to 4G)

Impact:

- Better in lossy or mobile network conditions
- Lower latency for first connect and reconnect in many scenarios

---

## 3) What PEM Is

PEM (Privacy-Enhanced Mail) is a common text-based format for certificates and keys.

Typical PEM certificate block:

```text
-----BEGIN CERTIFICATE-----
MIID....(Base64)...
-----END CERTIFICATE-----
```

Typical private key block:

```text
-----BEGIN PRIVATE KEY-----
MIIE....(Base64)...
-----END PRIVATE KEY-----
```

### Common PEM Contents

- Server certificate (public)
- Private key (must be kept secret)
- CA chain (intermediate/root certificates)

### Role in HTTPS

- During TLS handshake, server sends certificate for client verification
- Server proves possession of the matching private key
- Client validates trust chain and hostname before trusting

### Common Deployment Files

- `server.crt` (or `fullchain.pem`)
- `server.key` (or `privkey.pem`)
- Some systems combine cert + key + chain into one `pem` file

### Practical PEM Certificate Update on OpenBMC (Redfish / Paths / Restart)

The following maps to bmcweb implementation in `redfish-core/lib/certificate_service.hpp`:

- HTTPS certificate collection: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/`
- Single HTTPS certificate: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/{id}`
- Generic replace action: `/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/`

Backend constants in bmcweb:

- service: `xyz.openbmc_project.Certs.Manager.Server.Https`
- object base path: `/xyz/openbmc_project/certs/server/https`

#### 1) Redfish Operation Path (Recommended)

1. Check current HTTPS certificate list

```bash
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

2. Upload a new certificate to HTTPS collection (POST)

```bash
curl -k -u root:0penBmc \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }' \
  https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

3. Or use ReplaceCertificate action (target a specific cert)

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

Note: bmcweb only accepts `CertificateType = PEM` for this action.

#### 2) Path Mapping (File/Object View)

Conceptually:

- Redfish `.../HTTPS/Certificates/{id}`
- Maps to D-Bus object path: `/xyz/openbmc_project/certs/server/https/{id}`
- Managed by `xyz.openbmc_project.Certs.Manager.Server.Https` for Install/Replace

Text flow diagram:

```text
Admin uploads PEM (Redfish)
   |
   +-- POST /Managers/bmc/NetworkProtocol/HTTPS/Certificates
   |          or
   +-- POST /CertificateService/Actions/CertificateService.ReplaceCertificate
   |
+bmcweb parses certificate body
   |
+calls D-Bus service:
  xyz.openbmc_project.Certs.Manager.Server.Https
   |
+updates object path:
  /xyz/openbmc_project/certs/server/https/<id>
   |
+HTTPS endpoint starts using new cert (implementation/platform dependent)
```

#### 3) Do You Need to Restart Service?

- In many cases, install/replace through the certificate manager service takes effect automatically.
- If you still see the old certificate, restart HTTPS service on BMC shell:

```bash
systemctl restart bmcweb.service
```

- Verify new certificate is active:

```bash
openssl s_client -connect <bmc>:443 -servername <bmc> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

## 4) Handshake Text Diagrams

Below is a simplified view of major steps for secure connection setup.

### A. HTTPS over TCP (Common for HTTP/1.1 and HTTP/2)

First TCP 3-way handshake, then TLS handshake.

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

Key points:

- TCP and TLS are two separate handshake stages
- HTTP/2 commonly runs over TLS and is negotiated by ALPN (`h2`)

### B. HTTP/3 over QUIC (UDP)

QUIC includes transport handshake + TLS 1.3, so setup is more compact.

```text
Client                                Server
  | ---- Initial (ClientHello) -------> |
  | <- Initial (ServerHello, Cert, ...) |
  | ---- Handshake Finished ----------> |   (QUIC + TLS ready)
  | ===== HTTP/3 Request (stream) =====>|
  | <==== HTTP/3 Response (stream) =====|
```

Key points:

- Avoids the old split of TCP handshake first, TLS handshake second
- Can reduce latency and blocking under specific network conditions

---

## 5) Backend Example: How bmcweb Handles HTTP/1.x, HTTP/2, and TLS

Project: <https://github.com/openbmc/bmcweb>

The snippets below come from bmcweb (main branch) and answer how it is actually implemented.

### A. How bmcweb Implements TLS with C++ / Boost.Asio

Key statement: `http/http_connection.hpp` does not directly parse PEM.

- This file handles connection-level TLS flow (SSL detect, handshake, ALPN routing)
- PEM loading happens during SSL context initialization (`src/ssl_key_handler.cpp`)

#### A-1) Connection Layer: TLS Detect + Handshake in `http/http_connection.hpp`

Source: `http/http_connection.hpp`

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

Explanation:

- New connection first runs `async_detect_ssl` to determine TLS vs plaintext
- If TLS, it calls non-blocking `async_handshake`
- After handshake success, it routes into HTTP/1.x or HTTP/2 flow

#### A-2) PEM Loading: Build SSL Context in `src/ssl_key_handler.cpp`

Source: `http/http_server.hpp`

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

Source: `src/ssl_key_handler.cpp`

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

Explanation:

- bmcweb loads certificate context during server startup via `loadCertificate()`
- `getSslServerContext()` prepares and validates `/etc/ssl/certs/https/server.pem`
- PEM data is loaded with `use_certificate_chain` and `use_private_key(..., pem)`
- `http/http_connection.hpp` then uses the prepared TLS context for incoming connections

TLS implementation flow diagram:

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

### B. TLS ALPN: If `h2` Is Negotiated, Switch to HTTP/2

Source: `src/ssl_key_handler.cpp`

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

Explanation:

- ALPN selects from client-supported protocols during TLS handshake
- Selecting `h2` means the connection will use HTTP/2

### C. After Handshake, If ALPN Is `h2`, Call `upgradeToHttp2()`

Source: `http/http_connection.hpp`

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

Explanation:

- This is the HTTPS protocol branch point
- `h2` goes to `HTTP2Connection`
- Others (such as `http/1.1`) continue with HTTP/1.x header parser

### D. HTTP/1.x Request Handling: Version Check + Keep-Alive Decision

Source: `http/http_connection.hpp`

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

Explanation:

- bmcweb does not split HTTP/1.0 and HTTP/1.1 into two totally separate handlers
- It reads `req->version()` and `req->keepAlive()` and relies on Beast semantics
- HTTP/1.0 requests are still handled; connection persistence follows request semantics

### E. h2c Support: Upgrade HTTP/1.1 to HTTP/2

Source: `http/http_connection.hpp`

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

And then:

```cpp
if (res.result() == boost::beast::http::status::switching_protocols)
{
  upgradeToHttp2();
  return;
}
```

Explanation:

- Plain HTTP can also move to h2c through Upgrade
- Server returns 101, then switches into `HTTP2Connection`

### F. HTTP/2 Internals: Frame Callback -> Build Request -> Submit Response

Source: `http/http2_connection.hpp`

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

Explanation:

- nghttp2 receives HEADERS/DATA and treats END_STREAM as request completion
- It calls existing app-layer handlers to generate response
- Response is encoded back into HTTP/2 frames via nghttp2

### G. Text Flow Diagrams

#### 1) HTTPS + ALPN (HTTP/1.x or HTTP/2)

```text
TCP accept
   |
detect SSL?
   |
   +-- no  -> HTTP (plaintext) path
   |
   +-- yes -> TLS handshake
         |
         +-- ALPN == h2 ?
           |
           +-- yes -> upgradeToHttp2() -> HTTP2Connection
           |
           +-- no  -> doReadHeaders() -> HTTP/1.x parser
```

#### 2) HTTP/1.1 h2c Upgrade Path

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

#### 3) HTTP/2 Request Lifecycle

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

### H. Quick Test Commands

```bash
# Test HTTP/1.0
curl -v --http1.0 https://<bmc-host>/redfish/v1

# Test HTTP/1.1
curl -v --http1.1 https://<bmc-host>/redfish/v1

# Test HTTP/2 (TLS ALPN)
curl -v --http2 https://<bmc-host>/redfish/v1
```

What to check:

- `ALPN, server accepted to use h2` (HTTP/2)
- Response start line shows `HTTP/1.1 ...` or `HTTP/2 ...`

---

## 6) Quick Conclusion

- Use HTTPS (TLS) for secure transport by default
- HTTP/2 primarily addresses HTTP/1.x connection and concurrency bottlenecks
- HTTP/3 (QUIC) is often better under latency/lossy conditions
- Whether a backend supports HTTP/2 depends on app server, TLS setup, and reverse proxy configuration together
