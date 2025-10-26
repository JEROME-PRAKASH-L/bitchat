# BitChat Noise Protocol Implementation Guide

## Overview

The Noise Protocol Framework is a modern cryptographic protocol framework for building secure, authenticated communication channels. BitChat implements the **Noise_XX_25519_ChaChaPoly_SHA256** pattern for end-to-end encryption and authentication of private messages in mesh networks.

---

## 1. Noise Protocol Basics

### What is Noise?

Noise is a framework for constructing cryptographic protocols. Instead of building security primitives from scratch, Noise provides:

- **Authenticated Key Exchange (AKE)**: Ensures both parties are who they claim to be
- **Forward Secrecy**: Compromised keys don't decrypt past messages
- **Simplicity**: Well-defined, minimal attack surface
- **Performance**: Lightweight and optimized for constrained environments

### Why BitChat Uses Noise

BitChat chose Noise for several reasons:

1. **Mutual Authentication**: Both parties verify each other without pre-shared keys
2. **Forward Secrecy**: Session keys don't compromise historical messages
3. **Proven Design**: Audited and used by Signal and WireGuard
4. **Minimal Overhead**: Suitable for bandwidth-constrained Bluetooth networks
5. **Flexibility**: Extensible for future enhancements

---

## 2. The XX Handshake Pattern

BitChat uses the **XX pattern**, which provides mutual authentication and forward secrecy without requiring knowledge of the peer's public key beforehand.

### XX Pattern Overview

```
Initiator                           Responder
  |                                   |
  |  -> e [ephemeral public key]     |
  |                                   |
  |  <- e, ee, s, es [ephemeral key, |
  |                 DH result,       |
  |            encrypted static key]|
  |                                   |
  |  -> s, se [encrypted static key]  |
  |                                   |
  |<------ Secure Channel Established ------>|
```

### Detailed Handshake Steps

#### **Message 1: Initiator → Responder**
- Initiator generates ephemeral key pair `(e_i_priv, e_i_pub)`
- Sends: `e_i_pub`
- Hash state is updated: `h = SHA256(h + e_i_pub)`

#### **Message 2: Responder → Initiator**
- Responder generates ephemeral key pair `(e_r_priv, e_r_pub)`
- Computes: `DH(e_i_pub, e_r_priv)` → shared secret `ee`
- Sends: `e_r_pub` + encrypted(`responder_static_pub`, `ee`)
- Also computes: `DH(e_i_pub, responder_static_priv)` → shared secret `es`
- Hash state is updated with both `e_r_pub` and ciphertext

#### **Message 3: Initiator → Responder**
- Initiator decrypts responder's static key using `DH(initiator_ephemeral_priv, e_r_pub)`
- Verifies the static key
- Computes: `DH(initiator_static_priv, e_r_pub)` → shared secret `se`
- Sends: encrypted(`initiator_static_pub`, derived key)
- Hash state is updated

#### **Post-Handshake**
Both parties derive:
- **Send Cipher**: For encrypting outgoing messages
- **Receive Cipher**: For decrypting incoming messages
- **Handshake Hash**: For channel binding and integrity verification

---

## 3. Cryptographic Primitives

### Key Agreement: Curve25519
- **Algorithm**: Elliptic Curve Diffie-Hellman
- **Security Level**: 128 bits
- **Key Size**: 32 bytes
- **Properties**: Small keys, fast computation, resistance to timing attacks

### Cipher: ChaCha20-Poly1305
- **Encryption**: ChaCha20 stream cipher
- **Authentication**: Poly1305 MAC
- **Combined**: AEAD cipher with integrated authentication
- **Benefits**: Faster than AES on some CPUs, proven secure

### Hash Function: SHA-256
- **Output Size**: 256 bits (32 bytes)
- **Used For**: KDF, HKDF, hashing, integrity verification
- **Properties**: Cryptographically secure, widely trusted

---

## 4. Session State Management

### Session States

