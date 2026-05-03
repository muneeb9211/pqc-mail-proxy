# PQC Mail Proxy

[![Rust](https://img.shields.io/badge/rust-stable-orange.svg)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/License-Proprietary-red.svg)](#license)
[![ML-KEM-1024](https://img.shields.io/badge/KEM-ML--KEM--1024-green.svg)](#cryptographic-design)
[![NIST Level 5](https://img.shields.io/badge/NIST-Level%205-brightgreen.svg)](#standards-compliance)

A transparent **post-quantum secure** email encryption proxy written in **~10,600 lines of Rust** across 58 source files. It operates between standard email clients (Thunderbird, Outlook, Apple Mail) and mail servers, providing **end-to-end encryption** using NIST-standardized post-quantum algorithms — without requiring any modifications to the email client or server infrastructure.

> **Note:** The source code is currently kept private as a manuscript describing the design and implementation is in preparation for publication. This repository provides the full architectural overview, protocol specification, and cryptographic design. For access to the codebase (e.g., for academic review, peer evaluation, or collaboration), please contact **amarsohail@protonmail.com**.

---

## Motivation

Current email encryption standards (PGP, S/MIME) rely on RSA and elliptic-curve cryptography, both of which are vulnerable to cryptanalysis by sufficiently powerful quantum computers running Shor's algorithm [[1]](#references). NIST has finalized ML-KEM (FIPS 203) as the post-quantum key encapsulation standard, but no existing solution combines it with a **ratcheting protocol** to provide forward secrecy for asynchronous email communication.

This project addresses that gap: it replaces the classical X3DH key agreement [[2]](#references) with **ML-KEM-1024 encapsulation** and implements a **post-quantum Double Ratchet** [[3]](#references) that provides both **forward secrecy** and **break-in recovery** — security properties that existing PQ email encryption tools do not offer together.

### Key Contributions

- **PQ-initialized Double Ratchet** — Replaces X3DH's ECDH handshake with ML-KEM-1024 encapsulation/decapsulation for session bootstrap, achieving post-quantum security from the first message
- **Transparent proxy architecture** — Zero modification required to email clients or servers; works with any SMTP/IMAP/POP3 implementation
- **Three transport modes** — Fan-Out (per-recipient encryption), SCMW (multi-recipient efficiency), and Offload (large payload external storage) to handle real-world email patterns
- **Device-sealed key management** — Private keys stored in OS-native hardware-backed keystores (macOS Keychain, Windows DPAPI, Linux libsecret) with memory locking and zeroization

---

## Cryptographic Design

| Function | Algorithm | Standard | Purpose |
|----------|-----------|----------|---------|
| KEM | **ML-KEM-1024** | NIST FIPS 203 | Post-quantum key encapsulation (Level 5) |
| AEAD | **AES-256-GCM** | NIST SP 800-38D | Authenticated message encryption |
| KDF | **HKDF-SHA-512** | RFC 5869 | Key derivation from shared secrets |
| Hash | **SHA-256** | FIPS 180-4 | User identity hashing, payload validation |
| RNG | **OS CSPRNG + HKDF** | — | Deterministic per-context expansion |

### Security Properties

| Property | Mechanism |
|----------|-----------|
| **Forward Secrecy** | Per-message keys derived via symmetric ratchet; old keys securely erased |
| **Break-in Recovery** | Periodic ML-KEM ratchet restores security after temporary compromise |
| **Post-Quantum Security** | ML-KEM-1024 provides NIST Level 5 resistance to quantum attacks |
| **Zero-Knowledge Backend** | Key server stores only ciphertext and hashes; never sees plaintext |
| **Secure Key Storage** | OS-native keystore with memory locking (`mlock`/`VirtualLock`) and zeroization |

---

## Architecture

```
                           PQC Mail Proxy
  ┌──────────────┐     ┌──────────────────────┐     ┌───────────────┐
  │ Email Client  │────>│  SMTP Proxy  (:1025)  │────>│  SMTP Server   │
  │ (Thunderbird, │     │  Encrypt outbound     │     │  (Gmail, etc.) │
  │  Outlook,     │     ├──────────────────────┤     ├───────────────┤
  │  Apple Mail)  │<────│  IMAP Proxy  (:1143)  │<────│  IMAP Server   │
  │               │     │  Decrypt inbound      │     │               │
  │               │<────│  POP3 Proxy  (:1100)  │<────│  POP3 Server   │
  └──────────────┘     │  Decrypt inbound      │     └───────────────┘
                       └──────────┬───────────┘
                                  │
                       ┌──────────▼───────────┐
                       │    FS-API Server       │
                       │  Pre-key distribution  │
                       │  Device registration   │
                       │  OTPK claim / rotate   │
                       └────────────────────────┘
```

---

## Workspace Structure

The project follows a modular Rust workspace architecture, separating concerns into five crates:

```
pqc-mail-proxy/
│
├── lettuce-core/             # Deliverable 1: Core Cryptographic Engine
│   ├── pq.rs                 # ML-KEM-1024 key generation, encapsulation, decapsulation
│   ├── sym.rs                # AES-256-GCM authenticated encryption
│   ├── kdf.rs                # HKDF-SHA-512 key derivation
│   ├── keystore.rs           # OS keystore (Keychain / DPAPI / libsecret)
│   ├── memlock.rs            # Memory locking to prevent key swapping to disk
│   └── rng.rs                # CSPRNG with HKDF expansion
│
├── lettuce-identity/         # Identity and Pre-Key Management
│   ├── identity.rs           # Long-term Identity Key (IK) with PQ signatures
│   ├── prekey.rs             # Signed Pre-Key (SPK) + One-Time Pre-Keys (OTPKs)
│   ├── bundle.rs             # Pre-key bundle assembly for FS-API registration
│   └── ktc.rs                # Key Transfer Capsule (ML-KEM + AES-GCM)
│
├── lettuce-session/          # Deliverable 2: Double Ratchet Module
│   ├── session.rs            # Session state: symmetric + KEM ratchet management
│   ├── ratchet.rs            # Chain key advancement, KEM rekey operations
│   └── sealing.rs            # Device-sealed session state serialization
│
├── lettuce-proxy/            # Deliverables 3-4: SMTP/IMAP/POP3 Proxy
│   ├── smtp_outbound.rs      # SMTP proxy: intercept → encrypt → relay
│   ├── imap_proxy.rs         # IMAP proxy: intercept FETCH → decrypt
│   ├── pop3_proxy.rs         # POP3 proxy: intercept RETR → decrypt
│   ├── pq_bootstrap.rs       # PQ session bootstrap via FS-API pre-key bundles
│   ├── encryption.rs         # MIME encryption: plaintext → keypact.enc envelope
│   ├── policy.rs             # Domain-based encryption policy engine
│   ├── config.rs             # TOML configuration parser
│   ├── setup.rs              # Interactive device registration wizard
│   ├── transport/
│   │   ├── fanout.rs         # Mode A: per-recipient ciphertext
│   │   ├── scmw.rs           # Mode B: single ciphertext, multiple wrapped DEKs
│   │   └── offload.rs        # Mode C: large payload offload to external storage
│   └── inbound/
│       ├── dispatcher.rs     # Inbound decryption orchestrator
│       ├── bootstrap.rs      # Receiver-side session bootstrap from KEM ciphertext
│       ├── mime.rs           # MIME parsing and encrypted payload extraction
│       └── cache.rs          # Warm index to skip redundant decryptions
│
└── lettuce-cli/              # CLI for device setup and key management
```

---

## Protocol Design

### Session Bootstrap (PQ Replacement for X3DH)

Signal's X3DH key agreement [[2]](#references) uses elliptic-curve Diffie-Hellman, which is vulnerable to quantum attacks. This system replaces it entirely with ML-KEM encapsulation:

```
 Sender (Alice)                         FS-API Server                       Receiver (Bob)
      │                                      │                                    │
      │  1. GET /prekey/bundle/{hash}/{dev}   │                                    │
      │─────────────────────────────────────> │                                    │
      │  <── PreKeyBundle (IK, SPK, OTPK[i]) │                                    │
      │                                      │                                    │
      │  2. (ss, ct) = ML-KEM.Encaps(OTPK[i].pub)                                │
      │  3. root_key = HKDF-SHA-512(ss)      │                                    │
      │  4. Init Double Ratchet (Initiator)   │                                    │
      │                                      │                                    │
      │  5. POST /otpk/claim {otpk_id}       │                                    │
      │─────────────────────────────────────> │                                    │
      │                                      │                                    │
      │  6. Send encrypted msg + kem_ct + otpk_id ──────────────────────────────> │
      │                                      │                                    │
      │                                      │  7. ss = ML-KEM.Decaps(ct, OTPK.sk)│
      │                                      │  8. root_key = HKDF-SHA-512(ss)    │
      │                                      │  9. Init Double Ratchet (Responder) │
      │                                      │  10. Decrypt message                │
```

### Double Ratchet with PQ Rekey

After session establishment, forward secrecy is maintained through two ratchet mechanisms:

- **Symmetric Ratchet** — After each message, the chain key is advanced via HKDF, a new per-message key is derived, and the old chain key is securely erased
- **KEM Ratchet** — Periodically, a new ML-KEM encapsulation is performed to update the root key, providing post-quantum break-in recovery even if a ratchet state is temporarily compromised

### Transport Modes

| Mode | Mechanism | Use Case |
|------|-----------|----------|
| **Fan-Out** | One ciphertext per recipient, each encrypted with their session key | Single recipient, BCC (maximum privacy) |
| **SCMW** | Body encrypted once with random DEK; DEK wrapped per-recipient via session | Multiple recipients (To, CC) |
| **Offload** | Encrypted payload uploaded to Files API; email contains only the reference | Large attachments (>14 MB threshold) |

Mode selection is automatic: `Offload > Fan-Out (if BCC) > SCMW (otherwise)`

### Encrypted Email Format

```
X-KeyPact: encrypted=true; enc=v1; aead=AES-256-GCM; kem=ML-KEM-1024; method=scmw; ct=inline
```

The encrypted body is packaged as a `keypact.enc` JSON attachment containing the ciphertext, nonce, per-recipient wrapped DEKs, and sender identification for session lookup.

---

## Security Hardening

| Measure | Implementation |
|---------|---------------|
| Key Storage | OS-native keystore with hardware-backed protection where available |
| Memory Protection | `mlock` / `VirtualLock` prevents key material from swapping to disk |
| Key Zeroization | `Zeroize` trait overwrites all sensitive data when dropped |
| Zero-Knowledge Server | FS-API stores only ciphertext and hashes — never plaintext or private keys |
| No Plaintext Logging | All log output sanitized to exclude cryptographic material |
| Fail-Closed | Any integrity error aborts the operation; no silent fallthrough |
| DKIM/SPF/DMARC | Email authentication headers preserved post-encryption |

---

## Cross-Platform Support

| Platform | Architecture | Keystore Backend | Memory Lock |
|----------|-------------|------------------|-------------|
| macOS | x86_64 / ARM64 | Keychain Services | `mlock` |
| Windows | x86_64 | Credential Manager (DPAPI) | `VirtualLock` |
| Linux | x86_64 | Secret Service (libsecret) | `mlock` |

---

## Standards Compliance

| Standard | Description | Usage |
|----------|-------------|-------|
| **NIST FIPS 203** | ML-KEM Post-Quantum KEM | ML-KEM-1024 (Level 5 — 256-bit quantum security) |
| **NIST SP 800-38D** | AES in Galois/Counter Mode | AES-256-GCM authenticated encryption |
| **RFC 5869** | HKDF Key Derivation Function | HKDF-SHA-512 for all key derivation |
| **Signal Protocol** | Double Ratchet Algorithm | PQ-initialized variant with KEM ratchet |
| **RFC 5321/5322** | SMTP / Internet Message Format | Full compliance; SPF/DKIM/DMARC preserved |

---

## References

1. P. W. Shor, "Algorithms for quantum computation: discrete logarithms and factoring," *Proceedings 35th Annual Symposium on Foundations of Computer Science*, 1994.

2. M. Marlinspike and T. Perrin, "The X3DH Key Agreement Protocol," Signal Foundation, 2016. Available: https://signal.org/docs/specifications/x3dh/

3. T. Perrin and M. Marlinspike, "The Double Ratchet Algorithm," Signal Foundation, 2016. Available: https://signal.org/docs/specifications/doubleratchet/

4. National Institute of Standards and Technology, "Module-Lattice-Based Key-Encapsulation Mechanism Standard," FIPS 203, August 2024. Available: https://csrc.nist.gov/pubs/fips/203/final

5. National Institute of Standards and Technology, "Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC," SP 800-38D, November 2007.

6. H. Krawczyk and P. Eronen, "HMAC-based Extract-and-Expand Key Derivation Function (HKDF)," RFC 5869, May 2010.

---

## Project Metrics

| Metric | Value |
|--------|-------|
| Language | Rust |
| Total Lines of Code | ~10,600 |
| Source Files | 58 |
| Workspace Crates | 5 |
| Target Platforms | 3 (macOS, Windows, Linux) |
| Cryptographic Algorithms | 4 (ML-KEM-1024, AES-256-GCM, HKDF-SHA-512, SHA-256) |

---

## License

Copyright (c) 2025-2026 Muhammad Amar Sohail. All rights reserved.

This documentation is provided for informational and academic review purposes only. The source code is proprietary. For licensing inquiries or academic collaboration, contact **amarsohail@protonmail.com**.

This project was developed as an implementation of the KeyPact PQ-Mail Proxy Specification v1.3 (October 2025).
