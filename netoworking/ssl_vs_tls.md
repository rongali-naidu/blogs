
## SSL vs TLS

**SSL** (Secure Sockets Layer) is the predecessor to TLS. SSL 3.0 was the last version before it was renamed/redesigned as TLS 1.0. All SSL versions (1.0, 2.0, 3.0) are now considered insecure and deprecated. TLS is effectively "SSL's successor" — when people say "SSL" today they usually mean TLS.

## TLS 1.0/1.1 vs TLS 1.2 vs TLS 1.3

| Aspect | What it means | TLS 1.0/1.1 | TLS 1.2 | TLS 1.3 |
|--------|---------------|-------------|---------|---------|
| **Status** | Whether the protocol is actively supported and considered safe for use | Deprecated (RFC 8996, 2021) | Current standard, widely deployed | Latest (RFC 8446, 2018) |
| **Handshake** | Number of network round-trips to establish a secure connection before application data can flow | 2-RTT | 2-RTT | 1-RTT (0-RTT resumption available) |
| **Cipher suites** | The combination of algorithms (encryption + MAC + key exchange) both sides agree on during the handshake to protect data | Supports weak ciphers (RC4, DES, 3DES) | Adds AEAD ciphers (AES-GCM, ChaCha20-Poly1305) | AEAD only; just 5 cipher suites total |
| **Key exchange** | How client and server generate a shared secret without transmitting it over the wire; happens early in the handshake | RSA, DHE, ECDHE | RSA, DHE, ECDHE | ECDHE/DHE only — RSA key exchange removed |
| **Hash algorithms** | Cryptographic hash used in the PRF (pseudorandom function) to derive keys, and in digital signatures to authenticate the handshake | MD5/SHA-1 in PRF and signatures | SHA-256+ required | SHA-256/SHA-384 only |
| **Forward secrecy** | Ensures that compromising the server's long-term private key doesn't let an attacker decrypt previously recorded sessions | Optional, rarely negotiated | Supported, commonly negotiated | Mandatory — always on |
| **Removed in this version** | Legacy algorithms/features stripped out to reduce attack surface | — | MD5/SHA-1 from signatures | RSA key exchange, static DH, CBC mode, RC4, DES, 3DES, SHA-1 signatures, compression, renegotiation |
| **BEAST/POODLE** | Known protocol-level attacks that exploit weaknesses in CBC chaining (BEAST) or padding (POODLE) during record-layer encryption | TLS 1.0 vulnerable to BEAST | Not affected | Not affected |
| **0-RTT** | Ability to send encrypted application data in the very first handshake flight on resumed connections, reducing latency to zero round-trips | No | No | Yes (optional, with replay risk) |
| **Encrypted handshake** | Whether the handshake messages themselves (especially the server certificate) are encrypted, hiding server identity from passive eavesdroppers | No — server cert sent in clear | No — server cert sent in clear | Yes — server cert encrypted after ServerHello |
| **Compliance** | Whether the protocol version satisfies regulatory frameworks (PCI DSS, HIPAA, FedRAMP, etc.) | Fails PCI DSS, HIPAA, etc. | Meets current standards | Meets/exceeds all current standards |

**Key differences of TLS 1.3:**

- **Simpler and faster** — reduced cipher suite options from hundreds to 5, eliminating misconfiguration risk. 1-RTT handshake cuts latency.
- **Mandatory forward secrecy** — no RSA key exchange means a compromised private key can't decrypt past traffic.
- **Encrypted handshake** — server certificate is hidden from passive observers (privacy improvement).
- **0-RTT resumption** — allows sending data on the first flight for repeat connections, but carries replay attack risk so it's opt-in and should only be used for idempotent requests.
- **No downgrade attacks** — includes handshake transcript hashing and downgrade sentinel values to detect tampering.

**Practical guidance:** TLS 1.2 is the minimum for production. TLS 1.3 is preferred where supported. Disable 1.0/1.1 entirely.