```
┌─────────────────┐
│  Uninitialized  │
└────────┬────────┘
         │
    startHandshake()
         │
         ▼
┌─────────────────┐
│   Handshaking   │
└────────┬────────┘
         │
 processHandshakeMessage()
         │
         ▼
┌─────────────────┐
│  Established    │
└────────┬────────┘
         │
    reset() or expire
         │
         ▼
┌─────────────────┐
│  Uninitialized  │
└─────────────────┘
```

### State Transitions

- **Uninitialized → Handshaking**: Call `startHandshake()` (initiator only)
- **Handshaking → Established**: Complete 3-message exchange
- **Established → Uninitialized**: Call `reset()` or session timeout

### Thread Safety

All state transitions are protected by a concurrent dispatch queue (`sessionQueue`) with barrier flags to ensure atomic operations.

---

## 5. Key Derivation

### HKDF Process

After the handshake, Noise uses HKDF (HMAC-based Key Derivation Function) to derive transport keys:

```
handshake_hash = final_h_value
    ↓
HKDF(handshake_hash) → key_material
    ↓
split key_material → [send_key, receive_key, other_keys...]
```

### Transport Key Derivation

Both parties derive:
- **Send Cipher Key**: Used for encrypting outgoing transport messages
- **Receive Cipher Key**: Used for decrypting incoming transport messages
- **Channel Binding**: Derived from handshake hash for verification

---

## 6. Replay Protection

### Nonce Management

Each transport cipher maintains an incrementing counter (nonce) for each message:

```
Message 1: nonce = 0
Message 2: nonce = 1
Message 3: nonce = 2
...
```

### Sliding Window Replay Detection

The `NoiseCipherState` implements a sliding window mechanism:

1. **Track received nonces** within a window (typically 64 messages)
2. **Reject duplicates**: If a nonce is seen twice, discard the message
3. **Reject out-of-order**: Messages with very old nonces are rejected
4. **Allow reordering**: Recent out-of-order messages are buffered

This prevents attackers from replaying old encrypted messages.

---

## 7. Rate Limiting

### DoS Prevention

The `NoiseRateLimiter` prevents attackers from exhausting resources via:

1. **Per-peer handshake rate limiting**: Max N handshakes per time window
2. **Concurrent handshake limiting**: Max M simultaneous handshakes
3. **Connection throttling**: Exponential backoff on repeated failures

### Configuration

Rate limiting parameters are defined in `NoiseSecurityConstants`:

```swift
static let MAX_HANDSHAKES_PER_PEER_PER_MINUTE = 10
static let MAX_CONCURRENT_HANDSHAKES = 50
static let HANDSHAKE_BACKOFF_INITIAL_MS = 1000
```

---

## 8. Fingerprint Verification

### What is a Fingerprint?

A fingerprint is a short, human-readable representation of a peer's identity:

```
Fingerprint = SHA256(Noise_Static_Public_Key)
```

### Out-of-Band Verification

Users can verify fingerprints through secure channels:

1. **QR Code**: Scan peer's QR code containing fingerprint
2. **Voice**: Read fingerprint aloud over a secure channel
3. **In-Person**: Compare displayed fingerprints face-to-face

### Implementation

```swift
let staticKey: Curve25519.KeyAgreement.PublicKey = ...
let fingerprint = SHA256(staticKey.rawRepresentation)
let fingerprintHex = fingerprint.map { String(format: "%02x", $0) }.joined()
```

---

## 9. Session Management in BitChat

### NoiseSessionManager

The `NoiseSessionManager` coordinates all active sessions:

```swift
// Creating a session
let session = try sessionManager.getOrCreateSession(
    peerID: peerID,
    role: .initiator,
    remoteKey: peerPublicKey
)

// Starting handshake
let message1 = try session.startHandshake()

// Processing incoming message
let message2 = try session.processHandshakeMessage(receivedData)

// Encrypting a message
let ciphertext = try session.encrypt(plaintext)

// Decrypting a message
let plaintext = try session.decrypt(ciphertext)
```

