# Changelog

All notable changes to the age-rt specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [0.2] - 2026-05-20

### Specification Published

Initial public draft of the age-rt v0.2 specification as a differential extension of age v1.

### Added

- Complete specification with 14 normative sections
- Variable-length chunk framing with 4-byte length prefixes
- Explicit final chunk semantics and truncation detection
- Differential comparison table with age v1
- Security considerations for unauthenticated length field
- Pseudocode examples for key derivation and AEAD construction
- Constants and parameters reference table
- Normative implementation requirements (MUST/SHOULD/MAY)

### Defined

- Format identifier: `github.com/parsimonit/age-rt-encryption/v0.2`
- File key size: 16 bytes (identical to age v1)
- Scrypt context: `b"age-encryption.org/v1/scrypt"` (identical to age v1 for black-box reuse)
- Maximum chunk size: 65536 bytes plaintext (64 KiB)
- Chunk length range: 16–65552 bytes ciphertext
- Dual-flag decoder algorithm for final chunk detection

### Clarified

- HMAC construction over header (Step B, format-agnostic)
- Scrypt key derivation (Step A, reusable black box from age v1)
- Empty AEAD AAD (no length in additional authenticated data)
- Mandatory empty final chunk when no pending data
- All age v1 recipient types supported without modification
- Maximum chunk size is fixed at 65536 bytes (not a variable parameter)
- Cryptographic primitive compatibility with age v1 (primitives identical, application differs)

### Security

- Intentional design: length prefix not in AEAD AAD
- Bounds checking prevents DoS via length inflation
- Truncation detection prevents silent data loss
- Nonce uniqueness requirements documented
- **Traffic analysis considerations:** Variable-length chunks reveal chunk size patterns (but not content)
- Mitigation strategies for length-sensitive applications (transport encryption layer or age v1)

---

**Note:** Version numbers follow the specification version, not semantic versioning. Breaking changes to the wire format will result in a new major version (e.g., v0.3, v1.0).
