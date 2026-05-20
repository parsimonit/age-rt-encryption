# age-rt: Real-Time Streaming Encryption

**age-rt** is a secure encryption format for live data streams, based on the widely-used [age encryption format](https://age-encryption.org/).

## What is age-rt?

age-rt extends age to support **real-time streaming** with variable-length chunks. Unlike standard age (which uses fixed 64 KiB chunks), age-rt lets you encrypt and send data immediately—perfect for live feeds, network protocols, and progressive data processing.

**Think of it as:** age for streaming video, live logs, sensor data, or any scenario where you can't wait to buffer 64 KiB before encrypting.

## Why age-rt?

Standard age encryption is excellent for files, but streaming use cases need:

- ✅ **Lower latency** — encrypt and send small chunks immediately
- ✅ **Bandwidth efficiency** — no padding to fixed chunk sizes
- ✅ **Message boundaries** — preserve your application's natural chunk structure

age-rt provides these while maintaining the same strong encryption (ChaCha20-Poly1305) and key management as age.

## Status

**age-rt v0.2** — Stable specification, reference implementation available

## Quick Links

- 📄 **[Read the specification](age-rt-spec-v0.2.md)** — Full technical specification
- 🐍 **[Python implementation](https://github.com/parsimonit/python-age-rt)** — Reference implementation
- 📜 **[Changelog](CHANGELOG.md)** — Version history

## Key Differences from age v1

| Feature | age v1 | age-rt v0.2 |
|---------|--------|-------------|
| Chunk size | Fixed 64 KiB | Variable (0–64 KiB) |
| Framing | Implicit | Explicit 4-byte length prefix |
| Latency | Must buffer to 64 KiB | Send immediately |
| Use case | Files, batch data | Live streams, real-time |

## Example

```python
from age_rt import AgeRTEncoder, AgeRTDecoder

# Encrypt a stream of small chunks
encoder = AgeRTEncoder.from_passphrase("my-secret")
for chunk in [b"live", b"sensor", b"data"]:
    encrypted = encoder.encode_chunk(chunk)
    send_immediately(encrypted)  # no buffering!

# Decrypt as data arrives
decoder = AgeRTDecoder("my-secret")
while data := receive():
    if plaintext := decoder.feed(data):
        process(plaintext)
```

## Security Note

⚠️ **Trade-off:** age-rt's variable-length chunks reveal the **size pattern** of your data (but not the content). An observer can see that you sent a 500-byte chunk, then a 1200-byte chunk, etc.

- **What's protected:** Your data is still fully encrypted and authenticated
- **What leaks:** Chunk size patterns (timing, length)
- **Mitigation:** Use TLS, WireGuard, or similar transport encryption for your network layer

This is an **intentional design choice**—streaming efficiency in exchange for some metadata visibility. If you need perfect length hiding, use standard age with fixed chunks.

## Technical Details

- **Encryption:** ChaCha20-Poly1305 AEAD (RFC 8439)
- **Key derivation:** HKDF-SHA256, scrypt (same as age v1)
- **Authentication:** 16-byte Poly1305 tag per chunk
- **Truncation detection:** Mandatory final chunk flag
- **Maximum chunk:** 65536 bytes (64 KiB)

All cryptographic primitives are identical to age v1. The only protocol difference is the chunk framing mechanism.

## Contributing

Found an issue? Have questions about the specification?

- Open an issue at [github.com/parsimonit/age-rt-encryption](https://github.com/parsimonit/age-rt-encryption)

## References

- **age v1 specification:** [age-encryption.org/v1](https://age-encryption.org/v1)
- **age project:** [github.com/FiloSottile/age](https://github.com/FiloSottile/age)
- **Python implementation:** [github.com/parsimonit/zebrastream-age-rt](https://github.com/parsimonit/python-age-rt)

## License

The age-rt specification is published under the [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) license (public domain).

---

**age-rt** — Real-time encryption for the streaming age 🚀