### Session Lifecycle

1. **Create**: `getOrCreateSession()` creates a new `NoiseSession`
2. **Handshake**: Exchange 3 messages to establish keys
3. **Transport**: Encrypt/decrypt application messages
4. **Cleanup**: `reset()` clears sensitive data or `expireOldSessions()` on timeout

---

## 10. Error Handling

### Common Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| `invalidState` | Operation not allowed in current state | Reset session and retry |
| `notEstablished` | Tried to encrypt before handshake complete | Complete handshake first |
| `decryptionFailed` | Authentication tag mismatch or tampering | Discard message, may indicate attack |
| `replayDetected` | Duplicate or very old message | Discard message (already done) |
| `rateLimitExceeded` | Too many handshake attempts | Exponential backoff, reset later |

### Best Practices

```swift
do {
    let plaintext = try session.decrypt(ciphertext)
    // Process plaintext
} catch NoiseSessionError.decryptionFailed {
    logger.warning("Decryption failed - possible tampering")
    // Discard message, may flag peer
} catch NoiseSessionError.notEstablished {
    logger.error("Session not established")
    // Trigger new handshake
}
```

---

## 11. Security Considerations

### Assumptions

- **Clock Skew**: Timestamps should be within reasonable bounds (not critical for Noise itself)
- **Random Generation**: All randomness must use cryptographically secure sources (`CryptoKit`)
- **Key Storage**: Long-term keys must be stored securely in Keychain

### Threats & Mitigations

| Threat | Mitigation |
|--------|-----------|
| **Man-in-the-Middle** | XX pattern provides mutual authentication |
| **Replay Attacks** | Nonce-based replay detection with sliding window |
| **Denial of Service** | Rate limiting and handshake backoff |
| **Key Compromise** | Forward secrecy via ephemeral keys; rotation via re-keying |
| **Traffic Analysis** | Message padding to fixed sizes (at transport layer) |

---

## 12. Testing

### Unit Tests

BitChat includes comprehensive tests in `bitchatTests/Noise/`:

- **Handshake Tests**: Verify correct XX pattern execution
- **Encryption Tests**: Test encrypt/decrypt round-trips
- **Replay Tests**: Verify duplicate detection
- **State Tests**: Validate state machine transitions
- **Rate Limit Tests**: Verify DoS protection

### Integration Tests

- **End-to-End**: Full handshake between two devices
- **Mesh Scenarios**: Multi-hop message routing with encryption
- **Failure Cases**: Network interruptions and recovery

---

## 13. Future Enhancements

### Potential Improvements

1. **Post-Quantum Cryptography**: Hybrid Noise + PQC for quantum safety
2. **Zero-RTT**: Skip handshake for repeat connections (XX → XK)
3. **Session Resumption**: Resume sessions without full handshake
4. **Key Rotation**: Periodic re-keying with new ephemeral keys
5. **Multi-Device Support**: Sessions spanning multiple devices

---

## References

- **Noise Protocol Framework**: https://noiseprotocol.org/
- **RFC 7748**: Elliptic Curves for Security (Curve25519)
- **RFC 8439**: ChaCha20 and Poly1305 AEAD Cipher Suites
- **RFC 8949**: SHA-256 Hashing
- **BitChat Whitepaper**: See `WHITEPAPER.md`

---

## Questions & Troubleshooting

### Q: Why XX and not XK or NN?

**A:** XX provides mutual authentication without pre-sharing keys, essential for decentralized P2P networks where peers may not know each other beforehand.

### Q: What if a handshake fails?

**A:** The session returns to `uninitialized` state and can retry. Rate limiting prevents brute force attacks.

### Q: Are past messages safe if my key is compromised?

**A:** Yes, thanks to forward secrecy. Each session uses ephemeral keys that are discarded after use.

### Q: How long do sessions last?

**A:** Sessions are active until explicitly reset or expire due to timeout. See `NoiseSessionManager` for timeout configuration.

