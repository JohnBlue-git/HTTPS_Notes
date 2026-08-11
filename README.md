# HTTP / HTTPS / TLS / SSL / OpenSSL Notes

A beginner-to-advanced tutorial covering how the secure web actually works: HTTP itself, the cryptography TLS/SSL is built from, certificates and PKI, the OpenSSL toolkit, and how it all comes together across HTTP/1.0 through HTTP/3. Part 7 grounds all of it in one real, production C++ implementation ([openbmc/bmcweb](https://github.com/openbmc/bmcweb)).

## How to Use This Guide

- **New to HTTPS?** Read Parts 1 → 2 → 4 in order — that's the full conceptual picture.
- **Need to generate or inspect a certificate right now?** Jump to Part 3.
- **Debugging a handshake, or explaining one?** Part 5 has step-by-step diagrams.
- **Hardening a server / reviewing TLS config?** Part 6.
- **Want to see these concepts in real production code?** Part 7 walks through bmcweb.

## Table of Contents

**Part 1 — Foundations**
- [1) HTTP vs HTTPS](#1-http-vs-https)
- [2) What Encryption Solves](#2-what-encryption-solves)
- [3) Symmetric vs Asymmetric Cryptography](#3-symmetric-vs-asymmetric-cryptography)
- [4) Hashing MAC and Digital Signatures](#4-hashing-mac-and-digital-signatures)

**Part 2 — TLS/SSL In Depth**
- [5) SSL vs TLS History and Versions](#5-ssl-vs-tls-history-and-versions)
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

**Part 7 — Applied Case Study: OpenBMC bmcweb**
- [19) PEM Certificate Management via Redfish](#19-pem-certificate-management-via-redfish)
- [20) bmcweb Source Walkthrough](#20-bmcweb-source-walkthrough)

**Appendix**
- [A) Glossary](#a-glossary)
- [B) Quick Command Reference](#b-quick-command-reference)
- [C) Further Reading](#c-further-reading)

---

## 1) HTTP vs HTTPS

### What HTTP Is

HTTP (HyperText Transfer Protocol) is the application-layer protocol browsers and servers use to exchange requests and responses.

- Default port `80`
- Plaintext on the wire — anyone on the network path can read or modify it
- No identity verification — you can't tell if you're really talking to the server you think you are
- Vulnerable to eavesdropping, tampering, and MITM (man-in-the-middle) attacks

### What HTTPS Is

HTTPS is HTTP layered on top of TLS (Transport Layer Security, the modern name for what used to be called SSL — see [§5](#5-ssl-vs-tls-history-and-versions)).

- Default port `443`
- **Confidentiality**: traffic is encrypted, so eavesdroppers only see ciphertext
- **Integrity**: tampering with data in transit is detectable
- **Authentication**: a certificate proves the server is who it claims to be

None of that comes from HTTP itself — it's entirely provided by the TLS layer underneath. That's why Parts 2 and 3 exist: to explain the mechanism HTTPS is actually built on, not just the fact that it's "encrypted."

### One-Line Difference

- HTTP: a postcard — readable by anyone who handles it along the way
- HTTPS: a sealed, tamper-evident envelope, addressed to a verified recipient

---

## 2) What Encryption Solves

TLS exists to provide three properties. Every mechanism in the rest of this guide — key exchange, certificates, MACs — exists to deliver one of these:

| Property | Question it answers | How TLS provides it |
|---|---|---|
| **Confidentiality** | Can anyone else read this? | Symmetric encryption (e.g. AES) of the data |
| **Integrity** | Was this modified in transit? | AEAD ciphers / MAC — a modified ciphertext fails to decrypt/verify |
| **Authentication** | Am I talking to who I think? | Certificates + digital signatures, checked during the handshake |

A useful mental model: **the handshake's entire job is to let two strangers agree on a shared secret key over a public channel, while proving the server's identity, without ever transmitting that secret in a form an eavesdropper could reuse.** Everything in Parts 2–3 is in service of that one sentence.

Note what TLS does *not* do:

- It doesn't protect data once it's decrypted at either endpoint (that's application/OS security).
- It doesn't hide *that* a connection happened, or normally the destination hostname (SNI is visible on the wire unless ECH is used — see [§7](#7-sni-and-alpn-in-the-clienthello)).
- It doesn't verify the server is *trustworthy* or *not malicious* — only that it holds the private key for the identity named in its certificate.

---

## 3) Symmetric vs Asymmetric Cryptography

### Symmetric Encryption

One shared secret key encrypts and decrypts.

- Examples: AES-128/256, ChaCha20
- Fast — used for the actual bulk data (the whole HTTP request/response)
- Problem: both sides need the *same* key before they can talk — how do you deliver that key over a network someone might be watching?

### Asymmetric (Public-Key) Encryption

Two mathematically linked keys: a **public** key (share freely) and a **private** key (never shared).

- Examples: RSA, ECDSA/ECDHE (elliptic curve), Ed25519
- Data encrypted with the public key can only be decrypted with the matching private key (or, for signatures, the reverse: signed with the private key, verified with the public key)
- Solves the key-distribution problem — no shared secret needs to exist beforehand
- Much slower than symmetric encryption (roughly 100–1000x), so it's impractical for bulk data

### Why TLS Uses Both (Hybrid Encryption)

TLS uses asymmetric crypto only to *establish* a shared secret (via certificates + key exchange), then switches to fast symmetric encryption for the actual session:

```text
1. Asymmetric crypto  -> authenticate the server + agree on a shared secret
2. Symmetric crypto   -> encrypt/decrypt all actual HTTP traffic with that secret
```

This is why a cipher suite name like `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` lists both an asymmetric part (`ECDHE`, `RSA`) and a symmetric part (`AES_128_GCM`) — see [§6](#6-cipher-suites-explained).

---

## 4) Hashing MAC and Digital Signatures

### Hash Functions

A hash function takes arbitrary input and produces a fixed-size fingerprint (digest).

- Examples: SHA-256, SHA-384
- Same input always produces the same output; changing even one bit changes the output completely
- One-way: you can't recover the input from the digest
- Used to fingerprint certificates, derive keys, and build MACs — **not** encryption (there's no key, and it's not reversible)

### MAC / HMAC

A Message Authentication Code proves data wasn't altered *and* that the sender knew a shared secret.

- HMAC = a hash function combined with a secret key (e.g. `HMAC-SHA256`)
- Modern TLS mostly uses **AEAD** ciphers (AES-GCM, ChaCha20-Poly1305), which bundle encryption and integrity into one operation instead of encrypt-then-MAC as two separate steps

### Digital Signatures

A signature proves a message came from the holder of a specific private key, and wasn't altered.

```text
Sign:   signature = Encrypt_with_private_key( Hash(message) )
Verify: Hash(message) ==? Decrypt_with_public_key( signature )
```

(Real algorithms like RSA-PSS and ECDSA don't literally "encrypt the hash," but this captures the idea.)

This is exactly how a Certificate Authority signs a certificate: it hashes the certificate's contents and signs that hash with the CA's private key. Anyone with the CA's public key can then verify the certificate hasn't been forged or altered — this is the mechanical basis for the entire chain of trust in [§8](#8-certificates-and-pki).

---

## 5) SSL vs TLS History and Versions

"SSL" and "TLS" are often used interchangeably, but SSL is the deprecated predecessor:

| Version | Year | Status |
|---|---|---|
| SSL 1.0 | — | Never publicly released (fatally flawed) |
| SSL 2.0 | 1995 | Prohibited (RFC 6176, 2011) |
| SSL 3.0 | 1996 | Deprecated (RFC 7568, 2015) — broken by POODLE |
| TLS 1.0 | 1999 | Deprecated (RFC 8996, 2021) |
| TLS 1.1 | 2006 | Deprecated (RFC 8996, 2021) |
| TLS 1.2 | 2008 | Still widely used, secure when configured correctly |
| TLS 1.3 | 2018 | Current standard (RFC 8446), simpler and faster |

Practical takeaways:

- If a product/vendor says "SSL" today, they almost certainly mean TLS — genuine SSL has been unsafe to use for over a decade.
- A modern server should support **TLS 1.2 and TLS 1.3 only**, and reject SSL 2.0/3.0 and TLS 1.0/1.1 entirely (see [§17](#17-hardening-checklist)).
- TLS 1.3 isn't just "TLS 1.2 with a bigger version number" — it removed multiple legacy mechanisms outright (static RSA key exchange, CBC ciphers, custom Diffie-Hellman groups, compression) instead of merely discouraging them. Fewer options means fewer ways to misconfigure it.

---

## 6) Cipher Suites Explained

A cipher suite is the bundle of algorithms negotiated during the handshake. TLS 1.2 names them as four parts:

```text
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 |    |     |         |         |
 |    |     |         |         +-- Hash used for the handshake's HMAC/PRF
 |    |     |         +------------ Bulk symmetric cipher (AES-128 in GCM/AEAD mode)
 |    |     +---------------------- Authentication: server proves identity via RSA signature
 |    +---------------------------- Key exchange: Ephemeral Elliptic-Curve Diffie-Hellman
 +--------------------------------- Protocol
```

- **Key exchange** (`ECDHE`, `DHE`, or legacy static `RSA`) — how the shared secret is derived. `ECDHE`/`DHE` provide **Perfect Forward Secrecy (PFS)**: even if the server's long-term private key leaks later, past sessions can't be decrypted, because each session used a fresh, ephemeral key that was never stored. Static `RSA` key exchange has no PFS — one leaked key compromises every past session ever recorded — which is why TLS 1.3 removed it entirely.
- **Authentication** (`RSA`, `ECDSA`) — the algorithm behind the signature that proves the server holds the certificate's private key.
- **Bulk cipher** (`AES_128_GCM`, `AES_256_GCM`, `CHACHA20_POLY1305`) — should always be an **AEAD** (Authenticated Encryption with Associated Data) cipher, which provides confidentiality and integrity together. Avoid CBC-mode ciphers (`AES_128_CBC`) and stream ciphers like RC4 — both have known historical attacks (see [§16](#16-historical-attacks-and-why-modern-defaults-exist)).
- **Hash** (`SHA256`, `SHA384`) — used inside the handshake's key derivation, not for the bulk data itself.

TLS 1.3 simplified this. Key exchange is *always* (EC)DHE (no field for it in the name anymore), and cipher suites only name the AEAD cipher + hash:

```text
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
```

Authentication algorithm is negotiated separately via the certificate type and the `signature_algorithms` extension.

Check what your local OpenSSL supports:

```bash
openssl ciphers -v 'ALL'          # every cipher suite OpenSSL knows
openssl ciphers -v 'HIGH:!aNULL'  # a reasonably strong subset
```

---

## 7) SNI and ALPN in the ClientHello

Before any encryption is set up, the very first message of a handshake — the `ClientHello` — already carries two important, plaintext extensions.

### SNI (Server Name Indication)

Problem: one server can host many HTTPS domains on a single IP address, each with a *different* certificate. But the server has to pick which certificate to present before the handshake gets far enough to know what the client asked for at the HTTP level (the `Host` header is still encrypted at this point).

- SNI solves this: the client includes the target hostname, in plaintext, in the `ClientHello`
- The server reads it and selects the matching certificate before continuing the handshake
- Trade-off: the hostname is visible to anyone observing the connection (network devices, ISPs) even though everything after the handshake is encrypted
- Newer mitigation: **ECH (Encrypted Client Hello)**, a TLS 1.3 extension that encrypts SNI too — supported by some browsers/CDNs but not yet universal

### ALPN (Application-Layer Protocol Negotiation)

Problem: a client and server need to agree on which application protocol (HTTP/1.1, HTTP/2...) to speak, without extra round trips.

- The client lists supported protocols in its `ClientHello` (e.g. `[h2, http/1.1]`)
- The server picks one and returns it in the `ServerHello`
- One `443` connection can transparently serve HTTP/1.1 or HTTP/2 clients — no separate "upgrade" step needed

```text
ClientHello (SNI: example.com, ALPN: [h2, http/1.1])
            |
            v
ServerHello (ALPN selected: h2)
            |
            v
Use HTTP/2 frames on this connection
```

Relationship to HTTP versions:

- HTTP/1.0 and early HTTP/1.1 deployments predate ALPN and don't rely on it
- HTTP/2 over TLS is negotiated as `h2` via ALPN in virtually all real deployments (the spec technically allows unencrypted `h2c`, but browsers only support `h2` over TLS)
- HTTP/3 runs over QUIC, which embeds TLS 1.3 directly; negotiation still happens via ALPN, using the identifier `h3`

---

## 8) Certificates and PKI

A certificate binds a public key to an identity (a hostname), and is itself signed by someone else vouching for that binding. That's Public Key Infrastructure (PKI).

### X.509 Certificate Structure

The standard certificate format is X.509. Key fields:

- **Subject** — who this certificate identifies (e.g. `CN=example.com`)
- **Subject Alternative Name (SAN)** — the actual list of hostnames the cert is valid for; modern clients ignore the `CN` field entirely and only check SAN
- **Issuer** — which CA signed this certificate
- **Validity** — `Not Before` / `Not After` dates
- **Subject Public Key Info** — the public key this certificate vouches for
- **Extensions** — `Key Usage`, `Extended Key Usage`, `Basic Constraints` (is this a CA?), `Authority Key Identifier`, OCSP/CRL locations
- **Signature** — the issuer's signature over everything above

View a real one:

```bash
openssl x509 -in server.crt -noout -text
```

### Chain of Trust

Certificates are verified as a chain, not in isolation:

```text
Root CA (self-signed, pre-installed in OS/browser trust store)
   |
   |  signs
   v
Intermediate CA
   |
   |  signs
   v
Leaf / server certificate (example.com)
```

- **Root CAs** are kept offline as much as possible — if a root key leaks, every certificate it ever issued is suspect.
- **Intermediate CAs** do the actual day-to-day signing, so a compromise is contained to certs *that intermediate* issued, and the intermediate can be revoked without touching the root.
- **Self-signed certificates** have no chain — nobody vouches for them but the server itself. Fine for local development or closed internal systems (like a BMC's default HTTPS certificate, see [§19](#19-pem-certificate-management-via-redfish)); clients will show a trust warning because nothing in their trust store leads back to it.

### How a Client Actually Validates a Certificate

1. Build a chain from the leaf certificate up to a root the client already trusts
2. Verify each signature in the chain (each cert really was signed by the next one up)
3. Check the current date falls within every certificate's validity window
4. Check the requested hostname matches an entry in the leaf's SAN list
5. Check revocation status — CRL (Certificate Revocation List) or OCSP (Online Certificate Status Protocol); **OCSP stapling** lets the server fetch and attach this proof itself, avoiding an extra client round trip and a privacy leak (the CA would otherwise learn every site you visit)
6. Check `Basic Constraints`/`Key Usage` extensions are consistent with each cert's role (e.g. only a CA-flagged cert can sign other certs)

If any step fails, the connection should be rejected — not silently downgraded to unencrypted.

### Getting a Certificate

- **Self-signed**: you sign your own certificate. No external trust, useful for internal/test systems.
- **Private/internal CA**: your organization runs its own CA and installs its root into your own devices' trust stores. Common for internal infrastructure and IoT/BMC fleets.
- **Public CA**: a CA already trusted by browsers/OSes signs it (e.g. via **Let's Encrypt**, using the automated **ACME** protocol — free, domain-validated, 90-day certificates renewed automatically).

### Certificate Transparency (CT)

Public CAs are required to publish every certificate they issue to public, append-only CT logs. This lets domain owners detect if a CA mis-issued a certificate for their name, and modern browsers require a valid CT proof (SCT) before trusting a public certificate.

---

## 9) The TLS Handshake Step by Step

The goal, restated from [§2](#2-what-encryption-solves): authenticate the server and derive a shared symmetric key, using only messages sent over a public, unencrypted-so-far connection.

### TLS 1.2 Full Handshake

```text
Client                                             Server
  | -- ClientHello --------------------------------> |
  |    (client random, cipher suites, SNI, ALPN,     |
  |     supported_groups, key_share candidates)      |
  |                                                  |
  | <- ServerHello --------------------------------- |
  | <- Certificate --------------------------------- |
  | <- ServerKeyExchange (ECDHE params, signed) ---- |
  | <- ServerHelloDone ----------------------------- |
  |                                                  |
  | -- ClientKeyExchange (ECDHE params) -----------> |
  | -- [ChangeCipherSpec] -------------------------> |
  | -- Finished (encrypted) -----------------------> |
  |                                                  |
  | <- [ChangeCipherSpec] -------------------------- |
  | <- Finished (encrypted) ------------------------ |
  |                                                  |
  | === Application Data (encrypted, both ways) ==== |
```

What happens at each stage:

1. **ClientHello** — client proposes TLS versions, cipher suites, a random nonce, and extensions (SNI, ALPN, supported groups for key exchange)
2. **ServerHello** — server picks the version/cipher suite, sends its own random nonce
3. **Certificate** — server sends its cert chain
4. **ServerKeyExchange** — server sends its ephemeral (EC)DHE public value, signed with its certificate's private key (this signature is what ties the ephemeral key exchange to a verified identity)
5. Client verifies the certificate chain (see [§8](#8-certificates-and-pki)) and the signature, then sends its own ephemeral key share
6. Both sides now independently compute the same shared secret via Diffie-Hellman math, and derive symmetric session keys from it
7. **Finished** messages (the first encrypted messages) let each side confirm the other derived the same keys, and that nothing in the handshake was tampered with
8. Application data flows, encrypted with the negotiated AEAD cipher

This costs roughly **2 round trips** before the first byte of application data.

### Abbreviated Handshake (Session Resumption)

Repeating the full handshake for every connection to a site you just visited is wasteful. TLS caches enough state to skip most of it:

- **Session ID** — server keeps session state, client just reminds it of the ID
- **Session Tickets** (RFC 5077) — server encrypts its session state into a ticket and hands it to the client; no server-side storage needed, so this scales better

```text
Client                                Server
  | -- ClientHello + session ticket --> |
  | <- ServerHello (resumed) ---------- |
  | <- [ChangeCipherSpec], Finished --- |
  | -- [ChangeCipherSpec], Finished --> |
  | ======== Application Data ========  |
```

This cuts the handshake to **1 round trip**, and skips certificate verification and the asymmetric key exchange entirely (the original session's derived secret is reused to derive new keys).

TLS 1.3's handshake and 0-RTT resumption are different enough to deserve their own diagram — see [§14](#14-tls-13-handshake-and-0-rtt).

### mTLS (Mutual TLS)

Everything above authenticates the *server* to the client. Some deployments (service-to-service APIs, BMC-to-BMC management traffic, zero-trust internal networks) also need the *client* authenticated:

- Server sends an additional `CertificateRequest` message
- Client responds with its own `Certificate` + a `CertificateVerify` signature proving it holds that certificate's private key
- Both directions are now cryptographically authenticated

---

## 10) Certificate and Key File Formats

The same certificate or key can be stored in several container formats. Confusing them is one of the most common real-world TLS headaches.

### PEM (Privacy-Enhanced Mail)

Text-based, Base64-encoded, delimited by header/footer lines. By far the most common format on Linux/OpenSSL-based systems.

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

- Human-readable-ish, diffable, easy to concatenate — a PEM "chain" file is just multiple `BEGIN/END CERTIFICATE` blocks back to back
- A single `.pem` file can also bundle a certificate **and** its private key together (two blocks, one file) — bmcweb's `server.pem` does exactly this, see [§20](#20-bmcweb-source-walkthrough)
- Common extensions: `.pem`, `.crt`, `.cer`, `.key` — the extension is a *convention*, not a guarantee of contents; always inspect with `openssl x509 -text` or `openssl pkey -text` if unsure

### DER (Distinguished Encoding Rules)

The binary encoding of the same underlying ASN.1 structure PEM wraps in Base64.

- Not human-readable, but more compact
- Common on Windows/Java systems, and in some embedded contexts
- Extensions: `.der`, sometimes also `.cer`

### PKCS#7 / P7B

A bundle of certificates (a chain), but **never a private key**.

- Used mostly on Windows/Java for distributing a cert + intermediate chain together
- Extensions: `.p7b`, `.p7c`

### PKCS#12 / PFX

A single binary, password-protected container bundling a certificate, its private key, and optionally the chain — everything needed to stand up a server in one file.

- Common for importing into Windows/Java keystores or browsers
- Extensions: `.p12`, `.pfx`

### CSR (Certificate Signing Request, PKCS#10)

Not a certificate — a *request* for one.

- Contains your public key + identity info (CN, SAN, org), signed with your own private key to prove you hold it
- You send this to a CA; the CA validates it and returns a signed certificate
- Extension: `.csr`

### Quick Reference

| Extension | Typically contains | Format |
|---|---|---|
| `.pem` | cert, key, chain, or CSR (any of the above) | Text/Base64 |
| `.crt` / `.cer` | certificate | Usually PEM, sometimes DER |
| `.key` | private key | Usually PEM |
| `.csr` | signing request | Text/Base64 |
| `.der` | certificate or key | Binary |
| `.p7b` / `.p7c` | cert chain, no key | Binary |
| `.p12` / `.pfx` | cert + key + chain, password-protected | Binary |

When in doubt, don't trust the extension — open it with the commands in [§11](#11-openssl-practical-cheat-sheet).

---

## 11) OpenSSL Practical Cheat Sheet

OpenSSL is both a crypto library (`libssl` for the TLS protocol, `libcrypto` for the underlying primitives) and the `openssl` command-line tool built on top of it — the tool is what this section covers.

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

`-addext subjectAltName` matters: modern clients ignore `CN` and reject certificates that lack a matching SAN entry (see [§8](#8-certificates-and-pki)).

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

Compares the public key embedded in each — works for RSA and EC alike:

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

| Symptom | Likely cause |
|---|---|
| `unable to get local issuer certificate` | Intermediate chain not sent by the server, or not passed to `verify` |
| Server fails to start / "key values mismatch" | Cert and key files don't actually pair — check with the pubkey-hash comparison above |
| Browser shows "self-signed" warning | Working as intended — no CA vouches for this cert (see [§8](#8-certificates-and-pki)) |
| `certificate has expired` | Check `-dates` output; renew |
| Hostname mismatch warning despite a valid cert | Cert's SAN list doesn't include the hostname you connected with |

---

## 12) HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.0 (1996, RFC 1945)

- One request/response per TCP connection; connection typically closes after
- No mandatory `Host` header — one IP could really only cleanly serve one site
- No header compression; every request repeats them in full

Impact: multi-resource pages (HTML + CSS + JS + images) pay a new TCP (and, over HTTPS, a new TLS) handshake for almost every request.

### HTTP/1.1 (1997, revised 2014 as RFC 7230–7235)

The version most of the web still runs on underneath HTTP/2's shadow.

- **Persistent connections by default** — one TCP connection can carry many requests
- **Mandatory `Host` header** — enables name-based virtual hosting (many domains, one IP)
- **Chunked transfer encoding** — a response body can stream without knowing its total length up front
- **Pipelining** — technically allows sending multiple requests without waiting for each response, but suffers from head-of-line blocking (one slow response blocks everything queued behind it) and is effectively unused by real browsers

### HTTP/2 (2015, RFC 7540)

- **Binary framing** instead of text parsing
- **Multiplexing** — many concurrent streams over a single connection, solving HTTP/1.1's "open 6 connections per host" workaround
- **Header compression (HPACK)** — headers are compressed and diffed against previous requests
- **Server Push** — server can proactively send resources it expects the client will need; in practice this has been deprecated and removed from most browsers (e.g. Chrome removed it in 2022) due to poor real-world cache efficiency
- Requires TLS in practice — no major browser supports HTTP/2 over plaintext (`h2c`)
- Still suffers **TCP-level** head-of-line blocking: one lost packet stalls *all* multiplexed streams, because they all share one TCP byte stream

### HTTP/3 (2022, RFC 9114, over QUIC RFC 9000)

- Runs over **QUIC** (UDP-based) instead of TCP
- TLS 1.3 is built into the transport handshake itself, not layered on top
- Each stream is independently reliable — a lost packet only stalls the one stream it belongs to, fixing HTTP/2's remaining head-of-line blocking
- **Connection migration** — a connection survives a network change (e.g. Wi-Fi to cellular) because it's identified by a Connection ID, not the IP/port tuple
- Can offer **0-RTT** reconnection to a previously-visited server (with the same replay caveats as TLS 1.3 0-RTT — see [§14](#14-tls-13-handshake-and-0-rtt))

### Comparison

| | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|---|
| Transport | TCP | TCP | TCP | QUIC (UDP) |
| Connections needed | Many | Fewer (persistent) | One (multiplexed) | One (multiplexed) |
| Header compression | No | No | HPACK | QPACK |
| Head-of-line blocking | Yes (severe) | Yes | Transport-level only | No |
| TLS integration | Separate layer | Separate layer | Separate layer, via ALPN | Built into transport |

Which version is actually used on a given connection is decided by ALPN during the TLS handshake (or QUIC's equivalent) — see [§7](#7-sni-and-alpn-in-the-clienthello).

---

## 13) TCP and TLS 1.2 Handshake

Quick reference combining both layers — the conceptual step-by-step is in [§9](#9-the-tls-handshake-step-by-step).

```text
Client                                Server
  | -------- SYN ----------------------> |
  | <----- SYN-ACK --------------------- |
  | -------- ACK ----------------------> |   (TCP connected)
  | -------- ClientHello --------------> |
  | <------- ServerHello + Cert -------- |
  | -------- Key Exchange/Finished ----> |
  | <------- Finished ------------------ |   (TLS established)
  | ======== HTTP Request (encrypted) ==>|
  | <===== HTTP Response (encrypted) ====|
```

Key points:

- TCP and TLS are two independent handshakes, stacked — TCP guarantees ordered delivery, TLS adds confidentiality/integrity/authentication on top
- HTTP/1.1 and HTTP/2 both commonly run on exactly this stack; ALPN during the TLS handshake decides which of the two the connection will speak

---

## 14) TLS 1.3 Handshake and 0-RTT

TLS 1.3 cuts the handshake from 2 round trips to 1 by having the client guess a key exchange group and send its key share in the very first message, instead of waiting to be told which group the server wants.

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
  | <- Finished -------------------------------------- |
  |                                                    |
  | -- Finished -------------------------------------> |
  | ====== Application Data (both directions) ======== |
```

- The server can derive the shared secret as soon as it sees the client's `key_share`, so almost everything after `ServerHello` — certificate included — is already encrypted
- Client can send application data immediately after its own `Finished` — encrypted app data goes out on the **same flight** as the handshake's last message
- Static RSA key exchange no longer exists in TLS 1.3 — every handshake uses (EC)DHE, so every session gets Perfect Forward Secrecy by default (see [§6](#6-cipher-suites-explained))

### 0-RTT Resumption (and Its Trade-off)

If the client has a session ticket (PSK) from a previous connection to the same server, it can skip straight to sending encrypted application data in its very first flight:

```text
Client                                 Server
  | -- ClientHello + PSK + early_data --> |
  | ==== 0-RTT Application Data ========> |
  | <- ServerHello + Finished ----------- |
  | <==== Application Data ===============|
```

- Zero round trips before the client's first request is *sent* — a real latency win for repeat visits
- **Caveat: 0-RTT data is not forward-secret and is replayable.** Nothing in it is tied to a fresh, unpredictable value the server generates, so a network attacker who captures a 0-RTT request can resend it, and the server has no built-in way to tell it apart from the original. This is why 0-RTT is recommended only for idempotent operations (safe `GET`s), never for anything with a side effect (payments, form submissions).

---

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

- QUIC folds the transport handshake and the TLS 1.3 handshake into one exchange — there's no separate "TCP connect, then TLS handshake" phase
- Each HTTP/3 request/response runs on its own independently-reliable QUIC stream, so one dropped packet only stalls the stream it belongs to
- QUIC connections are identified by a Connection ID rather than the traditional (source IP, source port, dest IP, dest port) tuple, which is what makes connection migration possible

---

## 16) Historical Attacks and Why Modern Defaults Exist

This is why the "just use modern defaults" advice in [§17](#17-hardening-checklist) exists — every one of today's defaults was a direct response to a real, named attack. (Educational summary only, for understanding *why* these defaults exist — not exploit instructions.)

| Attack | Year | Root cause | Why it no longer matters if you follow §17 |
|---|---|---|---|
| **BEAST** | 2011 | Predictable IVs in TLS 1.0 CBC-mode ciphers | Fixed in TLS 1.1+; avoid CBC, prefer AEAD |
| **CRIME / BREACH** | 2012/13 | Compression ratio leaks secrets in the compressed stream | TLS-level compression is disabled by default; be careful compressing responses containing secrets |
| **Heartbleed** | 2014 | Buffer over-read bug in OpenSSL's heartbeat extension implementation | A library bug, not a protocol flaw — patched; the lesson was to rotate keys/certs after any such disclosure, since past traffic *could* have been exposed |
| **POODLE** | 2014 | Padding oracle in SSL 3.0's CBC mode | Disable SSL 3.0 entirely (it offers no real compatibility benefit today) |
| **Downgrade attacks** | ongoing | Attacker forces a handshake to negotiate a weaker, breakable version/cipher | `TLS_FALLBACK_SCSV`, refusing to offer legacy versions at all, and HSTS (prevents even the first connection from trying plain HTTP) |
| **ROBOT** | 2017 | Revival of a 1998 Bleichenbacher padding oracle against RSA key exchange | Avoid static RSA key exchange — prefer ECDHE, which TLS 1.3 makes mandatory |

The common thread: nearly every one of these targeted either an *optional legacy mechanism* (SSL 3.0, CBC ciphers, static RSA key exchange) that a fully modern configuration simply doesn't offer, or a specific implementation bug rather than the protocol itself. This is also why TLS 1.3 was designed by deleting options rather than adding a flag to disable them.

---

## 17) Hardening Checklist

- **Protocol versions**: support TLS 1.2 and TLS 1.3 only. Disable SSL 2.0/3.0 and TLS 1.0/1.1 explicitly — don't just rely on "not offering" them by default, since some stacks still enable them.
- **Cipher suites**: allow AEAD ciphers only (`*_GCM`, `*_CHACHA20_POLY1305`). Disable CBC-mode suites, RC4, 3DES, and anything with `NULL` or `EXPORT` in the name.
- **Key exchange**: prefer ECDHE over static RSA key exchange, for Perfect Forward Secrecy (automatic under TLS 1.3).
- **Key size/type**: RSA ≥ 2048 bits (3072–4096 for long-lived keys); EC P-256 or P-384.
- **Certificates**: always include a correct SAN list; keep validity periods short where automation allows (Let's Encrypt's 90-day default forces renewal automation, which is itself a resilience win).
- **HSTS** (`Strict-Transport-Security` header): tells browsers to never attempt plain HTTP for this host again, closing the window for downgrade/SSL-stripping attacks on subsequent visits.
- **OCSP stapling**: server fetches its own revocation proof and attaches it to the handshake — faster and more private than the client querying the CA directly.
- **Automate renewal**: expired certificates are one of the most common self-inflicted outages; ACME-based renewal (certbot, etc.) removes the human step.
- **Protect private keys**: restrictive file permissions, avoid transmitting keys over any unencrypted channel (including internal ones), rotate immediately after any suspected exposure.
- **Test the actual deployed configuration**, not just the intent — see [§18](#18-testing-and-diagnostic-tools).

---

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

- `Protocol` / `New, TLSv1.x` line — confirms which TLS version was actually negotiated
- `ALPN, server accepted to use h2` — confirms HTTP/2 was selected
- The response status line (`HTTP/1.1 ...` vs `HTTP/2 ...`) confirms it from the HTTP side too

Other tools worth knowing about (mentioned for awareness, not covered in depth here):

- **testssl.sh** — scripted, comprehensive scan of a server's protocol/cipher/vulnerability posture
- **nmap `ssl-enum-ciphers` script** — enumerates supported ciphers per TLS version
- **Qualys SSL Labs' SSL Test** — a hosted scanner producing a letter-grade report for any public server
- Browser DevTools → Security tab — shows the negotiated protocol/cipher for the current page

---

## 19) PEM Certificate Management via Redfish

Everything above is now applied to one real system: [openbmc/bmcweb](https://github.com/openbmc/bmcweb), the HTTP server used by OpenBMC-based BMC firmware. This section covers the Redfish-facing certificate management API; [§20](#20-bmcweb-source-walkthrough) covers the C++ implementation underneath it.

This maps to bmcweb's `redfish-core/lib/certificate_service.hpp`:

- HTTPS certificate collection: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/`
- Single HTTPS certificate: `/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/{id}`
- Generic replace action: `/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/`

Backend constants in bmcweb:

- D-Bus service: `xyz.openbmc_project.Certs.Manager.Server.Https`
- D-Bus object base path: `/xyz/openbmc_project/certs/server/https`

### 1) Redfish Operation Path (Recommended)

1. Check the current HTTPS certificate list:

```bash
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

2. Upload a new certificate to the HTTPS collection (POST) — the `CertificateString` is PEM content, see [§10](#10-certificate-and-key-file-formats):

```bash
curl -k -u root:0penBmc \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "CertificateString": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }' \
  https://<bmc>/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/
```

3. Or use the `ReplaceCertificate` action to target a specific existing certificate:

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

- In most cases, install/replace through the certificate manager service takes effect automatically.
- If you still see the old certificate, restart the HTTPS service on the BMC shell:

```bash
systemctl restart bmcweb.service
```

- Verify the new certificate is active — this is the `openssl s_client` pattern from [§18](#18-testing-and-diagnostic-tools), applied to the BMC itself:

```bash
openssl s_client -connect <bmc>:443 -servername <bmc> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

## 20) bmcweb Source Walkthrough

Source: [github.com/openbmc/bmcweb](https://github.com/openbmc/bmcweb) (main branch). These are the actual points in the codebase where the concepts from Parts 2 and 4 are implemented in C++ with Boost.Asio.

### A. TLS Implementation: Connection Layer vs PEM Loading

`http/http_connection.hpp` does **not** directly parse PEM — it only handles connection-level TLS flow (SSL detection, handshake, ALPN routing). PEM loading happens separately, during SSL context initialization in `src/ssl_key_handler.cpp`.

#### A-1) Connection Layer: TLS Detect + Handshake

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

This is the practical form of the TCP-then-TLS diagram from [§13](#13-tcp-and-tls-12-handshake): a new connection runs `async_detect_ssl` to distinguish TLS from plaintext, then — if TLS — performs the non-blocking handshake covered conceptually in [§9](#9-the-tls-handshake-step-by-step).

#### A-2) PEM Loading: Building the SSL Context

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

- bmcweb loads the certificate context once at server startup via `loadCertificate()`
- `getSslServerContext()` prepares and validates `/etc/ssl/certs/https/server.pem` — note it's a single combined PEM file containing both certificate and key, one of the formats from [§10](#10-certificate-and-key-file-formats)
- PEM data is loaded with `use_certificate_chain` and `use_private_key(..., pem)` — this is the C++/OpenSSL API equivalent of what `openssl x509`/`openssl pkey` inspect from the command line

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

### B. ALPN: Selecting HTTP/2

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

This is the server side of the ALPN negotiation from [§7](#7-sni-and-alpn-in-the-clienthello) — nghttp2 picks `h2` if the client offered it.

### C. After the Handshake: Route to HTTP/2 if ALPN Selected It

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

This is the actual branch point: `h2` routes to `HTTP2Connection`; anything else (`http/1.1`, or no ALPN at all) falls through to the HTTP/1.x header parser.

### D. HTTP/1.x Handling: Version Check and Keep-Alive

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

bmcweb doesn't split HTTP/1.0 and HTTP/1.1 into separate handlers — it reads `req->version()` and `req->keepAlive()` and relies on Boost.Beast's semantics, so both versions from [§12](#12-http10-vs-http11-vs-http2-vs-http3) are handled by the same code path; connection persistence follows whatever the request itself signals.

### E. h2c Support: Upgrading Plaintext HTTP/1.1 to HTTP/2

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

```cpp
if (res.result() == boost::beast::http::status::switching_protocols)
{
  upgradeToHttp2();
  return;
}
```

Plaintext HTTP can also move to `h2c` via the `Upgrade` header mechanism — the server replies `101 Switching Protocols`, then switches into `HTTP2Connection`. (As noted in [§12](#12-http10-vs-http11-vs-http2-vs-http3), no major browser actually does this over plaintext, but the code path exists for non-browser clients.)

### F. HTTP/2 Internals: Frame Callback to Response

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

nghttp2 receives `HEADERS`/`DATA` frames and treats `END_STREAM` as request-complete; the existing application handler generates a response, which nghttp2 re-encodes back into HTTP/2 frames — the concrete version of the binary framing/multiplexing described in [§12](#12-http10-vs-http11-vs-http2-vs-http3).

### G. Flow Diagrams

**HTTPS + ALPN routing:**

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

**HTTP/1.1 h2c upgrade path:**

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

### H. Quick Test Commands

```bash
# Test HTTP/1.0
curl -v --http1.0 https://<bmc-host>/redfish/v1

# Test HTTP/1.1
curl -v --http1.1 https://<bmc-host>/redfish/v1

# Test HTTP/2 (TLS ALPN)
curl -v --http2 https://<bmc-host>/redfish/v1
```

What to check (same signals as [§18](#18-testing-and-diagnostic-tools)):

- `ALPN, server accepted to use h2` (HTTP/2)
- Response start line shows `HTTP/1.1 ...` or `HTTP/2 ...`

---

## A) Glossary

| Term | Meaning |
|---|---|
| **TLS** | Transport Layer Security — the current protocol; see [§5](#5-ssl-vs-tls-history-and-versions) |
| **SSL** | Secure Sockets Layer — TLS's deprecated predecessor, name still used colloquially |
| **PKI** | Public Key Infrastructure — the system of CAs, certificates, and trust chains |
| **CA** | Certificate Authority — an entity that signs certificates, vouching for identity |
| **Root CA** | A CA whose certificate is self-signed and pre-installed in trust stores |
| **Intermediate CA** | A CA whose certificate is itself signed by a root, used for day-to-day signing |
| **CSR** | Certificate Signing Request — sent to a CA to obtain a signed certificate |
| **SAN** | Subject Alternative Name — the hostname list a certificate is actually valid for |
| **CN** | Common Name — legacy identity field, no longer trusted by modern clients (use SAN) |
| **PEM** | Base64 text container format for certs/keys, delimited by `-----BEGIN/END-----` |
| **DER** | Binary encoding of the same certificate/key structures PEM wraps in text |
| **Cipher suite** | The bundle of key exchange, authentication, cipher, and hash algorithms negotiated for a session |
| **AEAD** | Authenticated Encryption with Associated Data — a cipher mode giving confidentiality + integrity together (e.g. AES-GCM) |
| **ECDHE** | Elliptic-Curve Diffie-Hellman, Ephemeral — a key exchange method providing forward secrecy |
| **Forward Secrecy** | Property that a leaked long-term key can't be used to decrypt past recorded sessions |
| **ALPN** | Application-Layer Protocol Negotiation — picks HTTP/1.1 vs h2 vs h3 during the handshake |
| **SNI** | Server Name Indication — the plaintext hostname sent early in the handshake so the server can pick a certificate |
| **ECH** | Encrypted Client Hello — a newer extension that encrypts SNI too |
| **HSTS** | HTTP header that forces browsers to only ever use HTTPS for a host |
| **OCSP** | Online Certificate Status Protocol — real-time certificate revocation check |
| **OCSP stapling** | Server-side fetches its own OCSP proof and attaches it, avoiding a client round trip |
| **CRL** | Certificate Revocation List — an alternative, list-based revocation mechanism |
| **mTLS** | Mutual TLS — both client and server present certificates and authenticate each other |
| **0-RTT** | "Zero round trip" data sent alongside a TLS 1.3 resumption handshake, before it completes |
| **MITM** | Man-in-the-middle — an attacker positioned between client and server on the network path |

---

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

## Quick Conclusion

- HTTPS = HTTP + TLS; TLS's entire job is confidentiality, integrity, and authentication ([§2](#2-what-encryption-solves))
- Modern TLS means TLS 1.2/1.3 only, ECDHE key exchange, AEAD ciphers — every legacy alternative exists because of a named historical attack ([§16](#16-historical-attacks-and-why-modern-defaults-exist))
- A certificate is only as trustworthy as the chain behind it — self-signed is fine for closed systems, not for anything public ([§8](#8-certificates-and-pki))
- OpenSSL's CLI covers generation, inspection, conversion, and live testing with a small, memorable set of commands ([§11](#11-openssl-practical-cheat-sheet), [Appendix B](#b-quick-command-reference))
- HTTP/2 and HTTP/3 mainly attack connection/concurrency bottlenecks left over from HTTP/1.x, not security — but HTTP/3 folds TLS 1.3 directly into its transport handshake ([§12](#12-http10-vs-http11-vs-http2-vs-http3), [§14](#14-tls-13-handshake-and-0-rtt), [§15](#15-http3-over-quic))
- Whether a given backend "supports" all of this depends on the application server, TLS library, and any reverse proxy in front of it acting together — [§19](#19-pem-certificate-management-via-redfish)–[§20](#20-bmcweb-source-walkthrough) show exactly how one real implementation (bmcweb) wires it up end to end
