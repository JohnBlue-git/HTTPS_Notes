# HTTP / HTTPS / TLS / SSL / OpenSSL Notes

一份由淺入深的教學文件，說明「安全網路」實際上是如何運作的：HTTP 本身、TLS/SSL 所奠基的密碼學、憑證與 PKI、OpenSSL 工具組，以及這一切如何在 HTTP/1.0 到 HTTP/3 之間串連起來。Part 7 則把上述所有觀念套用到一個真實、正式上線運作的 C++ 實作（[openbmc/bmcweb](https://github.com/openbmc/bmcweb)）上。

## 使用方式

- **對 HTTPS 還不熟？** 依序閱讀 Part 1 → 2 → 4，就能建立完整的觀念圖像。
- **現在就要產生或檢查憑證？** 直接跳到 Part 3。
- **要除錯或講解一次握手（handshake）過程？** Part 5 有逐步文字圖示。
- **要強化伺服器安全性／檢視 TLS 設定？** 看 Part 6。
- **想看看這些觀念在真實正式環境的程式碼中如何實現？** Part 7 會逐步走過 bmcweb。

## 目錄

**Part 1 — 基礎觀念**
- [1) HTTP vs HTTPS](#cn-1)
- [2) 加密解決了什麼問題](#cn-2)
- [3) 對稱式與非對稱式密碼學](#cn-3)
- [4) 雜湊、MAC 與數位簽章](#cn-4)

**Part 2 — 深入 TLS/SSL**
- [5) SSL 與 TLS 的歷史與版本](#cn-5)
- [6) Cipher Suite 詳解](#cn-6)
- [7) ClientHello 中的 SNI 與 ALPN](#cn-7)
- [8) 憑證與 PKI](#cn-8)
- [9) TLS 握手逐步解析](#cn-9)

**Part 3 — 憑證檔案與 OpenSSL**
- [10) 憑證與金鑰檔案格式](#cn-10)
- [11) OpenSSL 實用指令速查](#cn-11)

**Part 4 — HTTP 協定版本**
- [12) HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3](#cn-12)

**Part 5 — 握手文字圖示**
- [13) TCP 與 TLS 1.2 握手](#cn-13)
- [14) TLS 1.3 握手與 0-RTT](#cn-14)
- [15) 基於 QUIC 的 HTTP/3](#cn-15)

**Part 6 — 安全性強化**
- [16) 歷史攻擊與現代預設值存在的原因](#cn-16)
- [17) 強化檢查清單](#cn-17)
- [18) 測試與診斷工具](#cn-18)

**Part 7 — 應用案例：OpenBMC bmcweb**
- [19) 透過 Redfish 管理 PEM 憑證](#cn-19)
- [20) bmcweb 原始碼走讀](#cn-20)

**附錄**
- [A) 詞彙表](#cn-a)
- [B) 指令速查表](#cn-b)
- [C) 延伸閱讀](#cn-c)

---

<a id="cn-1"></a>
## 1) HTTP vs HTTPS

### HTTP 是什麼

HTTP（HyperText Transfer Protocol）是瀏覽器與伺服器用來交換請求與回應的應用層協定。

- 預設埠號 `80`
- 在網路上以明文傳輸 — 任何位於網路路徑上的人都能讀取或竄改內容
- 沒有身分驗證機制 — 你無法確認自己真的在跟你以為的那台伺服器溝通
- 容易遭受竊聽、竄改與 MITM（中間人攻擊，man-in-the-middle）攻擊

### HTTPS 是什麼

HTTPS 是架在 TLS（Transport Layer Security，也就是過去俗稱 SSL 的現代名稱 — 見 [§5](#cn-5)）之上的 HTTP。

- 預設埠號 `443`
- **機密性（Confidentiality）**：流量經過加密，竊聽者只看得到密文
- **完整性（Integrity）**：傳輸過程中若遭竄改可被偵測出來
- **身分驗證（Authentication）**：憑證證明伺服器的身分與其宣稱的一致

以上這些都不是 HTTP 本身提供的 — 全部都是底層 TLS 層所提供的。這也是為什麼要有 Part 2 與 Part 3：用來說明 HTTPS 實際建構於其上的機制，而不只是「它有加密」這個表面事實。

### 一句話差異

- HTTP：像一張明信片 — 沿途經手的任何人都能讀到內容
- HTTPS：像一個密封、防拆封的信封，寄給經過驗證的收件人

---

<a id="cn-2"></a>
## 2) 加密解決了什麼問題

TLS 的存在是為了提供三項特性。本指南後續的每一個機制 — 金鑰交換、憑證、MAC — 都是為了達成下面其中一項：

| 特性 | 回答的問題 | TLS 如何提供 |
|---|---|---|
| **機密性（Confidentiality）** | 其他人看得到內容嗎？ | 對資料做對稱式加密（例如 AES） |
| **完整性（Integrity）** | 傳輸過程中有沒有被竄改？ | AEAD 加密演算法／MAC — 遭竄改的密文會無法解密／驗證 |
| **身分驗證（Authentication）** | 我正在跟我以為的對象溝通嗎？ | 憑證＋數位簽章，在握手過程中被檢查 |

一個好用的心智模型：**握手的全部任務，就是讓兩個素未謀面的陌生人，在一個公開的通道上就一把共享密鑰達成共識，同時證明伺服器的身分，而且過程中完全不會以竊聽者能夠重複利用的形式傳送這把密鑰。** Part 2–3 的所有內容，都是在為這一句話服務。

請注意 TLS *不會*做到的事：

- 它不會保護資料在任一端被解密之後的安全（那是應用程式／作業系統層的安全責任）。
- 它不會隱藏「有一個連線發生過」這件事，通常也不會隱藏目的地主機名稱（除非使用 ECH，否則 SNI 在網路上是可見的 — 見 [§7](#cn-7)）。
- 它不會驗證伺服器是否*值得信任*或*沒有惡意* — 它只證明伺服器持有其憑證上所宣稱身分對應的私鑰。

---

<a id="cn-3"></a>
## 3) 對稱式與非對稱式密碼學

### 對稱式加密

用同一把共享密鑰加密與解密。

- 範例：AES-128/256、ChaCha20
- 速度快 — 用於實際的大量資料傳輸（整個 HTTP 請求／回應）
- 問題：雙方在通訊之前就得先擁有「同一把」金鑰 — 但要怎麼在一個可能被別人監看的網路上把這把金鑰送過去？

### 非對稱式（公開金鑰）加密

兩把數學上相關聯的金鑰：**公鑰**（可自由分享）與**私鑰**（絕不分享）。

- 範例：RSA、ECDSA/ECDHE（橢圓曲線）、Ed25519
- 用公鑰加密的資料只有對應的私鑰能解密（若是簽章則相反：用私鑰簽署、用公鑰驗證）
- 解決了金鑰分發的問題 — 雙方事前不需要存在任何共享密鑰
- 速度比對稱式加密慢得多（大約慢 100–1000 倍），因此不適合用於大量資料傳輸

### 為什麼 TLS 兩者都用（混合式加密）

TLS 只用非對稱式密碼學來*建立*共享密鑰（透過憑證＋金鑰交換），接著就切換成快速的對稱式加密來處理實際的連線階段：

```text
1. Asymmetric crypto  -> authenticate the server + agree on a shared secret
2. Symmetric crypto   -> encrypt/decrypt all actual HTTP traffic with that secret
```

這就是為什麼像 `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` 這樣的 cipher suite 名稱，會同時列出非對稱式的部分（`ECDHE`、`RSA`）與對稱式的部分（`AES_128_GCM`） — 見 [§6](#cn-6)。

---

<a id="cn-4"></a>
## 4) 雜湊、MAC 與數位簽章

### Hash 雜湊函數

雜湊函數接受任意長度的輸入，產生固定長度的指紋（digest）。

- 範例：SHA-256、SHA-384
- 相同的輸入永遠產生相同的輸出；只要改變一個位元，輸出就會完全不同
- 單向：無法從 digest 反推回原始輸入
- 用於替憑證產生指紋、衍生金鑰、建構 MAC — **不是**加密（沒有金鑰，也無法反向還原）

### MAC / HMAC

Message Authentication Code（訊息驗證碼）用來證明資料沒有被竄改，*而且*證明發送者知道某個共享密鑰。

- HMAC = 雜湊函數 + 密鑰（例如 `HMAC-SHA256`）
- 現代 TLS 大多使用 **AEAD** 加密演算法（AES-GCM、ChaCha20-Poly1305），把加密與完整性驗證合併成同一個運算，而不是像過去那樣先加密、再算 MAC，分成兩個步驟

### 數位簽章

簽章證明一則訊息確實來自某把特定私鑰的持有者，且內容未遭竄改。

```text
Sign:   產生簽章 signature = Encrypt( Hash(message), private key )
      |
      v
  message, signature
      |
      v
Verify: 計算雜湊 Hash(message) == 解密簽章 Decrypt( signature, public key )
```

（實際的演算法如 RSA-PSS 與 ECDSA 並非真的「把雜湊值加密」，但這個示意能表達其中的概念。）

憑證授權機構（CA）簽署憑證的方式正是如此：對憑證內容取雜湊值，再用 CA 的私鑰簽署這個雜湊值。任何持有該 CA 公鑰的人，都能驗證這張憑證是否遭到偽造或竄改 — 這正是 [§8](#cn-8) 整條信任鏈背後的機制基礎。

---

<a id="cn-5"></a>
## 5) SSL 與 TLS 的歷史與版本

「SSL」與「TLS」常被交替使用，但 SSL 其實是已被淘汰的前身：

| 版本 | 年份 | 狀態 |
|---|---|---|
| SSL 1.0 | — | 從未公開發布（存在致命缺陷） |
| SSL 2.0 | 1995 | 已被禁止使用（RFC 6176，2011） |
| SSL 3.0 | 1996 | 已淘汰（RFC 7568，2015） — 被 POODLE 攻破 |
| TLS 1.0 | 1999 | 已淘汰（RFC 8996，2021） |
| TLS 1.1 | 2006 | 已淘汰（RFC 8996，2021） |
| TLS 1.2 | 2008 | 仍被廣泛使用，只要設定正確就是安全的 |
| TLS 1.3 | 2018 | 現行標準（RFC 8446），更精簡也更快 |

實務上的重點：

- 若產品／廠商今天還在說「SSL」，他們幾乎肯定是指 TLS — 真正的 SSL 已經超過十年不安全可用。
- 現代伺服器應該**只支援 TLS 1.2 與 TLS 1.3**，並完全拒絕 SSL 2.0/3.0 與 TLS 1.0/1.1（見 [§17](#cn-17)）。
- TLS 1.3 不只是「版本號比較大的 TLS 1.2」 — 它直接把多個舊機制整個移除（靜態 RSA 金鑰交換、CBC 加密模式、自訂 Diffie-Hellman 群組、壓縮），而不只是「不建議使用」。選項越少，設定錯誤的機會就越少。

---

<a id="cn-6"></a>
## 6) Cipher Suite 詳解

Cipher suite 是握手過程中協商出來的一組演算法組合。TLS 1.2 用四個部分來為它們命名：

```text
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 |    |     |         |         |
 |    |     |         |         +-- Hash used for the handshake's HMAC/PRF
 |    |     |         +------------ Bulk symmetric cipher (AES-128 in GCM/AEAD mode)
 |    |     +---------------------- Authentication: server proves identity via RSA signature
 |    +---------------------------- Key exchange: Ephemeral Elliptic-Curve Diffie-Hellman
 +--------------------------------- Protocol
```

- **金鑰交換**（`ECDHE`、`DHE`，或舊式的靜態 `RSA`） — 決定共享密鑰如何產生。`ECDHE`/`DHE` 提供**完全前向保密（Perfect Forward Secrecy, PFS）**：就算伺服器的長期私鑰之後外流，過去的連線階段依然無法被解密，因為每個 session 都使用一把從未被儲存下來、用完即棄的暫時金鑰。靜態 `RSA` 金鑰交換沒有 PFS — 一把外流的金鑰就能讓過去所有被記錄下來的連線全部曝光 — 這正是 TLS 1.3 徹底移除它的原因。
- **身分驗證**（`RSA`、`ECDSA`） — 用來證明伺服器持有該憑證對應私鑰的簽章演算法。
- **大量資料加密演算法**（`AES_128_GCM`、`AES_256_GCM`、`CHACHA20_POLY1305`） — 應該永遠選用 **AEAD**（Authenticated Encryption with Associated Data）加密演算法，同時提供機密性與完整性。應避免 CBC 模式加密演算法（`AES_128_CBC`）與 RC4 之類的串流加密演算法 — 這兩者都有已知的歷史攻擊案例（見 [§16](#cn-16)）。
- **雜湊**（`SHA256`、`SHA384`） — 用在握手過程的金鑰衍生內部運算，而不是用來處理大量資料本身。

TLS 1.3 把這件事簡化了。金鑰交換*永遠*是 (EC)DHE（名稱裡已經不再有對應欄位），cipher suite 只列出 AEAD 加密演算法＋雜湊：

```text
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
```

身分驗證演算法則改由憑證類型與 `signature_algorithms` 擴充欄位另外協商。

檢查你本機的 OpenSSL 支援哪些選項：

```bash
openssl ciphers -v 'ALL'          # every cipher suite OpenSSL knows
openssl ciphers -v 'HIGH:!aNULL'  # a reasonably strong subset
```

---

<a id="cn-7"></a>
## 7) ClientHello 中的 SNI 與 ALPN

在任何加密建立之前，握手的第一則訊息 — `ClientHello` — 就已經帶有兩個重要的明文擴充欄位。

### SNI（Server Name Indication）

問題：一台伺服器可能在同一個 IP 位址上架設許多不同的 HTTPS 網域，每個網域使用*不同*的憑證。但伺服器必須在握手進行到足以得知 HTTP 層級的客戶端請求內容之前（此時 `Host` header 都還處於加密狀態），就先決定要出示哪一張憑證。

- SNI 解決了這個問題：client 在 `ClientHello` 中以明文夾帶目標主機名稱
- server 讀取後，在握手繼續進行前選出對應的憑證
- 取捨：只要有人在觀察這條連線（網路設備、ISP），就能看到主機名稱，即使握手之後的所有內容都已加密
- 較新的緩解方式：**ECH（Encrypted Client Hello）**，一個會把 SNI 也一併加密的 TLS 1.3 擴充功能 — 部分瀏覽器／CDN 已支援，但尚未全面普及

### ALPN（Application-Layer Protocol Negotiation）

問題：client 與 server 需要在不增加額外往返次數的情況下，就要使用哪個應用層協定（HTTP/1.1、HTTP/2……）達成共識。

- client 在自己的 `ClientHello` 中列出所支援的協定（例如 `[h2, http/1.1]`）
- server 選定其中一個，並在 `ServerHello` 中回傳
- 同一個 `443` 連線可以無縫地同時服務 HTTP/1.1 與 HTTP/2 的 client — 不需要另外的「升級」步驟

```text
ClientHello (SNI: example.com, ALPN: [h2, http/1.1])
            |
            v
ServerHello (ALPN selected: h2)
            |
            v
Use HTTP/2 frames on this connection
```

與各 HTTP 版本的關係：

- HTTP/1.0 與早期的 HTTP/1.1 部署方式都早於 ALPN 出現，因此不依賴它
- 在幾乎所有實際部署中，TLS 上的 HTTP/2 都是透過 ALPN 協商為 `h2`（規格上技術層面允許未加密的 `h2c`，但瀏覽器只支援跑在 TLS 上的 `h2`）
- HTTP/3 跑在 QUIC 之上，而 QUIC 直接內嵌了 TLS 1.3；協商依然透過 ALPN 進行，識別碼是 `h3`

---

<a id="cn-8"></a>
## 8) 憑證與 PKI

憑證把一把公鑰跟一個身分（主機名稱）綁在一起，而且這張憑證本身是由另一方簽署、為這個綁定關係背書。這就是公開金鑰基礎建設（Public Key Infrastructure, PKI）。

### X.509 憑證結構

標準的憑證格式是 X.509。主要欄位：

- Certificate Info
  - **Subject** — 這張憑證所代表的對象是誰（例如 `CN=example.com`）
  - **Subject Alternative Name（SAN）** — 這張憑證實際生效的主機名稱清單；現代 client 完全忽略 `CN` 欄位，只檢查 SAN
  - **Issuer** — 由哪個 CA 簽署這張憑證
  - **Validity** — `Not Before` / `Not After` 有效日期
  - **Subject Public Key Info** — 這張憑證所背書的公鑰
  - **Extensions** — `Key Usage`、`Extended Key Usage`、`Basic Constraints`（是否為 CA？）、`Authority Key Identifier`、OCSP/CRL 位置
- **CA Signature** — 發證單位對上述所有內容的簽章

實際檢視一張憑證：

```bash
openssl x509 -in server.crt -noout -text
```

### 信任鏈（Chain of Trust）

憑證是以一條鏈的方式被驗證，而不是單獨驗證：

```text
Root CA (self-signed, pre-installed in OS/browser trust store)
   |
   |  signs
   v
Intermediate CA
   |
   |  signs
   v
Leaf / server certificate (server.crt)
```

- **Root CA** 會盡可能保持離線 — 一旦 root 私鑰外流，它曾經簽發過的每一張憑證都會變得可疑。
- **Intermediate CA** 負責實際日常的簽署工作，因此一旦遭入侵，影響範圍只限於*該中繼憑證*簽出去的憑證，而且可以直接撤銷該中繼憑證，不需動到 root。
- **自簽憑證（Self-signed certificates）** 沒有信任鏈 — 除了伺服器自己以外沒有人為它背書。適合用在本機開發或封閉的內部系統（例如 BMC 預設的 HTTPS 憑證，見 [§19](#cn-19)）；client 會顯示信任警告，因為它們的信任庫裡沒有任何路徑能連回這張憑證。

### Client 實際如何驗證一張憑證

1. 從 leaf 憑證往上建立一條鏈，直到抵達 client 已經信任的某個 root
2. 驗證鏈上每一個簽章（每張憑證確實是由上一層簽署的）
3. 檢查目前日期是否落在每一張憑證的有效期間內
4. 檢查所請求的主機名稱是否符合 leaf 憑證 SAN 清單裡的某一項
5. 檢查撤銷狀態 — CRL（Certificate Revocation List）或 OCSP（Online Certificate Status Protocol）；**OCSP stapling** 讓伺服器自行取得並附上這項證明，避免 client 多一次往返，也避免隱私外洩（否則 CA 就能得知你造訪過的每個網站）
6. 檢查 `Basic Constraints`/`Key Usage` 擴充欄位是否與每張憑證的角色一致（例如只有標示為 CA 的憑證才能簽署其他憑證）
7. 認證 Certificate 跟 CA Signature
```text
Sign: 產生 CA Signature = Encrpt( Hash(CertInfo || Server Public Key), CA Private Key )
        |
        v
  CertInfo ( with Server Public Key ), CA signature
        |
        v
Verify: 計算 Cert 雜湊 Hash( CertInfo || Server Public Key ) == 解密 CA 簽章 Decrpt( CA Signature, CA Public Key )
```

只要任何一個步驟失敗，連線就應該被拒絕 — 而不是靜默降級成不加密連線。

### 取得憑證的方式

- **自簽（Self-signed）**：自己簽自己的憑證。沒有外部信任，適合內部／測試系統。
- **私有／內部 CA**：組織自行架設 CA，並把它的 root 安裝進自家裝置的信任庫中。常見於內部基礎設施與 IoT/BMC 機群。
- **公開 CA**：由瀏覽器／作業系統本來就信任的 CA 簽署（例如透過 **Let's Encrypt**，使用自動化的 **ACME** 協定 — 免費、僅驗證網域、90 天效期並自動續期）。

### 憑證透明化（Certificate Transparency, CT）

公開 CA 被要求把它們簽發的每一張憑證都公布到公開、只能附加（append-only）的 CT log 中。這讓網域擁有者能夠察覺是否有 CA 誤發了以自己名義申請的憑證，現代瀏覽器也要求公開憑證必須附上有效的 CT 證明（SCT）才會被信任。

---

<a id="cn-9"></a>
## 9) TLS 握手逐步解析

延續 [§2](#cn-2) 的目標重述一次：僅透過目前為止都還是公開、未加密的連線所傳送的訊息，驗證伺服器身分並衍生出一把共享的對稱金鑰。

### TLS 1.2 完整握手

```text
=================================================================================================================
【 階段一：事前準備與信任鏈建立 (PKI Setup) 】
=================================================================================================================

  【 Root / Intermediate CA 】
    │ - CA Private Key (CA 嚴密保管，用來發憑證)
    │ - CA Public Key  (內建於 Client Trust Store / 作業系統中)
    │
    │  (1) Server 提交 CSR (包含 Server Public Key)
    │  (2) CA 驗證身份後發放憑證：
    └───────────────┐
                    ▼
          ┌─────────────────────────────────────────────────────────┐
          │  Server Certificate (server.crt)                        │
          ├─────────────────────────────────────────────────────────┤
          │  • Domain Name: *.example.com                           │
          │  • Server Public Key (長期非對稱公鑰)                     │
          │  • CA Signature = Sign(Cert Hash, CA Private Key)       │
          └─────────────────────────────────────────────────────────┘


=================================================================================================================
【 階段二：TLS 1.2 Handshake (身分驗證與金鑰交換) 】
=================================================================================================================

Client                                                                Server
 (擁有 CA Public Key)                                                 (擁有 Server Private Key & Certificate)
  │                                                                                 │
  │ ─── 1. ClientHello ───────────────────────────────────────────────────────────> │
  │       (ClientRandom, Cipher Suites, SNI, ALPN, Supported Groups)                │
  │                                                                                 │
  │ <── 2. ServerHello ──────────────────────────────────────────────────────────── │
  │       (ServerRandom, Selected Cipher Suite)                                     │
  │                                                                                 │
  │ <── 3. Certificate ──────────────────────────────────────────────────────────── │
  │       (server.crt；內含 Server Public Key 與 CA Signature)                       │
  │                                                                                 │
  │ <── 4. ServerKeyExchange ────────────────────────────────────────────────────── │
  │  ECDHE Signature = E( Hash(ClientRandom || ServerRandom || Server ECDHE PubKey) |
  |                     , Server Private Key )                                      |
  │       (Server ECDHE PubKey + ECDHE Signature)                                   │
  │                                                                                 │
  │ <── 5. ServerHelloDone ──────────────────────────────────────────────────────── │
  │                                                                                 │
  │  [Client 驗證階段]                                                               │
  │  A. 用 CA Public Key 驗證 Certificate                                            │
  │     → 確認憑證是 CA 簽發，且從中取出 Server Public Key                               │
  │   1. 解密 CA 簽章： Hash_ca = Decrpt( CA Signature, CA Public Key )               │
  │   2. 計算 Cert 雜湊： Hash_cert = Hash( CertInfo || Server Public Key )           │
  │   3. 公式驗證：     Hash_ca  ==  Hash_cert                                        │
  │      └──> 驗證通過：從 CertInfo 提取出 Server Public Key                           │
  │                                                                                 │
  │  B. 用 Server Public Key 驗證 Server ECDHE 簽章                                   │
  │     → 證明資料未被竄改，且 Server 確實持有對應私鑰                                    │
  │   驗證：H(ClientRandom || ServerRandom || ECDHE PubKey)                          |
  |       == D(ECDHE Signature, Server Public Key)                                  │
  │       └──> 驗證通過：確認 ECDHE 參數安全，且 Server 持有私鑰                          │
  │                                                                                 │
  │ ─── 6. ClientKeyExchange ─────────────────────────────────────────────────────> │
  │       (Client ECDHE PubKey)                                                     │
  │                                                                                 │
  │ ─── 7. [ChangeCipherSpec] & Finished ─────────────────────────────────────────> │
  │                                                                                 │
  │ <── 8. [ChangeCipherSpec] & Finished ────────────────────────────────────────── │
  │                                                                                 │
  │  [雙方獨立算出同一把對稱金鑰]                                                       │
  │  Client: Compute(Client ECDHE PrivKey + Server ECDHE PubKey + Randoms)          │
  │                                                                                 │
  │  【 Session Key / AES Key 】 <─────── 兩者一致 ────────                           │
  │                                                                                 │
  │  Server: Compute(Server ECDHE PrivKey + Client ECDHE PubKey + Randoms)          │
  │                                                                                 │

=================================================================================================================
【 階段三：Application Data 傳輸 (對稱加密) 】
=================================================================================================================

  Client                                                                         Server
    │                                                                               │
    │ === 9. HTTP / HTTPS 資料傳輸 (用【對稱金鑰 Session Key】雙向加密) ===              │
    │                                                                               │
```

這張圖其實把握手拆成三個層次：

1. **PKI Setup**：先讓 client 擁有 CA 的信任根，讓它知道「這張 server 憑證是由可信任的 CA 簽發」
2. **TLS Handshake**：透過 `ClientHello` / `ServerHello` / `Certificate` / `ServerKeyExchange` 進行身分驗證與金鑰協商
3. **Application Data**：握手完成後，雙方切換到高效率的對稱式加密，開始傳送實際的 HTTP 資料

每個階段發生的事：

1. **ClientHello** — client 提出 TLS 版本、cipher suite、一個隨機亂數，以及各種擴充欄位（SNI、ALPN、`supported_groups` 等）
2. **ServerHello** — server 選定版本／cipher suite，送出自己的隨機亂數，並回應協商結果
3. **Certificate** — server 送出它的憑證鏈，讓 client 能驗證 server 身分
4. **ServerKeyExchange** — server 送出暫時性的 `(EC)DHE` 公開值，並用該憑證對應的私鑰簽署；這個簽章把「臨時金鑰交換」與「已驗證身分」綁在一起
5. **Client 驗證** — client 先驗證憑證鏈（見 [§8](#cn-8)），確認憑證是由信任 CA 簽發，再用 server 公鑰驗證 `ServerKeyExchange` 的簽章；若成功，表示資料未被篡改，且 server 確實持有對應私鑰
6. **ClientKeyExchange** — client 送出自己的暫時金鑰分享值，雙方各自以自己的私鑰與對方的公開值，透過 Diffie-Hellman 計算出同一把共享密鑰
7. **Finished** — 兩邊各自發送加密後的 `Finished` 訊息，確認彼此都衍生出相同的金鑰，且握手過程中沒有任何內容遭到修改
8. **Application Data** — 握手完成後，雙方開始用協商好的 AEAD 演算法（例如 AES-GCM、ChaCha20-Poly1305）加密實際 HTTP 請求／回應

這些步驟的核心目標就是：

- 驗證伺服器身分：避免中間人偽造
- 協商共享對稱金鑰：讓後續流量可以用高速、低成本的對稱式加密
- 用 `Finished` 確認金鑰一致：避免「兩邊各自算出不同鑰匙」或資料被篡改

在第一個位元組的應用層資料送出之前，大約會花掉 **2 個往返（round trip）**。

### 簡化握手（Session Resumption）

每次連到剛造訪過的網站都重新走一次完整握手很浪費。TLS 會快取足夠的狀態，藉此跳過大部分流程：
- **Session ID** — server 保留 session 狀態，client 只需提醒它那個 ID
- **Session Tickets**（RFC 5077） — server 把自己的 session 狀態加密成一張 ticket 交給 client；伺服器端不需要儲存任何東西，因此擴充性更好

```text
Client                                Server
  | -- ClientHello + session ticket --> |
  | <- ServerHello (resumed) ---------- |
  | <- [ChangeCipherSpec], Finished --- |
  | -- [ChangeCipherSpec], Finished --> |
  | ======== Application Data ========  |
```

這能把握手縮短成 **1 RTT 往返**，並完全跳過憑證驗證與非對稱式金鑰交換（重複使用原始 session 衍生出的密鑰來產生新的金鑰）。

TLS 1.3 的握手與 0-RTT 恢復機制差異夠大，值得另外畫一張圖 — 見 [§14](#cn-14)。

### mTLS（Mutual TLS）

以上內容都是在向 client 驗證*伺服器*的身分。有些部署場景（服務對服務的 API、BMC 對 BMC 的管理流量、零信任內部網路）也需要驗證*client* 的身分：

- server 送出額外的 `CertificateRequest` 訊息
- client 回覆自己的 `Certificate`，以及一個 `CertificateVerify` 簽章，證明自己持有該憑證對應的私鑰
- 至此雙方的身分都已透過密碼學方式互相驗證

```text
=================================================================================================================
【 階段一：事前準備與信任鏈建立 (PKI Setup) 】
=================================================================================================================

  【 Root / Intermediate CA 】
    │ - CA Private Key (CA 嚴密保管，用來發憑證)
    │ - CA Public Key  (雙方各自預先內建於對方的 Trust Store 中)
    │
    ├─ (1) Server 提交 CSR ──> 發放 Server Certificate (server.crt) ──> 存於 Server
    └─ (2) Client 提交 CSR ──> 發放 Client Certificate (client.crt) ──> 存於 Client (mTLS 補充)


=================================================================================================================
【 階段二：TLS 1.2 Handshake (mTLS 雙向身分驗證與金鑰交換) 】
=================================================================================================================

Client                                                                Server
 (擁有 Server CA Public Key                                            (擁有 Client CA Public Key
  及 Client Private Key & client.crt)                                   及 Server Private Key & server.crt)
  │                                                                                 │
  │ ─── 1. ClientHello ───────────────────────────────────────────────────────────> │
  │       (ClientRandom, Cipher Suites, SNI, ALPN, Supported Groups)                │
  │                                                                                 │
  │ <── 2. ServerHello ──────────────────────────────────────────────────────────── │
  │       (ServerRandom, Selected Cipher Suite)                                     │
  │                                                                                 │
  │ <── 3. Certificate ──────────────────────────────────────────────────────────── │
  │       (server.crt；內含 Server Public Key 與 CA Signature)                       │
  │                                                                                 │
  │ <── 4. ServerKeyExchange ────────────────────────────────────────────────────── │
  │       (Server ECDHE PubKey + ECDHE Signature)                                   │
  │                                                                                 │
  │ <── 4.5 CertificateRequest ───────────────────────────────────────────────────  │ <== [mTLS 補充]
  │       (Server 要求 Client 提供憑證，可附上受信任 CA 清單)                            │
  │                                                                                 │
  │ <── 5. ServerHelloDone ──────────────────────────────────────────────────────── │
  │                                                                                 │
  │  [Client 驗證 Server 階段]                                                       │
  │  A. 用 CA Public Key 驗證 server.crt (取出 Server Public Key)                     │
  │  B. 用 Server Public Key 驗證 Server ECDHE 簽章 (確認 Server 身份與參數安全)         │
  │                                                                                 │
  │ ─── 5.5 Certificate ──────────────────────────────────────────────────────────> │ <== [mTLS 補充]
  │       (client.crt；內含 Client Public Key 與 CA Signature)                       │
  │                                                                                 │
  │ ─── 6. ClientKeyExchange ─────────────────────────────────────────────────────> │
  │       (Client ECDHE PubKey)                                                     │
  │                                                                                 │
  │ ─── 6.5 CertificateVerify ───────────────────────────────────────────────────>  │ <== [mTLS 補充]
  │       Signature = E( Hash(所有歷史 Handshake 訊息), Client Private Key )          │
  │                                                                                 │
  │                                                                                 │
  │        [Server 驗證 Client 階段]                                                 │ <== [mTLS 補充]
  │        A. 用 CA Public Key 驗證 client.crt → 確認憑證合法並取出【Client Public Key】 │
  │        B. 用 Client Public Key 驗證 Client ECDHE 簽章 ((確認 Client 身份與參數安全)) │
  │                                                                                 │
  │ ─── 7. [ChangeCipherSpec] & Finished ─────────────────────────────────────────> │
  │                                                                                 │
  │ <── 8. [ChangeCipherSpec] & Finished ────────────────────────────────────────── │
  │                                                                                 │
  │  [雙方獨立算出同一把對稱金鑰]                                                       │
  │  Client: Compute(Client ECDHE PrivKey + Server ECDHE PubKey + Randoms)          │
  │                                                                                 │
  │  【 Session Key / AES Key 】 <─────── 兩者一致 ────────                           │
  │                                                                                 │
  │  Server: Compute(Server ECDHE PrivKey + Client ECDHE PubKey + Randoms)          │
  │                                                                                 │

=================================================================================================================
【 階段三：Application Data 傳輸 (對稱加密) 】
=================================================================================================================

  Client                                                                         Server
    │                                                                               │
    │ === 9. HTTP / HTTPS 資料傳輸 (用【對稱金鑰 Session Key】雙向加密) ===              │
    │                                                                               │
```

---

<a id="cn-10"></a>
## 10) 憑證與金鑰檔案格式

同一張憑證或金鑰可以用好幾種容器格式儲存。搞混這些格式是實務上最常見的 TLS 頭痛問題之一。

### PEM（Privacy-Enhanced Mail）

文字格式、Base64 編碼，以開頭／結尾標記行分隔。在 Linux／以 OpenSSL 為基礎的系統上，這是目前為止最常見的格式。

```text
-----BEGIN CERTIFICATE-----
MIID....(Base64)...
-----END CERTIFICATE-------
```

```text
-----BEGIN PRIVATE KEY-----
MIIE....(Base64)...
-----END PRIVATE KEY-------
```

- 大致上可讀、可以拿去做 diff、也容易串接 — 一個 PEM「鏈」檔案其實就是好幾個 `BEGIN/END CERTIFICATE` 區塊前後相接而成
- 單一 `.pem` 檔案也可以同時把憑證*和*它的私鑰包在一起（兩個區塊、一個檔案） — bmcweb 的 `server.pem` 就是這樣做的，見 [§20](#cn-20)
- 常見副檔名：`.pem`、`.crt`、`.cer`、`.key` — 副檔名只是一種*慣例*，不保證內容一定如此；不確定時務必用 `openssl x509 -text` 或 `openssl pkey -text` 檢查

### DER（Distinguished Encoding Rules）

跟 PEM 用 Base64 包起來的其實是同一種底層 ASN.1 結構，只是 DER 是它的二進位編碼版本。

- 不可人工閱讀，但比較精簡
- 常見於 Windows／Java 系統，以及部分嵌入式環境
- 副檔名：`.der`，有時也用 `.cer`

### PKCS#7 / P7B

一組憑證的集合（一條鏈），但**絕不包含私鑰**。

- 主要用在 Windows／Java，把憑證與中繼鏈打包一起發送
- 副檔名：`.p7b`、`.p7c`

### PKCS#12 / PFX

單一二進位、有密碼保護的容器，把憑證、私鑰，以及選擇性的憑證鏈全部包在一起 — 只要這一個檔案就能架起一台伺服器。

- 常用於匯入 Windows／Java 的 keystore 或瀏覽器
- 副檔名：`.p12`、`.pfx`

### CSR（Certificate Signing Request，PKCS#10）

不是憑證 — 而是對憑證的*申請*。

- 包含你的公鑰＋身分資訊（CN、SAN、組織），並用你自己的私鑰簽署以證明你確實持有它
- 你把這個檔案送給 CA；CA 驗證後回傳一張已簽署的憑證
- 副檔名：`.csr`

### 快速對照表

| 副檔名 | 通常內容 | 格式 |
|---|---|---|
| `.pem` | 憑證、金鑰、鏈，或 CSR（以上皆有可能） | 文字／Base64 |
| `.crt` / `.cer` | 憑證 | 通常是 PEM，偶爾是 DER |
| `.key` | 私鑰 | 通常是 PEM |
| `.csr` | 簽署請求 | 文字／Base64 |
| `.der` | 憑證或金鑰 | 二進位 |
| `.p7b` / `.p7c` | 憑證鏈，不含金鑰 | 二進位 |
| `.p12` / `.pfx` | 憑證＋金鑰＋鏈，有密碼保護 | 二進位 |

有疑慮時別相信副檔名，直接用 [§11](#cn-11) 裡的指令打開來看。

---

<a id="cn-11"></a>
## 11) OpenSSL 實用指令速查

OpenSSL 既是一套密碼學函式庫（`libssl` 負責 TLS 協定、`libcrypto` 負責底層演算法原語），也是建構在其上的 `openssl` 命令列工具 — 本節要講的是這個命令列工具。

### 產生私鑰

```bash
# RSA (traditional, widely compatible)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key

# EC (smaller keys, faster, equally strong at much shorter key lengths)
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out server.key
```

### 產生 CSR（送給 CA）

```bash
openssl req -new -key server.key -out server.csr \
  -subj "/C=US/O=MyOrg/CN=example.com"
```

### 產生自簽憑證

```bash
# From an existing key
openssl req -x509 -new -key server.key -days 365 -out server.crt \
  -subj "/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com"

# One-liner: key + self-signed cert together
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout server.key -out server.crt -days 365 \
  -subj "/CN=example.com" \
  -addext "subjectAltName=DNS:example.com"
```

`-addext subjectAltName` 很重要：現代 client 會忽略 `CN`，並拒絕沒有對應 SAN 項目的憑證（見 [§8](#cn-8)）。

### 檢視內容

```bash
openssl x509 -in server.crt -noout -text                       # full certificate
openssl x509 -in server.crt -noout -subject -issuer -dates     # quick summary
openssl req  -in server.csr -noout -text                       # a CSR
openssl pkey -in server.key -noout -text                       # a private key (any type)
```

### 驗證憑證鏈

```bash
openssl verify -CAfile ca.crt server.crt
# with a separate intermediate:
openssl verify -CAfile root-ca.crt -untrusted intermediate.crt server.crt
```

### 確認憑證與金鑰是否真的成對

比對兩者內嵌的公鑰 — RSA 與 EC 都適用：

```bash
openssl x509 -in server.crt -noout -pubkey | openssl sha256
openssl pkey  -in server.key -pubout        | openssl sha256
# identical output = they match
```

### 格式互轉

```bash
# PEM -> DER
openssl x509 -in server.crt -outform der -out server.der

# DER -> PEM
openssl x509 -in server.der -inform der -outform pem -out server.pem

# PEM (cert + key [+ chain]) -> PKCS#12
openssl pkcs12 -export -in server.crt -inkey server.key \
  -certfile ca-chain.crt -out server.pfx

# PKCS#12 -> PEM
openssl pkcs12 -in server.pfx -out server.pem -nodes
```

### 與正式運作中的伺服器對話

```bash
# Full handshake dump
openssl s_client -connect example.com:443 -servername example.com

# Just the certificate summary (non-interactive)
openssl s_client -connect example.com:443 -servername example.com \
  </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# Check which ALPN protocol gets negotiated
openssl s_client -connect example.com:443 -servername example.com \
  -alpn h2,http/1.1 </dev/null 2>&1 | grep -i alpn

# Force a specific protocol version (to confirm old versions are actually rejected)
openssl s_client -connect example.com:443 -tls1_2
```

### 其他實用指令

```bash
openssl ciphers -v                 # list cipher suites this OpenSSL build supports
openssl rand -base64 32            # generate a random secret
openssl dgst -sha256 file.txt      # hash a file
openssl speed aes-128-gcm          # benchmark a cipher on this machine
```

### 常見錯誤

| 症狀 | 可能原因 |
|---|---|
| `unable to get local issuer certificate` | 伺服器沒有送出中繼鏈，或呼叫 `verify` 時沒有帶入 |
| 伺服器啟動失敗／出現「key values mismatch」 | 憑證與金鑰檔其實不成對 — 用上面的 pubkey-hash 比對法檢查 |
| 瀏覽器顯示「自簽憑證」警告 | 這是正常現象 — 沒有 CA 替這張憑證背書（見 [§8](#cn-8)） |
| `certificate has expired` | 檢查 `-dates` 的輸出；記得續期 |
| 憑證有效卻出現主機名稱不符警告 | 憑證的 SAN 清單裡沒有包含你連線時使用的主機名稱 |

---

<a id="cn-12"></a>
## 12) HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.0（1996 年，RFC 1945）

- 每個 TCP 連線只處理一組請求／回應；連線通常隨後就關閉
- 沒有強制要求 `Host` header — 一個 IP 實際上只能乾淨地服務一個網站
- 沒有 header 壓縮；每個請求都得把 header 完整重複傳送一次

影響：多資源頁面（HTML + CSS + JS + 圖片）幾乎每個請求都要再付一次新的 TCP（若是 HTTPS，還要再加一次新的 TLS）握手成本。

### HTTP/1.1（1997 年，2014 年修訂為 RFC 7230–7235）

現今大部分網路流量骨子裡其實還是跑在這個版本上，只是被 HTTP/2 的光環蓋過去了。

- **預設使用持久連線（Persistent connections）** — 一個 TCP 連線可以承載許多個請求
- **強制要求 `Host` header** — 讓「以名稱為基礎的虛擬主機（name-based virtual hosting）」成為可能（一個 IP 服務多個網域）
- **Chunked transfer encoding** — 回應本文可以在不預先知道總長度的情況下邊產生邊串流傳送
- **Pipelining** — 技術上允許不等前一個回應就送出下一個請求，但會遭受 head-of-line blocking 問題（一個慢回應會擋住排在它後面的所有請求），實務上瀏覽器幾乎都不使用它

概念圖：Persistent connections
```text
HTTP/1.0：每個請求都要新開一條 TCP 連線

Client                Server
  | -- Req1 -->        |
  |                    |
  | -- Req2 -->        |
  |                    |
  | -- Req3 -->        |
  |                    |
  | -- Req4 -->        |
  |                    |
  └─────────────┬──────┘
                │ 4 個獨立連線
                │ + 4 次 TCP 握手
                │ + 4 次 TLS 握手（若走 HTTPS）


HTTP/1.1：同一條 TCP 連線可承載多個請求

Client                   Server
  | -- Req1 ------------------------> |
  | -- Req2 ------------------------> |
  | -- Req3 ------------------------> |
  | -- Req4 ------------------------> |
  | <--- Res1 ----------------------- |
  | <--- Res2 ----------------------- |
  | <--- Res3 ----------------------- |
  | <--- Res4 ----------------------- |
  └───────────────────────────────┬───┘
                                  │ 1 條連線重複使用
                                  │ TCP / TLS 握手成本大幅降低

底層連線的唯一識別：5-Tuple（五元組）
對作業系統與 Web 伺服器來說，一個 HTTPS 連線是由 TCP 5-Tuple（五元組） 唯一確定的：
  源 IP 地址 (Source IP)
  源端口 (Source Port) — Client 端隨機分派
  目的 IP 地址 (Destination IP)
  目的端口 (Destination Port) — 通常是 443
  傳輸層協定 (TCP 或 UDP)

只要 Client 端發起連線時，這個五元組相同，伺服器就會認定這是 「同一個 Connection」。TLS 握手完成後算出的對稱金鑰（Session Key） 也會直接綁定在這個 Connection 上。

這就是「持久連線」的價值：**把原本很多個短命連線，合併成一條長連線**，讓請求/回應能反覆重用，降低握手與重複建立連線的成本。
```

### HTTP/2（2015 年，RFC 7540）

- 使用**二進位分幀（Binary framing）**取代文字解析
- **Header 壓縮（HPACK）** — header 會被壓縮，並與先前的請求做差異比對
- **多工（Multiplexing）** — 同一個連線上可以有許多個並行的 stream，解決了 HTTP/1.1「每個 host 開 6 條連線」這種變通做法
- **Server Push** — 伺服器可以主動推送它預期 client 會需要的資源；實務上這項功能已被淘汰，並從大多數瀏覽器中移除（例如 Chrome 已在 2022 年移除），原因是現實世界中的快取效益不佳
- 實務上必須搭配 TLS — 沒有任何主流瀏覽器支援明文的 HTTP/2（`h2c`）
- 仍然會遇到 **TCP 層級**的 head-of-line blocking：一個封包遺失就會卡住*所有*多工的 stream，因為它們共用同一條 TCP byte stream

概念圖：HPACK header compression
```text
HTTP/1.1：每個請求都把幾乎相同的 Header 重複發送

Request 1
GET /index.html
Host: example.com
User-Agent: curl/8.0
Accept: */*

Request 2
GET /app.js
Host: example.com
User-Agent: curl/8.0
Accept: */*

=> 很多 header 重複，浪費頻寬


HTTP/2 / HPACK：先建立動態字典（Dynamic Table），後續只傳「差異」

Dynamic Table（伺服器 / client 共用）
------------------------------------------------
| 索引 | 欄位名稱         | 欄位值                 |
| 1    | :method          | GET                 |
| 2    | :scheme          | https               |
| 3    | :authority       | example.com         |
| 4    | user-agent       | curl/8.0            |
| 5    | accept           | */*                 |
------------------------------------------------

Request 2 的 Header 其實很多都已經知道：

原始：
  :method = GET
  :authority = example.com
  user-agent = curl/8.0
  accept = */*

HPACK 壓縮後只傳：
  [Index 1] [Index 3] [Index 4] [Index 5]

=> 只傳「引用索引」或「與前次差異的增量」
=> 這就是 HPACK： header 不是整份重送，而是對先前資料做差異比對


概念上的心智模型：
- 先共享一份常見 Header 字典
- 後續請求只傳「我用的是哪個索引」或「新值和舊值差什麼」
- 這樣可以大幅降低 Header 重複度，提升載入效率

這是 HTTP/2 的另一個關鍵優化：**header 不再像 HTTP/1.x 一樣整段重複重送，而是透過索引表與差異比對來壓縮**。這特別適合多個資源同時下載時，因為每個請求的 header 會有大量重複字串。
```

概念圖：Binary framing
```text
HTTP/1.x：訊息是文字型態，必須逐行解析

GET /index.html HTTP/1.1
Host: example.com
User-Agent: curl/8.0
Accept: */*


HTTP/2：訊息被切成二進位 frame，再交給 stream / priority / length 管理

+-----------------------------------------------------------+
| Frame Header                                              |
|  - Length                                                 |
|  - Type                                                   |
|  - Flags                                                  |
|  - Stream ID                                              |
+---------------------- +------------------------------------+
                       |
                       v
              +------------------+
              | DATA / HEADERS  |
              |  frame payload   |
              +------------------+

        例如：
        HEADERS frame  -> 這個 stream 的 header
        DATA frame     -> 這個 stream 的 body
        SETTINGS frame -> 協商連線層參數


好處：
- 不再依賴「\r\n」分隔來解析
- frame 可以被高效地多工、重排、流量控制
- 一個 stream 的資料不必和另一個 stream 的資料混在同一份純文字訊息裡

這個差異很重要：**HTTP/1.x 是以「文字請求/回應」為核心，而 HTTP/2 是以「二進位 frame stream」為核心**。也就是說，HTTP/2 不再像 HTTP/1.x 一樣用一大段可讀文字去描述整個請求，而是把它拆成一個個 frame，然後由 stream 來組裝這些資料。
```

概念圖：Multiplexing vs HOL blocking
```text
HTTP/1.1：每個請求幾乎要自己佔用一條連線

Req A ──┐
Req B ──┼──> 連線 1
Req C ──┤
Req D ──┘

Req E ──┐
Req F ──┼──> 連線 2
Req G ──┤
Req H ──┘

=> 需要很多條連線，且請求只好排隊等待


HTTP/2：同一條 TCP 連線上有多個獨立 stream

TCP Connection
-------------------------------------------------
| Stream 1 | Stream 2 | Stream 3 | Stream 4 | 
| HTML     | CSS      | JS       | IMG      | 
|  Req     |  Req     |  Req     |  Req     | 
-------------------------------------------------

「多工」的意思是：
- 多個 stream 並行存在於同一個連線內
- 不是每個請求都要拆成新連線
- 一個大資源不會完全封住其他小資源的傳輸
- 伺服器收到混雜在一起的 Frame 後，直接依據 Stream ID 重新組裝出對應的 Request，完全不需要排隊，實現場平行的雙向多工傳輸

但注意：
  TCP 仍是一條單一 byte stream
  一個封包遺失 => TCP 重新整理順序 => 所有 stream 都可能被卡住

     [封包遺失]
           ↓
   TCP 重傳 / 排序修正
           ↓
   所有 stream 共同等待
```

### HTTP/3（2022 年，RFC 9114，基於 QUIC，RFC 9000）

- 跑在 **QUIC**（基於 UDP）之上，而不是 TCP
- TLS 1.3 直接內建在傳輸層的握手過程中，而不是疊加在上層
- 每個 stream 都各自獨立可靠 — 封包遺失只會卡住它所屬的那一個 stream，修正了 HTTP/2 遺留下來的 head-of-line blocking 問題
- **連線遷移（Connection migration）** — 連線能撐過網路環境的變化（例如從 Wi-Fi 切到行動網路），因為連線是以 Connection ID 識別，而不是 IP/port 組合
- 可以對曾經造訪過的伺服器提供 **0-RTT** 重新連線（與 TLS 1.3 0-RTT 有相同的重送風險考量 — 見 [§14](#cn-14)）

概念圖：stream independency
```text
QUIC Connection (Connection ID)
-------------------------------------------------
| Stream 1 | Stream 2 | Stream 3 | Stream 4 | 
| HTML     | CSS      | JS       | IMG      | 
| 可靠傳輸  | 可靠傳輸   | 可靠傳輸  | 可靠傳輸  | 
-------------------------------------------------

一個封包遺失時，只有它所屬的 stream 受影響
其他 stream 可繼續傳輸，不會全部卡住
```

概念圖：Connection migration
```text
HTTP/2 / TCP：連線識別依賴 IP + Port

Client (Wi‑Fi)      ──────── 連線 ────────>   Server
  IP: 10.0.0.5:52134

切換到 4G / 行動網路後：
Client (4G)          ──────── 連線 ────────>   Server
  IP: 192.168.1.50:43122

=> 這兩者其實是「不同的 TCP 連線」
=> 連線中斷 / 重新建立 / 重新握手


HTTP/3 / QUIC：連線識別依賴 Connection ID

Client (Wi‑Fi)               Server
   [Connection ID: CID-42]  <──────>  [Connection ID: CID-42]
         │
         ├─ 切換到 4G 時，IP 變了，但 Connection ID 不變
         │
         └─ 仍然視為同一條連線

=> 連線可「遷移」而不必重建整個會話
=> 對移動端與切換網路的場景更加友善

這也是 HTTP/3 一個很關鍵的變化：**它從「以 IP/Port 來識別連線」改成「以 Connection ID 來識別連線」**，因此在 Wi‑Fi ↔ 行動網路切換時，連線可以繼續存在，不一定立即失效。
```

### 比較表

| 比較項目 | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
| :--- | :--- | :--- | :--- | :--- |
| **傳輸層** | TCP | TCP | TCP | **QUIC (UDP)** |
| **支援/主流 TLS 版本** | TLS 1.0 / 1.1 / 1.2 *(早期 SSL 3.0)* | TLS 1.2 / 1.3 *(可不用 TLS，走明文 HTTP)* | **TLS 1.2 / 1.3** *(瀏覽器與規範強制要求)* | **僅支援 TLS 1.3** *(標準規範內建)* |
| **TLS 整合方式** | 獨立層 (Over TLS) | 獨立層 (Over TLS) | 獨立層，透過 **ALPN** 協商 (`h2`) | **原生內建**於 QUIC 傳輸層 |
| **首次連線握手延遲** *(傳輸層 + TLS)* | **3 RTT**<br>*(1 TCP + 2 TLS 1.2)* | **2~3 RTT**<br>*(1 TCP + 1~2 TLS)* | **2~3 RTT**<br>*(1 TCP + 1~2 TLS)* | **1 RTT**<br>*(QUIC 傳輸層與 TLS 1.3 握手合併)* |
| **快速恢復連線延遲** *(Resumption)* | **2 RTT** *(Session ID)* | **1~2 RTT** *(Session Ticket)* | **1~2 RTT** *(Session Ticket / PSK)* | **0 RTT** *(TLS 1.3 0-RTT PSK)* |
| **所需連線數** | 多 | 較少（持久連線） | 一條（多工） | 一條（多工） |
| **Header 壓縮** | 無 | 無 | HPACK | QPACK |
| **Head-of-line blocking** | 有（嚴重） | 有 | 僅傳輸層 (TCP 隊頭阻塞) | **完全無** |

某條連線最終實際使用哪個版本，是在 TLS 握手期間由 ALPN（或 QUIC 對應的機制）決定的 — 見 [§7](#cn-7)。

---

<a id="cn-13"></a>
## 13) TCP 與 TLS 1.2 握手

結合兩層的快速參考圖 — 觀念性的逐步說明在 [§9](#cn-9)。

```text
Client                                Server
  | -------- SYN ----------------------> |
  | <----- SYN-ACK --------------------- |
  | -------- ACK ----------------------> |   (TCP connected)
  |                                      |
  | -------- ClientHello --------------> |
  | <------- ServerHello + Cert -------- |
  | -------- Key Exchange/Finished ----> |
  | <------- Finished ------------------ |   (TLS established)
  | ======== HTTP Request (encrypted) ==>|
  | <===== HTTP Response (encrypted) ====|
```

重點：

- TCP 與 TLS 是兩個各自獨立、疊在一起的握手 — TCP 保證循序送達，TLS 在上面加上機密性／完整性／身分驗證
- HTTP/1.1 與 HTTP/2 通常都是跑在完全相同的這套堆疊上；TLS 握手期間的 ALPN 會決定連線最終要說哪一種協定

---

<a id="cn-14"></a>
## 14) TLS 1.3 握手與 0-RTT

TLS 1.3 讓 client 在第一則訊息裡就直接「猜」一個金鑰交換群組並送出自己的金鑰分享值，而不是等 server 告知想用哪個群組，藉此把握手從 2 個往返縮短為 1 個。

### 完整的 1-RTT 握手

```text
Client                                              Server
  | -- ClientHello + key_share ----------------------> |
  |                                                    |
  | <- ServerHello + key_share ----------------------- |
  |    [from here on, server's messages are encrypted] |
  | <- EncryptedExtensions --------------------------- |
  | <- Certificate ----------------------------------- |
  | <- CertificateVerify ----------------------------- |
  | <- Finished -------------------------------------- |
  |                                                    |
  | -- Finished -------------------------------------> |
  | ====== Application Data (both directions) ======== |
```

- server 一看到 client 的 `key_share` 就能立刻算出共享密鑰，所以從 `ServerHello` 之後幾乎所有內容 — 包括憑證 — 都已經是加密的
- client 送出自己的 `Finished` 之後可以立即送出應用層資料 — 加密後的應用層資料會跟握手最後一則訊息**在同一波（flight）**一起送出
- TLS 1.3 已經不存在靜態 RSA 金鑰交換 — 每次握手都使用 (EC)DHE，所以每個 session 預設都具備完全前向保密（見 [§6](#cn-6)）

### 0-RTT 恢復（及其取捨）

如果 client 手上有先前連到同一台伺服器所拿到的 session ticket（PSK），它可以在第一波訊息中就直接送出加密的應用層資料：

```text
Client                                 Server
  | -- ClientHello + PSK + early_data --> |
  | ==== 0-RTT Application Data ========> |
  | <- ServerHello + Finished ----------- |
  | <==== Application Data ===============|
```

- client 的第一個請求*送出*之前，往返次數是零 — 對回訪用戶來說是實實在在的延遲優勢
- **要注意：0-RTT 資料不具前向保密性，而且可被重送（replayable）。** 這些資料裡沒有任何內容跟 server 產生的、無法預測的新鮮值綁在一起，所以攔截到 0-RTT 請求的網路攻擊者可以直接重送它，而 server 沒有內建方法能分辨這與原始請求的差異。這正是為什麼 0-RTT 只建議用在冪等（idempotent）操作（安全的 `GET` 請求）上，絕不能用在任何有副作用的操作（付款、表單送出）。

---

<a id="cn-15"></a>
## 15) 基於 QUIC 的 HTTP/3

```text
Client                                Server
  | ---- Initial (ClientHello) -------> |
  | <- Initial (ServerHello, Cert, ...) |
  | ---- Handshake Finished ----------> |   (QUIC + TLS ready)
  | ===== HTTP/3 Request (stream) =====>|
  | <==== HTTP/3 Response (stream) =====|
```

重點：

- QUIC 把傳輸層握手與 TLS 1.3 握手合併成單一次交換 — 不再有獨立的「先 TCP 連線、再走 TLS 握手」這兩個階段
- 每個 HTTP/3 請求／回應都跑在自己獨立可靠的 QUIC stream 上，所以一個封包遺失只會卡住它所屬的那個 stream
- QUIC 連線是以 Connection ID 識別，而不是傳統的（來源 IP、來源埠、目的 IP、目的埠）四元組，這正是連線遷移之所以可行的原因

---

<a id="cn-16"></a>
## 16) 歷史攻擊與現代預設值存在的原因

這正是為什麼 [§17](#cn-17) 會建議「直接用現代預設值」 — 今天的每一項預設值，都是針對某個真實、有名有姓的攻擊所做出的直接回應。（僅為教育性摘要，用意是理解這些預設值*為什麼*存在 — 不是攻擊教學。）

| 攻擊 | 年份 | 根本原因 | 只要遵循 §17，為什麼就不再是問題 |
|---|---|---|---|
| **BEAST** | 2011 | TLS 1.0 CBC 模式加密演算法的 IV 可被預測 | 已在 TLS 1.1+ 修正；避免 CBC，改用 AEAD |
| **CRIME / BREACH** | 2012/13 | 壓縮率會洩漏壓縮串流中的機密資訊 | TLS 層級的壓縮預設為關閉；壓縮含機密內容的回應時仍需小心 |
| **Heartbleed** | 2014 | OpenSSL 心跳（heartbeat）擴充功能實作中的緩衝區過度讀取錯誤 | 屬於函式庫層級的錯誤，並非協定本身的缺陷 — 已修補；教訓是任何此類漏洞公開後都應輪換金鑰／憑證，因為過去的流量有可能已經外洩 |
| **POODLE** | 2014 | SSL 3.0 CBC 模式中的 padding oracle 漏洞 | 直接停用整個 SSL 3.0（如今已沒有任何實質相容性效益） |
| **降級攻擊（Downgrade attacks）** | 持續中 | 攻擊者強迫握手協商出較弱、可被破解的版本／加密演算法 | `TLS_FALLBACK_SCSV`、乾脆完全不提供舊版本，以及 HSTS（連第一次連線都不讓它嘗試明文 HTTP） |
| **ROBOT** | 2017 | 針對 RSA 金鑰交換的攻擊，是 1998 年 Bleichenbacher padding oracle 攻擊的復活版 | 避免使用靜態 RSA 金鑰交換 — 改用 ECDHE，TLS 1.3 已將其列為強制項目 |

共同的脈絡是：這裡幾乎每一項攻擊，鎖定的不是某個可選的舊機制（SSL 3.0、CBC 加密演算法、靜態 RSA 金鑰交換） — 這些東西在完全現代化的設定中根本不會被提供 — 就是針對特定實作的錯誤，而非協定本身的問題。這也是為什麼 TLS 1.3 的設計方式是直接刪掉選項，而不是加一個開關讓人手動關閉它們。

---

<a id="cn-17"></a>
## 17) 強化檢查清單

- **通訊協定版本**：只支援 TLS 1.2 與 TLS 1.3。明確停用 SSL 2.0/3.0 與 TLS 1.0/1.1 — 不要只依賴「預設不提供」，因為有些協定堆疊仍然預設開啟它們。
- **Cipher suite**：只允許 AEAD 加密演算法（`*_GCM`、`*_CHACHA20_POLY1305`）。停用 CBC 模式的 suite、RC4、3DES，以及任何名稱中帶有 `NULL` 或 `EXPORT` 的選項。
- **金鑰交換**：優先選用 ECDHE 而非靜態 RSA 金鑰交換，以取得完全前向保密（在 TLS 1.3 下已自動達成）。
- **金鑰大小／類型**：RSA ≥ 2048 位元（長效期金鑰建議 3072–4096 位元）；EC 建議 P-256 或 P-384。
- **憑證**：務必包含正確的 SAN 清單；在自動化允許的前提下盡量縮短有效期（Let's Encrypt 預設的 90 天效期，正是強迫自動化續期，而這本身就是一種韌性優勢）。
- **HSTS**（`Strict-Transport-Security` header）：告訴瀏覽器此後永遠不要再對這個主機嘗試明文 HTTP，藉此在後續造訪時關閉降級攻擊／SSL 剝除攻擊的可乘之機。
- **OCSP stapling**：伺服器自行取得撤銷證明並附加到握手中 — 比 client 直接查詢 CA 更快，也更能保護隱私。
- **自動化續期**：憑證過期是最常見的自我造成服務中斷的原因之一；採用以 ACME 為基礎的續期機制（certbot 等）可省去人工介入的步驟。
- **保護私鑰**：設定嚴格的檔案權限，避免透過任何未加密通道傳輸金鑰（包括內部通道），一旦懷疑外洩就立刻輪換。
- **測試實際部署的設定，而非只測試預期設定** — 見 [§18](#cn-18)。

---

<a id="cn-18"></a>
## 18) 測試與診斷工具

```bash
# Full handshake + certificate detail against a live host
openssl s_client -connect host:443 -servername host

# Confirm a legacy version is actually rejected (should fail to connect)
openssl s_client -connect host:443 -tls1_1

# Confirm negotiated protocol/version via curl
curl -v --http1.0 https://host/
curl -v --http1.1 https://host/
curl -v --http2   https://host/
curl -v --http3   https://host/   # requires a curl build with HTTP/3 support
```

輸出中要留意的重點：

- `Protocol` / `New, TLSv1.x` 那一行 — 確認實際協商出來的 TLS 版本
- `ALPN, server accepted to use h2` — 確認選定的是 HTTP/2
- 回應狀態列（`HTTP/1.1 ...` 或 `HTTP/2 ...`）也能從 HTTP 這一側再確認一次

其他值得認識的工具（僅提及讓你知道有這些選項，本文不深入介紹）：

- **testssl.sh** — 以腳本自動掃描伺服器的協定／cipher／弱點狀況，相當全面
- **nmap 的 `ssl-enum-ciphers` 腳本** — 列舉每個 TLS 版本各自支援哪些 cipher
- **Qualys SSL Labs 的 SSL Test** — 線上代管的掃描工具，會針對任何公開伺服器產出等第評分報告
- 瀏覽器開發者工具 → Security 分頁 — 顯示目前頁面實際協商出的協定／cipher

---

<a id="cn-19"></a>
## 19) 透過 Redfish 管理 PEM 憑證

以上所有內容現在套用到一個真實系統上：[openbmc/bmcweb](https://github.com/openbmc/bmcweb)，OpenBMC 系 BMC firmware 所使用的 HTTP 伺服器。本節說明面向 Redfish 的憑證管理 API；[§20](#cn-20) 則說明底層的 C++ 實作。

這對應到 bmcweb 的 `redfish-core/lib/certificate_service.hpp`：

- HTTPS 憑證集合：`/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/`
- 單張 HTTPS 憑證：`/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/{id}`
- 通用替換動作：`/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/`

bmcweb 內部對應的後端常數：

- D-Bus service：`xyz.openbmc_project.Certs.Manager.Server.Https`
- D-Bus object base path：`/xyz/openbmc_project/certs/server/https`

### 1) Redfish 操作路徑（建議做法）

1. 查看目前的 HTTPS 憑證清單：

```bash
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

2. 上傳一張新憑證到 HTTPS collection（POST） — `CertificateString` 是 PEM 內容，見 [§10](#cn-10)：

```bash
curl -k -u root:0penBmc \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }' \
  https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

3. 或是使用 `ReplaceCertificate` action，指定要替換某張既有憑證：

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

注意：bmcweb 在這個 action 中只接受 `CertificateType = PEM`。

### 2) 路徑對應關係（檔案／物件視角）

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

### 3) 需要重新啟動服務嗎？

- 多數情況下，透過憑證管理 service 安裝／替換後會自動生效。
- 如果你仍然看到舊憑證，可以在 BMC shell 上重新啟動 HTTPS 服務：

```bash
systemctl restart bmcweb.service
```

- 驗證新憑證是否已經生效 — 這是 [§18](#cn-18) 那個 `openssl s_client` 手法，套用在 BMC 本身上：

```bash
openssl s_client -connect <bmc>:443 -servername <bmc> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

<a id="cn-20"></a>
## 20) bmcweb 原始碼走讀

原始碼：[github.com/openbmc/bmcweb](https://github.com/openbmc/bmcweb)（`main` 分支）。以下為程式碼庫中，將網路連線、TLS 握手、HTTP 路由與業務邏輯以 C++ 搭配 **Boost.Asio / Boost.Beast** 實作說明的完整走讀。

---

### A. TLS 實作：連線層 vs PEM 載入 vs 業務 Router

`http/http_connection.hpp` **不會**直接解析 PEM 憑證檔 — 它僅處理連線層級的 TLS 流程（偵測是否為 SSL、握手、ALPN 路由）。PEM 載入是在 SSL Context 初始化階段進行，位於 `src/ssl_key_handler.cpp`。

在架構設計上，**業務 Handler（Business Handler）與連線層（Connection Layer）已完全解耦**。完整的分層設計如下：

```text
[ Socket 物理層 ] (TCP / TLS Socket)
       ↓
[ 連線與解析層 ] http/http_connection.hpp (Connection::handle)
       ↓
[ 路由分派層   ] http/routing.hpp (Router::handle)
       ↓
[ 業務邏輯層   ] redfish-core/lib/*.hpp (handleXxxGet / handleXxxPost)
       ↓
[ 系統服務層   ] D-Bus Call / DB / Custom Logic

```

### B. 端到端（End-to-End）完整請求生命週期

一個 HTTP/HTTPS 請求從「TCP/TLS Socket 建立」**到**「Response 寫回 Socket」的完整流程如下：

```text
[SOCKET START]
  1. Boost.Asio Server Acceptor (http/http_server.hpp: doAccept)
     ↓
  2. TLS 握手與資料讀取 (http/http_connection.hpp: start -> async_handshake -> doRead)
     ↓
  3. HTTP Request 解析完成 (http/http_connection.hpp: afterReadHeaders -> handle)
     ↓
  4. 路由匹配與分派 (http/routing.hpp: Router::handle -> rule.handle)
     ↓
  5. 執行 Redfish 業務邏輯 (redfish-core/lib/*.hpp: handleXxx -> 非同步 D-Bus Async I/O)
     ↓
[BUSINESS LOGIC COMPLETED]
  6. AsyncResp 引用計數歸零解構 (include/async_resp.hpp: ~AsyncResp -> res.end)
     ↓
  7. 觸發請求完成 Callback (http/http_connection.hpp: completeRequest)
     ↓
  8. 物理封包寫回 Socket (http/http_connection.hpp: doWrite -> boost::beast::http::async_write)
[SOCKET END]

```

### C. 核心原始碼逐段剖析

#### 【階段 1】建立 Listen 與 Accept 新連線

* **檔案位置：** `http/http_server.hpp`
* **說明：** bmcweb 啟動時會建立 `boost::asio::ip::tcp::acceptor`。當收到新 TCP 連線時，會實例化 `Connection` 物件並呼叫 `start()`。

```cpp
// http/http_server.hpp
void doAccept()
{
    acceptor->async_accept(
        *httpStream, [this, httpStream](const boost::system::error_code& ec) {
            if (!ec)
            {
                // 建立 Connection 物件實例並啟動連線處理
                std::make_shared<Connection<Adaptor, Handler>>(
                    handler, std::move(*httpStream), sslContext)
                    ->start();
            }
            doAccept();
        });
}
```

#### 【階段 2】TLS Handshake 與位元流讀取

* **檔案位置：** `http/http_connection.hpp`
* **說明：** 若啟用 TLS，連線啟動時會透過 Boost.Asio 執行 `async_handshake`。握手成功後進入 `doRead()` 循環讀取 Socket 位元組，直到 HTTP Header 讀取完畢後觸發 `afterReadHeaders()`。

```cpp
// http/http_connection.hpp
void start()
{
    if constexpr (std::is_same_v<Adaptor, boost::asio::ssl::stream<boost::asio::ip::tcp::socket>>)
    {
        // 執行 TLS 握手
        adaptor.async_handshake(
            boost::asio::ssl::stream_base::server,
            [self(shared_from_this())](const boost::system::error_code& ec) {
                if (!ec) { self->doRead(); }
            });
    }
    else 
    { 
        doRead(); 
    }
}

void doRead()
{
    // 透過 Boost.Beast 讀取 HTTP Request Header
    boost::beast::http::async_read_header(
        adaptor, buffer, *parser,
        [self(shared_from_this())](const boost::system::error_code& ec, std::size_t bytesTransferred) {
            self->afterReadHeaders(self, ec, bytesTransferred);
        });
}

void afterReadHeaders(const std::shared_ptr<self_type>& /*self*/,
                      const boost::system::error_code& ec,
                      std::size_t bytesTransferred)
{
    /* 身份驗證、 Header 格式檢查等 */
    if (parser->is_done())
    {
        handle();  // 標頭解析完成，進入業務邏輯分派
        return;
    }
    doRead();
}
```

#### 【階段 3】連線層 → 路由層交接

* **檔案位置：** `http/http_connection.hpp`
* **說明：** 建立 `AsyncResp` 傳送物件，並將 `completeRequest` 註冊為 Response 完成時的回呼函式，最後將 Request 丟給 Router 處理。

```cpp
// http/http_connection.hpp
void handle()
{
    auto asyncResp = std::make_shared<bmcweb::AsyncResp>();
    asyncResp->res.setCompleteRequestHandler(
        [self(shared_from_this())](Response& thisRes) {
            self->completeRequest(thisRes);  // 當 Response 完成時回傳連線層
        });
    
    if (doUpgrade(asyncResp))  // WebSocket / HTTP2 Upgrade 檢查
    {
        return;
    }
    
    handler->handle(req, asyncResp);  // 傳遞給 Router 進行匹配
}
```

#### 【階段 4】路由分派 (Router)

* **檔案位置：** `http/routing.hpp`
* **說明：** `Router::handle()` 根據 URL Path 與 HTTP Method 尋找匹配的 Rule，並轉發給對應的業務 Handler。

```cpp
// http/routing.hpp
void handle(const std::shared_ptr<Request>& req,
            const std::shared_ptr<bmcweb::AsyncResp>& asyncResp)
{
    FindRouteResponse foundRoute = findRoute(*req);
    
    if (foundRoute.route.rule == nullptr)
    {
        // 404 Not Found 或 405 Method Not Allowed 處理
        asyncResp->res.result(boost::beast::http::status::not_found);
        return;
    }
    
    BaseRule& rule = *foundRoute.route.rule;
    std::vector<std::string> params = std::move(foundRoute.route.params);
    
    BMCWEB_LOG_DEBUG("Matched rule '{}' {} / {}", rule.rule,
                     req->methodString(), rule.getMethods());
    
    rule.handle(*req, asyncResp, params);  // 呼叫具體的業務 Handler
}
```

#### 【階段 5】執行業務 Handler 與 RAII 機制

* **檔案位置：** `redfish-core/lib/service_root.hpp` 與 `include/async_resp.hpp`
* **說明：** 業務邏輯層（如 Redfish API）進行 D-Bus 呼叫並填寫 JSON 回應。`AsyncResp` 採用 **RAII 技術**，當非同步呼叫全部結束、`AsyncResp` 引用計數歸零解構時，會觸發 `res.end()`。

```cpp
// redfish-core/lib/service_root.hpp
inline void handleServiceRootGet(
    App& app, const crow::Request& req,
    const std::shared_ptr<bmcweb::AsyncResp>& asyncResp)
{
    if (!redfish::setUpRedfishRoute(app, req, asyncResp))
    {
        return;
    }
    
    // 填寫 JSON 回應資料（亦可能在此發起非同步 D-Bus 請求）
    asyncResp->res.jsonValue["@odata.type"] = "#ServiceRoot.v1_13_0.ServiceRoot";
    
    // 當函式結束且非同步 D-Bus Callback 皆完成時，shared_ptr<AsyncResp> 解構
}

// include/async_resp.hpp
struct AsyncResp
{
    crow::Response res;
    ~AsyncResp()
    {
        // 當 AsyncResp 引用計數歸零，解構子自動觸發 res.end()
        // res.end() 會呼叫先前設定好的 completeRequestHandler (即 completeRequest)
        res.end();
    }
};
```

---

#### 【階段 6】標頭補全 (Complete Request)

* **檔案位置：** `http/http_connection.hpp`
* **說明：** 當 `res.end()` 被觸發後，流程回到連線層的 `completeRequest()`，為 Response 補上 Security Headers 與 HTTP 狀態。

```cpp
// http/http_connection.hpp
void completeRequest(Response& thisRes)
{
    // 補齊 HTTP 安全標頭 (Security Headers) 與 Content-Type 等
    addSecurityHeaders(*req, thisRes);
    
    // 設定 Keep-Alive 狀態
    res.keepAlive(req->keepAlive());
    
    // 進入物理封包發送階段
    doWrite();
}

```

---

#### 【階段 7】將 Response 寫回 Socket (True End)

* **檔案位置：** `http/http_connection.hpp`
* **說明：** 透過 Boost.Beast 的 `async_write` 將完裝好的 HTTP Response 透過 TLS/TCP Stream 寫回 Client。若連線為 Keep-Alive 則清空 Parser 並重置連線準備讀取下一筆 Request，否則調用 `close()` 關閉 Socket。

```cpp
// http/http_connection.hpp
void doWrite()
{
    // 透過 Boost.Beast 將 HTTP Response 寫回 Socket/TLS Adaptor
    boost::beast::http::async_write(
        adaptor, res.stringResponse().value(),
        [self(shared_from_this())](const boost::system::error_code& ec, std::size_t bytesTransferred) {
            if (ec) { return; }
            
            // 若為 Keep-Alive 連線，重置 Parser 並繼續執行 doRead() 監聽下一次請求
            if (self->res.keepAlive())
            {
                self->parser.emplace();
                self->doRead();
            }
            else
            {
                self->close(); // 否則主動關閉 Socket 連線
            }
        });
}
```

以下是連線層和 PEM 的具體實作位置。

#### A-1) 連線層：TLS 偵測 + 握手

原始碼：`http/http_connection.hpp`

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

這正是 [§13](#cn-13) 那張「先 TCP 再 TLS」圖示的實際落地版本：新連線進來會先跑 `async_detect_ssl` 來分辨是 TLS 還是明文，如果是 TLS，就執行 [§9](#cn-9) 中觀念性介紹過的非阻塞式握手。

#### A-2) PEM 載入：建立 SSL Context

原始碼：`http/http_server.hpp`

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

原始碼：`src/ssl_key_handler.cpp`

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

- bmcweb 在伺服器啟動時透過 `loadCertificate()` 載入一次憑證 context
- `getSslServerContext()` 會準備並驗證 `/etc/ssl/certs/https/server.pem` — 注意這是單一合併的 PEM 檔，同時包含憑證與金鑰，正是 [§10](#cn-10) 提到的格式之一
- PEM 資料透過 `use_certificate_chain` 與 `use_private_key(..., pem)` 載入 — 這就是命令列的 `openssl x509`／`openssl pkey` 檢視方式，對應到 C++/OpenSSL API 上的版本

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

### B. ALPN：選定 HTTP/2

原始碼：`src/ssl_key_handler.cpp`

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

這是 [§7](#cn-7) 提到的 ALPN 協商在伺服器端的實作 — 如果 client 有提供 `h2`，nghttp2 就會選它。

### C. 握手之後：若 ALPN 選中就導向 HTTP/2

原始碼：`http/http_connection.hpp`

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

這就是實際的分流點：`h2` 會導向 `HTTP2Connection`；其他情況（`http/1.1`，或根本沒有 ALPN）則會繼續走 HTTP/1.x 的 header parser。

### D. HTTP/1.x 處理：版本檢查與 Keep-Alive

原始碼：`http/http_connection.hpp`

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

bmcweb 並沒有把 HTTP/1.0 與 HTTP/1.1 拆成兩套獨立的 handler — 它讀取 `req->version()` 與 `req->keepAlive()`，並依賴 Boost.Beast 本身的語意處理，因此 [§12](#cn-12) 提到的這兩個版本都是走同一套程式碼路徑；連線是否維持則完全依請求本身傳達的語意而定。

### E. h2c 支援：把明文 HTTP/1.1 升級成 HTTP/2

原始碼：`http/http_connection.hpp`

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

```cpp
if (res.result() == boost::beast::http::status::switching_protocols)
{
  upgradeToHttp2();
  return;
}
```

明文 HTTP 也能透過 `Upgrade` header 機制切換到 `h2c` — 伺服器回覆 `101 Switching Protocols`，接著切換進入 `HTTP2Connection`。（如同 [§12](#cn-12) 提到的，沒有任何主流瀏覽器真的會在明文連線上這麼做，但這條程式碼路徑是為非瀏覽器 client 準備的。）

### F. HTTP/2 內部運作：從 Frame Callback 到回應

原始碼：`http/http2_connection.hpp`

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

nghttp2 接收 `HEADERS`／`DATA` frame，並在遇到 `END_STREAM` 時視為請求已接收完成；既有的應用層 handler 會產生回應，再由 nghttp2 重新編碼回 HTTP/2 frame — 這就是 [§12](#cn-12) 所描述的二進位分幀／多工，在實際程式碼中的具體版本。

### G. 流程圖

**HTTPS + ALPN 路由：**

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

**HTTP/1.1 h2c 升級路徑：**

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

**HTTP/2 請求生命週期：**

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

### H. 快速測試指令

```bash
# Test HTTP/1.0
curl -v --http1.0 https://<bmc-host>/redfish/v1

# Test HTTP/1.1
curl -v --http1.1 https://<bmc-host>/redfish/v1

# Test HTTP/2 (TLS ALPN)
curl -v --http2 https://<bmc-host>/redfish/v1
```

要檢查的重點（與 [§18](#cn-18) 相同的訊號）：

- `ALPN, server accepted to use h2`（HTTP/2）
- 回應起始行顯示 `HTTP/1.1 ...` 或 `HTTP/2 ...`

---

<a id="cn-a"></a>
## A) 詞彙表

| Term | Meaning |
|---|---|
| **TLS** | Transport Layer Security —— 現行的協定；見 [§5](#cn-5) |
| **SSL** | Secure Sockets Layer —— TLS 已淘汰的前身，這個名稱至今仍被口語沿用 |
| **PKI** | Public Key Infrastructure（公開金鑰基礎建設） —— 由 CA、憑證與信任鏈組成的體系 |
| **CA** | Certificate Authority（憑證授權機構） —— 簽署憑證、為身分背書的單位 |
| **Root CA** | 憑證為自簽、並預先安裝在信任庫中的 CA |
| **Intermediate CA** | 憑證由 root 簽署、用於日常簽署工作的 CA |
| **CSR** | Certificate Signing Request —— 送給 CA 以取得已簽署憑證的申請 |
| **SAN** | Subject Alternative Name —— 憑證實際生效的主機名稱清單 |
| **CN** | Common Name —— 舊式的身分欄位，現代 client 已不再信任（改用 SAN） |
| **PEM** | 以 Base64 文字封裝憑證／金鑰的容器格式，以 `-----BEGIN/END-----` 分隔 |
| **DER** | PEM 用文字包裝的同一種憑證／金鑰結構，其二進位編碼版本 |
| **Cipher suite** | 一次 session 中協商出的金鑰交換、身分驗證、加密與雜湊演算法組合 |
| **AEAD** | Authenticated Encryption with Associated Data —— 同時提供機密性＋完整性的加密模式（例如 AES-GCM） |
| **ECDHE** | Elliptic-Curve Diffie-Hellman, Ephemeral —— 一種提供前向保密的金鑰交換方式 |
| **Forward Secrecy** | 長期金鑰外流也無法用來解密過去已記錄連線的特性 |
| **ALPN** | Application-Layer Protocol Negotiation —— 在握手過程中選定 HTTP/1.1、h2 或 h3 |
| **SNI** | Server Name Indication —— 握手初期以明文送出的主機名稱，讓伺服器能選出對應憑證 |
| **ECH** | Encrypted Client Hello —— 把 SNI 也一併加密的較新擴充功能 |
| **HSTS** | 強制瀏覽器此後只用 HTTPS 存取某主機的 HTTP header |
| **OCSP** | Online Certificate Status Protocol —— 即時查詢憑證撤銷狀態 |
| **OCSP stapling** | 由伺服器端自行取得 OCSP 證明並附加上去，避免 client 多一次往返 |
| **CRL** | Certificate Revocation List —— 另一種以清單為基礎的撤銷機制 |
| **mTLS** | Mutual TLS —— client 與 server 互相出示憑證、驗證彼此身分 |
| **0-RTT** | 在 TLS 1.3 恢復握手完成之前，隨著它一起送出的「零往返」資料 |
| **MITM** | Man-in-the-middle（中間人攻擊） —— 攻擊者在網路路徑上，位於 client 與 server 之間 |

---

<a id="cn-b"></a>
## B) 指令速查表

```bash
# --- Generate ---
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key
openssl req -x509 -newkey rsa:2048 -nodes -keyout server.key -out server.crt \
  -days 365 -subj "/CN=example.com" -addext "subjectAltName=DNS:example.com"

# --- Inspect ---
openssl x509 -in server.crt -noout -text
openssl x509 -in server.crt -noout -subject -issuer -dates

# --- Verify ---
openssl verify -CAfile ca.crt server.crt

# --- Convert ---
openssl x509 -in server.crt -outform der -out server.der
openssl pkcs12 -export -in server.crt -inkey server.key -out server.pfx

# --- Test a live server ---
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
curl -v --http2 https://host/
```

---

<a id="cn-c"></a>
## C) 延伸閱讀

- [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446) — TLS 1.3
- [RFC 8996](https://www.rfc-editor.org/rfc/rfc8996) — 淘汰 TLS 1.0 與 1.1
- [RFC 5246](https://www.rfc-editor.org/rfc/rfc5246) — TLS 1.2
- [RFC 6066](https://www.rfc-editor.org/rfc/rfc6066) — TLS 擴充功能，包括 SNI
- [RFC 7540](https://www.rfc-editor.org/rfc/rfc7540) — HTTP/2
- [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114) — HTTP/3
- [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000) — QUIC 傳輸層
- [openbmc/bmcweb](https://github.com/openbmc/bmcweb) — Part 7 引用的程式碼庫

---

## 快速結論

- HTTPS = HTTP + TLS；TLS 的全部工作就是機密性、完整性與身分驗證（[§2](#cn-2)）
- 現代 TLS 意味著只用 TLS 1.2/1.3、ECDHE 金鑰交換、AEAD 加密演算法 —— 每一個被淘汰的舊選項背後，都對應著一個有名有姓的歷史攻擊（[§16](#cn-16)）
- 一張憑證的可信度取決於它背後的信任鏈 —— 自簽憑證用在封閉系統沒問題，但不適合任何公開場合（[§8](#cn-8)）
- OpenSSL 的命令列工具只用一小組好記的指令，就涵蓋了產生、檢視、轉換與線上測試（[§11](#cn-11)、[附錄 B](#cn-b)）
- HTTP/2 與 HTTP/3 主要解決的是 HTTP/1.x 遺留下來的連線／併發瓶頸，而不是安全性問題 —— 但 HTTP/3 把 TLS 1.3 直接內建進了它的傳輸層握手中（[§12](#cn-12)、[§14](#cn-14)、[§15](#cn-15)）
- 一個後端是否「支援」以上這一切，取決於應用程式伺服器、TLS 函式庫，以及前方任何反向代理三者共同配合的結果 —— [§19](#cn-19)–[§20](#cn-20) 完整示範了一個真實實作（bmcweb）如何從頭到尾把這些串接起來
