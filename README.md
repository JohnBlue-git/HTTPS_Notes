# HTTP / HTTPS / TLS / SSL / OpenSSL Notes

A beginner-to-advanced tutorial covering how the secure web actually works: HTTP itself, the cryptography TLS/SSL is built from, certificates and PKI, the OpenSSL toolkit, and how it all comes together across HTTP/1.0 through HTTP/3. Part 7 grounds all of it in one real, production C++ implementation ([openbmc/bmcweb](https://github.com/openbmc/bmcweb)).

## How to Use This Guide

- **New to HTTPS?** Read Parts 1 → 2 → 4 in order — that's the full conceptual picture.
- **Need to generate or inspect a certificate right now?** Jump to Part 3.
- **Debugging a handshake, or explaining one?** Part 5 has step-by-step text diagrams.
- **Hardening a server / reviewing TLS config?** Part 6.
- **Want to see these concepts in real production code?** Part 7 walks through bmcweb.

## Table of Contents

**Part 1 — Foundations**
- [1) HTTP vs HTTPS](#1-http-vs-https)
- [2) What Encryption Solves](#2-what-encryption-solves)
- [3) Symmetric vs Asymmetric Cryptography](#3-symmetric-vs-asymmetric-cryptography)
- [4) Hashing, MAC, and Digital Signatures](#4-hashing-mac-and-digital-signatures)

**Part 2 — TLS/SSL In Depth**
- [5) SSL vs TLS: History and Versions](#5-ssl-vs-tls-history-and-versions)
- [6) Cipher Suites Explained](#6-cipher-suites-explained)
- [7) SNI and ALPN in the ClientHello](#7-sni-and-alpn-in-the-clienthello)
- [8) Certificates and PKI](#8-certificates-and-pki)
- [9) The TLS Handshake Step by Step](#9-the-tls-handshake-step-by-step)

**Part 3 — Certificate Files and OpenSSL**
- [10) Certificate and Key File Formats](#10-certificate-and-key-file-formats)
- [11) OpenSSL Practical Cheat Sheet](#11-openssl-practical-cheat-sheet)

**Part 4 — HTTP Protocol Versions**
- [12) HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3](#12-http10-vs-http11-vs-http2-vs-http3)

**Part 5 — Handshake Text Diagrams**
- [13) TCP and TLS 1.2 Handshake](#13-tcp-and-tls-12-handshake)
- [14) TLS 1.3 Handshake and 0-RTT](#14-tls-13-handshake-and-0-rtt)
- [15) HTTP/3 over QUIC](#15-http3-over-quic)

**Part 6 — Security Hardening**
- [16) Historical Attacks and Why Modern Defaults Exist](#16-historical-attacks-and-why-modern-defaults-exist)
- [17) Hardening Checklist](#17-hardening-checklist)
- [18) Testing and Diagnostic Tools](#18-testing-and-diagnostic-tools)
- [19) Case Study: What `curl -k` Actually Skips](#19-case-study-what-curl--k-actually-skips)

**Part 7 — Applied Case Study: OpenBMC bmcweb**
- [20) PEM Certificate Management via Redfish](#20-pem-certificate-management-via-redfish)
- [21) bmcweb Source Walkthrough](#21-bmcweb-source-walkthrough)

**Appendix**
- [A) Glossary](#a-glossary)
- [B) Quick Command Reference](#b-quick-command-reference)
- [C) Further Reading](#c-further-reading)

---

<a id="1-http-vs-https"></a>
## 1) HTTP vs HTTPS

### What HTTP Is

HTTP (HyperText Transfer Protocol) is the application-layer protocol browsers and servers use to exchange requests and responses.

- Default port `80`
- Transmitted in plaintext over the network — anyone on the network path can read or tamper with the content
- No authentication mechanism — you can't confirm you're actually talking to the server you think you are
- Vulnerable to eavesdropping, tampering, and MITM (man-in-the-middle) attacks

### What HTTPS Is

HTTPS is HTTP layered on top of TLS (Transport Layer Security — the modern name for what used to be called SSL, see [§5](#5-ssl-vs-tls-history-and-versions)).

- Default port `443`
- **Confidentiality**: traffic is encrypted, so eavesdroppers only see ciphertext
- **Integrity**: tampering in transit can be detected
- **Authentication**: a certificate proves the server's identity matches what it claims

None of this is provided by HTTP itself — it all comes from the TLS layer underneath. That's why Part 2 and Part 3 exist: to explain the mechanism HTTPS is actually built on, rather than just the surface fact that "it's encrypted."

### One-Line Difference

- HTTP: like a postcard — anyone who handles it along the way can read the contents
- HTTPS: like a sealed, tamper-evident envelope addressed to a verified recipient

---

<a id="2-what-encryption-solves"></a>
## 2) What Encryption Solves

TLS exists to provide three properties. Every mechanism covered later in this guide — key exchange, certificates, MACs — exists to achieve one of these:

| Property | Question It Answers | How TLS Provides It |
|---|---|---|
| **Confidentiality** | Can anyone else see the content? | Symmetric encryption of the data (e.g. AES) |
| **Integrity** | Was it tampered with in transit? | AEAD ciphers / MACs — tampered ciphertext fails to decrypt/verify |
| **Authentication** | Am I talking to who I think I am? | Certificates + digital signatures, checked during the handshake |

A useful mental model: **the entire job of the handshake is to let two strangers agree on a shared secret over a public channel, prove the server's identity, and do it all without ever sending that secret in a form an eavesdropper could reuse.** Everything in Parts 2–3 is in service of that one sentence.

Note what TLS does *not* do:

- It does not protect data once it's decrypted at either end (that's the application/OS layer's responsibility).
- It does not hide the fact that a connection happened, and usually doesn't hide the destination hostname either (SNI is visible on the wire unless ECH is used — see [§7](#7-sni-and-alpn-in-the-clienthello)).
- It does not verify that the server is *trustworthy* or *not malicious* — it only proves the server holds the private key matching the identity claimed in its certificate.

---

<a id="3-symmetric-vs-asymmetric-cryptography"></a>
## 3) Symmetric vs Asymmetric Cryptography

### Symmetric Encryption

Uses the same shared key to encrypt and decrypt.

- Examples: AES-128/256, ChaCha20
- Fast — used for the actual bulk data transfer (the whole HTTP request/response)
- Problem: both sides need to already have the "same" key before communicating — but how do you send that key over a network someone might be watching?

### Asymmetric (Public-Key) Encryption

Two mathematically related keys: a **public key** (freely shareable) and a **private key** (never shared).

- Examples: RSA, ECDSA/ECDHE (elliptic curve), Ed25519
- Data encrypted with the public key can only be decrypted with the matching private key (for signing it's the reverse: sign with the private key, verify with the public key)
- Solves the key-distribution problem — no shared secret needs to exist between the two sides beforehand
- Much slower than symmetric encryption (roughly 100–1000x), so unsuitable for bulk data transfer

### Why TLS Uses Both (Hybrid Encryption)

TLS only uses asymmetric cryptography to *establish* a shared key (via certificates + key exchange), then switches to fast symmetric encryption for the actual connection:

```text
1. Asymmetric crypto  -> authenticate the server + agree on a shared secret
2. Symmetric crypto   -> encrypt/decrypt all actual HTTP traffic with that secret
```

This is why a cipher suite name like `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` lists both an asymmetric part (`ECDHE`, `RSA`) and a symmetric part (`AES_128_GCM`) — see [§6](#6-cipher-suites-explained).

---

<a id="4-hashing-mac-and-digital-signatures"></a>
## 4) Hashing, MAC, and Digital Signatures

### Hash Functions

A hash function takes input of any length and produces a fixed-length fingerprint (digest).

- Examples: SHA-256, SHA-384
- The same input always produces the same output; changing even one bit changes the output completely
- One-way: you can't work backward from a digest to the original input
- Used to fingerprint certificates, derive keys, and build MACs — **not** encryption (no key, and not reversible)

### MAC / HMAC

A Message Authentication Code proves data hasn't been tampered with, *and* proves the sender knows a particular shared key.

- HMAC = hash function + key (e.g. `HMAC-SHA256`)
- Modern TLS mostly uses **AEAD** ciphers (AES-GCM, ChaCha20-Poly1305), which combine encryption and integrity checking into a single operation, instead of the older two-step "encrypt, then separately compute a MAC" approach

### Digital Signatures

A signature proves a message really came from the holder of a specific private key, and that the content hasn't been tampered with.

```text
Sign:   signature = Encrypt( Hash(message), private key )
      |
      v
  message, signature
      |
      v
Verify: Hash(message) == Decrypt( signature, public key )
```

(Real algorithms like RSA-PSS and ECDSA don't literally "encrypt the hash," but this diagram captures the idea.)

This is exactly how a Certificate Authority (CA) signs a certificate: hash the certificate's contents, then sign that hash with the CA's private key. Anyone holding the CA's public key can verify whether the certificate has been forged or tampered with — this is the mechanism underlying the entire chain of trust in [§8](#8-certificates-and-pki).

---

<a id="5-ssl-vs-tls-history-and-versions"></a>
## 5) SSL vs TLS: History and Versions

"SSL" and "TLS" are often used interchangeably, but SSL is actually the deprecated predecessor:

| Version | Year | Status |
|---|---|---|
| SSL 1.0 | — | Never publicly released (had fatal flaws) |
| SSL 2.0 | 1995 | Prohibited (RFC 6176, 2011) |
| SSL 3.0 | 1996 | Deprecated (RFC 7568, 2015) — broken by POODLE |
| TLS 1.0 | 1999 | Deprecated (RFC 8996, 2021) |
| TLS 1.1 | 2006 | Deprecated (RFC 8996, 2021) |
| TLS 1.2 | 2008 | Still widely used; secure if configured correctly |
| TLS 1.3 | 2018 | Current standard (RFC 8446), leaner and faster |

Practical takeaways:

- If a product or vendor today says "SSL," they almost certainly mean TLS — actual SSL has been unsafe to use for over a decade.
- Modern servers should **support only TLS 1.2 and TLS 1.3**, and fully reject SSL 2.0/3.0 and TLS 1.0/1.1 (see [§17](#17-hardening-checklist)).
- TLS 1.3 isn't just "TLS 1.2 with a bigger version number" — it outright removes several legacy mechanisms (static RSA key exchange, CBC cipher modes, custom Diffie-Hellman groups, compression) rather than merely discouraging them. Fewer options means fewer ways to misconfigure it.

---

<a id="6-cipher-suites-explained"></a>
## 6) Cipher Suites Explained

A cipher suite is the set of algorithms negotiated during the handshake. TLS 1.2 names them with four parts:

```text
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 |    |     |         |         |
 |    |     |         |         +-- Hash used for the handshake's HMAC/PRF
 |    |     |         +------------ Bulk symmetric cipher (AES-128 in GCM/AEAD mode)
 |    |     +---------------------- Authentication: server proves identity via RSA signature
 |    +---------------------------- Key exchange: Ephemeral Elliptic-Curve Diffie-Hellman
 +--------------------------------- Protocol
```

- **Key exchange** (`ECDHE`, `DHE`, or the legacy static `RSA`) — determines how the shared secret is generated. `ECDHE`/`DHE` provide **Perfect Forward Secrecy (PFS)**: even if the server's long-term private key leaks later, past sessions still can't be decrypted, because every session uses a fresh, never-stored, throwaway ephemeral key. Static `RSA` key exchange has no PFS — one leaked key exposes every past recorded connection — which is exactly why TLS 1.3 removed it entirely.
- **Authentication** (`RSA`, `ECDSA`) — the signature algorithm used to prove the server holds the private key matching its certificate.
- **Bulk cipher** (`AES_128_GCM`, `AES_256_GCM`, `CHACHA20_POLY1305`) — should always be an **AEAD** (Authenticated Encryption with Associated Data) cipher, providing both confidentiality and integrity. Avoid CBC-mode ciphers (`AES_128_CBC`) and stream ciphers like RC4 — both have known historical attacks (see [§16](#16-historical-attacks-and-why-modern-defaults-exist)).
- **Hash** (`SHA256`, `SHA384`) — used internally for key derivation during the handshake, not for the bulk data itself.

TLS 1.3 simplified this. Key exchange is *always* (EC)DHE (there's no longer a field for it in the name), and cipher suites only list the AEAD cipher + hash:

```text
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
```

The authentication algorithm is negotiated separately, via the certificate type and the `signature_algorithms` extension.

Check what your local OpenSSL supports:

```bash
openssl ciphers -v 'ALL'          # every cipher suite OpenSSL knows
openssl ciphers -v 'HIGH:!aNULL'  # a reasonably strong subset
```

---

<a id="7-sni-and-alpn-in-the-clienthello"></a>
## 7) SNI and ALPN in the ClientHello

Before any encryption is established, the very first handshake message — `ClientHello` — already carries two important plaintext extensions.

### SNI (Server Name Indication)

Problem: a single server may host many different HTTPS domains on the same IP address, each with a *different* certificate. But the server has to decide which certificate to present before the handshake has progressed far enough to know the HTTP-level client request (at which point the `Host` header is still encrypted).

- SNI solves this: the client sends the target hostname in plaintext inside `ClientHello`
- the server reads it and picks the matching certificate before the handshake continues
- Trade-off: anyone observing the connection (network gear, an ISP) can see the hostname, even though everything after the handshake is encrypted
- Newer mitigation: **ECH (Encrypted Client Hello)**, a TLS 1.3 extension that encrypts SNI too — supported by some browsers/CDNs but not yet universal

### ALPN (Application-Layer Protocol Negotiation)

Problem: the client and server need to agree on which application-layer protocol to use (HTTP/1.1, HTTP/2, ...) without adding extra round trips.

- the client lists the protocols it supports in its own `ClientHello` (e.g. `[h2, http/1.1]`)
- the server picks one and returns it in `ServerHello`
- the same `443` connection can seamlessly serve both HTTP/1.1 and HTTP/2 clients — no separate "upgrade" step needed

```text
ClientHello (SNI: example.com, ALPN: [h2, http/1.1])
            |
            v
ServerHello (ALPN selected: h2)
            |
            v
Use HTTP/2 frames on this connection
```

Relationship to each HTTP version:

- HTTP/1.0 and early HTTP/1.1 deployments predate ALPN, so they don't rely on it
- in virtually all real-world deployments, HTTP/2 over TLS is negotiated as `h2` via ALPN (the spec technically allows unencrypted `h2c`, but browsers only support `h2` over TLS)
- HTTP/3 runs over QUIC, which embeds TLS 1.3 directly; negotiation still happens via ALPN, with the identifier `h3`

---

<a id="8-certificates-and-pki"></a>
## 8) Certificates and PKI

A certificate binds a public key to an identity (a hostname), and the certificate itself is signed by another party vouching for that binding. This is Public Key Infrastructure (PKI).

### X.509 Certificate Structure

The standard certificate format is X.509. Main fields:

- Certificate Info
  - **Subject** — who this certificate represents (e.g. `CN=example.com`)
  - **Subject Alternative Name (SAN)** — the list of hostnames this certificate is actually valid for; modern clients ignore the `CN` field entirely and only check SAN
  - **Issuer** — which CA signed this certificate
  - **Validity** — the `Not Before` / `Not After` validity dates
  - **Subject Public Key Info** — the public key this certificate vouches for
  - **Extensions** — `Key Usage`, `Extended Key Usage`, `Basic Constraints` (is this a CA?), `Authority Key Identifier`, OCSP/CRL locations
- **CA Signature** — the issuer's signature over all of the above

Inspect an actual certificate:

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

- **Root CAs** are kept offline as much as possible — if a root's private key leaks, every certificate it has ever signed becomes suspect.
- **Intermediate CAs** handle the actual day-to-day signing, so a compromise is scoped to *only that intermediate's* issued certificates, and that intermediate can be revoked directly without touching the root.
- **Self-signed certificates** have no chain of trust — nobody vouches for them but the server itself. Fine for local development or closed internal systems (e.g. a BMC's default HTTPS certificate, see [§20](#20-pem-certificate-management-via-redfish)); clients will show a trust warning because there's no path in their trust store back to this certificate.

### How a Client Actually Validates a Certificate

1. Build a chain from the leaf certificate upward until reaching a root the client already trusts
2. Verify every signature in the chain (each certificate really was signed by the one above it)
3. Check that the current date falls within each certificate's validity period
4. Check that the requested hostname matches an entry in the leaf certificate's SAN list
5. Check revocation status — CRL (Certificate Revocation List) or OCSP (Online Certificate Status Protocol); **OCSP stapling** lets the server fetch and attach this proof itself, saving the client an extra round trip and avoiding a privacy leak (otherwise the CA would learn every site you visit)
6. Check that the `Basic Constraints`/`Key Usage` extensions match each certificate's role (e.g. only certificates marked as a CA may sign other certificates)
7. Authenticate the Certificate against the CA Signature:
```text
Sign: CA Signature = Encrypt( Hash(CertInfo || Server Public Key), CA Private Key )
        |
        v
  CertInfo ( with Server Public Key ), CA signature
        |
        v
Verify: Hash( CertInfo || Server Public Key ) == Decrypt( CA Signature, CA Public Key )
```

If any single step fails, the connection should be rejected — not silently downgraded to an unencrypted one.

### Getting a Certificate

- **Self-signed**: sign your own certificate. No external trust; fine for internal/test systems.
- **Private/internal CA**: an organization runs its own CA and installs its root into the trust store of its own devices. Common for internal infrastructure and IoT/BMC fleets.
- **Public CA**: signed by a CA the browser/OS already trusts (e.g. via **Let's Encrypt**, using the automated **ACME** protocol — free, domain-validation only, 90-day validity with automatic renewal).

### Certificate Transparency (CT)

Public CAs are required to publish every certificate they issue to a public, append-only CT log. This lets domain owners notice if a CA mistakenly issued a certificate in their name, and modern browsers require public certificates to carry a valid CT proof (SCT) to be trusted.

---

<a id="9-the-tls-handshake-step-by-step"></a>
## 9) The TLS Handshake Step by Step

Picking back up the goal from [§2](#2-what-encryption-solves): authenticate the server and derive a shared symmetric key, using only messages sent over what is, so far, a public and unencrypted connection.

### TLS 1.2 Full Handshake

```text
=================================================================================================================
【 Phase 1: Trust Chain Setup (PKI Setup) 】
=================================================================================================================

  【 Root / Intermediate CA 】
    │ - CA Private Key (kept strictly by the CA, used to issue certificates)
    │ - CA Public Key  (built into the Client Trust Store / OS)
    │
    │  (1) Server submits a CSR (containing the Server Public Key)
    │  (2) CA verifies identity and issues a certificate:
    └───────────────┐
                    ▼
          ┌─────────────────────────────────────────────────────────┐
          │  Server Certificate (server.crt)                        │
          ├─────────────────────────────────────────────────────────┤
          │  • Domain Name: *.example.com                           │
          │  • Server Public Key (long-term asymmetric key)         │
          │  • CA Signature = Sign(Cert Hash, CA Private Key)       │
          └─────────────────────────────────────────────────────────┘


=================================================================================================================
【 Phase 2: TLS 1.2 Handshake (Authentication and Key Exchange) 】
=================================================================================================================

Client                                                                Server
 (has the CA Public Key)                                              (has the Server Private Key & Certificate)
  │                                                                                 │
  │ ─── 1. ClientHello ───────────────────────────────────────────────────────────> │
  │       (ClientRandom, Cipher Suites, SNI, ALPN, Supported Groups)                │
  │                                                                                 │
  │ <── 2. ServerHello ──────────────────────────────────────────────────────────── │
  │       (ServerRandom, Selected Cipher Suite)                                     │
  │                                                                                 │
  │ <── 3. Certificate ──────────────────────────────────────────────────────────── │
  │        CA Signature =                                                           |
  |          E( Hash(CertInfo || Server Public Key), Server Private Key )           |
  │       (server.crt; contains the Server Public Key & CA Signature)               │
  │                                                                                 │
  │ <── 4. ServerKeyExchange ────────────────────────────────────────────────────── │
  │        ECDHE Signature =                                                        |
  |          E( Hash(ClientRandom || ServerRandom || Server ECDHE PubKey)           |
  |                                                 , Server Private Key )          |
  │       (Server ECDHE PubKey + ECDHE Signature)                                   │
  │                                                                                 │
  │ <── 5. ServerHelloDone ──────────────────────────────────────────────────────── │
  │                                                                                 │
  │  [Client Verification Phase]                                                    │
  │  A. Verify the Certificate using the CA Public Key                              │
  │     → Confirm the cert was issued by the CA, and extract the Server Public Key  │
  │   1. Compute the cert hash:  Hash_cert = Hash( CertInfo || Server Public Key )  │
  │   2. Decrypt the CA signature:  Hash_ca = D( CA Signature, CA Public Key )      │
  │   3. Compare:                Hash_cert  ==  Hash_ca                             │
  │      └──> Verified: Server Public Key extracted from CertInfo                   │
  │                                                                                 │
  │  B. Verify the Server's ECDHE signature using the Server Public Key             │
  │     → Prove the data wasn't tampered with, and the Server holds the matching key│
  │   Check:   D(ECDHE Signature, Server Public Key)                                |
  |         == H(ClientRandom || ServerRandom || ECDHE PubKey)                      │
  │       └──> Verified: ECDHE parameters are safe, Server holds the private key    │
  │                                                                                 │
  │ ─── 6. ClientKeyExchange ─────────────────────────────────────────────────────> │
  │       (Client ECDHE PubKey)                                                     │
  │                                                                                 │
  │ ─── 7. [ChangeCipherSpec] & Finished ─────────────────────────────────────────> │
  │                                                                                 │
  │ <── 8. [ChangeCipherSpec] & Finished ────────────────────────────────────────── │
  │                                                                                 │
  │  [Both sides independently derive the same symmetric key]                       │
  │  Client: Compute(Client ECDHE PrivKey + Server ECDHE PubKey + Randoms)          │
  │                                                                                 │
  │  【 Session Key / AES Key 】 <─────── identical on both sides ────────          │
  │                                                                                 │
  │  Server: Compute(Server ECDHE PrivKey + Client ECDHE PubKey + Randoms)          │
  │                                                                                 │

=================================================================================================================
【 Phase 3: Application Data Transfer (Symmetric Encryption) 】
=================================================================================================================

  Client                                                                         Server
    │                                                                               │
    │ === 9. HTTP / HTTPS data transfer (encrypted with the 【Session Key】) ===    │
    │                                                                               │
```

This diagram actually splits the handshake into three layers:

1. **PKI Setup**: first give the client a root of trust for the CA, so it knows "this server certificate was issued by a trusted CA"
2. **TLS Handshake**: authentication and key agreement via `ClientHello` / `ServerHello` / `Certificate` / `ServerKeyExchange`
3. **Application Data**: once the handshake is done, both sides switch to efficient symmetric encryption and start sending actual HTTP data

What happens at each step:

1. **ClientHello** — the client proposes a TLS version, cipher suites, a random nonce, and various extensions (SNI, ALPN, `supported_groups`, etc.)
2. **ServerHello** — the server picks the version/cipher suite, sends its own random nonce, and responds with the negotiated choices
3. **Certificate** — the server sends its certificate chain so the client can verify the server's identity
4. **ServerKeyExchange** — the server sends a temporary `(EC)DHE` public value, signed with the private key matching its certificate; this signature binds "the ephemeral key exchange" to "the verified identity"
5. **Client verification** — the client first verifies the certificate chain (see [§8](#8-certificates-and-pki)), confirming it was issued by a trusted CA, then verifies the `ServerKeyExchange` signature with the server's public key; success means the data wasn't tampered with, and the server really holds the matching private key
6. **ClientKeyExchange** — the client sends its own temporary key-share value; each side combines its own private key with the other's public value via Diffie-Hellman to compute the same shared secret
7. **Finished** — both sides send an encrypted `Finished` message, confirming both derived the same key and that nothing in the handshake was altered
8. **Application Data** — once the handshake completes, both sides encrypt actual HTTP requests/responses using the negotiated AEAD cipher (e.g. AES-GCM, ChaCha20-Poly1305)

The core goals of these steps are:

- Authenticate the server's identity: prevent a man-in-the-middle from impersonating it
- Negotiate a shared symmetric key: so subsequent traffic can use fast, cheap symmetric encryption
- Confirm key agreement with `Finished`: catch "each side derived a different key" or tampered data

It takes roughly **2 round trips** before the first byte of application data can be sent.

### Abbreviated Handshake (Session Resumption)

Doing a full handshake every time you reconnect to a site you just visited is wasteful. TLS caches enough state to skip most of it:

- **Session ID** — the server keeps the session state; the client just reminds it of the ID
- **Session Tickets** (RFC 5077) — the server encrypts its own session state into a ticket handed to the client; the server needs to store nothing, which scales better

```text
Client                                Server
  | -- ClientHello + session ticket --> |
  | <- ServerHello (resumed) ---------- |
  | <- [ChangeCipherSpec], Finished --- |
  | -- [ChangeCipherSpec], Finished --> |
  | ======== Application Data ========  |
```

This shortens the handshake to **1 round trip**, and skips certificate verification and asymmetric key exchange entirely (reusing the key derived from the original session to produce a new one).

TLS 1.3's handshake and 0-RTT resumption mechanism differ enough to deserve their own diagram — see [§14](#14-tls-13-handshake-and-0-rtt).

### mTLS (Mutual TLS)

Everything above proves the *server's* identity to the client. Some deployments (service-to-service APIs, BMC-to-BMC management traffic, zero-trust internal networks) also need to verify the *client's* identity:

- the server sends an additional `CertificateRequest` message
- the client replies with its own `Certificate`, plus a `CertificateVerify` signature proving it holds the matching private key
- both identities are now cryptographically verified to each other

```text
=================================================================================================================
【 Phase 1: Trust Chain Setup (PKI Setup) 】
=================================================================================================================

  【 Root / Intermediate CA 】
    │ - CA Private Key (kept strictly by the CA, used to issue certificates)
    │ - CA Public Key  (pre-installed in each side's own Trust Store)
    │
    ├─ (1) Server submits a CSR ──> issued Server Certificate (server.crt) ──> stored on Server
    └─ (2) Client submits a CSR ──> issued Client Certificate (client.crt) ──> stored on Client (mTLS addition)


=================================================================================================================
【 Phase 2: TLS 1.2 Handshake (mTLS Mutual Authentication and Key Exchange) 】
=================================================================================================================

Client                                                                Server
 (has the Server CA Public Key                                        (has the Client CA Public Key
  and Client Private Key & client.crt)                                 and Server Private Key & server.crt)
  │                                                                                 │
  │ ─── 1. ClientHello ───────────────────────────────────────────────────────────> │
  │       (ClientRandom, Cipher Suites, SNI, ALPN, Supported Groups)                │
  │                                                                                 │
  │ <── 2. ServerHello ──────────────────────────────────────────────────────────── │
  │       (ServerRandom, Selected Cipher Suite)                                     │
  │                                                                                 │
  │ <── 3. Certificate ──────────────────────────────────────────────────────────── │
  │       (server.crt; contains the Server Public Key & CA Signature)               │
  │                                                                                 │
  │ <── 4. ServerKeyExchange ────────────────────────────────────────────────────── │
  │       (Server ECDHE PubKey + ECDHE Signature)                                   │
  │                                                                                 │
  │ <── 4.5 CertificateRequest ───────────────────────────────────────────────────  │ <== [mTLS addition]
  │       (Server requests a Client certificate, optionally with a trusted CA list) │
  │                                                                                 │
  │ <── 5. ServerHelloDone ──────────────────────────────────────────────────────── │
  │                                                                                 │
  │  [Client verifies Server]                                                       │
  │  A. Verify server.crt using the CA Public Key (extract the Server Public Key)   │
  │  B. Verify the Server's ECDHE signature using the Server Public Key             │
  │     (confirms Server identity and that the parameters are safe)                 │
  │                                                                                 │
  │ ─── 5.5 Certificate ──────────────────────────────────────────────────────────> │ <== [mTLS addition]
  │       (client.crt; contains the Client Public Key & CA Signature)               │
  │                                                                                 │
  │ ─── 6. ClientKeyExchange ─────────────────────────────────────────────────────> │
  │       (Client ECDHE PubKey)                                                     │
  │                                                                                 │
  │ ─── 6.5 CertificateVerify ───────────────────────────────────────────────────>  │ <== [mTLS addition]
  │       Signature = E( Hash(all prior Handshake messages), Client Private Key )   │
  │                                                                                 │
  │                                                                                 │
  │        [Server verifies Client]                                                 │ <== [mTLS addition]
  │        A. Verify client.crt with the CA Public Key → get the Client Public Key  │
  │        B. Verify the Client's ECDHE signature using the Client Public Key       │
  │           (confirms Client identity and that the parameters are safe)           │
  │                                                                                 │
  │ ─── 7. [ChangeCipherSpec] & Finished ─────────────────────────────────────────> │
  │                                                                                 │
  │ <── 8. [ChangeCipherSpec] & Finished ────────────────────────────────────────── │
  │                                                                                 │
  │  [Both sides independently derive the same symmetric key]                       │
  │  Client: Compute(Client ECDHE PrivKey + Server ECDHE PubKey + Randoms)          │
  │                                                                                 │
  │  【 Session Key / AES Key 】 <─────── identical on both sides ────────          │
  │                                                                                 │
  │  Server: Compute(Server ECDHE PrivKey + Client ECDHE PubKey + Randoms)          │
  │                                                                                 │

=================================================================================================================
【 Phase 3: Application Data Transfer (Symmetric Encryption) 】
=================================================================================================================

  Client                                                                         Server
    │                                                                               │
    │ === 9. HTTP / HTTPS data transfer (encrypted with the 【Session Key】) ===    │
    │                                                                               │
```

---

<a id="10-certificate-and-key-file-formats"></a>
## 10) Certificate and Key File Formats

The same certificate or key can be stored in several different container formats. Confusing these is one of the most common practical TLS headaches.

### PEM (Privacy-Enhanced Mail)

A text format, Base64-encoded, delimited by begin/end marker lines. By far the most common format on Linux/OpenSSL-based systems.

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

- Roughly human-readable, diff-friendly, and easy to concatenate — a PEM "chain" file is just several `BEGIN/END CERTIFICATE` blocks back to back
- A single `.pem` file can also hold a certificate *and* its private key together (two blocks, one file) — that's exactly how bmcweb's `server.pem` does it, see [§21](#21-bmcweb-source-walkthrough)
- Common extensions: `.pem`, `.crt`, `.cer`, `.key` — the extension is only a *convention* and doesn't guarantee the actual content; when unsure, check with `openssl x509 -text` or `openssl pkey -text`

### DER (Distinguished Encoding Rules)

The same underlying ASN.1 structure PEM wraps in Base64, just as a binary encoding.

- Not human-readable, but more compact
- Common on Windows/Java systems and some embedded environments
- Extension: `.der`, sometimes also `.cer`

### PKCS#7 / P7B

A bundle of certificates (a chain), but **never contains a private key**.

- Mainly used on Windows/Java to ship a certificate together with its intermediate chain
- Extensions: `.p7b`, `.p7c`

### PKCS#12 / PFX

A single, password-protected binary container holding the certificate, private key, and optionally the certificate chain — one file is enough to stand up a server.

- Commonly used to import into Windows/Java keystores or browsers
- Extensions: `.p12`, `.pfx`

### CSR (Certificate Signing Request, PKCS#10)

Not a certificate — a *request* for one.

- Contains your public key + identity info (CN, SAN, organization), signed with your own private key to prove you hold it
- You send this file to a CA; the CA verifies it and returns a signed certificate
- Extension: `.csr`

### Quick Reference

| Extension | Usually Contains | Format |
|---|---|---|
| `.pem` | Certificate, key, chain, or CSR (any of these) | Text/Base64 |
| `.crt` / `.cer` | Certificate | Usually PEM, occasionally DER |
| `.key` | Private key | Usually PEM |
| `.csr` | Signing request | Text/Base64 |
| `.der` | Certificate or key | Binary |
| `.p7b` / `.p7c` | Certificate chain, no key | Binary |
| `.p12` / `.pfx` | Certificate + key + chain, password-protected | Binary |

When in doubt, don't trust the extension — open it directly with the commands in [§11](#11-openssl-practical-cheat-sheet).

---

<a id="11-openssl-practical-cheat-sheet"></a>
## 11) OpenSSL Practical Cheat Sheet

OpenSSL is both a cryptography library (`libssl` handles the TLS protocol, `libcrypto` the underlying algorithm primitives) and the `openssl` command-line tool built on top of it — this section covers the command-line tool.

### Generate a Private Key

```bash
# RSA (traditional, widely compatible)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key

# EC (smaller keys, faster, equally strong at much shorter key lengths)
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out server.key
```

### Generate a CSR (to send to a CA)

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

`-addext subjectAltName` matters: modern clients ignore `CN` and reject certificates without a matching SAN entry (see [§8](#8-certificates-and-pki)).

### Inspect Things

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

Compare the public key embedded in each — works for both RSA and EC:

```bash
openssl x509 -in server.crt -noout -pubkey | openssl sha256
openssl pkey  -in server.key -pubout        | openssl sha256
# identical output = they match
```

### Convert Between Formats

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

### Talk to a Live Server

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

### Misc Useful Commands

```bash
openssl ciphers -v                 # list cipher suites this OpenSSL build supports
openssl rand -base64 32            # generate a random secret
openssl dgst -sha256 file.txt      # hash a file
openssl speed aes-128-gcm          # benchmark a cipher on this machine
```

### Common Errors

| Symptom | Likely Cause |
|---|---|
| `unable to get local issuer certificate` | The server isn't sending its intermediate chain, or it wasn't passed to `verify` |
| Server fails to start / "key values mismatch" | The certificate and key file don't actually match — check with the pubkey-hash method above |
| Browser shows a "self-signed certificate" warning | Expected — no CA vouches for this certificate (see [§8](#8-certificates-and-pki)) |
| `certificate has expired` | Check the `-dates` output; time to renew |
| Certificate is valid but hostname-mismatch warning appears | The certificate's SAN list doesn't include the hostname you connected with |

---

<a id="12-http10-vs-http11-vs-http2-vs-http3"></a>
## 12) HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.0 (1996, RFC 1945)

- Each TCP connection handles one request/response pair; the connection is usually then closed
- No mandatory `Host` header — one IP can really only cleanly serve one website
- No header compression; every request repeats the full headers again

Impact: a multi-resource page (HTML + CSS + JS + images) pays a fresh TCP (and for HTTPS, a fresh TLS) handshake cost for almost every request.

### HTTP/1.1 (1997, revised 2014 as RFC 7230–7235)

Most of today's web traffic still runs on this version under the hood, just overshadowed by HTTP/2's reputation.

- **Persistent connections by default** — a single TCP connection can carry many requests
- **`Host` header required** — makes name-based virtual hosting possible (one IP serving multiple domains)
- **Chunked transfer encoding** — a response body can be streamed out as it's produced, without knowing the total length in advance
- **Pipelining** — technically allows sending the next request without waiting for the previous response, but suffers from head-of-line blocking (one slow response blocks everything queued behind it) — in practice browsers almost never use it

Conceptual diagram: Persistent connections
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
                │ + 4 TLS handshakes (if over HTTPS)


HTTP/1.1: a single TCP connection can carry multiple requests

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
                                  │ TCP/TLS handshake cost drastically reduced

The one identifier of an underlying connection: the 5-Tuple
To the OS and a web server, an HTTPS connection is uniquely identified by a TCP 5-Tuple:
  Source IP address
  Source port — assigned randomly by the client
  Destination IP address
  Destination port — usually 443
  Transport protocol (TCP or UDP)

As long as this 5-tuple stays the same for a given client-initiated connection, the server treats it as "the same connection." The symmetric key (Session Key) derived once the TLS handshake completes is bound directly to this connection.

This is the value of "persistent connections": **collapsing what used to be many short-lived connections into one long-lived one**, so requests/responses can be reused repeatedly, cutting the cost of repeated handshakes and connection setup.
```

### HTTP/2 (2015, RFC 7540)

- Uses **binary framing** instead of text parsing
- **Header compression (HPACK)** — headers are compressed and diffed against previous requests
- **Multiplexing** — many concurrent streams on one connection, replacing HTTP/1.1's "open 6 connections per host" workaround
- **Server Push** — the server can proactively push resources it expects the client will need; in practice this feature has been deprecated and removed from most browsers (e.g. Chrome removed it in 2022), because real-world caching behavior made it a net loss
- Requires TLS in practice — no major browser supports plaintext HTTP/2 (`h2c`)
- Still suffers **TCP-level** head-of-line blocking: one lost packet stalls *all* multiplexed streams, because they share the same TCP byte stream

Conceptual diagram: HPACK header compression
```text
HTTP/1.1: nearly identical headers are repeated on every request

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

=> Lots of repeated headers, wasted bandwidth


HTTP/2 / HPACK: build a shared dynamic table first, then send only "the diff"

Dynamic Table (shared between server / client)
------------------------------------------------
| Index | Field Name       | Field Value        |
| 1     | :method          | GET                |
| 2     | :scheme          | https              |
| 3     | :authority       | example.com        |
| 4     | user-agent       | curl/8.0           |
| 5     | accept           | */*                |
------------------------------------------------

Much of Request 2's headers are already known:

Raw:
  :method = GET
  :authority = example.com
  user-agent = curl/8.0
  accept = */*

After HPACK compression, only this is sent:
  [Index 1] [Index 3] [Index 4] [Index 5]

=> Only "index references" or "deltas from last time" are sent
=> That's HPACK: headers aren't resent in full, they're diffed against prior state


Conceptual mental model:
- First share a dictionary of common headers
- Subsequent requests only send "which index I'm using" or "what changed vs. the old value"
- This drastically cuts header repetition and improves load efficiency

This is another key HTTP/2 optimization: **headers are no longer resent in full like HTTP/1.x — they're compressed via an index table and diffing**. This especially helps when many resources load at once, since each request's headers share a lot of repeated text.
```

Conceptual diagram: Binary framing
```text
HTTP/1.x: messages are text, parsed line by line

GET /index.html HTTP/1.1
Host: example.com
User-Agent: curl/8.0
Accept: */*


HTTP/2: messages are chopped into binary frames, managed by stream/priority/length

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
        HEADERS frame  -> this stream's headers
        DATA frame     -> this stream's body
        SETTINGS frame -> negotiates connection-level parameters


Benefits:
- No longer relies on "\r\n" delimiters for parsing
- Frames can be efficiently multiplexed, reordered, and flow-controlled
- One stream's data doesn't have to be mixed into the same plaintext message as another stream's

This distinction matters: **HTTP/1.x is built around "text request/response," while HTTP/2 is built around "binary frame streams."** In other words, HTTP/2 no longer describes an entire request as one big block of readable text — it splits it into individual frames, which streams then reassemble.
```

Conceptual diagram: Multiplexing vs HOL blocking
```text
HTTP/1.1: each request basically needs its own connection

Req A ──┐
Req B ──┼──> Connection 1
Req C ──┤
Req D ──┘

Req E ──┐
Req F ──┼──> Connection 2
Req G ──┤
Req H ──┘

=> Needs many connections, and requests still end up queuing


HTTP/2: multiple independent streams on one TCP connection

TCP Connection
-------------------------------------------------
| Stream 1 | Stream 2 | Stream 3 | Stream 4 |
| HTML     | CSS      | JS       | IMG      |
|  Req     |  Req     |  Req     |  Req     |
-------------------------------------------------

"Multiplexing" means:
- Multiple streams exist in parallel within one connection
- Not every request needs to be split into a new connection
- One large resource doesn't completely block smaller ones
- The server reassembles frames arriving interleaved back into the right request by Stream ID, with no queuing needed — true parallel, bidirectional multiplexing

But note:
  TCP is still a single byte stream
  One lost packet => TCP has to reorder/retransmit => every stream can stall

     [packet loss]
           ↓
   TCP retransmit / reordering
           ↓
   all streams wait together
```

### HTTP/3 (2022, RFC 9114, over QUIC RFC 9000)

- Runs over **QUIC** (UDP-based) instead of TCP
- TLS 1.3 is built directly into the transport-layer handshake, rather than layered on top
- Each stream is independently reliable — a lost packet only stalls the one stream it belongs to, fixing the head-of-line blocking HTTP/2 inherited from TCP
- **Connection migration** — a connection can survive network changes (e.g. switching from Wi-Fi to cellular), because the connection is identified by a Connection ID rather than an IP/port pair
- Can offer **0-RTT** reconnection to previously visited servers (with the same replay-risk considerations as TLS 1.3 0-RTT — see [§14](#14-tls-13-handshake-and-0-rtt))

Conceptual diagram: stream independence
```text
QUIC Connection (Connection ID)
-------------------------------------------------
| Stream 1 | Stream 2 | Stream 3 | Stream 4 |
| HTML     | CSS      | JS       | IMG      |
| reliable | reliable | reliable | reliable |
-------------------------------------------------

When one packet is lost, only the stream it belongs to is affected
Other streams keep transmitting — nothing else stalls
```

Conceptual diagram: Connection migration
```text
HTTP/2 / TCP: connection identity depends on IP + Port

Client (Wi‑Fi)      ──────── connection ────────>   Server
  IP: 10.0.0.5:52134

After switching to 4G / cellular:
Client (4G)          ──────── connection ────────>   Server
  IP: 192.168.1.50:43122

=> These are actually "two different TCP connections"
=> Disconnect / reconnect / re-handshake


HTTP/3 / QUIC: connection identity depends on Connection ID

Client (Wi‑Fi)               Server
   [Connection ID: CID-42]  <──────>  [Connection ID: CID-42]
         │
         ├─ IP changes when switching to 4G, but the Connection ID doesn't
         │
         └─ Still treated as the same connection

=> The connection can "migrate" without rebuilding the whole session
=> Much friendlier for mobile devices and network switching

This is another key HTTP/3 change: **it moves from "identifying a connection by IP/Port" to "identifying it by Connection ID,"** so a connection can keep living across a Wi-Fi ↔ cellular switch instead of dying immediately.
```

### Comparison

| Aspect | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
| :--- | :--- | :--- | :--- | :--- |
| **Transport layer** | TCP | TCP | TCP | **QUIC (UDP)** |
| **TLS versions supported/typical** | TLS 1.0 / 1.1 / 1.2 *(early on, SSL 3.0)* | TLS 1.2 / 1.3 *(can also run plaintext HTTP)* | **TLS 1.2 / 1.3** *(required by browsers and spec)* | **TLS 1.3 only** *(built into the standard)* |
| **TLS integration** | Separate layer (over TLS) | Separate layer (over TLS) | Separate layer, negotiated via **ALPN** (`h2`) | **Natively built into** the QUIC transport layer |
| **First-connection handshake latency** *(transport + TLS)* | **3 RTT**<br>*(1 TCP + 2 TLS 1.2)* | **2–3 RTT**<br>*(1 TCP + 1–2 TLS)* | **2–3 RTT**<br>*(1 TCP + 1–2 TLS)* | **1 RTT**<br>*(QUIC transport and TLS 1.3 handshake merged)* |
| **Resumption latency** | **2 RTT** *(Session ID)* | **1–2 RTT** *(Session Ticket)* | **1–2 RTT** *(Session Ticket / PSK)* | **0 RTT** *(TLS 1.3 0-RTT PSK)* |
| **Connections needed** | Many | Fewer (persistent connections) | One (multiplexed) | One (multiplexed) |
| **Header compression** | None | None | HPACK | QPACK |
| **Head-of-line blocking** | Yes (severe) | Yes | Transport-level only (TCP HOL blocking) | **None** |

Which version a given connection actually ends up using is decided during the TLS handshake by ALPN (or QUIC's equivalent mechanism) — see [§7](#7-sni-and-alpn-in-the-clienthello).

---

<a id="13-tcp-and-tls-12-handshake"></a>
## 13) TCP and TLS 1.2 Handshake

A quick combined reference diagram for both layers — the conceptual step-by-step is in [§9](#9-the-tls-handshake-step-by-step).

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

- TCP and TLS are two independent handshakes stacked on top of each other — TCP guarantees in-order delivery, TLS adds confidentiality/integrity/authentication on top
- HTTP/1.1 and HTTP/2 typically run over exactly this same stack; ALPN during the TLS handshake decides which protocol the connection ultimately speaks

---

<a id="14-tls-13-handshake-and-0-rtt"></a>
## 14) TLS 1.3 Handshake and 0-RTT

TLS 1.3 lets the client "guess" a key-exchange group and send its own key-share value right in the first message, instead of waiting for the server to say which group it wants — shortening the handshake from 2 round trips to 1.

### Full 1-RTT Handshake

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

- The server can compute the shared secret as soon as it sees the client's `key_share`, so almost everything after `ServerHello` — including the certificate — is already encrypted
- The client can send application data immediately after its own `Finished` — the encrypted application data goes out **in the same flight** as the final handshake message
- Static RSA key exchange no longer exists in TLS 1.3 — every handshake uses (EC)DHE, so every session has Perfect Forward Secrecy by default (see [§6](#6-cipher-suites-explained))

### 0-RTT Resumption (and Its Trade-off)

If the client has a session ticket (PSK) from a previous connection to the same server, it can send encrypted application data in the very first flight:

```text
Client                                 Server
  | -- ClientHello + PSK + early_data --> |
  | ==== 0-RTT Application Data ========> |
  | <- ServerHello + Finished ----------- |
  | <==== Application Data ===============|
```

- Zero round trips before the client's first request even goes out — a real latency win for returning users
- **Caveat: 0-RTT data has no forward secrecy, and it's replayable.** None of this data is bound to any unpredictable fresh value generated by the server, so a network attacker who captures the 0-RTT request can simply resend it, and the server has no built-in way to tell it apart from the original. This is exactly why 0-RTT is only recommended for idempotent operations (safe `GET` requests), and never for anything with side effects (payments, form submissions).

---

<a id="15-http3-over-quic"></a>
## 15) HTTP/3 over QUIC

```text
Client                                Server
  | ---- Initial (ClientHello) -------> |
  | <- Initial (ServerHello, Cert, ...) |
  | ---- Handshake Finished ----------> |   (QUIC + TLS ready)
  | ===== HTTP/3 Request (stream) =====>|
  | <==== HTTP/3 Response (stream) =====|
```

Key points:

- QUIC merges the transport-layer handshake and the TLS 1.3 handshake into a single exchange — there's no longer a separate "TCP connect, then TLS handshake" pair of phases
- Every HTTP/3 request/response runs on its own independently reliable QUIC stream, so a lost packet only stalls the stream it belongs to
- A QUIC connection is identified by a Connection ID rather than the traditional (source IP, source port, destination IP, destination port) 4-tuple, which is exactly what makes connection migration possible

---

<a id="16-historical-attacks-and-why-modern-defaults-exist"></a>
## 16) Historical Attacks and Why Modern Defaults Exist

This is exactly why [§17](#17-hardening-checklist) recommends "just use modern defaults" — every default in use today is a direct response to some real, named attack. (This is an educational summary meant to explain *why* these defaults exist — not an attack tutorial.)

| Attack | Year | Root Cause | Why It's a Non-Issue If You Follow §17 |
|---|---|---|---|
| **BEAST** | 2011 | Predictable IVs in TLS 1.0's CBC-mode ciphers | Fixed in TLS 1.1+; avoid CBC, use AEAD instead |
| **CRIME / BREACH** | 2012/13 | Compression ratio leaks secrets from the compressed stream | Compression is disabled by default at the TLS layer; still be careful compressing responses that contain secrets |
| **Heartbleed** | 2014 | A buffer over-read bug in OpenSSL's heartbeat extension implementation | A library-level bug, not a protocol flaw — patched; the lesson is to rotate keys/certificates after any such disclosure, since past traffic may have already leaked |
| **POODLE** | 2014 | A padding-oracle flaw in SSL 3.0's CBC mode | Disable SSL 3.0 entirely (there's no real compatibility benefit left today) |
| **Downgrade attacks** | Ongoing | An attacker forces the handshake to negotiate a weaker, breakable version/cipher | `TLS_FALLBACK_SCSV`, simply not offering old versions at all, and HSTS (which won't even let the first connection attempt plaintext HTTP) |
| **ROBOT** | 2017 | An attack on RSA key exchange — a revival of the 1998 Bleichenbacher padding-oracle attack | Avoid static RSA key exchange — use ECDHE instead; TLS 1.3 makes this mandatory |

The common thread: nearly every attack here either targets an optional legacy mechanism (SSL 3.0, CBC ciphers, static RSA key exchange) — none of which is even offered in a fully modern configuration — or exploits a bug in a specific implementation, not a flaw in the protocol itself. That's also why TLS 1.3 is designed by deleting options outright, rather than adding a switch someone has to remember to turn off.

---

<a id="17-hardening-checklist"></a>
## 17) Hardening Checklist

- **Protocol versions**: support only TLS 1.2 and TLS 1.3. Explicitly disable SSL 2.0/3.0 and TLS 1.0/1.1 — don't rely on "not offered by default," since some protocol stacks still enable them by default.
- **Cipher suites**: allow only AEAD ciphers (`*_GCM`, `*_CHACHA20_POLY1305`). Disable CBC-mode suites, RC4, 3DES, and anything with `NULL` or `EXPORT` in the name.
- **Key exchange**: prefer ECDHE over static RSA key exchange for Perfect Forward Secrecy (automatic under TLS 1.3).
- **Key size/type**: RSA ≥ 2048 bits (3072–4096 bits recommended for long-lived keys); EC should use P-256 or P-384.
- **Certificates**: always include a correct SAN list; keep validity as short as automation allows (Let's Encrypt's default 90-day validity forces automated renewal, which is itself a resilience win).
- **HSTS** (the `Strict-Transport-Security` header): tells the browser to never attempt plaintext HTTP to this host again, closing off downgrade/SSL-stripping attacks on subsequent visits.
- **OCSP stapling**: the server fetches its own revocation proof and attaches it to the handshake — faster than the client querying the CA directly, and better for privacy.
- **Automated renewal**: certificate expiry is one of the most common self-inflicted outages; ACME-based renewal (certbot, etc.) removes the need for manual intervention.
- **Protect private keys**: strict file permissions, never transmit keys over any unencrypted channel (including internal ones), and rotate immediately if compromise is suspected.
- **Test the actual deployed configuration, not just the intended one** — see [§18](#18-testing-and-diagnostic-tools).

---

<a id="18-testing-and-diagnostic-tools"></a>
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
- The response status line (`HTTP/1.1 ...` or `HTTP/2 ...`) also confirms it from the HTTP side

Other tools worth knowing about (mentioned so you know they exist — not covered in depth here):

- **testssl.sh** — scripted, fairly comprehensive scan of a server's protocols/ciphers/vulnerabilities
- **nmap's `ssl-enum-ciphers` script** — enumerates which ciphers each TLS version supports
- **Qualys SSL Labs' SSL Test** — a hosted online scanner that produces a graded report for any public server
- Browser DevTools → Security tab — shows the protocol/cipher actually negotiated for the current page

---

<a id="19-case-study-what-curl--k-actually-skips"></a>
## 19) Case Study: What `curl -k` Actually Skips

`-k` (long form `--insecure`) is probably the most misunderstood flag in TLS tooling. Two common misconceptions are "`-k` turns off encryption" and "`-k` falls back to HTTP" — both are wrong. The TLS handshake still runs from start to finish, traffic is still encrypted, and **the only thing skipped is certificate verification**.

### Step by step, in curl's actual order of operations

For `curl -k https://example.com/`, here's what really happens between pressing enter and getting a response:

**1. URL parsing and DNS resolution** — ✅ still done

Unaffected. `-k` plays no part in name resolution.

**2. TCP three-way handshake** — ✅ still done

Connects to port 443 (see [§13](#13-tcp-and-tls-12-handshake)); nothing to do with `-k`.

**3. ClientHello sent** — ✅ still done

TLS version list, cipher suite list, random, SNI, ALPN, and `supported_groups` all go out unchanged (see [§7](#7-sni-and-alpn-in-the-clienthello)). **The server cannot tell whether the client used `-k`** — this is purely client-side behavior from beginning to end.

**4. ServerHello received** — ✅ still done

TLS version and cipher suite are negotiated normally; `-k` does not cause a weaker suite to be selected.

**5. Certificate message received** — ✅ still done

The server still sends its full chain, and curl still receives and parses it into X.509 structures. `-k` does not make the server send less, nor make curl skip reading it.

**6. Certificate verification** — ❌ **skipped ← the one and only thing `-k` changes**

Normally curl performs all of the following at this point; `-k` disables the entire group at once:

| Check | What it validates |
|---|---|
| Chain validation | Verify each signature with the issuer's public key, up to a root CA in the trust store ([§8](#8-certificates-and-pki)) |
| Validity period | Whether `notBefore` / `notAfter` covers the current time |
| Hostname match | Whether the URL's host matches the certificate's SAN ([§8](#8-certificates-and-pki)) |
| Usage constraints | Whether `basicConstraints`, `keyUsage`, `extendedKeyUsage` permit this use |
| Revocation status | CRL/OCSP (depending on build and configuration) |

**7. CertificateVerify (TLS 1.3) / ServerKeyExchange (TLS 1.2) signature verification** — ✅ **still done**

This is the part people most often get wrong. curl still uses the public key from the certificate to verify this signature, confirming the peer **really does hold the private key matching that certificate** (see [§9](#9-the-tls-handshake-step-by-step)). What's skipped is "is this certificate trustworthy," not "does the peer hold its private key."

**8. ECDHE key exchange** — ✅ still done

Both sides derive the shared secret as usual, and PFS still holds (see [§6](#6-cipher-suites-explained)).

**9. Finished message exchange** — ✅ still done

Both sides compare transcript hashes, confirming the handshake wasn't tampered with.

**10. Application data** — ✅ still done

Encrypted with the negotiated AEAD algorithm; both confidentiality and integrity are intact.

### Verified in practice

Running `curl -kv` against a server with a self-signed certificate — every handshake message is present:

```text
* ALPN, offering h2
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN),  TLS handshake, Server hello (2):
* TLSv1.3 (IN),  TLS handshake, Certificate (11):      <- certificate still received
* TLSv1.3 (IN),  TLS handshake, CERT verify (15):      <- signature still verified
* TLSv1.3 (IN),  TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

More telling still: curl actually **computes the verification result and simply ignores it**:

```bash
curl -k -o /dev/null -s -w "verify_result=%{ssl_verify_result}\n" https://self-signed.host/
# verify_result=18     <- 18 = X509_V_ERR_DEPTH_ZERO_SELF_SIGNED_CERT
```

The failure code `18` is still calculated and reported — `-k` just stops curl from treating it as a reason to abort.

### `-k` actually flips two independent switches

At the libcurl level these are two separate options:

| Option | Check it governs | Effect of `-k` |
|---|---|---|
| `CURLOPT_SSL_VERIFYPEER` | Whether the chain leads back to a trusted CA | Set to `0` |
| `CURLOPT_SSL_VERIFYHOST` | Whether the hostname matches the SAN | Set to `0` |

There's no way to disable just one from the command line — `-k` is all or nothing. If the only problem is a hostname mismatch, the right fix is `--resolve`, not `-k` (see below).

### Threat model: what it stops and what it doesn't

```text
Passive eavesdropper (can capture packets, cannot modify traffic)

  Client <════════════ encrypted ════════════> Server
                   ^ attacker sees ciphertext only
  -> -k still protects you completely

Active attacker (can reroute, ARP-spoof, or control an intermediate node)

  Client <═══ encrypted ═══> [MITM] <═══ encrypted ═══> Server
                   ^ both legs are genuine TLS
                   ^ but the MITM holds both session keys and sees all plaintext
  -> -k offers no protection whatsoever
```

An attacker only needs to generate a private key and sign a self-signed certificate, and `-k` will accept it without complaint (see [§8](#8-certificates-and-pki)). What you get is a flawlessly encrypted tunnel — straight to the attacker.

The two guarantees, side by side:

- **With `-k`**: "I have an encrypted channel to *someone* who holds the private key for this certificate."
- **Without `-k`**: "…and that someone really is `example.com`, vouched for by a CA I trust."

### Three common verification failures and their proper fixes

`-k` makes all of these go away, but each has a more precise remedy:

```bash
# Symptom 1: self-signed, or issued by a private CA that isn't in the trust store
curl: (60) SSL certificate problem: self-signed certificate
# Fix: name the CA you want to trust, instead of disabling verification
curl --cacert ca.crt https://host/
curl --cacert server.crt https://host/     # self-signed: trust that exact certificate

# Symptom 2: CA is fine, but connecting by IP breaks the hostname match
curl: (60) SSL: no alternative certificate subject name matches target host name '192.168.1.100'
# Fix: keep full chain validation, just point the name at that IP
curl --cacert ca.crt --resolve host.example:443:192.168.1.100 https://host.example/

# Symptom 3: certificate has expired
curl: (60) SSL certificate problem: certificate has expired
# Fix: issue a new certificate. This is a real problem, not something to flag your way past
```

If you can't obtain the CA certificate at all, public key pinning is a reasonable fallback — it doesn't validate a chain, but it does bind the connection to one specific key:

```bash
# Compute the SHA-256 fingerprint of the certificate's public key
openssl x509 -in server.crt -pubkey -noout \
  | openssl pkey -pubin -outform der \
  | openssl dgst -sha256 -binary | base64

curl --pinnedpubkey "sha256//<value from above>" https://host/
```

### When it's acceptable and when it isn't

| Situation | Verdict |
|---|---|
| Local development against a cert you just made with `openssl req -x509` | ✅ Acceptable |
| First-time setup of a BMC's factory self-signed cert on an isolated management network ([§20](#20-pem-certificate-management-via-redfish)) | ✅ Acceptable |
| Debugging, to establish whether the problem is the certificate or something else | ✅ Acceptable (confirm connectivity with `-k`, then go back and fix verification) |
| Production automation scripts, CI/CD | ❌ Use `--cacert` instead |
| Any connection crossing an untrusted network | ❌ Equivalent to no protection at all |

This is also why the Redfish examples in [§20](#20-pem-certificate-management-via-redfish) all carry `-k`: a BMC ships with a self-signed certificate ([§8](#8-certificates-and-pki)), so there's no path back to it in any client's trust store, and these operations typically happen on an isolated management network. Once a proper certificate from the organization's internal CA has been installed via Redfish, `-k` should be replaced with `--cacert`.

---

<a id="20-pem-certificate-management-via-redfish"></a>
## 20) PEM Certificate Management via Redfish

Now let's apply everything above to a real system: [openbmc/bmcweb](https://github.com/openbmc/bmcweb), the HTTP server used by OpenBMC-family BMC firmware. This section covers the Redfish-facing certificate management API; [§21](#21-bmcweb-source-walkthrough) covers the underlying C++ implementation.

This maps to bmcweb's `redfish-core/lib/certificate_service.hpp`:

- HTTPS certificate collection: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/`
- A single HTTPS certificate: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/{id}`
- The generic replace action: `/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/`

The backend constants bmcweb maps these to internally:

- D-Bus service: `xyz.openbmc_project.Certs.Manager.Server.Https`
- D-Bus object base path: `/xyz/openbmc_project/certs/server/https`

### 1) Redfish Workflow (Recommended)

1. List the current HTTPS certificates:

```bash
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

2. Upload a new certificate to the HTTPS collection (POST) — `CertificateString` is PEM content, see [§10](#10-certificate-and-key-file-formats):

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

Note: bmcweb only accepts `CertificateType = PEM` for this action.

### 2) Path Mapping (File/Object View)

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

### 3) Do You Need to Restart the Service?

- In most cases, the certificate management service applies a newly installed/replaced certificate automatically.
- If you still see the old certificate, you can restart the HTTPS service from the BMC shell:

```bash
systemctl restart bmcweb.service
```

- Confirm the new certificate is actually in effect — the same `openssl s_client` trick from [§18](#18-testing-and-diagnostic-tools), pointed at the BMC itself:

```bash
openssl s_client -connect <bmc>:443 -servername <bmc> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

<a id="21-bmcweb-source-walkthrough"></a>
## 21) bmcweb Source Walkthrough

Source: [github.com/openbmc/bmcweb](https://github.com/openbmc/bmcweb) (`master` branch). What follows is a complete walkthrough of how the codebase implements networking, the TLS handshake, HTTP routing, and business logic in C++ with **Boost.Asio / Boost.Beast**.

---

### Architecture Overview

`http/http_connection.hpp` **does not** parse PEM certificate files directly — it only handles connection-level TLS mechanics (detecting SSL, the handshake, ALPN routing). PEM loading happens during SSL Context initialization at server startup, in `src/ssl_key_handler.cpp` (see Stage 0 below).

The business handler layer and the connection layer are separate, connected through a Router:

```text
[ Socket physical layer ] (TCP / TLS Socket)
       ↓
[ Connection & parsing layer ] http/http_connection.hpp (Connection::handle)
       ↓
[ Routing layer ] http/routing.hpp (Router::handle)
       ↓
[ Business logic layer ] redfish-core/lib/*.hpp (handleXxxGet / handleXxxPost)
       ↓
[ System service layer ] D-Bus Call / DB / Custom Logic
```

### Full Request Lifecycle

Here's the complete flow of an HTTP/HTTPS request from "TCP/TLS Socket established" **to** "Response written back to the Socket" (stage numbers match the walkthrough below):

```text
[Once, at server startup]
  0. Load the PEM certificate, build the SSL Context (http_server.hpp: loadCertificate -> ssl_key_handler.cpp: getSslServerContext)

[Per connection / per request]
[SOCKET START]
  1. Boost.Asio Server Acceptor (http_server.hpp: doAcceptOne -> afterAccept -> Connection::start)
     ↓
  2. TLS detection, handshake, and ALPN negotiation (http_connection.hpp: start -> async_detect_ssl -> afterDetectSsl -> [async_handshake -> afterSslHandshake])
     │
     ├── ALPN selects h2 -> upgradeToHttp2(), then follows the "Side Branch: HTTP/2" path below
     │
     ↓ (plaintext HTTP, or TLS without h2)
  3. HTTP header read (http_connection.hpp: doReadHeaders -> afterReadHeaders -> handle)
     ↓
  4. handle(): version check, keep-alive, auth, upgrade check (http_connection.hpp: handle -> doUpgrade)
     ↓ (a normal request)
  5. Route matching, privilege check, dispatch (routing.hpp: Router::handle -> validatePrivilege -> rule.handle)
     ↓
  6. Run the Redfish business logic (redfish-core/lib/*.hpp: handleXxx -> async D-Bus I/O)
     ↓
[BUSINESS LOGIC COMPLETED]
  7. AsyncResp's refcount hits zero and it destructs (async_resp.hpp: ~AsyncResp -> res.end)
     ↓
  8. Complete-request callback fires, security headers get added (http_connection.hpp: completeRequest -> completeResponseFields -> addSecurityHeaders)
     ↓
  9. Bytes get written back to the socket (http_connection.hpp: doWrite -> boost::beast::async_write -> afterDoWrite)
     │
     └── Keep-Alive -> back to step 3 to read the next request; otherwise gracefulClose()
[SOCKET END]
```

### Source Walkthrough by Stage

#### Stage 0: Server Startup — Loading the PEM Certificate

* **Location:** `http/http_server.hpp` (`loadCertificate`), `src/ssl_key_handler.cpp` (the rest)
* **What happens:** bmcweb calls `loadCertificate()` once at startup (and again on `SIGHUP`), which prepares and validates `/etc/ssl/certs/https/server.pem` — a single combined PEM file holding both the certificate and the key, one of the formats covered in [§10](#10-certificate-and-key-file-formats). The PEM data is loaded via `use_certificate_chain` and `use_private_key(..., pem)`, corresponding to the command-line `openssl x509`/`openssl pkey`. This step also registers ALPN's server-side callback (`alpnSelectProtoCallback`, used in Stage 2) with the SSL Context.

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

#### Stage 1: Listening and Accepting New Connections

* **Location:** `http/http_server.hpp`
* **What happens:** bmcweb sets up a `boost::asio::ip::tcp::acceptor` at startup. When a new TCP connection arrives, `afterAccept()` instantiates a `Connection` object and posts `start()` onto the io_context via `boost::asio::post`.

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

* **Location:** `http/http_connection.hpp` (the ALPN callback lives in `src/ssl_key_handler.cpp`)
* **What happens:** `start()` first uses `async_detect_ssl` to determine whether the connection is plaintext or TLS; only if it's TLS does it run `async_handshake`. Once the handshake completes, if HTTP/2 is enabled, `afterSslHandshake()` checks the ALPN negotiation result: if `h2` was selected, it calls `upgradeToHttp2()` directly to switch to HTTP/2 (see "Side Branch: HTTP/2" below); otherwise it calls `doReadHeaders()` to move into Stage 3.

```cpp
// http/http_connection.hpp
void start()
{
    readClientIp();
    /* connectionCount limit check, mTLS prep (if enabled) omitted */
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
        doReadHeaders();  // Plaintext HTTP: straight into Stage 3
    }
}

void afterSslHandshake(const std::shared_ptr<self_type>& /*self*/,
                       const boost::system::error_code& ec,
                       size_t bytesParsed)
{
    buffer.consume(bytesParsed);
    if (ec) { return; }  // Handshake failed: no active close, left to the deadline timer

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
                upgradeToHttp2();  // -> Side Branch: HTTP/2
                return;
            }
        }
    }

    doReadHeaders();  // h2 wasn't selected: move into Stage 3
}
```

The actual place the server "picks" the ALPN protocol is the callback registered with OpenSSL back in Stage 0:

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

This is the concrete, real-code version of the ALPN negotiation from [§7](#7-sni-and-alpn-in-the-clienthello), the "TCP first, then TLS" diagram from [§13](#13-tcp-and-tls-12-handshake), and the non-blocking handshake from [§9](#9-the-tls-handshake-step-by-step) all at once: if the client offers `h2` in `ClientHello`, nghttp2 picks it for OpenSSL; the instant the handshake ends, `afterSslHandshake()` reads that result to decide which way to branch.

---

#### Stage 3: HTTP Header Read

* **Location:** `http/http_connection.hpp`
* **What happens:** `doReadHeaders()` reads the HTTP headers via Boost.Beast; once done, `afterReadHeaders()` checks whether header parsing is finished — if so it calls `handle()` to move into Stage 4, otherwise it calls `doRead()` (which continues reading the body via `async_read_some`).

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
    /* Authentication, Content-Length checks, etc. omitted */
    if (parse.is_done())
    {
        handle();  // Header parsing complete, move into Stage 4
        return;
    }
    doRead();  // Body still needs reading
}
```

---

#### Stage 4: handle() — Version Check, Keep-Alive, Auth, and Upgrade Check

* **Location:** `http/http_connection.hpp`
* **What happens:** `handle()` first checks the `Host` header for HTTP/1.1 and reads out `keepAlive` (the HTTP/1.0 vs 1.1 difference mentioned in [§12](#12-http10-vs-http11-vs-http2-vs-http3) comes down to `req->version()`/`req->keepAlive()` here, not two separate code paths); then it runs the auth check, creates an `AsyncResp` and registers `completeRequest` as the completion callback; `doUpgrade()` checks whether this request wants to switch to WebSocket / SSE — if so it hands off directly to `handler->handleUpgrade()` and returns `true` (and `handle()` returns right there); otherwise the request is handed to the Router (Stage 5).

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

There's also an h2c branch inside `doUpgrade()` whose behavior is worth calling out specifically:

```cpp
// http/http_connection.hpp (excerpt from doUpgrade)
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

Note: the `res` here is a **member of the Connection itself** (not `asyncResp->res`). In other words, when `Upgrade: h2c` is detected, `doUpgrade()` only marks the Connection's `res` as `101 Switching Protocols` and adds the matching headers, but the function itself still returns `false` — so the same request still gets routed normally by `handler->handle(req, asyncResp)` afterward. It's not until that (Router-processed) response actually flows back through `completeRequest()` and is written to the socket that `afterDoWrite()` (see Stage 9) checks whether `res.result()` is `switching_protocols`, and only then calls `upgradeToHttp2()` to switch the connection over. Put differently, whether h2c actually manages to switch depends on whether the final `res` written back still carries that 101 status — something that isn't obvious just from reading the code, and worth verifying with an actual packet capture if you need to rely on it. ([§12](#12-http10-vs-http11-vs-http2-vs-http3) also notes that no major browser actually attempts h2c over a plaintext connection — this path mainly exists for non-browser clients.)

---

#### Stage 5: Route Dispatch and Privilege Check

* **Location:** `http/routing.hpp`
* **What happens:** `Router::handle()` looks up the matching Rule by URL path and HTTP method; if the request already carries a session, it first calls `validatePrivilege()` to do the permission check, and only forwards to the matching business handler once that passes.

```cpp
// http/routing.hpp
void handle(const std::shared_ptr<Request>& req,
            const std::shared_ptr<bmcweb::AsyncResp>& asyncResp)
{
    FindRouteResponse foundRoute = findRoute(*req);

    if (foundRoute.route.rule == nullptr)
    {
        // No matching route: try a dedicated 404 / 405 route, then respond with the right status
        asyncResp->res.result(boost::beast::http::status::not_found);
        return;
    }

    BaseRule& rule = *foundRoute.route.rule;
    std::vector<std::string> params = std::move(foundRoute.route.params);

    BMCWEB_LOG_DEBUG("Matched rule '{}' {} / {}", rule.rule,
                     req->methodString(), rule.getMethods());

    if (req->session == nullptr)
    {
        rule.handle(*req, asyncResp, params);  // Call the concrete business handler
        return;
    }
    // A logged-in session: run the privilege check first, then call the handler
    validatePrivilege(req, asyncResp, rule,
                      [req, asyncResp, &rule, params = std::move(params)]() {
                          rule.handle(*req, asyncResp, params);
                      });
}
```

---

#### Stage 6: Running the Business Handler and AsyncResp's RAII Mechanism

* **Location:** `redfish-core/lib/service_root.hpp` and `include/async_resp.hpp`
* **What happens:** The business logic layer (e.g. a Redfish API) makes D-Bus calls and fills in the JSON response. `AsyncResp` uses **RAII**: once all async calls finish and `AsyncResp`'s refcount hits zero, its destructor fires `res.end()`.

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

    // The actual JSON response is filled in inside handleServiceRootGetImpl()
    // (which may also kick off async D-Bus requests)
    handleServiceRootGetImpl(asyncResp);

    // Once this function returns and any async D-Bus callbacks complete, the shared_ptr<AsyncResp> destructs
}

// include/async_resp.hpp
class AsyncResp
{
  public:
    crow::Response res;

    ~AsyncResp()
    {
        // When AsyncResp's refcount hits zero, the destructor fires res.end() automatically
        // res.end() calls the completeRequestHandler set earlier (i.e. completeRequest)
        res.end();
    }
};
```

---

#### Stage 7: completeRequest — Adding Security Headers

* **Location:** `http/http_connection.hpp` (`addSecurityHeaders` is actually defined in `include/security_headers.hpp`)
* **What happens:** Once `res.end()` fires, control returns to the connection layer's `completeRequest()`. It calls `completeResponseFields()` (`http/complete_response_fields.hpp`), and it's *that* function which calls `addSecurityHeaders(res)` to add the security headers.

```cpp
// http/http_connection.hpp
void completeRequest(Response& thisRes)
{
    res = std::move(thisRes);
    res.keepAlive(keepAlive);

    // Internally calls addSecurityHeaders(res) etc. to add security headers
    completeResponseFields(accept, acceptEncoding, res);
    res.addHeader(boost::beast::http::field::date, getCachedDateStr());

    doWrite();
}
```

---

#### Stage 8: doWrite / afterDoWrite — Writing Back to the Socket

* **Location:** `http/http_connection.hpp`
* **What happens:** `doWrite()` wraps `res` in a `boost::beast::http::message_generator` and writes it back to the Socket/TLS adaptor via `boost::beast::async_write` (not `http::async_write`). Once the write completes, `afterDoWrite()` is where the real decision happens: if the `res` just written has status `switching_protocols` (see the h2c discussion in Stage 4), it calls `upgradeToHttp2()`; if it's Keep-Alive, it resets the parser and goes back to Stage 3 to keep reading; otherwise it closes the connection.

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
        upgradeToHttp2();  // h2c upgrade: -> Side Branch: HTTP/2
        return;
    }

    if (!keepAlive)
    {
        gracefulClose();  // Not Keep-Alive: close the connection
        return;
    }

    // Keep-Alive: clear the Response/reset the parser, back to Stage 3 for the next request
    res.clear();
    initParser();
    doReadHeaders();
}
```

---

#### Side Branch: If the Connection Switches to HTTP/2

Whether it's ALPN selecting `h2` in Stage 2, or a plaintext h2c upgrade in Stage 4/8 — once `upgradeToHttp2()` is called, this connection is handed off to `HTTP2Connection` in `http/http2_connection.hpp`. It no longer uses Boost.Beast's header/body parser; instead it uses **nghttp2** to handle binary frames directly (the binary framing/multiplexing mentioned in [§12](#12-http10-vs-http11-vs-http2-vs-http3), here as actual code):

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
                return onRequestRecv(frame.hd.stream_id);  // A stream's request is fully received
            }
            break;
        default:
            break;
    }
    return 0;
}
```

`onRequestRecv()` likewise builds a `Request`/`AsyncResp` and hands it to the same Router (equivalent to the Stage 5–6 logic), except that at the end it doesn't call `doWrite()` — it converts the response into HTTP/2 frames and sends those instead:

```cpp
int rv = ngSession.submitResponse(streamId, hdr, &dataPrd);
if (rv != 0)
{
    BMCWEB_LOG_ERROR("Fatal error: {}", nghttp2_strerror(rv));
    close();
    return -1;
}
```

### Flow Diagrams at a Glance

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
(a normal routing pass still runs first -> completeRequest swaps res for the routed result)
   |
doWrite() writes the final res back to the client
   |
afterDoWrite() checks res.result() == switching_protocols?
   |
   +-- yes -> upgradeToHttp2() -> HTTP2Connection
   +-- no  -> this connection never switches to HTTP/2
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

What to check (the same signals as [§18](#18-testing-and-diagnostic-tools)):

- `ALPN, server accepted to use h2` (HTTP/2)
- The response status line shows `HTTP/1.1 ...` or `HTTP/2 ...`

---

<a id="a-glossary"></a>
## A) Glossary

| Term | Meaning |
|---|---|
| **TLS** | Transport Layer Security — the current protocol; see [§5](#5-ssl-vs-tls-history-and-versions) |
| **SSL** | Secure Sockets Layer — TLS's deprecated predecessor, the name still used colloquially |
| **PKI** | Public Key Infrastructure — the system of CAs, certificates, and trust chains |
| **CA** | Certificate Authority — the entity that signs certificates and vouches for identity |
| **Root CA** | A CA whose certificate is self-signed and pre-installed in trust stores |
| **Intermediate CA** | A CA whose certificate was signed by a root, used for day-to-day signing |
| **CSR** | Certificate Signing Request — a request sent to a CA to obtain a signed certificate |
| **SAN** | Subject Alternative Name — the list of hostnames a certificate is actually valid for |
| **CN** | Common Name — a legacy identity field modern clients no longer trust (SAN is used instead) |
| **PEM** | A Base64-text container format for certificates/keys, delimited by `-----BEGIN/END-----` |
| **DER** | The binary-encoded version of the same certificate/key structure PEM wraps in text |
| **Cipher suite** | The combination of key exchange, authentication, encryption, and hash algorithms negotiated for a session |
| **AEAD** | Authenticated Encryption with Associated Data — a cipher mode providing both confidentiality and integrity (e.g. AES-GCM) |
| **ECDHE** | Elliptic-Curve Diffie-Hellman, Ephemeral — a key exchange method that provides forward secrecy |
| **Forward Secrecy** | The property that a leaked long-term key can't be used to decrypt past recorded connections |
| **ALPN** | Application-Layer Protocol Negotiation — selects HTTP/1.1, h2, or h3 during the handshake |
| **SNI** | Server Name Indication — the hostname sent in plaintext early in the handshake, so the server can pick the right certificate |
| **ECH** | Encrypted Client Hello — a newer extension that encrypts SNI as well |
| **HSTS** | An HTTP header that forces browsers to only use HTTPS for a host from then on |
| **OCSP** | Online Certificate Status Protocol — real-time certificate revocation lookup |
| **OCSP stapling** | The server fetches its own OCSP proof and attaches it, saving the client a round trip |
| **CRL** | Certificate Revocation List — another list-based revocation mechanism |
| **mTLS** | Mutual TLS — client and server each present a certificate and verify the other's identity |
| **0-RTT** | "Zero round trip" data sent alongside a TLS 1.3 resumption handshake, before it completes |
| **MITM** | Man-in-the-middle — an attacker positioned on the network path between client and server |

---

<a id="b-quick-command-reference"></a>
## B) Quick Command Reference

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

<a id="c-further-reading"></a>
## C) Further Reading

- [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446) — TLS 1.3
- [RFC 8996](https://www.rfc-editor.org/rfc/rfc8996) — Deprecating TLS 1.0 and 1.1
- [RFC 5246](https://www.rfc-editor.org/rfc/rfc5246) — TLS 1.2
- [RFC 6066](https://www.rfc-editor.org/rfc/rfc6066) — TLS extensions, including SNI
- [RFC 7540](https://www.rfc-editor.org/rfc/rfc7540) — HTTP/2
- [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114) — HTTP/3
- [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000) — QUIC transport
- [openbmc/bmcweb](https://github.com/openbmc/bmcweb) — the codebase referenced in Part 7

---

## Bottom Line

- HTTPS = HTTP + TLS; TLS's entire job is confidentiality, integrity, and authentication ([§2](#2-what-encryption-solves))
- Modern TLS means TLS 1.2/1.3 only, ECDHE key exchange, AEAD ciphers — every deprecated legacy option corresponds to a real, named historical attack ([§16](#16-historical-attacks-and-why-modern-defaults-exist))
- A certificate's trustworthiness comes down to the chain of trust behind it — self-signed certs are fine for closed systems but not for anything public-facing ([§8](#8-certificates-and-pki))
- OpenSSL's command-line tool covers generating, inspecting, converting, and live-testing with just a small, memorable set of commands ([§11](#11-openssl-practical-cheat-sheet), [Appendix B](#b-quick-command-reference))
- HTTP/2 and HTTP/3 mainly solve the connection/concurrency bottlenecks left over from HTTP/1.x, not security problems — though HTTP/3 does build TLS 1.3 directly into its transport-layer handshake ([§12](#12-http10-vs-http11-vs-http2-vs-http3), [§14](#14-tls-13-handshake-and-0-rtt), [§15](#15-http3-over-quic))
- Whether a given backend "supports" all of this depends on the application server, the TLS library, and any reverse proxy in front of it working together — [§20](#20-pem-certificate-management-via-redfish)–[§21](#21-bmcweb-source-walkthrough) walk through exactly how one real implementation (bmcweb) wires it all together end to end
