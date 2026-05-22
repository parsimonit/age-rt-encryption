# age-rt v0.2 Specification

**Version:** 0.2  
**Status:** Draft  
**Date:** May 2026

## §1 Overview

age-rt is a real-time streaming encryption format based on the [age v1 specification](https://age-encryption.org/v1). It extends age v1 with variable-length chunk framing to enable low-latency authenticated encryption for streaming data.

**Key characteristics:**

- **Variable-length chunks** with explicit 4-byte length prefixes (vs. age v1's implicit fixed 64 KiB chunks)
- **Cryptographic primitive compatibility** with age v1: identical algorithms (ChaCha20-Poly1305, HKDF-SHA256), key derivation, and recipient types
- **Streaming-friendly design**: chunks can be emitted and authenticated incrementally without buffering
- **Truncation detection**: mandatory final chunk flag enables reliable detection of incomplete streams

age-rt shares the core cryptographic foundation with age v1 while optimizing for use cases requiring immediate chunk-by-chunk delivery, such as live data feeds, network protocols, and progressive processing pipelines. See §13 for security considerations regarding variable-length chunks.

---

## §2 Relationship to age v1

age-rt is a **differential extension** of age v1. This section summarizes what is identical versus what diverges.

### Identical to age v1

| Component | Details |
|---|---|
| AEAD algorithm | ChaCha20-Poly1305 (RFC 8439) |
| File key size | 16 bytes (128 bits) |
| Payload nonce size | 16 bytes (CSPRNG) |
| Payload key derivation | `HKDF-SHA256(ikm=file_key, salt=nonce, info=b"payload")` |
| AEAD nonce structure | 11-byte chunk counter + 1-byte final flag |
| AEAD AAD | Empty (no additional authenticated data) |
| Header HMAC | `HKDF-SHA256(ikm=file_key, salt=None, info=b"header")` |
| Recipient types | All age v1 types supported (X25519, scrypt, SSH keys, etc.) |
| Scrypt parameters | N=2^18, r=8, p=1; S=`b"age-encryption.org/v1/scrypt" \|\| salt` |
| Base64 encoding | Unpadded, canonical, 64-column wrapped |

### Diverges from age v1

| Aspect | age v1 | age-rt v0.2 |
|---|---|---|
| **Format identifier** | `age-encryption.org/v1` | `github.com/parsimonit/age-rt-encryption/v0.2` |
| **Chunk framing** | Implicit fixed 64 KiB | Explicit 4-byte big-endian length prefix per chunk |
| **Chunk size** | Fixed 65536 bytes (except final) | Variable 0–65536 bytes |
| **Wire chunk format** | Raw ciphertext only | `[4B length][ciphertext]` |
| **Final chunk** | Short chunk signals end | Empty chunk MUST be sent if no pending data; flag=0x01 required |
| **Maximum ciphertext per chunk** | 65552 bytes (65536 + 16-byte tag) | 65552 bytes (same constraint) |

---

## §3 Identifier

The canonical age-rt v0.2 format identifier is:

```
github.com/parsimonit/age-rt-encryption/v0.2
```

This identifier MUST appear as the first line of the age header. Conforming implementations MUST use the maximum chunk size of 65536 bytes as specified in §7 and §11.

---

## §4 Header Format

The age-rt header follows the age v1 header grammar with one change: the version line uses the age-rt identifier.

### Structure

```
<identifier>\n
-> <recipient-type> <args>...\n
<base64-wrapped-stanza-body>\n
---[ <base64-mac>]\n
```

**Identifier line:** Exactly `github.com/parsimonit/age-rt-encryption/v0.2` followed by a newline.

**Recipient stanza:** Identical format to age v1. Multiple recipient stanzas are permitted. See §5 for details.

**Stanza body encoding:** Base64-unpadded, wrapped at 64 columns. The final line of the body MUST be strictly less than 64 characters. If the unwrapped base64 string is an exact multiple of 64 characters, an empty line MUST be appended to satisfy this constraint.

**MAC line:** The literal string `---` followed by a space, followed by the unpadded base64-encoded HMAC, followed by a newline.

### HMAC Construction

The HMAC authenticates all header bytes from the identifier through the `---` separator (inclusive), but excluding the space, MAC value, and final newline.

```python
# Pseudocode
mac_data = header_bytes_through_separator  # ends with b"---"
hmac_key = HKDF-SHA256(
    algorithm=SHA256,
    length=32,
    salt=None,           # 32 zero bytes
    info=b"header"
).derive(file_key)

mac = HMAC-SHA256(key=hmac_key, message=mac_data)
```

The MAC is encoded as unpadded base64 and appended: `--- <mac>\n`.

### Example Header

```
github.com/parsimonit/age-rt-encryption/v0.2
-> scrypt hLRBE6F/M0GEOlNx7ILsKg 18
mJ6V1FRGzABPNMp99VLJjQnLPy6HFY6xHVQrGB0rTAU
--- Zht8b3EvpLgCGxJ7RUFOvSZXzDLYLmvPIhjFqWLq5GM
```

(Stanza body may span multiple lines if >64 characters unencoded.)

---

## §5 Key Establishment

age-rt supports **all age v1 recipient types** without modification. Key establishment produces a 16-byte file key, identical to age v1.

### Scrypt Recipient Type

The scrypt (passphrase-based) recipient is specified here as an example. For full details on X25519, SSH keys, and other recipient types, see the [age v1 specification](https://age-encryption.org/v1).

**Stanza format:**

```
-> scrypt <base64-salt> <log2(N)>
<base64-wrapped-file-key>
```

**Parameters:**
- `salt`: 16 random bytes, base64-encoded (unpadded)
- `log2(N)`: Decimal string `18` (work factor N=2^18=262144)

**Scrypt derivation:**

```python
# Pseudocode (Step A: passphrase → wrap_key)
S = b"age-encryption.org/v1/scrypt" + salt  # 16-byte salt
wrap_key = scrypt(
    password=passphrase.encode('utf-8'),
    salt=S,
    N=2^18,
    r=8,
    p=1,
    dkLen=32
)

# Wrap the file key
wrapped_file_key = ChaCha20-Poly1305.encrypt(
    key=wrap_key,
    nonce=b"\x00" * 12,
    plaintext=file_key,     # 16 bytes
    aad=None
)  # Returns 32 bytes (16 + 16-byte tag)
```

**Critical note:** The scrypt salt input `S` is **identical to age v1**: `b"age-encryption.org/v1/scrypt" || salt`. This enables black-box reuse of age v1 scrypt implementations for Step A (stanza unwrap). The age-rt format is distinguished by the HMAC in Step B (§4), which covers the age-rt identifier.

**File key size:** The file key MUST be exactly **16 bytes** for all recipient types.

---

## §6 Payload Structure

The payload immediately follows the header and consists of:

```
[16-byte nonce][chunk₀][chunk₁]...[chunkₙ]
```

**Payload nonce:** 16 bytes generated by a cryptographically secure random number generator (CSPRNG). This value is used to derive the payload encryption key (§8) and MUST be unique per stream.

**Chunks:** Variable-length encrypted chunks, each framed as specified in §7.

---

## §7 Chunk Format

Each chunk on the wire is formatted as:

```
[4-byte length][ciphertext]
```

**Length field:**
- **Encoding:** 4-byte unsigned integer, big-endian (network byte order)
- **Value:** `length = len(ciphertext) = len(plaintext) + 16`
- **Range:** Minimum 16 (empty plaintext + AEAD tag), maximum 65552 (65536-byte plaintext + 16-byte tag)

**Ciphertext:**
- ChaCha20-Poly1305 authenticated encryption of the plaintext chunk
- Includes a 16-byte authentication tag appended by the AEAD

**Bounds checking:** Decoders MUST validate that `16 ≤ length ≤ 65552` before allocating memory or reading data. Values outside this range MUST cause immediate decryption failure.

**Security note:** The length field is **NOT included in the AEAD additional authenticated data (AAD)**. This is intentional. An attacker modifying the length will cause AEAD authentication failure, but cannot extract plaintext. The bounds check prevents excessive memory allocation (DoS mitigation). See §13 for details.

---

## §8 Payload Key and AEAD Construction

### Payload Key Derivation

The payload encryption key is derived once per stream from the file key and nonce:

```python
# Pseudocode
payload_key = HKDF-SHA256(
    algorithm=SHA256,
    length=32,
    salt=payload_nonce,   # 16-byte nonce from §6
    info=b"payload"
).derive(file_key)        # 16-byte file key from recipient stanza
```

A single `ChaCha20Poly1305` cipher instance is initialized with this key and used for all chunks in the stream.

### Per-Chunk AEAD

Each chunk is encrypted with a unique 12-byte nonce derived from the chunk index and finalization flag:

```python
# Pseudocode
def make_aead_nonce(chunk_index: int, is_final: bool) -> bytes:
    """
    Construct 12-byte AEAD nonce.
    
    chunk_index: 0-based sequence number (0, 1, 2, ...)
    is_final: True if this is the final chunk
    """
    flag = 0x01 if is_final else 0x00
    return chunk_index.to_bytes(11, byteorder='big') + bytes([flag])
```

**Encryption (encoder):**

```python
# Pseudocode
aead_nonce = make_aead_nonce(chunk_index, is_final)
ciphertext = ChaCha20Poly1305(payload_key).encrypt(
    nonce=aead_nonce,
    plaintext=chunk_data,
    aad=None              # Empty AAD
)
wire_chunk = len(ciphertext).to_bytes(4, 'big') + ciphertext
```

**Decryption (decoder):**

```python
# Pseudocode (dual-flag algorithm; see §9)
ciphertext = read_bytes(length)  # length from 4-byte prefix

# Try non-final flag first
try:
    nonce = make_aead_nonce(chunk_index, is_final=False)
    plaintext = ChaCha20Poly1305(payload_key).decrypt(
        nonce=nonce,
        ciphertext=ciphertext,
        aad=None
    )
    # Success → continue to next chunk
    return plaintext
except AuthenticationError:
    pass  # Try final flag

# Try final flag
try:
    nonce = make_aead_nonce(chunk_index, is_final=True)
    plaintext = ChaCha20Poly1305(payload_key).decrypt(
        nonce=nonce,
        ciphertext=ciphertext,
        aad=None
    )
    # Success → stream complete
    return plaintext
except AuthenticationError:
    raise DecryptionError("Chunk authentication failed")
```

---

## §9 Final Chunk Semantics

The encoder MUST signal stream completion using the final chunk flag in the AEAD nonce.

### Encoder Requirements

1. The **final chunk** (last chunk in the stream) MUST set `is_final=True` in its AEAD nonce (flag byte = `0x01`). This signals stream termination to the decoder.

2. The final chunk MAY contain plaintext of any valid length (0–65536 bytes).

3. If the encoder has no more plaintext to send after emitting all data chunks, it MUST send an additional empty chunk with `is_final=True`:
   - Length: `0x00000010` (4-byte big-endian integer = 16)
   - Ciphertext: 16 bytes (encryption of zero-length plaintext)

4. Zero-length chunks with `is_final=False` (flag=`0x00`) are valid non-final chunks.

### Decoder Algorithm

Decoders MUST use a **dual-flag attempt** for each chunk:

1. Attempt decryption with `is_final=False` (flag=`0x00`)
   - On success: yield plaintext, continue to next chunk
   - On failure: proceed to step 2

2. Attempt decryption with `is_final=True` (flag=`0x01`)
   - On success: yield plaintext, mark stream complete, stop
   - On failure: abort with authentication error

This algorithm enables the decoder to detect the final chunk without advance knowledge of stream length.

---

## §10 Truncation Detection

A stream is **truncated** if the transport reaches EOF before receiving a chunk with `is_final=True`.

**Detection:** If the decoder successfully decrypts a chunk with `flag=0x00` and then encounters EOF (no more data available), the stream was truncated.

**Decoder obligation:** Upon detecting truncation, the decoder MUST:
- Abort processing immediately
- Signal an error (e.g., raise `TruncationError`)
- NOT deliver the incomplete data as a complete stream

**Rationale:** Truncation indicates either data loss (network/storage failure) or an active attack. Silent acceptance of truncated streams would violate age-rt's integrity guarantee.

---

## §11 Constants and Parameters

| Constant | Value | Description |
|---|---|---|
| `FILE_KEY_BYTES` | 16 | File key size (128 bits) |
| `PAYLOAD_NONCE_BYTES` | 16 | Payload nonce size |
| `AEAD_NONCE_BYTES` | 12 | ChaCha20-Poly1305 nonce size |
| `AEAD_TAG_BYTES` | 16 | ChaCha20-Poly1305 authentication tag size |
| `CHUNK_LENGTH_BYTES` | 4 | Chunk length prefix size |
| `MAX_CHUNK_PLAINTEXT` | 65536 | Maximum plaintext bytes per chunk (64 KiB) |
| `MAX_CHUNK_CIPHERTEXT` | 65552 | Maximum ciphertext bytes per chunk (plaintext + tag) |
| `MIN_CHUNK_CIPHERTEXT` | 16 | Minimum ciphertext bytes (empty plaintext + tag) |
| `SCRYPT_SALT_BYTES` | 16 | Scrypt salt size |
| `SCRYPT_N` | 262144 | Scrypt work factor (2^18) |
| `SCRYPT_r` | 8 | Scrypt block size parameter |
| `SCRYPT_p` | 1 | Scrypt parallelization parameter |
| `SCRYPT_DKLEN` | 32 | Scrypt output length (wrap key size) |

---

## §12 Implementation Requirements

### Encoder (Normative)

**MUST:**
- Generate a unique 16-byte file key using a CSPRNG
- Generate a unique 16-byte payload nonce using a CSPRNG
- Emit a chunk with `is_final=True` to finalize the stream (MAY be empty or non-empty)
- Validate chunk plaintext does not exceed 65536 bytes (`MAX_CHUNK_PLAINTEXT`)
- Encode length as 4-byte big-endian unsigned integer
- Use the identifier `github.com/parsimonit/age-rt-encryption/v0.2`

**SHOULD:**
- Emit chunks as soon as plaintext is available (minimize buffering)

**MAY:**
- Support zero-length non-final chunks (flag=0x00)

### Decoder (Normative)

**MUST:**
- Validate `16 ≤ chunk_length ≤ 65552` before reading ciphertext
- Attempt decryption with `is_final=False`, then `is_final=True` (dual-flag algorithm)
- Detect truncation (EOF after `flag=0x00` chunk) and abort with error
- Verify header HMAC before processing payload
- Reject unknown recipient types or invalid stanza formats
- Support the identifier `github.com/parsimonit/age-rt-encryption/v0.2`

**SHOULD:**
- Limit header size (e.g., 4 KiB) to prevent DoS

**MAY:**
- Support additional age-rt identifiers for future versions

### Error Handling

Implementations MUST distinguish:
- **Header errors** (parse failure, unknown identifier, HMAC mismatch, unsupported recipient)
- **Authentication errors** (chunk AEAD failure, wrong passphrase)
- **Truncation errors** (EOF with `flag=0x00`)
- **Protocol errors** (invalid length, oversized chunk, malformed base64)

---

## §13 Security Considerations

### Traffic Analysis and Chunk Length Patterns

Unlike age v1 (which uses fixed 64 KiB chunks), **age-rt's variable-length chunks reveal the exact size of each plaintext chunk** through the 4-byte length prefix.

**Security impact:**

- **Metadata leakage:** Attackers can observe chunk size patterns without decryption, potentially inferring information about message structure, content type, or application behavior
- **Authenticated encryption preserved:** The AEAD still protects plaintext content and detects tampering—only the _size_ of chunks is visible
- **No plaintext extraction:** Observing chunk sizes does not enable an attacker to decrypt content or forge valid chunks

**Design rationale:**

age-rt's variable-length chunks enable:
- **Lower latency:** Send data immediately without buffering to 64 KiB
- **Bandwidth efficiency:** No padding to fixed chunk sizes
- **Message boundary preservation:** Maintain application-level chunk structure

For real-time streaming use cases (live feeds, network protocols, sensor data), this trade-off between latency/efficiency and metadata privacy is often acceptable.

**Mitigation strategies:**

Applications requiring chunk length confidentiality SHOULD:
- Use an additional transport encryption layer (TLS, WireGuard, Tor, etc.)
- Or use age v1 with fixed 64 KiB chunks instead of age-rt

Applications where chunk size patterns are not sensitive (e.g., uniform sensor readings, fixed-size protocol messages) can use age-rt directly.

### Unauthenticated Length Prefix

The 4-byte chunk length is **intentionally not included in the AEAD AAD**. 

**Threat model:** An attacker modifying the length field can cause:
- **Authentication failure** (if length mismatch causes incorrect ciphertext read)
- **Decoder resource exhaustion** (if length is inflated)

**Mitigations:**
- Bounds check (`16 ≤ length ≤ 65552`) prevents over-allocation
- AEAD authentication detects any ciphertext tampering
- Attacker cannot extract plaintext or forge valid chunks

**Design rationale:** Including length in AAD would require reading the entire ciphertext before authentication, defeating the purpose of streaming. The current design enables chunk-by-chunk processing with equivalent security.

### Nonce Uniqueness

**Payload nonce:** MUST be generated by a CSPRNG. Nonce reuse with the same file key compromises confidentiality.

**AEAD nonces:** Derived deterministically from chunk index and final flag. Chunk counter MUST NOT overflow 2^88 chunks (practically impossible at 64 KiB/chunk = 19 ZiB total).

### Truncation Attacks

Without the final chunk requirement and truncation detection:
- An attacker could strip trailing chunks from a stream
- Recipient would accept partial data as complete

The `is_final` flag ensures integrity of stream boundaries by requiring explicit stream termination.

### Scrypt Work Factor

The fixed work factor (N=2^18) balances security and usability. As of 2026, this provides adequate resistance to brute-force passphrase attacks for high-entropy passphrases. Users with low-entropy passphrases should prefer X25519 recipient types with long-term keys.

### Scrypt Context String

Using `S = b"age-encryption.org/v1/scrypt" || salt` (identical to age v1) enables code reuse but does not compromise format separation. The HMAC (§4) authenticates the age-rt identifier, ensuring ciphertexts cannot be confused across formats.

---

## §14 References

- **age v1 specification:** [age-encryption.org/v1](https://age-encryption.org/v1) / [c2sp.org/age](https://c2sp.org/age)
- **RFC 8439:** ChaCha20 and Poly1305 for IETF Protocols
- **RFC 5869:** HMAC-based Extract-and-Expand Key Derivation Function (HKDF)
- **RFC 7748:** Elliptic Curves for Security (X25519)
- **RFC 7914:** The scrypt Password-Based Key Derivation Function

---

**End of Specification**
