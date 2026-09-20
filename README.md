# HTTP / HTTPS / TLS / SSL / OpenSSL Notes

A tutorial that goes from the basics to the details, explaining how "secure networking" actually works: HTTP itself, the cryptography that TLS/SSL is built on, certificates and PKI, the OpenSSL toolkit, and how all of it fits together from HTTP/1.0 through HTTP/3. Part 7 then applies everything above to a real, production C++ implementation ([openbmc/bmcweb](https://github.com/openbmc/bmcweb)).

## How to Use This Guide

- **New to HTTPS?** Read Part 1 → 2 → 4 in order to build the complete mental picture.
- **Need to generate or inspect a certificate right now?** Jump straight to Part 3.
- **Debugging or explaining a handshake?** Part 5 has step-by-step text diagrams.
- **Hardening a server / reviewing TLS configuration?** See Part 6.
- **Want to see how these concepts are implemented in real production code?** Part 7 walks through bmcweb step by step.

## Table of Contents

**Part 1 — Fundamentals**
- [1) HTTP vs HTTPS](#cn-1)
- [2) What Cryptography Solves](#cn-2)
- [3) Symmetric vs Asymmetric Cryptography](#cn-3)
- [4) Hashes, MACs, and Digital Signatures](#cn-4)

**Part 2 — TLS/SSL in Depth**
- [5) SSL and TLS: History and Versions](#cn-5)
- [6) Cipher Suites Explained](#cn-6)
- [7) SNI and ALPN in the ClientHello](#cn-7)
- [8) Certificates and PKI](#cn-8)
- [9) The TLS Handshake, Step by Step](#cn-9)

**Part 3 — Certificate Files and OpenSSL**
- [10) Certificate and Key File Formats](#cn-10)
- [11) OpenSSL Practical Command Cheat Sheet](#cn-11)

**Part 4 — HTTP Protocol Versions**
- [12) HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3](#cn-12)

**Part 5 — Handshake and Transport-Layer Evolution Diagrams**
- [13) TCP and the TLS 1.2 Handshake](#cn-13)
- [14) TLS 1.3 and QUIC / HTTP/3: Handshake and Evolution](#cn-14)

**Part 6 — Security Hardening**
- [16) Historical Attacks and Why Modern Defaults Exist](#cn-16)
- [17) Hardening Checklist](#cn-17)
- [18) Testing and Diagnostic Tools](#cn-18)
- [19) Case Study: What `curl -k` Actually Skips](#cn-19)

**Part 7 — Applied Case: OpenBMC bmcweb**
- [20) Managing PEM Certificates via Redfish](#cn-20)
- [21) bmcweb Source Code Walkthrough](#cn-21)
- [22) Case Study: How Session ID / Session Ticket Differ from Application-Layer Tokens](#cn-22)

**Appendices**
- [A) Glossary](#cn-a)
- [B) Command Quick Reference](#cn-b)
- [C) Further Reading](#cn-c)

---

<a id="cn-1"></a>
## 1) HTTP vs HTTPS

### What HTTP Is

HTTP (HyperText Transfer Protocol) is the application-layer protocol that browsers and servers use to exchange requests and responses.

- Default port `80`
- Travels in plaintext on the network — anyone along the network path can read or tamper with the contents
- No authentication mechanism — you cannot confirm that you are really talking to the server you think you are
- Vulnerable to eavesdropping, tampering, and MITM (man-in-the-middle) attacks

### What HTTPS Is

HTTPS is HTTP layered on top of TLS (Transport Layer Security — the modern name for what used to be commonly called SSL; see [§5](#cn-5)).

- Default port `443`
- **Confidentiality**: traffic is encrypted, so an eavesdropper only sees ciphertext
- **Integrity**: tampering in transit can be detected
- **Authentication**: the certificate proves that the server's identity matches what it claims

None of these are provided by HTTP itself — all of them come from the underlying TLS layer. That is why Part 2 and Part 3 exist: to explain the mechanisms HTTPS is actually built on, rather than just the surface-level fact that "it's encrypted."

### The One-Sentence Difference

- HTTP: like a postcard — anyone who handles it along the way can read the contents
- HTTPS: like a sealed, tamper-evident envelope addressed to a verified recipient

---

<a id="cn-2"></a>
## 2) What Cryptography Solves

TLS exists to provide three properties. Every mechanism in the rest of this guide — key exchange, certificates, MACs — serves one of the following:

| Property | Question it answers | How TLS provides it |
|---|---|---|
| **Confidentiality** | Can anyone else see the contents? | Symmetric encryption of the data (e.g., AES) |
| **Integrity** | Was it tampered with in transit? | AEAD ciphers / MAC — tampered ciphertext fails to decrypt/verify |
| **Authentication** | Am I talking to who I think I'm talking to? | Certificates + digital signatures, checked during the handshake |

A useful mental model: **the entire job of the handshake is to let two strangers, over a public channel, agree on a shared secret key while proving the server's identity — and without ever transmitting that key in a form an eavesdropper could reuse.** Everything in Parts 2–3 serves this one sentence.

Note what TLS does *not* do:

- It does not protect data after it has been decrypted at either end (that is the responsibility of application / OS-level security).
- It does not hide the fact that a connection took place, and it usually does not hide the destination hostname (unless ECH is used, the SNI is visible on the network — see [§7](#cn-7)).
- It does not verify that the server is *trustworthy* or *non-malicious* — it only proves that the server holds the private key corresponding to the identity claimed on its certificate.

---

<a id="cn-3"></a>
## 3) Symmetric vs Asymmetric Cryptography

### Symmetric Encryption

The same shared secret key is used for both encryption and decryption.

- Examples: AES-128/256, ChaCha20
- Fast — used for the actual bulk data transfer (entire HTTP requests / responses)
- Problem: both parties must already hold the *same* key before communicating — but how do you get that key across a network that others may be watching?

### Asymmetric (Public-Key) Encryption

Two mathematically related keys: a **public key** (freely shareable) and a **private key** (never shared).

- Examples: RSA, ECDSA/ECDHE (elliptic curve), Ed25519
- Data encrypted with the public key can only be decrypted by the matching private key (for signatures it is the reverse: sign with the private key, verify with the public key)
- Solves the key-distribution problem — no prior shared secret needs to exist between the parties
- Much slower than symmetric encryption (roughly 100–1000× slower), so it is unsuitable for bulk data transfer

### Why TLS Uses Both (Hybrid Encryption)

TLS uses asymmetric cryptography only to *establish* a shared secret (via certificates + key exchange), then switches to fast symmetric encryption for the actual session:

```text
1. Asymmetric crypto  -> authenticate the server + agree on a shared secret
2. Symmetric crypto   -> encrypt/decrypt all actual HTTP traffic with that secret
```

This is why a cipher suite name like `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` lists both the asymmetric parts (`ECDHE`, `RSA`) and the symmetric part (`AES_128_GCM`) — see [§6](#cn-6).

---

<a id="cn-4"></a>
## 4) Hashes, MACs, and Digital Signatures

### Hash Functions

A hash function takes input of any length and produces a fixed-length fingerprint (digest).

- Examples: SHA-256, SHA-384
- The same input always produces the same output; changing a single bit produces a completely different output
- One-way: the original input cannot be recovered from the digest
- Used for certificate fingerprints, key derivation, and building MACs — it is **not** encryption (there is no key, and it cannot be reversed)

### MAC / HMAC

A Message Authentication Code proves that the data has not been tampered with, *and* proves that the sender knows a shared secret key.

- HMAC = hash function + secret key (e.g., `HMAC-SHA256`)
- Modern TLS mostly uses **AEAD** ciphers (AES-GCM, ChaCha20-Poly1305), which combine encryption and integrity verification into a single operation, instead of the older approach of encrypting first and then computing a MAC as a separate step

### Digital Signatures

A signature proves that a message really came from the holder of a particular private key, and that its content has not been altered.

```text
Sign:   compute signature = Encrypt( Hash(message), private key )
      |
      v
  message, signature
      |
      v
Verify: compute Hash(message) == decrypt signature: Decrypt( signature, public key )
```

(Real algorithms such as RSA-PSS and ECDSA do not literally "encrypt the hash," but this sketch conveys the idea.)

This is exactly how a Certificate Authority (CA) signs a certificate: it hashes the certificate contents, then signs that hash with the CA's private key. Anyone holding the CA's public key can verify whether the certificate has been forged or tampered with — this is the mechanism underlying the entire chain of trust in [§8](#cn-8).

---

<a id="cn-5"></a>
## 5) SSL and TLS: History and Versions

"SSL" and "TLS" are often used interchangeably, but SSL is actually the deprecated predecessor:

| Version | Year | Status |
|---|---|---|
| SSL 1.0 | — | Never publicly released (fatal flaws) |
| SSL 2.0 | 1995 | Prohibited (RFC 6176, 2011) |
| SSL 3.0 | 1996 | Deprecated (RFC 7568, 2015) — broken by POODLE |
| TLS 1.0 | 1999 | Deprecated (RFC 8996, 2021) |
| TLS 1.1 | 2006 | Deprecated (RFC 8996, 2021) |
| TLS 1.2 | 2008 | Still widely used; secure as long as it is configured correctly |
| TLS 1.3 | 2018 | Current standard (RFC 8446); leaner and faster |

Practical takeaways:

- If a product or vendor still says "SSL" today, they almost certainly mean TLS — real SSL has been unsafe to use for over a decade.
- A modern server should **support only TLS 1.2 and TLS 1.3**, and reject SSL 2.0/3.0 and TLS 1.0/1.1 outright (see [§17](#cn-17)).
- TLS 1.3 is not simply "TLS 1.2 with a bigger version number" — it removes several legacy mechanisms entirely (static RSA key exchange, CBC cipher modes, custom Diffie-Hellman groups, compression) rather than merely "discouraging" them. Fewer options means fewer chances to misconfigure.

---

<a id="cn-6"></a>
## 6) Cipher Suites Explained

A cipher suite is the combination of algorithms negotiated during the handshake. TLS 1.2 names one by concatenating its component algorithms:

```text
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 |    |     |         |         |
 |    |     |         |         +-- Hash used for the handshake's HMAC/PRF
 |    |     |         +------------ Bulk symmetric cipher (AES-128 in GCM/AEAD mode)
 |    |     +---------------------- Authentication: server proves identity via RSA signature
 |    +---------------------------- Key exchange: Ephemeral Elliptic-Curve Diffie-Hellman
 +--------------------------------- Protocol
```

- **Key exchange** (`ECDHE`, `DHE`, or the legacy static `RSA`) — determines how the shared secret is produced. `ECDHE`/`DHE` provide **Perfect Forward Secrecy (PFS)**: even if the server's long-term private key leaks later, past sessions still cannot be decrypted, because each session uses a throwaway ephemeral key that was never stored. Static `RSA` key exchange has no PFS — a single leaked key exposes every previously recorded connection — which is exactly why TLS 1.3 removed it completely.
- **Authentication** (`RSA`, `ECDSA`) — the signature algorithm used to prove that the server holds the private key matching the certificate.
- **Bulk cipher** (`AES_128_GCM`, `AES_256_GCM`, `CHACHA20_POLY1305`) — you should always choose an **AEAD** (Authenticated Encryption with Associated Data) cipher, which provides both confidentiality and integrity. Avoid CBC-mode ciphers (`AES_128_CBC`) and stream ciphers such as RC4 — both have well-known historical attacks (see [§16](#cn-16)).
- **Hash** (`SHA256`, `SHA384`) — used internally for key derivation during the handshake, not for processing the bulk data itself.

TLS 1.3 simplifies this. Key exchange is *always* (EC)DHE (so the name no longer has a field for it), and the cipher suite lists only the AEAD cipher + hash:

```text
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
```

The authentication algorithm is now negotiated separately, via the certificate type and the `signature_algorithms` extension.

Check which options your local OpenSSL supports:

```bash
openssl ciphers -v 'ALL'          # every cipher suite OpenSSL knows
openssl ciphers -v 'HIGH:!aNULL'  # a reasonably strong subset
```

---

<a id="cn-7"></a>
## 7) SNI and ALPN in the ClientHello

Before any encryption is established, the very first handshake message — `ClientHello` — already carries two important plaintext extensions.

### SNI (Server Name Indication)

The problem: one server may host many different HTTPS domains on the same IP address, each with a *different* certificate. But the server has to decide which certificate to present before the handshake has progressed far enough for it to see the HTTP-level client request (at this point the `Host` header is still encrypted).

- SNI solves this: the client includes the target hostname in plaintext in the `ClientHello`
- The server reads it and picks the matching certificate before the handshake continues
- Trade-off: anyone observing the connection (network equipment, an ISP) can see the hostname, even though everything after the handshake is encrypted
- Newer mitigation: **ECH (Encrypted Client Hello)**, a TLS 1.3 extension that encrypts the SNI as well — supported by some browsers / CDNs, but not yet universal

### ALPN (Application-Layer Protocol Negotiation)

The problem: the client and server need to agree on which application-layer protocol to use (HTTP/1.1, HTTP/2, ...) without adding extra round trips.

- The client lists its supported protocols in its `ClientHello` (e.g., `[h2, http/1.1]`)
- The server selects one and returns it in the `ServerHello`
- A single `443` connection can seamlessly serve both HTTP/1.1 and HTTP/2 clients — no separate "upgrade" step is needed

```text
ClientHello (SNI: example.com, ALPN: [h2, http/1.1])
            |
            v
ServerHello (ALPN selected: h2)
            |
            v
Use HTTP/2 frames on this connection
```

Relationship to the HTTP versions:

- HTTP/1.0 and early HTTP/1.1 deployments both predate ALPN, so they do not rely on it
- In nearly all real deployments, HTTP/2 over TLS is negotiated as `h2` via ALPN (the spec technically allows unencrypted `h2c`, but browsers only support `h2` running over TLS)
- HTTP/3 runs on top of QUIC, which embeds TLS 1.3 directly; negotiation still happens via ALPN, with the identifier `h3`

---

<a id="cn-8"></a>
## 8) Certificates and PKI

A certificate binds a public key to an identity (a hostname), and the certificate itself is signed by another party who vouches for that binding. This is Public Key Infrastructure (PKI).

### X.509 Certificate Structure

The standard certificate format is X.509. Main fields:

- Certificate Info
  - **Subject** — who this certificate represents (e.g., `CN=example.com`)
  - **Subject Alternative Name (SAN)** — the list of hostnames for which this certificate is actually valid; modern clients completely ignore the `CN` field and only check the SAN
  - **Issuer** — which CA signed this certificate
  - **Validity** — the `Not Before` / `Not After` dates
  - **Subject Public Key Info** — the public key this certificate vouches for
  - **Extensions** — `Key Usage`, `Extended Key Usage`, `Basic Constraints` (is it a CA?), `Authority Key Identifier`, OCSP/CRL locations
- **CA Signature** — the issuer's signature over all of the above

To inspect a certificate in practice:

```bash
openssl x509 -in server.crt -noout -text
```

### Chain of Trust

Certificates are validated as a chain, not individually:

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

- The **Root CA** is kept offline as much as possible — if the root private key leaks, every certificate it has ever issued becomes suspect.
- The **Intermediate CA** does the actual day-to-day signing, so if it is compromised, the blast radius is limited to certificates issued by *that intermediate*, and the intermediate can be revoked directly without touching the root.
- **Self-signed certificates** have no chain of trust — nobody but the server itself vouches for them. They are fine for local development or closed internal systems (for example, a BMC's default HTTPS certificate, see [§20](#cn-20)); clients show a trust warning because their trust store contains no path leading back to that certificate.

### How a Client Actually Validates a Certificate

1. Build a chain upward from the leaf certificate until it reaches a root the client already trusts
2. Verify every signature in the chain (each certificate really was signed by the one above it)
3. Check that the current date falls within each certificate's validity period
4. Check that the requested hostname matches one of the entries in the leaf certificate's SAN list
5. Check revocation status — CRL (Certificate Revocation List) or OCSP (Online Certificate Status Protocol); **OCSP stapling** lets the server fetch and attach this proof itself, saving the client an extra round trip and avoiding a privacy leak (otherwise the CA would learn every site you visit)
6. Check that the `Basic Constraints` / `Key Usage` extensions are consistent with each certificate's role (e.g., only certificates marked as CAs may sign other certificates)
7. Verify the Certificate against the CA Signature

```text
Sign: compute CA Signature = Encrypt( Hash(CertInfo || Server Public Key), CA Private Key )
        |
        v
  CertInfo ( with Server Public Key ), CA signature
        |
        v
Verify: compute Cert hash Hash( CertInfo || Server Public Key ) == decrypt CA signature Decrypt( CA Signature, CA Public Key )
```

If any single step fails, the connection should be rejected — not silently downgraded to an unencrypted connection.

### Ways to Obtain a Certificate

- **Self-signed**: sign your own certificate. No external trust; suitable for internal / test systems.
- **Private / internal CA**: an organization runs its own CA and installs its root into the trust store of its own devices. Common for internal infrastructure and IoT/BMC fleets.
- **Public CA**: signed by a CA that browsers / operating systems already trust (for example via **Let's Encrypt**, using the automated **ACME** protocol — free, domain-validation only, 90-day validity with automatic renewal).

### Certificate Transparency (CT)

Public CAs are required to publish every certificate they issue to public, append-only CT logs. This lets domain owners notice if a CA has mis-issued a certificate in their name, and modern browsers also require public certificates to carry valid CT proofs (SCTs) before they will be trusted.

---

<a id="cn-9"></a>
## 9) The TLS Handshake, Step by Step

Restating the goal from [§2](#cn-2): using only messages sent over a connection that is still public and unencrypted so far, verify the server's identity and derive a shared symmetric key.

### The Full TLS 1.2 Handshake

```text
====================================================================================================
[ Phase 1: Preparation and Chain-of-Trust Setup (PKI Setup) ]
====================================================================================================

  [ Root / Intermediate CA ]
    │ - CA Private Key (guarded closely by the CA; used to issue certificates)
    │ - CA Public Key  (built into the Client Trust Store / operating system)
    │
    │  (1) Server submits a CSR (contains the Server Public Key)
    │  (2) After verifying identity, the CA issues the certificate:
    └───────────────┐
                    ▼
          ┌─────────────────────────────────────────────────────────┐
          │  Server Certificate (server.crt)                        │
          ├─────────────────────────────────────────────────────────┤
          │  • Domain Name: *.example.com                           │
          │  • Server Public Key (long-term asymmetric public key)  │
          │  • CA Signature = Sign(Cert Hash, CA Private Key)       │
          └─────────────────────────────────────────────────────────┘


====================================================================================================
[ Phase 2: TLS 1.2 Handshake (Authentication and Key Exchange) ]
====================================================================================================

  Client                                                                                    Server
 (holds the CA Public Key)                            (holds the Server Private Key & Certificate)
  │                                                                                                │
  │ ─── 1. ClientHello ──────────────────────────────────────────────────────────────────────────> │
  │       (ClientRandom, Cipher Suites, SNI, ALPN, Supported Groups)                               │
  │                                                                                                │
  │ <── 2. ServerHello ─────────────────────────────────────────────────────────────────────────── │
  │       (ServerRandom, Selected Cipher Suite)                                                    │
  │                                                                                                │
  │ <── 3. Certificate ─────────────────────────────────────────────────────────────────────────── │
  │        CA Signature =                                                                          │
  │          E( Hash(CertInfo || Server Public Key), Server Private Key )                          │
  |          E: RSA-PSS, ECDSA, Static RSA should deprecated                                       |
  │       (server.crt: contains the Server Public Key and the CA Signature)                        │
  │                                                                                                │
  │ <── 4. ServerKeyExchange ───────────────────────────────────────────────────────────────────── │
  │        Server Params PubKey                                                                    │
  │        Server Params =                                                                         │
  │          E( Hash(ClientRandom || ServerRandom || Server Params PubKey),                        │
  │                                                      Server Private Key )                      │
  |          E: DHE, ECDHE                                                                         |
  │                                                                                                │
  │ <── 5. ServerHelloDone ─────────────────────────────────────────────────────────────────────── │
  │                                                                                                │
  │  [Client verification phase]                                                                   │
  │  A. Verify the Certificate using the CA Public Key                                             │
  │     → Confirms the certificate was issued by the CA, and extracts the Server Public Key        │
  │       Hash_cert = Hash( CertInfo || Server Public Key )                                        │
  │       Hash_ca = D( CA Signature, CA Public Key )                                               │
  │       Hash_cert == Hash_ca                                                                     │
  │                 └──> Verified: extract the Server Public Key from CertInfo                     │
  │                                                                                                │
  │  B. Verify the Server ECDHE signature using the Server Public Key                              │
  │     → Proves the data was not tampered with and the Server holds the matching private key      │
  │     → H(ClientRandom || ServerRandom || Server Params PubKey)                                  │
  │       == D(Server Params, Server Public Key)                                                   │
  │        └──> Verified: parameters are authentic and the Server holds the private key            │
  │                                                                                                │
  │ ─── 6. ClientKeyExchange ────────────────────────────────────────────────────────────────────> │
  │       (Client Params PubKey)                                                                   │
  │                                                                                                │
  │ ─── 7. [ChangeCipherSpec] & Finished ────────────────────────────────────────────────────────> │
  │                                                                                                │
  │ <── 8. [ChangeCipherSpec] & Finished ───────────────────────────────────────────────────────── │
  │                                                                                                │
  │  [Both sides independently derive the same symmetric key]                                      │
  │  Client: Compute(Client Params PrivKey + Server Params PubKey + Randoms)                       │
  │                                                                                                │
  │  [ Session Key / AES Key ] <─────── both match ────────                                        │
  │                                                                                                │
  │  Server: Compute(Server Params PrivKey + Client Params PubKey + Randoms)                       │
  │                                                                                                │

====================================================================================================
[ Phase 3: Application Data Transfer (Symmetric Encryption) ]
====================================================================================================

  Client                                                                                    Server
    │                                                                                              │
    │ === 9. HTTP / HTTPS data transfer (encrypted both ways with the [Session Key]) ===           │
    │                                                                                              │
```

This diagram actually breaks the handshake into three layers:

1. **PKI Setup**: first give the client the CA's trust root, so it knows "this server certificate was issued by a trusted CA"
2. **TLS Handshake**: authentication and key negotiation via `ClientHello` / `ServerHello` / `Certificate` / `ServerKeyExchange`
3. **Application Data**: once the handshake completes, both sides switch to efficient symmetric encryption and begin transmitting the actual HTTP data

What happens at each step:

1. **ClientHello** — the client proposes TLS versions, cipher suites, a random number, and assorted extensions (SNI, ALPN, `supported_groups`, etc.)
2. **ServerHello** — the server selects the version / cipher suite, sends its own random number, and answers the negotiation
3. **Certificate** — the server sends its certificate chain so the client can verify the server's identity
4. **ServerKeyExchange** — the server sends its ephemeral `(EC)DHE` public value and signs it with the private key matching that certificate; this signature binds the "ephemeral key exchange" to the "authenticated identity"
5. **Client verification** — the client first validates the certificate chain (see [§8](#cn-8)) to confirm the certificate was issued by a trusted CA, then uses the server's public key to verify the `ServerKeyExchange` signature; if that succeeds, the data was not tampered with and the server really holds the matching private key
6. **ClientKeyExchange** — the client sends its own ephemeral key share, and each side uses its own private value and the other's public value to compute the same shared secret via Diffie-Hellman
7. **Finished** — each side sends an encrypted `Finished` message, confirming that both derived the same keys and that nothing in the handshake was modified
8. **Application Data** — once the handshake is done, both sides use the negotiated AEAD algorithm (e.g., AES-GCM, ChaCha20-Poly1305) to encrypt the actual HTTP requests / responses

The core goals of these steps are:

- Authenticate the server's identity: prevent impersonation by a man-in-the-middle
- Negotiate a shared symmetric key: so later traffic can use fast, low-cost symmetric encryption
- Confirm key agreement via `Finished`: prevent "each side computing a different key" or the data being tampered with

Before the first byte of application data is sent, roughly **2 round trips** are spent.

### Abbreviated Handshake (Session Resumption)

First, it is important to distinguish two different levels of "reuse":

- **HTTP keep-alive / persistent connection**: a single TCP connection is used to carry multiple HTTP requests / responses. This is connection reuse at the HTTP layer and does not count toward the TLS handshake's RTT cost.
- **TLS Session Resumption**: the same client and server reuse previously established TLS session state to shorten the round trips of the next handshake. It happens at the TLS layer and is a handshake optimization — it is not an HTTP Header / Cookie mechanism.

The key difference between the two:

- **Keep-Alive does not affect RTT**: it does not "save some handshake rounds"; it "saves the cost of re-establishing TCP connections and repeatedly building new request paths." In other words, a persistent connection lets one path carry many requests; it does not change the latency of the TLS handshake.
- **Session Resumption does affect RTT**: it directly reduces the round trips needed to re-establish TLS "certificate verification + key exchange," usually cutting a full handshake from 2 RTT to 1 RTT, and even to TLS 1.3's 0-RTT.

Running a full handshake every time you connect to a site you just visited is wasteful. TLS caches enough state to skip most of the process:

- **Session ID** — the server keeps the session state, and the client only has to remind it of that ID
- **Session Tickets** (RFC 5077) — the server encrypts its session state into a ticket and hands it to the client; the server does not need to store anything, which scales better

This session state is typically stored in:

- **Client side**: the TLS library's session cache (e.g., inside OpenSSL / the browser / an HTTP client's memory, or in a long-lived cache file / memory)
- **Server side**: an in-memory session cache, or via session tickets so the server does not need to store much state
- **HTTP Cookie**: not a storage location for TLS session resumption; a Cookie is application-layer data belonging to HTTP and cannot directly substitute for the TLS session mechanism
- **HTTP Header**: the TLS extensions in the `ClientHello` (such as `session_id`, `session_ticket`) are sent within the TLS layer, and are not ordinary HTTP request headers

In practice, client / server code might look like this:

```c
// Example: TLS session cache on the OpenSSL / library side
SSL_CTX_set_session_cache_mode(ctx, SSL_SESS_CACHE_CLIENT | SSL_SESS_CACHE_SERVER);
SSL_CTX_set_tlsext_ticket_keys(ctx, key, sizeof(key));

// When reconnecting later, the client presents the old session in its ClientHello,
// and the server decides whether it can resume
if (SSL_session_reused(ssl) != 0) {
    printf("TLS session resumed\n");
}
```

```text
Client                                Server
  | -- ClientHello + session ticket --> |
  | <- ServerHello (resumed) ---------- |
  | <- [ChangeCipherSpec], Finished --- |
  | -- [ChangeCipherSpec], Finished --> |
  | ======== Application Data ========  |
```

This shortens the handshake to **1 RTT** and skips certificate verification and the asymmetric key exchange entirely (fresh keys are produced by reusing the secrets derived from the original session).

Note: this is a different level of optimization from HTTP keep-alive. A connection that uses keep-alive may still need to redo a handshake the next time it establishes a new TLS session; conversely, TLS session resumption does not mean HTTP requests automatically share the same TCP connection. A truly "long-lived connection" and a "reused handshake" are two different things.

TLS 1.3's handshake and its 0-RTT resumption differ enough to deserve a separate diagram — see [§14](#cn-14).

### mTLS (Mutual TLS)

Everything above authenticates the *server* to the client. Some deployment scenarios (service-to-service APIs, BMC-to-BMC management traffic, zero-trust internal networks) also need to authenticate the *client*:

- The server sends an additional `CertificateRequest` message
- The client replies with its own `Certificate`, plus a `CertificateVerify` signature proving it holds the private key matching that certificate
- At that point both sides' identities have been cryptographically verified to each other

```text
====================================================================================================
[ Phase 1: Preparation and Chain-of-Trust Setup (PKI Setup) ]
====================================================================================================

  [ Root / Intermediate CA ]
    │ - CA Private Key (guarded closely by the CA; used to issue certificates)
    │ - CA Public Key  (pre-installed by each side in the other's Trust Store)
    │
    ├─ (1) Server submits a CSR ──> Server Certificate (server.crt) issued ──> stored on the Server
    └─ (2) Client submits a CSR ──> Client Certificate (client.crt) issued ──> stored on the Client (mTLS addition)


====================================================================================================
[ Phase 2: TLS 1.2 Handshake (mTLS Mutual Authentication and Key Exchange) ]
====================================================================================================

  Client                                                                                    Server
 (holds the Server CA Public Key,                        (holds the Client CA Public Key,
  plus the Client Private Key & client.crt)              plus the Server Private Key & server.crt)
  │                                                                                                │
  │ ─── 1. ClientHello ──────────────────────────────────────────────────────────────────────────> │
  │       (ClientRandom, Cipher Suites, SNI, ALPN, Supported Groups)                               │
  │                                                                                                │
  │ <── 2. ServerHello ─────────────────────────────────────────────────────────────────────────── │
  │       (ServerRandom, Selected Cipher Suite)                                                    │
  │                                                                                                │
  │ <── 3. Certificate ─────────────────────────────────────────────────────────────────────────── │
  │       (server.crt; contains the Server Public Key and the CA Signature)                        │
  │                                                                                                │
  │ <── 4. ServerKeyExchange ───────────────────────────────────────────────────────────────────── │
  │       (Server Params PubKey + Server Params)                                                   │
  │                                                                                                │
  │ <── 4.5 CertificateRequest ─────────────────────────────────────────────────────────────────── │ <== [mTLS addition]
  │       (Server asks the Client for a certificate; may attach a list of trusted CAs)             │
  │                                                                                                │
  │ <── 5. ServerHelloDone ─────────────────────────────────────────────────────────────────────── │
  │                                                                                                │
  │  [Client verifies Server phase]                                                                │
  │  A. Verify server.crt using the CA Public Key (extract the Server Public Key)                  │
  │  B. Verify the Server Params using the Server Public Key                                       │
  │     (confirms Server identity and parameters)                                                  │
  │                                                                                                │
  │ ─── 5.5 Certificate ─────────────────────────────────────────────────────────────────────────> │ <== [mTLS addition]
  │       (client.crt; contains the Client Public Key and the CA Signature)                        │
  │                                                                                                │
  │ ─── 6. ClientKeyExchange ────────────────────────────────────────────────────────────────────> │
  │       (Client ECDHE PubKey)                                                                    │
  │                                                                                                │
  │ ─── 6.5 CertificateVerify ───────────────────────────────────────────────────────────────────> │ <== [mTLS addition]
  │       Signature = E( Hash(all prior Handshake messages), Client Private Key )                  │
  │                                                                                                │
  │      [Server verifies Client phase]                                                            │ <== [mTLS addition]
  │      A. Verify client.crt using the CA Public Key → confirm it is valid                        │
  │         and extract the [Client Public Key]                                                    │
  │      B. Verify the Client signature using the Client Public Key                                │
  │         (confirms Client identity and parameters)                                              │
  │                                                                                                │
  │ ─── 7. [ChangeCipherSpec] & Finished ────────────────────────────────────────────────────────> │
  │                                                                                                │
  │ <── 8. [ChangeCipherSpec] & Finished ───────────────────────────────────────────────────────── │
  │                                                                                                │
  │  [Both sides independently derive the same symmetric key]                                      │
  │  Client: Compute(Client Params PrivKey + Server Params PubKey + Randoms)                       │
  │                                                                                                │
  │  [ Session Key / AES Key ] <─────── both match ────────                                        │
  │                                                                                                │
  │  Server: Compute(Server Params PrivKey + Client Params PubKey + Randoms)                       │
  │                                                                                                │

====================================================================================================
[ Phase 3: Application Data Transfer (Symmetric Encryption) ]
====================================================================================================

  Client                                                                                    Server
    │                                                                                              │
    │ === 9. HTTP / HTTPS data transfer (encrypted both ways with the [Session Key]) ===           │
    │                                                                                              │
```

---

<a id="cn-10"></a>
## 10) Certificate and Key File Formats

The same certificate or key can be stored in several container formats. Mixing these formats up is one of the most common practical TLS headaches.

### PEM (Privacy-Enhanced Mail)

A text format, Base64-encoded, delimited by header / footer marker lines. On Linux / OpenSSL-based systems this is by far the most common format.

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

- Mostly human-readable, diff-able, and easy to concatenate — a PEM "chain" file is simply several `BEGIN/END CERTIFICATE` blocks placed one after another
- A single `.pem` file can also bundle a certificate *and* its private key together (two blocks, one file) — bmcweb's `server.pem` does exactly this, see [§21](#cn-21)
- Common extensions: `.pem`, `.crt`, `.cer`, `.key` — an extension is only a *convention* and does not guarantee the contents; when in doubt, always inspect with `openssl x509 -text` or `openssl pkey -text`

### DER (Distinguished Encoding Rules)

This is the same underlying ASN.1 structure that PEM wraps in Base64, just in its binary encoding.

- Not human-readable, but more compact
- Common on Windows / Java systems and in some embedded environments
- Extension: `.der`, sometimes `.cer`

### PKCS#7 / P7B

A collection of certificates (a chain), but it **never contains a private key**.

- Mainly used on Windows / Java to distribute a certificate together with its intermediate chain
- Extensions: `.p7b`, `.p7c`

### PKCS#12 / PFX

A single binary, password-protected container that bundles the certificate, the private key, and optionally the certificate chain — this one file is all you need to stand up a server.

- Commonly used for importing into Windows / Java keystores or browsers
- Extensions: `.p12`, `.pfx`

### CSR (Certificate Signing Request, PKCS#10)

Not a certificate — it is a *request* for one.

- Contains your public key + identity information (CN, SAN, organization), and is signed with your own private key to prove that you really hold it
- You send this file to a CA; after verification, the CA returns a signed certificate
- Extension: `.csr`

### Quick Reference Table

| Extension | Typical contents | Format |
|---|---|---|
| `.pem` | Certificate, key, chain, or CSR (any of these is possible) | Text / Base64 |
| `.crt` / `.cer` | Certificate | Usually PEM, occasionally DER |
| `.key` | Private key | Usually PEM |
| `.csr` | Signing request | Text / Base64 |
| `.der` | Certificate or key | Binary |
| `.p7b` / `.p7c` | Certificate chain, no key | Binary |
| `.p12` / `.pfx` | Certificate + key + chain, password-protected | Binary |

When in doubt, don't trust the extension — open the file with the commands in [§11](#cn-11).

---

<a id="cn-11"></a>
## 11) OpenSSL Practical Command Cheat Sheet

OpenSSL is both a cryptography library (`libssl` handles the TLS protocol, `libcrypto` handles the low-level algorithm primitives) and the `openssl` command-line tool built on top of it — this section covers the command-line tool.

### Generate a Private Key

```bash
# RSA (traditional, widely compatible)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key

# EC (smaller keys, faster, equally strong at much shorter key lengths)
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out server.key
```

### Generate a CSR (to Send to a CA)

```bash
openssl req -new -key server.key -out server.csr \
  -subj "/C=US/O=MyOrg/CN=example.com"
```

### Generate a Self-Signed Certificate

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

`-addext subjectAltName` matters: modern clients ignore `CN` and reject certificates that have no matching SAN entry (see [§8](#cn-8)).

### Inspect Contents

```bash
openssl x509 -in server.crt -noout -text                       # full certificate
openssl x509 -in server.crt -noout -subject -issuer -dates     # quick summary
openssl req  -in server.csr -noout -text                       # a CSR
openssl pkey -in server.key -noout -text                       # a private key (any type)
```

### Verify a Certificate Chain

```bash
openssl verify -CAfile ca.crt server.crt
# with a separate intermediate:
openssl verify -CAfile root-ca.crt -untrusted intermediate.crt server.crt
```

### Confirm a Certificate and Key Actually Match

Compare the public keys embedded in each — works for both RSA and EC:

```bash
openssl x509 -in server.crt -noout -pubkey | openssl sha256
openssl pkey  -in server.key -pubout        | openssl sha256
# identical output = they match
```

### Format Conversion

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

### Talking to a Live Server

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

### Other Useful Commands

```bash
openssl ciphers -v                 # list cipher suites this OpenSSL build supports
openssl rand -base64 32            # generate a random secret
openssl dgst -sha256 file.txt      # hash a file
openssl speed aes-128-gcm          # benchmark a cipher on this machine
```

### Common Errors

| Symptom | Likely cause |
|---|---|
| `unable to get local issuer certificate` | The server did not send the intermediate chain, or it was not supplied when calling `verify` |
| Server fails to start / "key values mismatch" | The certificate and key files do not actually match — check using the pubkey-hash comparison above |
| Browser shows a "self-signed certificate" warning | This is expected — no CA vouches for this certificate (see [§8](#cn-8)) |
| `certificate has expired` | Check the `-dates` output; remember to renew |
| Certificate is valid but a hostname-mismatch warning appears | The certificate's SAN list does not include the hostname you connected with |

---

<a id="cn-12"></a>
## 12) HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.0 (1996, RFC 1945)

- Each TCP connection handles a single request / response pair; the connection is usually closed afterward
- The `Host` header is not required — effectively one IP can cleanly serve only one website
- No header compression; every request has to repeat its full headers

Impact: a multi-resource page (HTML + CSS + JS + images) pays for a new TCP handshake (and, for HTTPS, another new TLS handshake) on almost every request.

### HTTP/1.1 (1997; revised in 2014 as RFC 7230–7235)

Most web traffic today still actually runs on this version underneath, though it has been overshadowed by the glamour of HTTP/2.

- **Persistent connections by default** — this is HTTP/1.1's **keep-alive mechanism**: one TCP connection can carry many requests, without needing a new connection for each
- **`Host` header required** — makes "name-based virtual hosting" possible (one IP serving multiple domains)
- **Chunked transfer encoding** — the response body can be streamed as it is generated, without knowing the total length up front
- **Pipelining** — technically allows sending the next request without waiting for the previous response, but it suffers from head-of-line blocking (one slow response blocks every request queued behind it), so in practice browsers almost never use it

"Persistent connection" and "keep-alive" here are really the same thing: the goal is to "reuse the same connection to handle multiple requests / responses," rather than reopening a TCP/TLS connection each time. This optimization is connection reuse at the HTTP layer and is a different concept from TLS session resumption; the former is not an RTT optimization, whereas the latter is an RTT optimization of the TLS handshake.

**Not every web server can perfectly support every optimization; it depends entirely on the server software's implementation, its version, and its backend configuration.**

The HTTP protocol itself is a "specification," like a legal document. It defines certain headers, semantics, and behaviors, but that does not mean "every server will implement it in full." Each vendor (Nginx, Apache, Node.js, or a lightweight server you wrote yourself in C++) can choose to **implement it fully**, **implement it partially**, or even **ignore certain headers entirely** when writing their code.

This also means:

- `Connection: keep-alive` may work perfectly in some server implementations
- Some servers may support only `gzip` and not `br`
- Some HTTP servers may accept `ETag` and `If-None-Match`, while others do only the most basic handling
- Some servers may implement `Accept-Encoding` but not fully implement compression, caching, and conditional requests altogether

So when learning about headers, do not mistake "allowed by the specification" for "supported by every deployment." What actually determines behavior is: **the implementation, the version, build options, reverse proxies, upstream services**, and **the libraries and runtime you use**.

Common individual HTTP/1.1 optimizations and related headers (illustrative — not all need to be enabled at the same time):

- **Keeping the connection alive / connection reuse**:
  - `Connection: keep-alive`
  - `Keep-Alive: timeout=30, max=100`
- **Content compression**:
  - `Accept-Encoding: gzip, br, deflate`
- **Cache control**:
  - `Cache-Control: public, max-age=300`
  - `ETag: "abc123"`
  - `If-None-Match: "abc123"`
- **Conditional requests**:
  - `If-Modified-Since: Tue, 19 Sep 2026 00:00:00 GMT`
- **Compression / data-format negotiation**:
  - `Accept: application/json, text/html`

These headers do not "make the TLS handshake faster"; they make HTTP more efficient on an existing connection: fewer repeated connections, less redundant transfer, and more effective caching and compression.

Concept diagram: persistent connections

```text
HTTP/1.0: every request opens a new TCP connection

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
                │ 4 separate connections
                │ + 4 TCP handshakes
                │ + 4 TLS handshakes (if HTTPS)


HTTP/1.1: one TCP connection can carry multiple requests

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
                                  │ 1 connection reused
                                  │ TCP / TLS handshake cost greatly reduced

The unique identity of the underlying connection: the 5-Tuple
To the operating system and the web server, an HTTPS connection is uniquely identified by its TCP 5-Tuple:
  Source IP address
  Source Port — randomly assigned by the client
  Destination IP address
  Destination Port — usually 443
  Transport protocol (TCP or UDP)

As long as this 5-tuple is the same when the client initiates the connection, the server treats it as "the same Connection." The symmetric key (Session Key) computed after the TLS handshake completes is also bound directly to this Connection.

This is the value of a "persistent connection": **merging many short-lived connections into one long-lived connection**, so requests / responses can reuse it repeatedly and reduce the cost of handshakes and repeated connection setup.
```

### HTTP/2 (2015, RFC 7540)

- Uses **binary framing** instead of text parsing
- **Header compression (HPACK)** — headers are compressed and diffed against previous requests
- **Multiplexing** — many concurrent streams can exist on a single connection, eliminating the HTTP/1.1 workaround of "opening 6 connections per host"
- **Server Push** — the server can proactively push resources it expects the client to need; in practice this feature has been deprecated and removed from most browsers (Chrome removed it in 2022), because the real-world caching benefit was poor
- In practice it must be paired with TLS — no major browser supports cleartext HTTP/2 (`h2c`)
- Still suffers from **TCP-level** head-of-line blocking: a single lost packet stalls *all* multiplexed streams, because they share one TCP byte stream

Concept diagram: HPACK header compression

```text
HTTP/1.1: every request resends nearly identical headers

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

=> Many headers are repeated, wasting bandwidth


HTTP/2 / HPACK: first build a dynamic table, then send only the "differences"

Dynamic Table (shared by server / client)
------------------------------------------------
| Index | Field name       | Field value       |
| 1     | :method          | GET               |
| 2     | :scheme          | https             |
| 3     | :authority       | example.com       |
| 4     | user-agent       | curl/8.0          |
| 5     | accept           | */*               |
------------------------------------------------

Many of Request 2's headers are already known:

Original:
  :method = GET
  :authority = example.com
  user-agent = curl/8.0
  accept = */*

After HPACK compression, only this is sent:
  [Index 1] [Index 3] [Index 4] [Index 5]

=> Only "index references" or "increments relative to the previous value" are sent
=> This is HPACK: headers are not resent in full, but diffed against earlier data


Conceptual mental model:
- First share a dictionary of common headers
- Later requests only send "which index I'm using" or "how the new value differs from the old one"
- This greatly reduces header duplication and improves load efficiency

This is another key HTTP/2 optimization: **headers are no longer resent in full as in HTTP/1.x, but compressed via an index table and diffing.** It is especially useful when many resources download at the same time, since each request's headers contain a lot of repeated strings.
```

Concept diagram: binary framing

```text
HTTP/1.x: messages are text and must be parsed line by line

GET /index.html HTTP/1.1
Host: example.com
User-Agent: curl/8.0
Accept: */*


HTTP/2: messages are cut into binary frames, then handed to stream / priority / length management

+-----------------------------------------------------------+
| Frame Header                                              |
|  - Length                                                 |
|  - Type                                                   |
|  - Flags                                                  |
|  - Stream ID                                              |
+---------------------- +-----------------------------------+
                       |
                       v
              +------------------+
              | DATA / HEADERS   |
              |  frame payload   |
              +------------------+

        For example:
        HEADERS frame  -> the headers of this stream
        DATA frame     -> the body of this stream
        SETTINGS frame -> negotiates connection-level parameters


Benefits:
- No more relying on "\r\n" delimiters for parsing
- Frames can be efficiently multiplexed, reordered, and flow-controlled
- One stream's data does not have to be mixed into the same plaintext message as another stream's

This difference matters: **HTTP/1.x is built around "text requests / responses," whereas HTTP/2 is built around "binary frame streams."** In other words, HTTP/2 no longer describes an entire request with one long block of readable text as HTTP/1.x does; it splits it into individual frames, and streams then reassemble that data.
```

Concept diagram: multiplexing vs HOL blocking

```text
HTTP/1.1: each request has to occupy a connection of its own, more or less

Req A ──┐
Req B ──┼──> Connection 1
Req C ──┤
Req D ──┘

Req E ──┐
Req F ──┼──> Connection 2
Req G ──┤
Req H ──┘

=> Many connections are needed, and requests have to queue and wait


HTTP/2: multiple independent streams on a single TCP connection

TCP Connection
----------------------------------------------
| Stream 1 | Stream 2 | Stream 3 | Stream 4  | 
| HTML     | CSS      | JS       | IMG       | 
|  Req     |  Req     |  Req     |  Req      | 
----------------------------------------------

"Multiplexing" means:
- Multiple streams coexist within the same connection
- Not every request needs to be split onto a new connection
- One large resource does not completely block the transfer of other small resources
- After receiving the interleaved frames, the server reassembles each request directly from its Stream ID, with no queuing at all, achieving parallel bidirectional multiplexing

But note:
  TCP is still a single byte stream
  One lost packet => TCP reorders / reassembles => all streams may get stuck

     [packet loss]
           ↓
   TCP retransmission / order repair
           ↓
   all streams wait together
```

### HTTP/3 (2022, RFC 9114, based on QUIC, RFC 9000)

- Runs on top of **QUIC** (built on UDP) instead of TCP
- TLS 1.3 is built directly into the transport-layer handshake, rather than layered on top
- Every stream is independently reliable — packet loss only stalls the one stream it belongs to, fixing the head-of-line blocking problem left over from HTTP/2
- **Connection migration** — a connection can survive changes in the network environment (e.g., switching from Wi-Fi to mobile data), because the connection is identified by a Connection ID rather than an IP/port combination
- Can offer **0-RTT** reconnection to previously visited servers (with the same replay-risk considerations as TLS 1.3 0-RTT — see [§14](#cn-14))

> The "evolution" here should be viewed as a whole from 1.0 → 1.1 → 2 → 3: §12 is an overview comparing HTTP versions along with their transport / connection models; §14 and §15 then supply the further transport-layer changes of TLS 1.3 and QUIC/HTTP/3. Folding §14/§15 entirely into §12 would mix up the "HTTP protocol version comparison" with the "TLS/QUIC transport-layer improvements," so the most natural way to write it is: §12 defines the main axis, and §14/§15 are transport-layer detail extensions under that axis. In practice they are linked evolutions, not several fully independent pieces.

Concept diagram: stream independence

```text
QUIC Connection (Connection ID)
-------------------------------------------------
| Stream 1 | Stream 2 | Stream 3 | Stream 4     | 
| HTML     | CSS      | JS       | IMG          | 
| reliable | reliable | reliable | reliable     | 
-------------------------------------------------

When a packet is lost, only the stream it belongs to is affected
The other streams can keep transferring, and are not all stalled
```

Concept diagram: connection migration

```text
HTTP/2 / TCP: connection identity depends on IP + Port

Client (Wi‑Fi)      ──────── connection ────────>   Server
  IP: 10.0.0.5:52134

After switching to 4G / mobile network:
Client (4G)          ──────── connection ────────>   Server
  IP: 192.168.1.50:43122

=> These are actually two "different TCP connections"
=> The connection breaks / has to be re-established / re-handshaken


HTTP/3 / QUIC: connection identity depends on the Connection ID

Client (Wi‑Fi)               Server
   [Connection ID: CID-42]  <──────>  [Connection ID: CID-42]
         │
         ├─ When switching to 4G, the IP changes but the Connection ID does not
         │
         └─ Still treated as the same connection

=> The connection can "migrate" without rebuilding the entire session
=> Friendlier to mobile use and network-switching scenarios

This is also a key change in HTTP/3: **it moves from "identifying a connection by IP/Port" to "identifying a connection by Connection ID,"** so when moving between Wi‑Fi ↔ mobile networks, the connection can keep existing and does not necessarily fail immediately.
```

### Comparison Table

| Comparison item | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
| :--- | :--- | :--- | :--- | :--- |
| **Transport layer** | TCP | TCP | TCP | **QUIC (UDP)** |
| **Supported / mainstream TLS versions** | TLS 1.0 / 1.1 / 1.2 *(SSL 3.0 in the early days)* | TLS 1.2 / 1.3 *(can also go without TLS, over plaintext HTTP)* | **TLS 1.2 / 1.3** *(required by browsers and the spec)* | **TLS 1.3 only** *(built into the standard)* |
| **TLS integration** | Separate layer (Over TLS) | Separate layer (Over TLS) | Separate layer, negotiated via **ALPN** (`h2`) | **Natively built into** the QUIC transport layer |
| **First-connection handshake latency** *(transport + TLS)* | **3 RTT**<br>*(1 TCP + 2 TLS 1.2)* | **2~3 RTT**<br>*(1 TCP + 1~2 TLS)* | **2~3 RTT**<br>*(1 TCP + 1~2 TLS)* | **1 RTT**<br>*(QUIC transport and TLS 1.3 handshake merged)* |
| **Fast-resumption latency** *(Resumption)* | **2 RTT** *(Session ID)* | **1~2 RTT** *(Session Ticket)* | **1~2 RTT** *(Session Ticket / PSK)* | **0 RTT** *(TLS 1.3 0-RTT PSK)* |
| **Connections needed** | Many | Fewer (persistent connections) | One (multiplexed) | One (multiplexed) |
| **Header compression** | None | None | HPACK | QPACK |
| **Head-of-line blocking** | Yes (severe) | Yes | Transport layer only (TCP head-of-line blocking) | **None at all** |

Which version a given connection ends up using is decided during the TLS handshake by ALPN (or QUIC's equivalent mechanism) — see [§7](#cn-7).

---

<a id="cn-13"></a>
## 13) TCP and the TLS 1.2 Handshake

A quick reference diagram combining the two layers — the conceptual step-by-step explanation is in [§9](#cn-9).

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

Key points:

- TCP and TLS are two separate, stacked handshakes — TCP guarantees in-order delivery, and TLS adds confidentiality / integrity / authentication on top
- HTTP/1.1 and HTTP/2 usually run on exactly this same stack; ALPN during the TLS handshake decides which protocol the connection ultimately speaks

---

<a id="cn-14"></a>
## 14) TLS 1.3 and QUIC / HTTP/3: Handshake and Evolution

This section looks at TLS 1.3's handshake optimizations and QUIC / HTTP/3's transport-layer rebuild along the same evolutionary axis. Conceptually:

- **TLS 1.3** is an improvement to the "secure handshake": shorter, cleaner, with stronger protection
- **QUIC / HTTP/3** is an improvement to the "connection model": it no longer depends on traditional TCP, and instead redesigns the connection and TLS together

They are not mutually exclusive concepts; they are different facets of the same wave of network-protocol evolution: TLS 1.3 is responsible for "how to establish an encrypted connection securely," while QUIC is responsible for "how to carry multiple HTTP streams more reliably and faster at the transport layer."

### TLS 1.3: The Full 1-RTT Handshake

TLS 1.3 has the client "guess" a key-exchange group and send its key share right in the first message, instead of waiting for the server to say which group it wants — cutting the handshake from 2 round trips to 1.

```text
Client                                              Server
  | -- ClientHello + key_share ----------------------> |
  |                                                    |
  | <- ServerHello + key_share ----------------------- |
  |    [from here on, server's messages are encrypted] |
  | <- EncryptedExtensions --------------------------- |
  | <- Certificate ----------------------------------- |
  | <- CertificateVerify ----------------------------- |
  |                                                    |
  | <- Finished -------------------------------------- |
  | -- Finished -------------------------------------> |
  | ====== Application Data (both directions) ======== |
```

- As soon as the server sees the client's `key_share`, it can compute the shared secret immediately, so almost everything after the `ServerHello` — including the certificate — is already encrypted
- After sending its own `Finished`, the client can immediately send application data — the encrypted application data goes out in the **same flight** as the handshake's last message
- Static RSA key exchange no longer exists in TLS 1.3 — every handshake uses (EC)DHE, so every session has Perfect Forward Secrecy by default (see [§6](#cn-6))

### TLS 1.3: 0-RTT Resumption (and Its Trade-offs)

If the client holds a session ticket (PSK) from a previous connection to the same server, it can send encrypted application data directly in the first flight:

```text
Client                                 Server
  | -- ClientHello + PSK + early_data --> |
  | ==== 0-RTT Application Data ========> |
  | <- ServerHello + Finished ----------- |
  | <==== Application Data ===============|
```

- Zero round trips before the client's first request is *sent* — a real latency advantage for returning users
- **Caveat: 0-RTT data has no forward secrecy and can be replayed.** Nothing in that data is bound to a fresh, unpredictable value generated by the server, so a network attacker who captures a 0-RTT request can simply resend it, and the server has no built-in way to tell it apart from the original. That is exactly why 0-RTT is only recommended for idempotent operations (safe `GET` requests), and must never be used for anything with side effects (payments, form submissions).

### QUIC / HTTP/3: Redoing the Handshake and Transport in One Diagram

Moving one layer beyond TLS 1.3's "faster handshake" to look at HTTP/3's QUIC:

```text
Client                                Server
  | ---- Initial (ClientHello) -------> |
  | <- Initial (ServerHello, Cert, ...) |
  | ---- Handshake Finished ----------> |   (QUIC + TLS ready)
  | ===== HTTP/3 Request (stream) =====>|
  | <==== HTTP/3 Response (stream) =====|
```

Key points:

- QUIC merges the transport-layer handshake and the TLS 1.3 handshake into a single exchange — there are no longer two separate phases of "first TCP connect, then run the TLS handshake"
- Every HTTP/3 request / response runs on its own independently reliable QUIC stream, so a lost packet only stalls the stream it belongs to
- A QUIC connection is identified by a Connection ID, rather than the traditional four-tuple of (source IP, source port, destination IP, destination port), which is why connection migration is possible
- This is also why HTTP/3's design looks like "rebuilding the TCP + TLS stack": it changes more than HTTP; it rewrites "how connections are established, how reliable delivery works, how multiplexing works, and how a session survives network changes" all at once.

The most important idea in this section: **TLS 1.3 is an evolution in handshake speed, while QUIC / HTTP/3 is a restructuring of the entire connection model.** The two complement each other, but they are not exchanges at the same layer.

---

<a id="cn-16"></a>
## 16) Historical Attacks and Why Modern Defaults Exist

This is exactly why [§17](#cn-17) recommends "just use modern defaults" — every default today is a direct response to some real, named attack. (This is only an educational summary, meant to help you understand *why* the defaults exist — it is not attack instruction.)

| Attack | Year | Root cause | Why it is no longer a problem if you follow §17 |
|---|---|---|---|
| **BEAST** | 2011 | The IV in TLS 1.0 CBC-mode ciphers was predictable | Fixed in TLS 1.1+; avoid CBC and use AEAD |
| **CRIME / BREACH** | 2012/13 | Compression ratio leaks secret information inside a compressed stream | TLS-level compression is off by default; still be careful when compressing responses that contain secrets |
| **Heartbleed** | 2014 | A buffer over-read bug in OpenSSL's heartbeat extension implementation | A library-level bug, not a flaw in the protocol itself — it has been patched; the lesson is that after any such vulnerability becomes public, keys / certificates should be rotated, because past traffic may already have leaked |
| **POODLE** | 2014 | A padding-oracle vulnerability in SSL 3.0 CBC mode | Simply disable SSL 3.0 entirely (there is now no practical compatibility benefit to keeping it) |
| **Downgrade attacks** | Ongoing | An attacker forces the handshake to negotiate a weaker, breakable version / cipher | `TLS_FALLBACK_SCSV`, not offering old versions at all, and HSTS (not even letting the first connection try plaintext HTTP) |
| **ROBOT** | 2017 | An attack on RSA key exchange — a revival of the 1998 Bleichenbacher padding-oracle attack | Avoid static RSA key exchange — use ECDHE, which TLS 1.3 makes mandatory |

The common thread: nearly every attack here targeted either an optional legacy mechanism (SSL 3.0, CBC ciphers, static RSA key exchange) — things that are simply not offered in a fully modern configuration — or a bug in a specific implementation, not the protocol itself. That is also why TLS 1.3 was designed by deleting options outright instead of adding a switch for people to turn them off manually.

---

<a id="cn-17"></a>
## 17) Hardening Checklist

- **Protocol versions**: support only TLS 1.2 and TLS 1.3. Explicitly disable SSL 2.0/3.0 and TLS 1.0/1.1 — don't rely solely on "not offered by default," because some protocol stacks still enable them by default.
- **Cipher suites**: allow only AEAD ciphers (`*_GCM`, `*_CHACHA20_POLY1305`). Disable CBC-mode suites, RC4, 3DES, and anything with `NULL` or `EXPORT` in its name.
- **Key exchange**: prefer ECDHE over static RSA key exchange to get Perfect Forward Secrecy (achieved automatically under TLS 1.3).
- **Key size / type**: RSA ≥ 2048 bits (3072–4096 bits recommended for long-lived keys); EC: P-256 or P-384 recommended.
- **Certificates**: always include a correct SAN list; keep validity periods as short as automation allows (Let's Encrypt's default 90-day validity forces automated renewal, which is itself a resilience advantage).
- **HSTS** (`Strict-Transport-Security` header): tells browsers never again to attempt plaintext HTTP to this host, closing the window for downgrade / SSL-stripping attacks on subsequent visits.
- **OCSP stapling**: the server fetches the revocation proof itself and attaches it to the handshake — faster than clients querying the CA directly, and better for privacy.
- **Automated renewal**: certificate expiry is one of the most common causes of self-inflicted outages; using ACME-based renewal (certbot, etc.) removes the manual step.
- **Protect private keys**: set strict file permissions, avoid transferring keys over any unencrypted channel (including internal ones), and rotate immediately if a leak is suspected.
- **Test the configuration that is actually deployed, not just the one you intended** — see [§18](#cn-18).

---

<a id="cn-18"></a>
## 18) Testing and Diagnostic Tools

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

What to look for in the output:

- The `Protocol` / `New, TLSv1.x` line — confirms the TLS version actually negotiated
- `ALPN, server accepted to use h2` — confirms HTTP/2 was selected
- The response status line (`HTTP/1.1 ...` or `HTTP/2 ...`) gives a second confirmation from the HTTP side

Other tools worth knowing (mentioned only so you know the options exist; not covered in depth here):

- **testssl.sh** — a script that automatically scans a server's protocols / ciphers / vulnerability status, quite comprehensive
- **nmap's `ssl-enum-ciphers` script** — enumerates which ciphers each TLS version supports
- **Qualys SSL Labs' SSL Test** — an online hosted scanner that produces a graded report for any public server
- Browser developer tools → Security tab — shows the protocol / cipher actually negotiated for the current page

---

<a id="cn-19"></a>
## 19) Case Study: What `curl -k` Actually Skips

`-k` (long form `--insecure`) is probably the most misunderstood of the TLS-related flags. Two common misconceptions are "adding `-k` means there is no encryption" and "adding `-k` is the same as falling back to HTTP" — both are wrong. The TLS handshake still runs completely from start to finish, and traffic is still encrypted; **the only thing skipped is the single step of "verifying the certificate."**

### A Step-by-Step Breakdown in curl's Execution Order

Taking `curl -k https://example.com/` as the example, here is what actually happens, in order, from typing the command to getting the response:

**1. Parse the URL, DNS lookup** — ✅ Done as usual

Unaffected. `-k` does not get involved in name resolution at all.

**2. TCP three-way handshake** — ✅ Done as usual

Connects to port 443 (see [§13](#cn-13)), unrelated to `-k`.

**3. Send ClientHello** — ✅ Done as usual

The list of TLS versions, the list of cipher suites, the random number, SNI, ALPN, `supported_groups` — none are omitted (see [§7](#cn-7)). **The server side cannot tell at all whether the client used `-k`** — this is purely client-side behavior from start to finish.

**4. Receive ServerHello** — ✅ Done as usual

The TLS version and cipher suite are negotiated as usual; `-k` does not cause a weaker combination to be picked.

**5. Receive the Certificate message** — ✅ Done as usual

The server still sends its full certificate chain, and curl still receives it completely and parses it into an X.509 structure. `-k` does not make the server send less, nor does it stop curl from receiving.

**6. Verify the certificate** — ❌ **Skipped ← this is the only step `-k` touches**

Normally curl performs the following checks at this step, and `-k` turns off the whole set at once:

| Check | What it does |
|---|---|
| Chain validation | Verify signatures layer by layer with the public key of the layer above, all the way up to a root CA in the trust store ([§8](#cn-8)) |
| Validity period | Whether `notBefore` / `notAfter` covers the current time |
| Hostname matching | Whether the host in the URL matches the certificate's SAN ([§8](#cn-8)) |
| Usage constraints | Whether `basicConstraints`, `keyUsage`, `extendedKeyUsage` permit this use |
| Revocation status | CRL / OCSP (depending on the build and configuration) |

**7. Verify the CertificateVerify (TLS 1.3) / ServerKeyExchange (TLS 1.2) signature** — ✅ **Done as usual**

This is the point most often misunderstood. curl still uses the public key in the certificate to verify this signature, confirming that the other side **really holds the private key matching that certificate** (see [§9](#cn-9)). What is skipped is only "whether this certificate is worth trusting," not "whether the other side holds its private key."

**8. ECDHE key exchange** — ✅ Done as usual

Both sides still compute the shared secret on their own, and PFS still holds (see [§6](#cn-6)).

**9. Finished message exchange** — ✅ Done as usual

Both sides compare the hash of the handshake transcript, confirming that nothing in the handshake was tampered with.

**10. Application Data** — ✅ Done as usual

Encrypted with the negotiated AEAD algorithm; confidentiality and integrity are both intact.

### Verified in Practice

Running `curl -kv` against a server with a self-signed certificate, not a single handshake message is missing:

```text
* ALPN, offering h2
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN),  TLS handshake, Server hello (2):
* TLSv1.3 (IN),  TLS handshake, Certificate (11):      <- certificate still received
* TLSv1.3 (IN),  TLS handshake, CERT verify (15):      <- signature still verified
* TLSv1.3 (IN),  TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

More interesting still, curl actually **computes the verification result anyway — it just ignores it**:

```bash
curl -k -o /dev/null -s -w "verify_result=%{ssl_verify_result}\n" https://self-signed.host/
# verify_result=18     <- 18 = X509_V_ERR_DEPTH_ZERO_SELF_SIGNED_CERT
```

The failure code `18` is still computed and reported; `-k` merely stops curl from treating it as a reason to abort the connection.

### `-k` Actually Turns Off Two Independent Switches at Once

At the libcurl level these are two mutually independent options:

| Option | Check it controls | Effect of `-k` |
|---|---|---|
| `CURLOPT_SSL_VERIFYPEER` | Whether the certificate chain leads back to a trusted CA | Set to `0` |
| `CURLOPT_SSL_VERIFYHOST` | Whether the hostname matches the SAN | Set to `0` |

On the command line there is no way to turn off just one — `-k` is all-or-nothing. If the problem is only that the hostname does not match, the correct approach is to use `--resolve` rather than `-k` (see below).

### Threat Model: Who It Stops and Who It Doesn't

```text
Passive eavesdropper (can only record packets, cannot alter traffic)

  Client <════════════ encrypted ════════════> Server
                  ^ The attacker only sees ciphertext
  → -k still fully protects against this

Active attacker (can change routing / ARP spoofing / controls an intermediate node)

  Client <═══ encrypted ═══> [MITM] <═══ encrypted ═══> Server
                  ^ Both legs are genuine TLS
                  ^ But the session keys for both legs are in the MITM's hands, so all plaintext is visible
  → -k gives no protection whatsoever
```

All the attacker has to do is generate their own private key and sign a self-signed certificate, and `-k` will accept it without complaint (see [§8](#cn-8)). What you get is a channel that is flawlessly encrypted, yet leads straight to the attacker.

A one-line comparison of the strength of the two guarantees:

- **With `-k`**: "I have established an encrypted channel with *some* party that holds the private key for this certificate."
- **Without `-k`**: "...and that party really is `example.com`, vouched for by a CA I trust."

### Three Common Verification Failures and Their Proper Fixes

`-k` can make all of the following errors disappear, but each one actually has a more precise fix:

```bash
# Symptom 1: self-signed certificate, or signed by a private CA that isn't in the trust store
curl: (60) SSL certificate problem: self-signed certificate
# Fix: explicitly specify the CA to trust, instead of turning verification off
curl --cacert ca.crt https://host/
curl --cacert server.crt https://host/     # self-signed: trust that certificate itself directly

# Symptom 2: the CA is fine, but connecting by IP causes a hostname mismatch
curl: (60) SSL: no alternative certificate subject name matches target host name '192.168.1.100'
# Fix: keep full chain validation, and just point the name at that IP
curl --cacert ca.crt --resolve host.example:443:192.168.1.100 https://host.example/

# Symptom 3: expired certificate
curl: (60) SSL certificate problem: certificate has expired
# Fix: replace it with a new certificate. This is a real problem and should not be bypassed with a flag
```

If you can't even get hold of the CA certificate, you can fall back to public-key pinning — although it does not validate the chain of trust, it at least binds to one specific key:

```bash
# First compute the SHA-256 fingerprint of the certificate's public key
openssl x509 -in server.crt -pubkey -noout \
  | openssl pkey -pubin -outform der \
  | openssl dgst -sha256 -binary | base64

curl --pinnedpubkey "sha256//<value computed above>" https://host/
```

### When It's Acceptable and When It Isn't

| Scenario | Recommendation |
|---|---|
| Local development, with a certificate you just generated using `openssl req -x509` | ✅ Acceptable |
| Initial setup of a BMC's factory self-signed certificate on an isolated management network ([§20](#cn-20)) | ✅ Acceptable |
| Debugging, when you first need to determine "is it a certificate problem or something else" | ✅ Acceptable (use `-k` to confirm you can connect, then go back and fix verification) |
| Production automation scripts, CI/CD | ❌ Use `--cacert` instead |
| Any connection crossing a network you don't control | ❌ Equivalent to no protection at all |

This also explains why the Redfish example commands in [§20](#cn-20) all carry `-k`: a BMC ships with a self-signed certificate ([§8](#cn-8)), the client's trust store has no path back to it to begin with, and such operations usually take place on an isolated management network. Once a proper certificate issued by the enterprise's internal CA has been installed via Redfish, `-k` should be replaced with `--cacert`.

---

<a id="cn-20"></a>
## 20) Managing PEM Certificates via Redfish

Everything above now gets applied to a real system: [openbmc/bmcweb](https://github.com/openbmc/bmcweb), the HTTP server used by OpenBMC-based BMC firmware. This section covers the Redfish-facing certificate-management API; [§21](#cn-21) covers the underlying C++ implementation.

This maps to bmcweb's `redfish-core/lib/certificate_service.hpp`:

- HTTPS certificate collection: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/`
- A single HTTPS certificate: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/{id}`
- Generic replace action: `/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/`

Backend constants that bmcweb maps to internally:

- D-Bus service: `xyz.openbmc_project.Certs.Manager.Server.Https`
- D-Bus object base path: `/xyz/openbmc_project/certs/server/https`

### 1) Redfish Operation Path (Recommended Approach)

1. View the current list of HTTPS certificates:

```bash
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

2. Upload a new certificate to the HTTPS collection (POST) — `CertificateString` is the PEM content, see [§10](#cn-10):

```bash
curl -k -u root:0penBmc \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }' \
  https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

3. Or use the `ReplaceCertificate` action, specifying which existing certificate to replace:

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

Note: in this action, bmcweb only accepts `CertificateType = PEM`.

### 2) Path Mapping (File / Object View)

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

### 3) Does the Service Need a Restart?

- In most cases, once the certificate has been installed / replaced through the certificate-management service, it takes effect automatically.
- If you still see the old certificate, you can restart the HTTPS service from the BMC shell:

```bash
systemctl restart bmcweb.service
```

- To verify the new certificate has taken effect — this is the `openssl s_client` technique from [§18](#cn-18), applied to the BMC itself:

```bash
openssl s_client -connect <bmc>:443 -servername <bmc> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

<a id="cn-21"></a>
## 21) bmcweb Source Code Walkthrough

Source: [github.com/openbmc/bmcweb](https://github.com/openbmc/bmcweb) (`master` branch). Below is a complete walkthrough of how the codebase implements network connections, the TLS handshake, HTTP routing, and business logic in C++ using **Boost.Asio / Boost.Beast**.

---

### Architecture Overview

`http/http_connection.hpp` does **not** parse PEM certificate files directly — it only handles connection-level TLS flow (detecting whether the connection is SSL, the handshake, ALPN routing). PEM loading happens during SSL Context initialization at server startup, in `src/ssl_key_handler.cpp` (see Stage 0 below).

The Business Handler and the Connection Layer are two separate layers, bridged by the Router:

```text
[ Socket physical layer ]  (TCP / TLS Socket)
       ↓
[ Connection & parsing layer ]  http/http_connection.hpp (Connection::handle)
       ↓
[ Routing / dispatch layer ]    http/routing.hpp (Router::handle)
       ↓
[ Business logic layer ]        redfish-core/lib/*.hpp (handleXxxGet / handleXxxPost)
       ↓
[ System services layer ]       D-Bus Call / DB / Custom Logic
```

### Complete Request Lifecycle

The complete flow of an HTTP/HTTPS request, **from** "TCP/TLS socket established" **to** "response written back to the socket," is shown below (stage numbers correspond to the per-stage analysis further down):

```text
[Server startup, once]
  0. Load PEM certificate, build SSL Context (http_server.hpp: loadCertificate -> ssl_key_handler.cpp: getSslServerContext)

[Per connection / per request]
[SOCKET START]
  1. Boost.Asio Server Acceptor (http_server.hpp: doAcceptOne -> afterAccept -> Connection::start)
     ↓
  2. TLS detection, handshake, and ALPN negotiation (http_connection.hpp: start -> async_detect_ssl -> afterDetectSsl -> [async_handshake -> afterSslHandshake])
     │
     ├── ALPN selects h2 -> upgradeToHttp2(), then follow the "Side branch: HTTP/2" at the end
     │
     ↓ (plaintext HTTP, or TLS without h2 selected)
  3. Read HTTP headers (http_connection.hpp: doReadHeaders -> afterReadHeaders -> handle)
     ↓
  4. handle(): version check, Keep-Alive, authentication, Upgrade check (http_connection.hpp: handle -> doUpgrade)
     ↓ (ordinary request)
  5. Route matching, privilege check, and dispatch (routing.hpp: Router::handle -> validatePrivilege -> rule.handle)
     ↓
  6. Run Redfish business logic (redfish-core/lib/*.hpp: handleXxx -> asynchronous D-Bus Async I/O)
     ↓
[BUSINESS LOGIC COMPLETED]
  7. AsyncResp reference count drops to zero and is destructed (async_resp.hpp: ~AsyncResp -> res.end)
     ↓
  8. Trigger the request-complete callback, add Security Headers (http_connection.hpp: completeRequest -> completeResponseFields -> addSecurityHeaders)
     ↓
  9. Physically write the packets back to the socket (http_connection.hpp: doWrite -> boost::beast::async_write -> afterDoWrite)
     │
     └── Keep-Alive -> go back to step 3 to read the next request; otherwise gracefulClose()
[SOCKET END]
```

### Per-Stage Source Code Analysis

#### Stage 0: Server Startup — Loading the PEM Certificate

* **File location:** `http/http_server.hpp` (`loadCertificate`), `src/ssl_key_handler.cpp` (the rest)
* **Description:** When bmcweb starts (and when it receives `SIGHUP`), it calls `loadCertificate()` once to prepare and validate `/etc/ssl/certs/https/server.pem` — a single combined PEM file that contains both the certificate and the key, exactly one of the formats mentioned in [§10](#cn-10). The PEM data is loaded through `use_certificate_chain` and `use_private_key(..., pem)`, which correspond to the command-line `openssl x509` / `openssl pkey`. This step also registers the server-side ALPN callback (`alpnSelectProtoCallback`, used in Stage 2) into the SSL Context.

```cpp
// http/http_server.hpp
void loadCertificate()
{
    if constexpr (BMCWEB_INSECURE_DISABLE_SSL)
    {
        return;
    }
    adaptorCtx = ensuressl::getSslServerContext();
}
```

```cpp
// src/ssl_key_handler.cpp
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
    if constexpr (BMCWEB_HTTP2)
    {
        SSL_CTX_set_alpn_select_cb(sslCtx.native_handle(),
                                   alpnSelectProtoCallback, nullptr);
    }
    ...
}

static bool getSslContext(boost::asio::ssl::context& mSslContext,
                          const std::string& sslPemFile)
{
    ...
    boost::asio::const_buffer buf(sslPemFile.data(), sslPemFile.size());
    mSslContext.use_certificate_chain(buf, ec);
    ...
    mSslContext.use_private_key(buf, boost::asio::ssl::context::pem, ec);
    ...
}

static std::string ensureCertificate()
{
    ...
    fs::path certFile = certPath / "server.pem";  // certPath = "/etc/ssl/certs/https/"
    ...
    return ensuressl::ensureOpensslKeyPresentAndValid(sslPemFile);
}
```

---

#### Stage 1: Setting Up Listen and Accepting a New Connection

* **File location:** `http/http_server.hpp`
* **Description:** On startup, bmcweb creates a `boost::asio::ip::tcp::acceptor`. When a new TCP connection arrives, `afterAccept()` instantiates a `Connection` object and uses `boost::asio::post` to queue `start()` onto the io_context for execution.

```cpp
// http/http_server.hpp
void doAcceptOne(Acceptor& acceptor)
{
    SocketPtr socket = std::make_unique<Adaptor>(getIoContext());
    Adaptor* socketPtr = socket.get();
    acceptor.acceptor.async_accept(
        *socketPtr, std::bind_front(&self_t::afterAccept, this, &acceptor,
                                    std::move(socket), acceptor.httpType));
}

void afterAccept(Acceptor* acceptor, SocketPtr socket, HttpType httpType,
                 const boost::system::error_code& ec)
{
    if (ec) { return; }

    boost::asio::ssl::stream<Adaptor> stream(std::move(*socket), *adaptorCtx);
    auto connection = std::make_shared<Connection<Adaptor, Handler>>(
        handler, httpType, std::move(timer), getCachedDateStr,
        std::move(stream));

    boost::asio::post(getIoContext(), [connection] { connection->start(); });
    doAcceptOne(*acceptor);
}
```

---

#### Stage 2: TLS Detection, Handshake, and ALPN Negotiation

* **File location:** `http/http_connection.hpp` (the ALPN callback is in `src/ssl_key_handler.cpp`)
* **Description:** `start()` first uses `async_detect_ssl` to determine whether the connection is plaintext or TLS; only if it is TLS does it run `async_handshake`. After the handshake completes, if HTTP/2 is enabled, `afterSslHandshake()` checks the ALPN negotiation result: if `h2` was selected, it directly calls `upgradeToHttp2()` to switch to HTTP/2 (see "Side branch: HTTP/2" at the end); otherwise it calls `doReadHeaders()` and moves on to Stage 3.

```cpp
// http/http_connection.hpp
void start()
{
    readClientIp();
    /* connectionCount limit check and mTLS preparation (if enabled) omitted */
    startDeadline(DeadlineTimerType::Default);

    boost::beast::async_detect_ssl(
        adaptor.next_layer(), buffer,
        std::bind_front(&self_type::afterDetectSsl, this,
                        shared_from_this()));
}

void afterDetectSsl(const std::shared_ptr<self_type>& /*self*/,
                    boost::beast::error_code ec, bool isTls)
{
    if (ec) { return; }

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
        doReadHeaders();  // Plaintext HTTP: go straight to Stage 3
    }
}

void afterSslHandshake(const std::shared_ptr<self_type>& /*self*/,
                       const boost::system::error_code& ec,
                       size_t bytesParsed)
{
    buffer.consume(bytesParsed);
    if (ec) { return; }  // Handshake failed: doesn't close actively; the deadline timer times it out

    /* mTLS session setup (if enabled) omitted */

    if constexpr (BMCWEB_HTTP2)
    {
        const unsigned char* alpn = nullptr;
        unsigned int alpnlen = 0;
        SSL_get0_alpn_selected(adaptor.native_handle(), &alpn, &alpnlen);
        if (alpn != nullptr)
        {
            std::string_view selectedProtocol(
                std::bit_cast<const char*>(alpn), alpnlen);
            if (selectedProtocol == "h2")
            {
                upgradeToHttp2();  // -> Side branch: HTTP/2
                return;
            }
        }
    }

    doReadHeaders();  // h2 not selected: go to Stage 3
}
```

The place where the server actually "selects" the ALPN protocol is the callback that Stage 0 registered with OpenSSL:

```cpp
// src/ssl_key_handler.cpp
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

This is where the ALPN negotiation from [§7](#cn-7), the "TCP first, then TLS" diagram from [§13](#cn-13), and the non-blocking handshake from [§9](#cn-9) all land in real code: if the client offers `h2` in its `ClientHello`, nghttp2 selects it on OpenSSL's behalf; as soon as the handshake ends, `afterSslHandshake()` immediately reads out that result to decide which way to route the connection.

---

#### Stage 3: Reading the HTTP Headers

* **File location:** `http/http_connection.hpp`
* **Description:** `doReadHeaders()` reads the HTTP headers through Boost.Beast; once done, `afterReadHeaders()` checks whether the headers have been fully parsed. If so, it calls `handle()` to enter Stage 4; otherwise it calls `doRead()` (which continues reading the body via `async_read_some`).

```cpp
// http/http_connection.hpp
void doReadHeaders()
{
    boost::beast::http::async_read_header(
        adaptor, buffer, *parser,
        std::bind_front(&self_type::afterReadHeaders, this,
                        shared_from_this()));
}

void afterReadHeaders(const std::shared_ptr<self_type>& /*self*/,
                      const boost::system::error_code& ec,
                      std::size_t bytesTransferred)
{
    /* authentication, Content-Length checks, etc. omitted */
    if (parse.is_done())
    {
        handle();  // Header parsing complete, enter Stage 4
        return;
    }
    doRead();  // Body still to be read; doRead() continues reading via async_read_some
}
```

---

#### Stage 4: handle() — Version Check, Keep-Alive, Authentication, and Upgrade Check

* **File location:** `http/http_connection.hpp`
* **Description:** `handle()` first performs the HTTP/1.1 `Host` header check and reads out `keepAlive` (the HTTP/1.0 vs 1.1 difference mentioned in [§12](#cn-12) comes down to `req->version()` / `req->keepAlive()` here, rather than two separate code paths); it then performs the authentication check, creates the `AsyncResp`, and registers `completeRequest` as its completion callback; `doUpgrade()` checks whether this request wants to switch to WebSocket / SSE, and if so hands it directly to `handler->handleUpgrade()` and returns `true` (`handle()` returns right there); if none of those apply, only then is the request handed to the Router (Stage 5).

The `keepAlive` here is **HTTP-layer** long-connection management, not TLS session resumption. The actual code reads:

```cpp
keepAlive = req->keepAlive();
...
res.keepAlive(keepAlive);
```

In other words: "whether this request can keep sharing the socket" is decided by the HTTP request / response themselves, not controlled by the TLS `session_id` / `session_ticket`. `session_id` / `session_ticket` are handled by OpenSSL at the TLS handshake layer, and this can also be observed in bmcweb's code: it does not hand-implement a custom `SessionTicket` structure; it only calls `SSL_set_session_id_context(...)` in the mTLS scenario, to set a session ID context on the current connection (the OpenSSL SSL object) so as to distinguish TLS contexts used for different purposes; the actual session cache and ticket resumption are handled by OpenSSL / Boost.Asio's default mechanisms.

The key points of bmcweb's actual design at present are:

- **HTTP keep-alive**: handled explicitly between `handle()` / `completeRequest()`, using `req->keepAlive()` and `res.keepAlive(keepAlive)`
- **TLS session resumption**: bmcweb does not add custom `session_id` / `session_ticket` objects at the application layer; it relies on the underlying OpenSSL TLS session cache and the default `SSL_CTX` settings
- **mTLS session ID context**: only in `prepareMutualTls()` does it use `SSL_set_session_id_context(...)` to set a fixed marker string `"bmcweb"`, used to help the SSL session distinguish contexts; this is not an HTTP Cookie and not an HTTP header, but identification information internal to TLS

```cpp
constexpr std::string_view id = "bmcweb";
SSL_set_session_id_context(adaptor.native_handle(), idCPtr, idLen);
```

So bmcweb's actual state is: **it has two kinds of state management — "long-lived connection keep-alive" and "mTLS session context" — but no explicit application-layer session ticket implementation**; if you want to dig to the bottom of it, you have to go back to OpenSSL and its SSL_CTX / TLS session cache implementation.

```cpp
// http/http_connection.hpp
void handle()
{
    req = std::make_shared<Request>(parser->release(), reqEc);
    /* reqEc error handling omitted */

    // Check for HTTP version 1.1.
    if (req->version() == 11)
    {
        if (req->getHeaderValue(field::host).empty())
        {
            res.result(boost::beast::http::status::bad_request);
            completeRequest(res);
            return;
        }
    }

    keepAlive = req->keepAlive();

    if (authenticationEnabled)
    {
        /* not logged in and not on the allowlist -> sendUnauthorized + completeRequest + return */
    }

    auto asyncResp = std::make_shared<bmcweb::AsyncResp>();
    asyncResp->res.setCompleteRequestHandler(
        [self(shared_from_this())](Response& thisRes) {
            self->completeRequest(thisRes);
        });

    if (doUpgrade(asyncResp))  // WebSocket / SSE / h2c check
    {
        return;
    }

    handler->handle(req, asyncResp);  // Hand off to the Router, see Stage 5
}
```

`doUpgrade()` also contains an h2c branch, whose behavior deserves particular attention:

```cpp
// http/http_connection.hpp (doUpgrade excerpt)
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
// websocket and SSE are only allowed on GET
if (req->req.method() == boost::beast::http::verb::get)
{
    if (isWebsocket || isSse) { ...; return true; }
}
return false;
```

Note: the `res` here is **the Connection's own member** (not `asyncResp->res`). In other words, when `Upgrade: h2c` is detected, `doUpgrade()` merely marks the Connection's `res` as `101 Switching Protocols` and adds the corresponding headers, but the function itself still returns `false` — so the same request will still be routed normally through `handler->handle(req, asyncResp)` afterward. Only when this (Router-processed) response actually flows back into `completeRequest()` and is written to the socket does `afterDoWrite()` (see Stage 9) check whether `res.result()` is `switching_protocols`, and only then call `upgradeToHttp2()` to switch the connection. Put differently, whether the h2c switch can really succeed depends on whether the final `res` written back still retains that 101 status — this is not easy to see at a glance from the code itself, so readers who want to dig deeper are advised to verify with a packet capture. ([§12](#cn-12) also mentions that no major browser will actually attempt h2c over a plaintext connection; this path is mainly intended for non-browser clients.)

---

#### Stage 5: Route Dispatch and Privilege Check

* **File location:** `http/routing.hpp`
* **Description:** `Router::handle()` looks for a matching Rule based on the URL Path and HTTP Method; if the request already carries a session, it first calls `validatePrivilege()` to do the privilege check, and only after that passes does it forward to the corresponding business Handler.

```cpp
// http/routing.hpp
void handle(const std::shared_ptr<Request>& req,
            const std::shared_ptr<bmcweb::AsyncResp>& asyncResp)
{
    FindRouteResponse foundRoute = findRoute(*req);

    if (foundRoute.route.rule == nullptr)
    {
        // No route found: try the dedicated 404 / 405 route, and finally respond with the corresponding status code
        asyncResp->res.result(boost::beast::http::status::not_found);
        return;
    }

    BaseRule& rule = *foundRoute.route.rule;
    std::vector<std::string> params = std::move(foundRoute.route.params);

    BMCWEB_LOG_DEBUG("Matched rule '{}' {} / {}", rule.rule,
                     req->methodString(), rule.getMethods());

    if (req->session == nullptr)
    {
        rule.handle(*req, asyncResp, params);  // Call the concrete business Handler
        return;
    }
    // Logged-in session: do the privilege check first, and only call the Handler once it passes
    validatePrivilege(req, asyncResp, rule,
                      [req, asyncResp, &rule, params = std::move(params)]() {
                          rule.handle(*req, asyncResp, params);
                      });
}
```

---

#### Stage 6: Running the Business Handler and AsyncResp's RAII Mechanism

* **File location:** `redfish-core/lib/service_root.hpp` and `include/async_resp.hpp`
* **Description:** The business logic layer (e.g., the Redfish API) makes D-Bus calls and fills in the JSON response. `AsyncResp` uses the **RAII technique**: when all asynchronous calls have finished and the `AsyncResp` reference count drops to zero and it is destructed, it triggers `res.end()`.

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

    // The logic that actually fills in the JSON response data is in handleServiceRootGetImpl()
    // (it may also start asynchronous D-Bus requests here)
    handleServiceRootGetImpl(asyncResp);

    // When the function ends and all asynchronous D-Bus callbacks have completed, shared_ptr<AsyncResp> is destructed
}

// include/async_resp.hpp
class AsyncResp
{
  public:
    crow::Response res;

    ~AsyncResp()
    {
        // When the AsyncResp reference count drops to zero, the destructor automatically triggers res.end()
        // res.end() calls the previously configured completeRequestHandler (i.e., completeRequest)
        res.end();
    }
};
```

---

#### Stage 7: completeRequest — Filling in Security Headers

* **File location:** `http/http_connection.hpp` (`addSecurityHeaders` is actually defined in `include/security_headers.hpp`)
* **Description:** After `res.end()` is triggered, the flow returns to the connection layer's `completeRequest()`. It calls `completeResponseFields()` (`http/complete_response_fields.hpp`), and it is inside that function that `addSecurityHeaders(res)` is called to add the Security Headers.

```cpp
// http/http_connection.hpp
void completeRequest(Response& thisRes)
{
    res = std::move(thisRes);
    res.keepAlive(keepAlive);

    // Internally calls addSecurityHeaders(res) etc. to add the Security Headers
    completeResponseFields(accept, acceptEncoding, res);
    res.addHeader(boost::beast::http::field::date, getCachedDateStr());

    doWrite();
}
```

---

#### Stage 8: doWrite / afterDoWrite — Writing Back to the Socket

* **File location:** `http/http_connection.hpp`
* **Description:** `doWrite()` wraps `res` into a `boost::beast::http::message_generator` and writes it back to the Socket/TLS Adaptor via `boost::beast::async_write` (not `http::async_write`). Once the write completes, `afterDoWrite()` is where the next step is really decided: if the `res` just written has status `switching_protocols` (see the h2c discussion in Stage 4), it calls `upgradeToHttp2()`; if Keep-Alive applies, it resets the Parser and goes back to Stage 3 to keep reading; otherwise it closes the connection.

```cpp
// http/http_connection.hpp
void doWrite()
{
    res.preparePayload(urlView);
    boost::beast::async_write(
        adaptor,
        boost::beast::http::message_generator(std::move(res.response)),
        std::bind_front(&self_type::afterDoWrite, this, shared_from_this()));
}

void afterDoWrite(const std::shared_ptr<self_type>& /*self*/,
                  const boost::system::error_code& ec,
                  std::size_t /*bytesTransferred*/)
{
    if (ec) { return; }

    if (res.result() == boost::beast::http::status::switching_protocols)
    {
        upgradeToHttp2();  // h2c upgrade: -> Side branch: HTTP/2
        return;
    }

    if (!keepAlive)
    {
        gracefulClose();  // Not Keep-Alive: close the connection
        return;
    }

    // Keep-Alive: clear the Response / reset the Parser, go back to Stage 3 to read the next Request
    res.clear();
    initParser();
    doReadHeaders();
}
```

---

#### Side Branch: If the Connection Goes HTTP/2

Whether it is ALPN selecting `h2` in Stage 2, or a plaintext h2c upgrade in Stage 4/8, once `upgradeToHttp2()` is called, this connection is handed over from then on to `HTTP2Connection` in `http/http2_connection.hpp` — it no longer uses Boost.Beast's header/body parser, but instead uses **nghttp2** to process binary frames directly (the Binary framing / multiplexing mentioned in [§12](#cn-12), here as real code):

```cpp
// http/http2_connection.hpp
int onFrameRecvCallback(const nghttp2_frame& frame)
{
    switch (frame.hd.type)
    {
        case NGHTTP2_DATA:
        case NGHTTP2_HEADERS:
            if ((frame.hd.flags & NGHTTP2_FLAG_END_STREAM) != 0)
            {
                return onRequestRecv(frame.hd.stream_id);  // A stream's request has been fully received
            }
            break;
        default:
            break;
    }
    return 0;
}
```

`onRequestRecv()` internally likewise builds a `Request` / `AsyncResp` and hands them to the same Router (equivalent to the logic of Stages 5–6); the only difference is that in the end it does not call `doWrite()`, but instead converts the response into HTTP/2 frames and sends them:

```cpp
int rv = ngSession.submitResponse(streamId, hdr, &dataPrd);
if (rv != 0)
{
    BMCWEB_LOG_ERROR("Fatal error: {}", nghttp2_strerror(rv));
    close();
    return -1;
}
```

### Flowchart Overview

**HTTPS + ALPN routing:**

```text
TCP accept
   |
detect SSL?
   |
   +-- no  -> HTTP (plaintext) path -> doReadHeaders()
   |
   +-- yes -> TLS handshake
         |
         +-- ALPN == h2 ?
           |
           +-- yes -> upgradeToHttp2() -> HTTP2Connection
           |
           +-- no  -> doReadHeaders() -> HTTP/1.x parser
```

**HTTP/1.1 h2c upgrade path:**

```text
HTTP/1.1 request
   |
check Connection: Upgrade + Upgrade: h2c
   |
decode HTTP2-Settings, mark res = 101 Switching Protocols
   |
(still runs normal routing once first -> completeRequest replaces res with the routing result)
   |
doWrite() writes the final res back to the client
   |
afterDoWrite() checks res.result() == switching_protocols?
   |
   +-- yes -> upgradeToHttp2() -> HTTP2Connection
   +-- no  -> this connection will not switch to HTTP/2
```

**HTTP/2 request lifecycle:**

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

### Quick Test Commands

```bash
# Test HTTP/1.0
curl -v --http1.0 https://<bmc-host>/redfish/v1

# Test HTTP/1.1
curl -v --http1.1 https://<bmc-host>/redfish/v1

# Test HTTP/2 (TLS ALPN)
curl -v --http2 https://<bmc-host>/redfish/v1
```

What to check (the same signals as [§18](#cn-18)):

- `ALPN, server accepted to use h2` (HTTP/2)
- The response start line shows `HTTP/1.1 ...` or `HTTP/2 ...`

---

<a id="cn-22"></a>
## 22) Case Study: How Session ID / Session Ticket Differ from Application-Layer Tokens

The easiest thing to get confused about here is mixing up "TLS sessions" with "web application tokens." Although they are all called "session" or "token," their **layer, purpose, storage location, and lifetime** are completely different.

The conclusion first:

- **TLS Session ID / Session Ticket**: belongs to **TLS / transport-layer security**, used to reuse handshake state and reduce RTT.
- **Session Token / Session ID**: belongs to the **web application layer**, used to identify a logged-in user's session.
- **JWT / Access Token / ID Token**: belongs to **authentication and authorization**, used to express "who" and "what they may do."
- **CSRF Token**: belongs to **attack defense**, preventing the browser from issuing cross-site forged requests; it is not used to authenticate a user's identity.
- **OTT / One-Time Token**: belongs to **one-time verification**, commonly used for password resets, email verification, and two-factor authentication.
- **API Key / Bearer Token**: belongs to **machine-to-machine or API authorization**.
- **Hardware Token / Security Token**: belongs to **physical devices or hardware MFA**.

#### First, an Overview Table: Which Things Are on the Same Layer?

| Name | Layer | Main purpose | Common storage location | Typical lifetime | Affects TLS handshake RTT? |
|---|---|---|---|---|---|
| **TLS Session ID** | TLS / Transport Security | Reuse previous handshake state | server memory / TLS library cache | Short-term (cache) | **Yes** |
| **TLS Session Ticket** | TLS / Transport Security | Hand handshake state to the client; the server doesn't need to hold all the state | client-side ticket + server key | Short to medium term | **Yes** |
| **Session Token / Session ID** | HTTP Application Layer | Identify a user's login session | Cookie / server-side session store | Generally the browser session | **No** |
| **JWT (Access Token / ID Token)** | OAuth / Identity | bearer / identity / authorization assertions | client local storage / memory / HTTP Authorization header | Can be short or long, depending on configuration | **No** |
| **CSRF Token** | Web App Security | Prevent cross-site request forgery | hidden form field / cookie + header | Duration of one session | **No** |
| **OTT / One-Time Token** | Auth / Verification | Verify a one-time operation | URL, email, SMS, within an OTP | Very short (seconds / minutes) | **No** |
| **API Key / Bearer Token** | API Auth | Machine-to-machine / API authorization | header / config / secret store | Can be long-lived | **No** |
| **Hardware Token / Security Token** | MFA / PKI | Provide physical / hardware authentication | YubiKey / smart card / TPM / HSM | Long-lived but revocable | **No** |
| **OAuth 2.0 / OIDC** | Authorization Framework | Request authorization, obtain tokens | client / browser / auth server | Tokens depend on policy | **No** |

> First establish this mental model: **TLS Session ID / Ticket are mechanisms that "make the handshake faster"; Session Token / JWT / CSRF Token are mechanisms that "let the application layer know who is doing what."** They are different layers and cannot be substituted for one another.

#### 1) TLS Session ID / Session Ticket: Reusing the Handshake, Not the User's Login Identity

This is something TLS does, and it is not the same problem as login, JWT, or Cookies at all.

```text
Client                                Server
  | -- ClientHello + old session id --> |
  |                                     |
  | <----- ServerHello (resumed) ------ |
  | <----- Finished ------------------  |
  | ----- Finished -------------------> |
  | ===== Application Data ===========  |
```

- **Session ID**: the server keeps the session state (usually in a memory cache or TLS cache); the client only supplies the ID
- **Session Ticket**: the server encrypts the state into a ticket and hands it to the client; the server does not need to keep all the data, so it scales better
- This data is for the **TLS library** to use, so that both sides do not have to redo full certificate verification and key exchange

It does not represent:

- That the user is logged in
- What role the user has
- That this connection necessarily belongs to some account

All it says is:

- "I have shaken hands with this server before"
- "The previous TLS state can be reused to reduce RTT"

#### 2) Session Token (Session ID): The Application Layer's "Login Session Identifier"

This is where the login session commonly seen in web apps comes in. Typical usage:

```text
Browser                                  Server
  | -- POST /login (username, pwd) -->     |
  |                                        |
  | <----- 200 OK + Set-Cookie: SID=abc123 |
  |                                        |
  | -- GET /profile Cookie: SID=abc123 --> |
  |                                        |
  | <----- 200 OK with user profile        | 
```

- The server creates a session record: `SID -> user_id, role, expiry, ...`
- The server asks the browser to hold on to this `SID`, usually in a Cookie
- On every subsequent request the Cookie is sent along, and the server knows which user the request came from
- **This `SID` is not a TLS session ID**, and it is not a JWT; it is an **application-layer session identity**

Common storage locations:

- Cookie (most common)
- Server-side session store (Redis, Memcached, DB, in-memory)
- `HttpOnly`, `Secure`, `SameSite` may be used to restrict it

#### 3) Access Token / ID Token: JWT Is Usually One Way of Expressing Them

These tokens are carriers of identity and authorization; they are usually not used to "keep a TCP/TLS connection alive," but to "describe the user and the authorization."

```text
Client                                 Auth Server                              API Server
  | -- redirect to login /oauth/authorize ---->  |                              |
  |                                       | -- verify login and consent -->      |
  | <--- 302 redirect with code --------- |                                      |
  | -- POST /token code=... ------------> |                                      |
  |                                       | -- issue access_token + id_token --> |
  | <---- access_token, id_token -------  |                                      |
  | -- GET /api/data Authorization: Bearer <access_token> -------------------->  |
  |                                                                              | -- verify JWT signature -->
  |                                                                              | -- check scopes/claims -->
  | <----------------------- 200 OK with data ---------------------------------- |
```

##### JWT (JSON Web Token)

JWT itself is not "a protocol"; it is a common **token format**. It usually looks like this:

```text
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

A JWT consists of three parts:

- `Header`: the algorithm type (HS256 / RS256)
- `Payload`: claims, such as `sub`, `aud`, `exp`, `scope`, `iss`
- `Signature`: signed with the server's private key, so the client cannot forge it

Common uses:

- **Access Token**: given to the API server, used to confirm whether the user has permission to access a resource
- **ID Token**: given to the client / frontend, used to express "who this user is"

**Key points:**

- A JWT is not a TLS Session Ticket
- A JWT is also not a Session ID
- A JWT is also not on the same layer as a Cookie
- It is usually a "declaration of identity and authorization," not an "identifier that maintains a session"

##### CSRF Token

The purpose of a CSRF Token is not login authentication, but "preventing Cross-Site Request Forgery."

The key point is this: **the browser automatically attaches Cookies, but a malicious site's JavaScript cannot read your site's Cookies.** This is exactly why CSRF Tokens work.

```text
Suppose you are logged in to site A, and site A has stored this in a Cookie for you:
  csrftoken=abc5566

Bad Site (malicious website)
  | -- triggers form / img / fetch to A --> |
  |                                       |
  |                                       | -- Browser automatically attaches A's Cookie
  |                                       |    e.g.: Cookie: csrftoken=abc5566
  |                                       | -- But the malicious site's JS cannot read this Cookie
  |                                       |    because the same-origin policy prevents it from accessing A's data
  |                                       |
  |                                       | -- The server then checks:
  |                                       |    Cookie = abc5566
  |                                       |    Header = empty / wrong / inconsistent
  |                                       |    => judged to be CSRF, request rejected
```

This flow is the core of CSRF:

- **The Cookie is carried along automatically**: because the browser says "this request is going to A's site, so I'll attach A's Cookie"
- **The Header cannot be read directly by the malicious site**: because the Same-Origin Policy prevents JavaScript from reading other sites' Cookies and data
- **The malicious site doesn't know the real token value**: it can send the request, but it cannot put the correct `X-XSRF-TOKEN: abc5566` in the Header, because it doesn't know what is in the cookie
- **The backend comparison mismatches**: if the Cookie is `abc5566` but the Header is empty or doesn't match, the server rejects it

Actual usage typically looks like this:

```text
Browser                                 Server
  | -- GET /form -> gets CSRF token -- >   |
  | <----- form+hidden csrf_token ----     |
  | -- POST /transfer body + csrf_token -> |
  |                                        | -- validate token matches session -->
  | <----- 200 OK ---------------------    |
```

There are two common approaches:

1. **Double Submit Cookie**
   - The Cookie stores `csrftoken=abc5566`
   - The Form or Header also carries an identical value `X-XSRF-TOKEN: abc5566`
   - The server compares whether the Cookie and the Header match

2. **Server-side Session CSRF**
   - The server generates a `csrf_token` and stores it in the session
   - The client only places it in the form / header
   - The server checks whether the token in the request matches the token in the session

The point here is not "Cookies can't be used," but: **a Cookie is attached automatically, but that does not guarantee it is held by the attacker; a Header, on the other hand, has to be sent directly from JavaScript, and in a cross-site scenario that is restricted by the same-origin policy, so it can serve as the verification value.**

- A CSRF Token is usually bound to the session
- It is placed in a hidden form field, or synchronized in a header / in a cookie + header double-submit pattern
- It is not the "user identity" itself, and it is not a "JWT"

##### The Easiest Points to Get Wrong

- **A JWT is a token**: it holds claims, which may represent identity and permissions
- **A CSRF Token is a defensive token**: it is not used to authenticate identity, but to avoid forged requests
- **A Session ID is a session identifier**: used to identify a particular login session
- **A TLS Session ID is a TLS-layer identifier**: used to reuse the handshake, not for login or authorization

#### 4) One-Time Token (OTT): Use Once, Then It's Invalid

This kind of token is commonly seen in:

- Password-reset links
- Email verification
- SMS verification codes
- Confirmation of high-risk operations

```text
User                                  Server
  | -- forgot password ------------>   |
  |                                    | -- create random nonce + expiry -->
  | <----- email link with token ----  | 
  | -- click link /reset?token=abc --> |
  |                                    | -- verify token once -->
  |                                    | -- invalidate token -->
  | <----- password reset success ---  |
```

Characteristics:

- Mostly short-lived
- Can only be used once
- Often uses a nonce, a timestamp, and an HMAC signature
- Does not represent an ongoing logged-in state

#### 5) API Key / Bearer Token: The Most Common for Machine-to-Machine

```text
Client                               API Server
  | -- GET /v1/users Authorization: Bearer <token> --> |
  |                                                    | -- verify token / API key -->
  | <--------------------- 200 OK -------------------- |
```

Or:

```text
Client                         API Server
  | -- X-API-Key: abc123 ------> |
  |                              | -- check key in allowlist / db -->
  | <------ 200 OK ------------  |
```

- **API Key**: usually simpler, commonly seen in machine-to-machine settings
- **Bearer Token**: treats the token as a "bearer credential"; whoever obtains the token can use it, making it a more direct authentication mechanism
- This is very different from a browser session: it usually does not depend on Cookies, and does not require the browser to support web-attack defenses such as `SameSite`

#### 6) Hardware Token / Security Token: Bound to a Physical Device

```text
User / Browser                     Server                     Hardware Token
  | -- login request --------------> |                          |
  |                                  | -- challenge + nonce --> |
  |                                  | <--- sign challenge ---- |
  | -- response + signature ------>  |                          |
  |                                  | -- verify signature --> |
  | <------ success ---------------  |
```

- YubiKey, smart card, TPM, HSM, and the like all belong to this category
- Commonly used for MFA, U2F, FIDO2, PKI
- These tokens emphasize "you possess some hardware root"; they are not a TLS handshake session, nor an HTTP cookie

#### 7) OAuth 2.0 / OIDC: Spawns Many Token Types

OAuth 2.0 is mainly an "authorization" framework and does not directly say "who is who." OIDC (OpenID Connect) adds identity authentication on top of it.

```text
Client                              Auth Server                              Resource Server
  | -- authorize / login ----------> |                                         |
  | <----- authorization code -----  |                                         |
  | -- exchange code for token -->   |                                         |
  |                                  | -- issue access_token + refresh_token ->|
  | <--- access_token -------------  |                                         |
  | -- call API with Bearer token ---------------------------->                |
  |                                                                            | -- validate token -->
  | <---------------------------------------------------------- 200 OK |
```

Common tokens:

- **Access Token**: used to fetch data from the resource server
- **Refresh Token**: used to obtain a new access token
- **ID Token**: states "who this user is"

Notes:

- OAuth 2.0 is not a single token; it is a set of authorization flows
- The access token it produces is not necessarily a JWT; it may also be an opaque token
- The ID token it produces is often a JWT, but its purpose differs from that of the access token

#### Overall Comparison: The Positioning of Each Token

| Type | Core purpose | Typical data location | Represents login state? | Suitable for long-term persistence? | Common examples |
|---|---|---|---|---|---|
| **TLS Session ID / Ticket** | Handshake optimization, reusing encryption state | TLS library cache / client ticket | No | Short-term | `session_id`, `session ticket` |
| **Session Token / Session ID** | Identify a website login session | Cookie + server store | **Yes** | Usually for the session's duration | `SID=abc123` |
| **Access Token** | API authorization | Authorization header / JWT | Not exactly "login"; more about permissions | Usually short to medium term | `Bearer eyJ...` |
| **ID Token** | Identity assertion | JWT / frontend | **Yes** (identity information) | Depends on policy | OIDC `id_token` |
| **JWT** | Formatted expression of permissions and identity | Header / local storage / cookie | May represent identity, or may just be a token | Can be short or long | `sub`, `aud`, `exp` |
| **CSRF Token** | Defend against cross-site forged requests | hidden field / cookie + header | No | Usually one session | `csrf_token=xyz` |
| **OTT** | One-time verification | email link / OTP / SMS | No | Very short | reset-password token |
| **API Key** | machine-to-machine authorization | header / config / secret store | No | Can be long-lived | `X-API-Key` |
| **Bearer Token** | Direct bearer authentication | Authorization header | No | Depends on configuration | `Authorization: Bearer ...` |
| **Hardware Token** | Strong authentication / MFA / device-bound | YubiKey / TPM / smart card | No | Long-lived | FIDO2 / U2F |
| **OAuth 2.0 Token** | Authorization flow and access-token requests | client storage / browser | Not necessarily | Can be short-term | `code`, `access_token`, `refresh_token` |

#### One-Line Summary

- **TLS Session ID / Session Ticket**: make the handshake faster
- **Session ID**: lets the server know who this logged-in user is
- **JWT / Access Token / ID Token**: let the server know "what permissions / identity the token I received represents"
- **CSRF Token**: lets the website know "this request wasn't secretly forged by another website"
- **OTT / OTP / API Key / Hardware Token**: all specific authentication or authorization tools for different scenarios

This point is very important: **Session ID / Session Ticket are identifiers related to "transport security" or a "login session," but they are not a JWT, not an API auth token, and certainly not a CSRF protection token.** They are the same word, "token," playing different roles at different layers.

---

<a id="cn-a"></a>
## A) Glossary

| Term | Meaning |
|---|---|
| **TLS** | Transport Layer Security — the current protocol; see [§5](#cn-5) |
| **SSL** | Secure Sockets Layer — TLS's deprecated predecessor; the name is still used colloquially today |
| **PKI** | Public Key Infrastructure — the system made up of CAs, certificates, and chains of trust |
| **CA** | Certificate Authority — the entity that signs certificates and vouches for identities |
| **Root CA** | A CA whose certificate is self-signed and pre-installed in trust stores |
| **Intermediate CA** | A CA whose certificate is signed by a root and which is used for day-to-day signing work |
| **CSR** | Certificate Signing Request — a request sent to a CA to obtain a signed certificate |
| **SAN** | Subject Alternative Name — the list of hostnames a certificate is actually valid for |
| **CN** | Common Name — the legacy identity field, no longer trusted by modern clients (SAN is used instead) |
| **PEM** | A container format that wraps certificates / keys in Base64 text, delimited by `-----BEGIN/END-----` |
| **DER** | The binary encoding of the same certificate / key structure that PEM wraps in text |
| **Cipher suite** | The combination of key-exchange, authentication, encryption, and hash algorithms negotiated for a session |
| **AEAD** | Authenticated Encryption with Associated Data — an encryption mode providing both confidentiality and integrity (e.g., AES-GCM) |
| **ECDHE** | Elliptic-Curve Diffie-Hellman, Ephemeral — a key-exchange method that provides forward secrecy |
| **Forward Secrecy** | The property that a leak of long-term keys cannot be used to decrypt previously recorded connections |
| **ALPN** | Application-Layer Protocol Negotiation — selects HTTP/1.1, h2, or h3 during the handshake |
| **SNI** | Server Name Indication — the hostname sent in plaintext early in the handshake so the server can choose the right certificate |
| **ECH** | Encrypted Client Hello — a newer extension that also encrypts the SNI |
| **HSTS** | An HTTP header that forces browsers to access a host over HTTPS only from then on |
| **OCSP** | Online Certificate Status Protocol — real-time lookup of a certificate's revocation status |
| **OCSP stapling** | The server itself fetches and attaches the OCSP proof, saving the client an extra round trip |
| **CRL** | Certificate Revocation List — another, list-based revocation mechanism |
| **mTLS** | Mutual TLS — client and server present certificates and authenticate each other |
| **0-RTT** | "Zero round trip" data sent along with a TLS 1.3 resumption handshake, before it completes |
| **MITM** | Man-in-the-middle — an attacker positioned on the network path between client and server |

---

<a id="cn-b"></a>
## B) Command Quick Reference

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
## C) Further Reading

- [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446) — TLS 1.3
- [RFC 8996](https://www.rfc-editor.org/rfc/rfc8996) — Deprecating TLS 1.0 and 1.1
- [RFC 5246](https://www.rfc-editor.org/rfc/rfc5246) — TLS 1.2
- [RFC 6066](https://www.rfc-editor.org/rfc/rfc6066) — TLS extensions, including SNI
- [RFC 7540](https://www.rfc-editor.org/rfc/rfc7540) — HTTP/2
- [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114) — HTTP/3
- [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000) — QUIC transport layer
- [openbmc/bmcweb](https://github.com/openbmc/bmcweb) — the codebase referenced in Part 7

---

## Quick Takeaways

- HTTPS = HTTP + TLS; TLS's entire job is confidentiality, integrity, and authentication ([§2](#cn-2))
- Modern TLS means only TLS 1.2/1.3, ECDHE key exchange, and AEAD ciphers — behind every deprecated legacy option stands a named historical attack ([§16](#cn-16))
- A certificate is only as trustworthy as the chain of trust behind it — self-signed certificates are fine for closed systems but unsuitable for any public setting ([§8](#cn-8))
- OpenSSL's command-line tool covers generating, inspecting, converting, and live-testing with just a small set of memorable commands ([§11](#cn-11), [Appendix B](#cn-b))
- HTTP/2 and HTTP/3 mainly address the connection / concurrency bottlenecks inherited from HTTP/1.x, not security problems — but HTTP/3 builds TLS 1.3 directly into its transport-layer handshake ([§12](#cn-12), [§14](#cn-14))
- Whether a backend "supports" all of the above depends on the application server, the TLS library, and any reverse proxy in front working together — [§20](#cn-20)–[§21](#cn-21) demonstrate end to end how a real implementation (bmcweb) wires all of this together
