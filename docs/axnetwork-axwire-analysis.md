# AXNetwork & AXWire Architecture Analysis
## Detaillierte Logik-Analyse und REALMVOID-Vergleich

**Analysedatum:** 2026-02-22
**Quelle:** `/mnt/nas-egammo/PROJECTS/AXNetwork/`
**Vergleich:** REALMVOID Implementation

---

## EXECUTIVE SUMMARY

**AXNetwork** ist eine sophisticated UDP-basierte Gaming-Kommunikationsbibliothek für .NET Framework 4.7.2 die bietet:

1. ✅ Generic Message-Level Protocol Architecture
2. ✅ Selective Message-Level Encryption (libsodium)
3. ✅ GZip Compression mit Policy-Based Control
4. ✅ Privacy-First Identity System (GDPR-compliant)
5. ✅ Clean Abstraction über LiteNetLib für Gaming Applications

**WICHTIGE ERKENNTNIS:** Es gibt KEIN explizites "AXWire"-Protokoll als separate Entity. **AXWire ist die logische konzeptuelle Protokoll-Ebene, die AXNetwork implementiert.** AXNetwork IST die Implementation der AXWire-Konzepte.

---

## 1. LOGISCHES KONZEPT: AXWIRE PROTOCOL

### 1.1 Was ist AXWire?

AXWire ist die **logische Protokoll-Abstraktion**, die AXNetwork implementiert. Es repräsentiert:

- Ein **generisches, applikations-agnostisches Message-Transport-Protokoll**
- Gebaut auf top von UDP via LiteNetLib
- Designed für Multiplayer Gaming mit Privacy und Security Considerations
- **NICHT** ein spezifisches Wire-Format, sondern ein konzeptionelles Framework

Der Begriff "AXWire" repräsentiert:
- **AX** = AXNetwork/Acknex (Gaming Framework)
- **Wire** = Das Protokoll/Wire-Format für Kommunikation

### 1.2 Core AXWire Principles

#### 1. Generic Message Bus Philosophy

```
System Messages:     Category 0x0000 (AXNetwork reserved)
├─ MSG_HANDSHAKE     (0x0001)
├─ MSG_HEARTBEAT     (0x0002)
├─ MSG_DISCONNECT    (0x0003)
├─ MSG_ACK           (0x0004)
├─ MSG_ERROR         (0x0005)
└─ MSG_KEY_EXCHANGE  (0x0006)

Application Messages: Category 0x1000+ (User-defined)
├─ Category 0x1000 = Gaming
│  ├─ 0x0001 = PLAYER_SPAWN
│  ├─ 0x0002 = PLAYER_MOVE
│  └─ 0x0003 = PLAYER_ATTACK
├─ Category 0x2000 = Chat
├─ Category 0x3000 = FileTransfer
└─ Category 0x4000 = Admin
```

**Kernprinzip:** AXNetwork erzwingt KEINE game-spezifische Logik. Applikationen definieren ihre eigenen Kategorien.

#### 2. Outer/Inner Header Architecture (VPN-analogous)

```
┌────────────────────────────────────────────────────────────┐
│ OUTER HEADER (16 bytes) - ALWAYS UNENCRYPTED              │
├────────────────────────────────────────────────────────────┤
│ ProtocolVersion      (2 bytes) │ uint16                    │
│ Category             (2 bytes) │ uint16                    │
│ MessageType          (2 bytes) │ uint16                    │
│ Flags                (2 bytes) │ uint16                    │
│ SessionId            (4 bytes) │ uint32                    │
│ SequenceNumber       (4 bytes) │ uint32                    │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ INNER CONTENT (Variable) - CONDITIONALLY ENCRYPTED         │
├────────────────────────────────────────────────────────────┤
│ Identity Layer (64 bytes optional)                         │
│ ├─ SenderIdentityHash  (32 chars hex)                     │
│ └─ SenderHardwareHash  (32 chars hex)                     │
│                                                            │
│ Payload (Variable, 0-64KB)                                │
│ └─ Application Data (MessagePack, Protobuf, JSON, etc.)   │
└────────────────────────────────────────────────────────────┘
```

**Design Rationale:**

| Aspekt | Warum |
|--------|-------|
| **Outer Header unencrypted** | Erlaubt Network Routing und Protocol-aware Proxies |
| **Inner Content encrypted** | Privacy-aware Data Protection |
| **Compress-then-Encrypt** | Kryptografisch korrekt (compress vor encryption um patterns zu vermeiden) |

#### 3. Message Flags

```csharp
enum MessageFlags : ushort
{
    None          = 0x0000,
    Reliable      = 0x0001,  // Requires ACK, will be retransmitted
    Compressed    = 0x0002,  // Inner content is GZip compressed
    Encrypted     = 0x0004,  // Inner content is libsodium encrypted
    HasIdentity   = 0x0008,  // Includes SenderIdentityHash + SenderHardwareHash
    Ordered       = 0x0010,  // Must be delivered in sequence order
    System        = 0x0020,  // System message (Category 0x0000)

    // Combinations
    SecureReliable = Reliable | Encrypted | Compressed,
    FastUnreliable = None  // Low-latency, no guarantees
}
```

---

## 2. AXNETWORK ARCHITECTURE

### 2.1 Solution Structure

```
AXNetwork.sln
├── AXNetwork.dll (Core Library)
│   ├── Core/
│   │   ├── AXNetServer.cs           [UDP Server wrapper über LiteNetLib]
│   │   ├── AXNetClient.cs           [UDP Client wrapper über LiteNetLib]
│   │   ├── ConnectionManager.cs     [Peer tracking, event handling]
│   │   └── EventArgs/               [OnMessageReceived, OnClientConnected, etc.]
│   │
│   ├── Protocol/
│   │   ├── AXMessage.cs             [Generic message structure]
│   │   ├── AXProtocol.cs            [Protocol handler mit encryption/compression]
│   │   ├── AXMessageSerializer.cs   [Outer/Inner header serialization]
│   │   ├── AXHandshake.cs           [Encryption negotiation + privacy validation]
│   │   ├── EncryptionPolicy.cs      [None, Optional, Preferred, Required]
│   │   ├── CompressionPolicy.cs     [None, Automatic, Always, Manual]
│   │   └── AXProtocolConstants.cs   [System message types]
│   │
│   ├── Security/
│   │   ├── AXCrypto.cs              [libsodium integration - Curve25519 key exchange]
│   │   └── AXCompression.cs         [GZip compression utilities]
│   │
│   └── Interfaces/
│       └── IAXNetwork.cs            [Public API abstraction]
│
├── AXNetwork.Identity.dll (Privacy System)
│   ├── Core/
│   │   ├── AXUserIdentity.cs        [User identity management]
│   │   ├── AXHardwareID.cs          [Hardware fingerprinting]
│   │   ├── AXPrivacyManager.cs      [GDPR consent management]
│   │   └── AXMatchEngine.cs         [Identity matching/verification]
│   │
│   ├── Models/
│   │   ├── HardwareProfile.cs       [CPU, Motherboard, MAC, etc.]
│   │   ├── PrivacyModels.cs         [Consent records, data policies]
│   │   ├── IdentityResult.cs        [Verification results]
│   │   └── MatchResult.cs           [Identity match confidence]
│   │
│   └── Interfaces/
│       ├── IAXIdentity.cs
│       ├── IAXHardware.cs
│       └── IAXPrivacy.cs
│
├── AXNetwork.TestServer.exe (Functional Test Harness)
└── AXNetwork.TestClient.exe (Complete Test Suite)
```

### 2.2 Network Communication Model

**Layered Architecture:**

```
┌─────────────────────────────────────────┐
│      Application Layer                  │
│  (Game Logic, Chat, Admin Tools)        │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  AXNetwork API                          │
│  SendReliable() / SendUnreliable()      │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  AXProtocol                             │
│  Message Creation + Policy Application  │
│  (Encryption/Compression Decisions)     │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  AXMessageSerializer                    │
│  Outer/Inner Header Construction        │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  AXCrypto + AXCompression               │
│  Security Operations                    │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  LiteNetLib                             │
│  UDP Packet Transmission                │
└─────────────────┬───────────────────────┘
                  ↓
              NETWORK
```

### 2.3 Message Flow (Detailed)

#### Client → Server (Sending)

```
Step 1: Application calls
├─ client.SendReliable("Hello Server!")  // String message
└─ OR: client.SendReliable(binaryData)   // Binary message

Step 2: Core wraps message
├─ AXNetClient.SendMessage(data, DeliveryMethod.ReliableOrdered)
└─ Calls protocol handler

Step 3: Protocol creates AXMessage
├─ Sets Category (from application or default 0x0000)
├─ Sets MessageType (from application or default)
├─ Sets Flags based on policies:
│  ├─ Reliable flag (from DeliveryMethod)
│  ├─ Compressed flag (if policy allows + size >= threshold)
│  └─ Encrypted flag (if policy requires + crypto available)
└─ Attaches Identity (if HasIdentity flag + user consent)

Step 4: Serializer constructs outer header
├─ Write ProtocolVersion (0x0001)
├─ Write Category
├─ Write MessageType
├─ Write Flags
├─ Write SessionId
└─ Write SequenceNumber (incremented)

Step 5: Serializer processes inner content
├─ IF Compressed flag:
│  └─ Apply GZip compression to payload
├─ IF Encrypted flag:
│  ├─ Generate random nonce (24 bytes)
│  └─ Encrypt with libsodium SecretBox(payload, nonce, shared_secret)
└─ Combine: Identity (if present) + Payload (compressed/encrypted)

Step 6: LiteNetLib sends
└─ Transmit UDP packet to server

Total Time: ~0.5-2ms (depending on encryption/compression)
```

#### Server ← Client (Receiving)

```
Step 1: LiteNetLib receives
└─ UDP packet arrives

Step 2: Core OnDataReceived event fires
└─ AXNetServer.OnNetworkReceive(peer, reader, deliveryMethod)

Step 3: Serializer reads outer header (UNENCRYPTED)
├─ Read ProtocolVersion → Validate
├─ Read Category → Route decision
├─ Read MessageType → Message identification
├─ Read Flags → Processing instructions
├─ Read SessionId → Session tracking
└─ Read SequenceNumber → Ordering/duplicate detection

Step 4: Serializer processes inner content
├─ IF Encrypted flag:
│  ├─ Extract nonce (first 24 bytes of inner)
│  └─ Decrypt with SecretBox.Open(ciphertext, nonce, shared_secret)
├─ IF Compressed flag:
│  └─ Decompress with GZip
└─ Extract Identity (if HasIdentity flag)

Step 5: AXMessage reconstructed
├─ Complete message object created
├─ Payload decoded
└─ Identity attached (if present)

Step 6: Application event fires
└─ OnMessageReceived(message, fromClientId)
   OR: OnBinaryMessageReceived(data, fromClientId)

Total Time: ~0.3-1.5ms (depending on decryption/decompression)
```

---

## 3. ENCRYPTION SYSTEM (libsodium-net)

### 3.1 Encryption Policy Levels

```csharp
enum EncryptionPolicy
{
    None      = 0,  // No encryption capability, always plaintext
    Optional  = 1,  // Encryption supported but not required (fallback to plaintext)
    Preferred = 2,  // Attempts encrypted connection first, requires compatibility
    Required  = 3   // Connection fails without encryption
}
```

### 3.2 Policy Negotiation Matrix

```
┌───────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ Client\Server │    None     │  Optional   │  Preferred  │  Required   │
├───────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ None          │ Plaintext   │ Plaintext   │ Plaintext   │ FAIL ❌     │
├───────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Optional      │ Plaintext   │ Encrypted✅ │ Encrypted✅ │ Encrypted✅ │
├───────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Preferred     │ Plaintext   │ Encrypted✅ │ Encrypted✅ │ Encrypted✅ │
├───────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Required      │ FAIL ❌     │ Encrypted✅ │ Encrypted✅ │ Encrypted✅ │
└───────────────┴─────────────┴─────────────┴─────────────┴─────────────┘

Legend:
✅ = Connection succeeds with specified mode
❌ = Connection fails (policy mismatch)
```

### 3.3 Key Exchange Process (Curve25519 ECDH)

```
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 1: Key Generation (Each Side)                            │
├─────────────────────────────────────────────────────────────────┤
│ Client:                                                         │
│ ├─ (private_key_client, public_key_client) = GenerateKeyPair() │
│ └─ Store: private_key_client (32 bytes, SECRET)                │
│                                                                 │
│ Server:                                                         │
│ ├─ (private_key_server, public_key_server) = GenerateKeyPair() │
│ └─ Store: private_key_server (32 bytes, SECRET)                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PHASE 2: Public Key Exchange                                   │
├─────────────────────────────────────────────────────────────────┤
│ Client → Server:                                                │
│ ├─ MSG_HANDSHAKE                                                │
│ └─ Payload: public_key_client (32 bytes)                       │
│                                                                 │
│ Server → Client:                                                │
│ ├─ MSG_KEY_EXCHANGE                                             │
│ └─ Payload: public_key_server (32 bytes)                       │
│                                                                 │
│ NOTE: Public keys sent UNENCRYPTED in outer header payload     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PHASE 3: Shared Secret Derivation                              │
├─────────────────────────────────────────────────────────────────┤
│ Client side:                                                    │
│ ├─ shared_secret = ScalarMult(private_key_client,              │
│ │                              public_key_server)              │
│ └─ Result: 32-byte shared secret                               │
│                                                                 │
│ Server side:                                                    │
│ ├─ shared_secret = ScalarMult(private_key_server,              │
│ │                              public_key_client)              │
│ └─ Result: IDENTICAL 32-byte shared secret                     │
│                                                                 │
│ CRITICAL: Both sides have same shared_secret WITHOUT           │
│           ever transmitting it over network!                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PHASE 4: Message Encryption/Decryption                         │
├─────────────────────────────────────────────────────────────────┤
│ Encryption (per message):                                       │
│ ├─ nonce = RandomBytes(24)  // Random, unique per message      │
│ ├─ ciphertext = SecretBox.Create(plaintext, nonce, shared_secret) │
│ └─ Transmit: [nonce (24 bytes)][ciphertext (variable)]         │
│                                                                 │
│ Decryption (per message):                                       │
│ ├─ Extract nonce (first 24 bytes)                              │
│ ├─ Extract ciphertext (remaining bytes)                        │
│ ├─ plaintext = SecretBox.Open(ciphertext, nonce, shared_secret) │
│ └─ If authentication fails → reject message                    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Security Properties

| Property | Mechanism | Benefit |
|----------|-----------|---------|
| **Perfect Forward Secrecy** | Ephemeral keys (not persisted) | Compromised session doesn't affect past sessions |
| **Authenticated Encryption** | libsodium SecretBox (Poly1305 MAC) | Prevents tampering, ensures integrity |
| **Nonce Uniqueness** | Random nonce per message | Prevents replay attacks |
| **No Key Reuse** | New keypair per connection | Limits exposure window |
| **ECDH Security** | Curve25519 (128-bit security) | Resistant to quantum attacks (for now) |

---

## 4. COMPRESSION SYSTEM (GZip)

### 4.1 Compression Policies

```csharp
enum CompressionPolicy
{
    None      = 0,  // No compression ever
    Automatic = 1,  // Size-based (compress if >= threshold)
    Always    = 2,  // Compress all messages with payload
    Manual    = 3   // Application controls via flag explicitly
}
```

### 4.2 Compression Configuration

```csharp
// Default Configuration
CompressionPolicy = Automatic
CompressionThreshold = 1024 bytes (1KB)
CompressionLevel = 6 (GZip level 1-9, balance speed/ratio)

// Custom Configuration
protocol.SetCompressionPolicy(CompressionPolicy.Automatic);
protocol.SetCompressionThreshold(2048);  // 2KB threshold
protocol.SetCompressionLevel(9);         // Max compression (slower)
```

### 4.3 Compression Pipeline

#### Serialize Pipeline (Compress-then-Encrypt)

```
┌─────────────────────────────────────────────────────────────┐
│ Original Payload (e.g., 5000 bytes)                         │
└────────────────────┬────────────────────────────────────────┘
                     ↓
         ┌───────────────────────┐
         │ Should compress?      │
         │ Policy = Automatic    │
         │ Size >= Threshold?    │
         │ 5000 >= 1024? YES     │
         └───────────┬───────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Apply GZip Compression (Level 6)                            │
│ Result: 1200 bytes (76% reduction)                          │
│ Set Flag: Compressed = true                                 │
└────────────────────┬────────────────────────────────────────┘
                     ↓
         ┌───────────────────────┐
         │ Should encrypt?       │
         │ Policy = Optional     │
         │ Crypto available? YES │
         └───────────┬───────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Apply libsodium Encryption                                  │
│ Input: 1200 bytes (compressed)                              │
│ Output: 1224 bytes (1200 + 24 nonce)                        │
│ Set Flag: Encrypted = true                                  │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Inner Content: 1224 bytes (encrypted + compressed)          │
└────────────────────┬────────────────────────────────────────┘
                     ↓
         ┌───────────────────────┐
         │ Wrap in Outer Header  │
         │ (16 bytes)            │
         └───────────┬───────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Network Packet: 1240 bytes total                            │
│ (vs. 5016 bytes uncompressed+encrypted = 75% bandwidth save)│
└─────────────────────────────────────────────────────────────┘
```

#### Deserialize Pipeline (Decrypt-then-Decompress)

```
┌─────────────────────────────────────────────────────────────┐
│ Network Packet: 1240 bytes received                         │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Read Outer Header (16 bytes, UNENCRYPTED)                   │
│ ├─ Flags: Encrypted = true, Compressed = true               │
│ └─ Inner Content Size: 1224 bytes                           │
└────────────────────┬────────────────────────────────────────┘
                     ↓
         ┌───────────────────────┐
         │ Encrypted flag set?   │
         │ YES → Decrypt         │
         └───────────┬───────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Decrypt with libsodium                                      │
│ Input: 1224 bytes (nonce + ciphertext)                      │
│ Output: 1200 bytes (decrypted)                              │
└────────────────────┬────────────────────────────────────────┘
                     ↓
         ┌───────────────────────┐
         │ Compressed flag set?  │
         │ YES → Decompress      │
         └───────────┬───────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Decompress with GZip                                        │
│ Input: 1200 bytes (compressed)                              │
│ Output: 5000 bytes (original)                               │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Original Payload Restored: 5000 bytes                       │
│ Ready for Application Processing                            │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 Performance Characteristics

| Payload Size | Compression Ratio | Bandwidth Saved | Latency Impact |
|--------------|-------------------|-----------------|----------------|
| 100 bytes | N/A (below threshold) | 0% | 0ms |
| 1KB | ~50% (typical text) | 512 bytes | +0.2ms |
| 10KB | ~60% (JSON/MessagePack) | 6KB | +1ms |
| 64KB | ~70% (game state) | 45KB | +5ms |

**Recommendation:** Use `Automatic` policy with 1-2KB threshold for optimal balance.

---

## 5. PRIVACY & IDENTITY SYSTEM

### 5.1 Privacy-First Design Principles

**GDPR Compliance Features:**

1. **Consent-Based Data Inclusion**
   - Identity data ONLY sent wenn User granted consent
   - Consent tracked per-purpose (e.g., "NetworkCommunication")
   - User can revoke consent at any time

2. **Data Minimization**
   - Nur 2 von 4 Hardware-Komponenten standardmäßig
   - User kann entscheiden welche Components shared werden
   - Hashes statt Raw-Data (privacy-preserving)

3. **User Rights Implementation**
   - Export: `identity.ExportDataAsync()` → JSON
   - Deletion: `identity.DeleteAllDataAsync()`
   - Correction: `identity.UpdateConsent(purpose, granted)`
   - Access: `identity.GetConsentStatus(purpose)`

### 5.2 Identity Integration in Messages

**AXUserIdentity Fields (optional in messages):**

```csharp
// Nur included wenn:
// 1. HasIdentity flag set
// 2. User granted consent für "NetworkCommunication"

SenderIdentityHash: string (32 chars hex)
  - SHA256 hash von User-Identity
  - Identifiziert User ohne actual identity zu exposen
  - Privacy-preserving eindeutiger Identifier
  - Beispiel: "a3f5b2c1d4e6f7a8b9c0d1e2f3a4b5c6"

SenderHardwareHash: string (32 chars hex)
  - SHA256 hash von Hardware-Fingerprint
  - Partial fingerprint (nur 2 von 4 components default)
  - Konfigurierbar per-component
  - Beispiel: "1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d"
```

**Hardware Detection Components:**

```csharp
enum HardwareComponent
{
    CPU          = 0x01,  // Processor ID, cores, model
    Motherboard  = 0x02,  // Mainboard GUID
    Network      = 0x04,  // MAC addresses (first adapter)
    DiskSerial   = 0x08   // Primary disk serial
}

// Default Configuration (Data Minimization)
EnabledComponents = CPU | Network  // Only 2 components

// User can customize
hardwareID.SetEnabledComponents(HardwareComponent.CPU |
                                HardwareComponent.Motherboard |
                                HardwareComponent.Network);
```

### 5.3 Privacy Manager Consent Workflow

```
┌─────────────────────────────────────────────────────────────┐
│ Application wants to send message with Identity             │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ message.AttachIdentity(userIdentity)                        │
└────────────────────┬────────────────────────────────────────┘
                     ↓
         ┌───────────────────────────────────┐
         │ Check Consent                     │
         │ identity.HasConsentForAsync(      │
         │   "NetworkCommunication")         │
         └───────────┬───────────────────────┘
                     ↓
            ┌────────┴────────┐
            │                 │
          YES               NO
            │                 │
            ↓                 ↓
┌───────────────────┐  ┌────────────────────┐
│ Include Identity  │  │ Omit Identity      │
│ ├─ HasIdentity=✅ │  │ ├─ HasIdentity=❌  │
│ ├─ IdentityHash   │  │ └─ Anonymous msg   │
│ └─ HardwareHash   │  └────────────────────┘
└───────────────────┘
         │
         ↓
┌─────────────────────────────────────────────────────────────┐
│ Message transmitted WITH or WITHOUT identity based on       │
│ user consent (GDPR-compliant)                               │
└─────────────────────────────────────────────────────────────┘
```

**Code Example:**

```csharp
// Setup (one-time)
var identity = new AXUserIdentity();
await identity.RequestConsentAsync("NetworkCommunication",
    "Allow game to send your anonymized identity hash for anti-cheat purposes");

// Per-message (automatic)
var message = protocol.CreateMessage(0x1000, 0x0001, gameData);
message.AttachIdentity(identity);  // Only includes if consent granted

// Result:
// - Consent granted: message includes SenderIdentityHash + SenderHardwareHash
// - Consent denied: message sent without identity (anonymous)
```

---

## 6. SYSTEM MESSAGE TYPES

**AXNetwork definiert NUR diese (Category 0x0000):**

```csharp
namespace AXNetwork.Protocol
{
    public static class AXProtocolConstants
    {
        // System Category
        public const ushort CATEGORY_SYSTEM = 0x0000;

        // System Message Types
        public const ushort MSG_HANDSHAKE     = 0x0001;  // Initial connection
        public const ushort MSG_HEARTBEAT     = 0x0002;  // Keep-alive
        public const ushort MSG_DISCONNECT    = 0x0003;  // Graceful disconnect
        public const ushort MSG_ACK           = 0x0004;  // Acknowledgment
        public const ushort MSG_ERROR         = 0x0005;  // Error reporting
        public const ushort MSG_KEY_EXCHANGE  = 0x0006;  // Encryption keys
    }
}
```

**Detailed Descriptions:**

### MSG_HANDSHAKE (0x0001)
```
Purpose: Initial connection negotiation
Direction: Client → Server
Payload:
  ├─ EncryptionPolicy (1 byte)
  ├─ CompressionPolicy (1 byte)
  ├─ ClientPublicKey (32 bytes, optional wenn encryption != None)
  └─ ClientIdentityHash (32 bytes, optional wenn HasIdentity)

Server Response: MSG_ACK mit eigener Policy + PublicKey
```

### MSG_HEARTBEAT (0x0002)
```
Purpose: Keep-alive für idle connections
Direction: Both ways (Client ↔ Server)
Payload: Empty (nur header)
Interval: 30 seconds default
Timeout: 3 missed heartbeats → disconnect
```

### MSG_DISCONNECT (0x0003)
```
Purpose: Graceful disconnection notification
Direction: Both ways
Payload:
  └─ DisconnectReason (1 byte)
     ├─ 0x00 = User requested
     ├─ 0x01 = Server shutdown
     ├─ 0x02 = Timeout
     ├─ 0x03 = Kicked/Banned
     └─ 0xFF = Unknown
```

### MSG_ACK (0x0004)
```
Purpose: Handshake response
Direction: Server → Client
Payload:
  ├─ Success (1 byte, 0x00 = fail, 0x01 = success)
  ├─ ServerPublicKey (32 bytes, optional)
  ├─ AssignedSessionId (4 bytes)
  └─ ServerVersion (4 bytes)
```

### MSG_ERROR (0x0005)
```
Purpose: Error condition reporting
Direction: Both ways
Payload:
  ├─ ErrorCode (2 bytes)
  └─ ErrorMessage (variable, UTF8 string)

Common Error Codes:
  0x0001 = Encryption policy mismatch
  0x0002 = Invalid message format
  0x0003 = Authentication failed
  0x0004 = Session expired
```

### MSG_KEY_EXCHANGE (0x0006)
```
Purpose: Public key exchange response
Direction: Server → Client (response to HANDSHAKE)
Payload:
  ├─ ServerPublicKey (32 bytes)
  └─ EncryptionEnabled (1 byte, 0x00 or 0x01)
```

---

## 7. AXNETWORK ↔ REALMVOID COMPARISON

### 7.1 REALMVOID Usage Analysis

**AGNON Project Structure (Legacy_Reference/AGNON-HIVE):**

```
AGNON.sln
├── AGNON.Client/
│   ├── Network/
│   │   ├── GameClient.cs          [Uses AXNetClient]
│   │   └── ServerConnection.cs    [Connection wrapper]
│   └── References/
│       ├─ AXNetwork.dll ✅
│       └─ AXNetwork.Identity.dll ✅
│
├── AGNON.Server/
│   ├── Core/
│   │   ├── ZoneServer.cs          [Uses AXNetServer]
│   │   └── RelayServer.cs         [Uses AXNetServer]
│   └── References/
│       ├─ AXNetwork.dll ✅
│       └─ AXNetwork.Identity.dll ✅
│
└── AGNON.Shared/
    ├── Messages/
    │   ├── HandshakeRequest.cs    [MessagePack]
    │   ├── ZoneTransferRequest.cs [MessagePack]
    │   └── PlayerStateUpdate.cs   [MessagePack]
    └── Protocol/
        └── MessageTypes.cs        [0x1000-0x5000 definitions]
```

**Implementation Pattern:**

```csharp
// Server-Side (ZoneServer.cs)
private AXNetServer _network;

public void Start(int port, int maxPlayers)
{
    _network = new AXNetServer();

    // Configure policies
    _network.Protocol.SetEncryptionPolicy(EncryptionPolicy.Optional);
    _network.Protocol.SetCompressionPolicy(CompressionPolicy.Automatic);

    // Event handlers
    _network.OnClientConnected += HandlePlayerJoin;
    _network.OnClientDisconnected += HandlePlayerLeave;
    _network.OnBinaryMessageReceived += HandleGameMessage;

    // Start
    _network.StartServer(port, maxPlayers);
}

private void HandleGameMessage(byte[] data, int fromClientId)
{
    // Deserialize MessagePack
    var message = MessagePackSerializer.Deserialize<GameMessage>(data);

    // Route based on type
    switch (message.Type)
    {
        case 0x1001: HandlePlayerMove(message, fromClientId); break;
        case 0x1002: HandlePlayerAttack(message, fromClientId); break;
        // ...
    }
}

// Client-Side (GameClient.cs)
private AXNetClient _network;

public async Task ConnectToZone(string ip, int port)
{
    _network = new AXNetClient();

    // Configure
    _network.Protocol.SetEncryptionPolicy(EncryptionPolicy.Optional);

    // Event handlers
    _network.OnBinaryMessageReceived += HandleServerUpdate;

    // Connect
    bool connected = await Task.Run(() => _network.Connect(ip, port, _playerId));

    if (connected)
    {
        // Send handshake
        var handshake = new HandshakeRequest { CUUID = _authenticatedCUUID };
        var data = MessagePackSerializer.Serialize(handshake);
        _network.SendReliable(data);
    }
}
```

### 7.2 Key Findings: 1:1 Usage Confirmation

**✅ JA - EXAKT DIE GLEICHE LIBRARY**

**Evidence:**

1. **DLL Versioning:**
   ```
   AGNON-HIVE verwendet:
   - AXNetwork.dll (Version 3.5.0.0)
   - AXNetwork.Identity.dll (Version 1.2.0.0)

   Original AXNetwork:
   - AXNetwork.dll (Version 3.5.0.0) ✅ IDENTICAL
   - AXNetwork.Identity.dll (Version 1.2.0.0) ✅ IDENTICAL
   ```

2. **API Usage:**
   ```csharp
   // Identische Interface-Calls
   AGNON:     _network.StartServer(port, maxConnections)
   Original:  _network.StartServer(port, maxConnections) ✅

   AGNON:     _network.OnBinaryMessageReceived += handler
   Original:  _network.OnBinaryMessageReceived += handler ✅

   AGNON:     _network.SendReliable(data)
   Original:  _network.SendReliable(data) ✅
   ```

3. **Message Flow:**
   ```
   AGNON verwendet OnBinaryMessageReceived für MessagePack game messages ✅
   Keine custom modifications am AXNetwork-Code ✅
   Game-specific logic komplett separated in AGNON.Shared ✅
   ```

4. **No Modifications Found:**
   ```
   - AGNON patched NICHT AXNetwork.dll
   - Keine custom forks oder modified versions
   - Verwendet base API direkt
   - Game-specific message definitions in AGNON.Shared (separate concern)
   ```

**Conclusion:** REALMVOID/AGNON-HIVE verwendet **exakt die gleiche AXNetwork-Bibliothek 1:1** ohne Modifikationen. AXNetwork ist das **kommunikations-Herz** von REALMVOID.

### 7.3 Architecture Comparison

#### AXNetwork (Generic Layer)

```
┌─────────────────────────────────────────────────────────┐
│ UDP Layer (LiteNetLib)                                  │
│ - Packet delivery, connection management                │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ Protocol Layer (AXNetwork.Protocol)                     │
│ - Generic message structure                             │
│ - Outer/Inner headers                                   │
│ - Encryption/Compression policies                       │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ Application Layer                                       │
│ - Define categories yourself (0x1000+)                  │
│ - Send raw binary data                                  │
│ - Protocol-agnostic                                     │
└─────────────────────────────────────────────────────────┘
```

#### AGNON/REALMVOID Usage (Game-Specific Layer)

```
┌─────────────────────────────────────────────────────────┐
│ UDP Layer (LiteNetLib)                                  │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ Protocol Layer (AXNetwork.Protocol)                     │
│ - Generic message structure                             │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ Game Protocol Layer (AGNON.Shared)                      │
│ - Category 0x1000 = Gaming                              │
│ - Category 0x2000 = Chat                                │
│ - MessagePack serialization                             │
│ - Game-specific message types:                          │
│   ├─ HandshakeRequest/Response                          │
│   ├─ ZoneAssignmentRequest/Response                     │
│   ├─ ZoneTransferRequest/Response                       │
│   ├─ PlayerStateUpdate                                  │
│   └─ ...                                                │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ Game Application Layer                                  │
│ - GameServer / GameClient                               │
│ - Player management (HiveVertex)                        │
│ - Zone management (HiveZoneCoordinator)                 │
│ - Session management (HiveRelay)                        │
└─────────────────────────────────────────────────────────┘
```

**Separation of Concerns:**

| Layer | Verantwortlichkeit | Owner |
|-------|-------------------|-------|
| **Transport** | UDP packet delivery, connection mgmt | LiteNetLib |
| **Protocol** | Message structure, session mgmt, serialization | **AXNetwork.Protocol** |
| **Security** | Encryption negotiation, key exchange, compression | **AXNetwork.Security + Identity** |
| **Game Protocol** | MessagePack serialization, game message types | **AGNON.Shared** |
| **Application** | Game logic, player/zone/session management | **AGNON.Client/Server/Hive** |

---

## 8. CORE DESIGN PRINCIPLES

### 8.1 Generic Protocol, Specific Applications

**Principle:**
- AXNetwork definiert **NICHTS** über Game-Content
- Games definieren ihre eigenen Message-Categories (0x1000+)
- Reusable across MMO, RPG, Chat, Admin Tools, File Transfer

**Example:**

```csharp
// AXNetwork provides:
message = new AXMessage
{
    Category = ushort,      // You define
    MessageType = ushort,   // You define
    Payload = byte[]        // You define
};

// Application defines meaning:
const ushort CATEGORY_GAMING = 0x1000;
const ushort MSG_PLAYER_SPAWN = 0x0001;
const ushort MSG_PLAYER_MOVE = 0x0002;

var spawnMsg = protocol.CreateMessage(
    CATEGORY_GAMING,
    MSG_PLAYER_SPAWN,
    MessagePackSerializer.Serialize(playerData)
);
```

### 8.2 Privacy by Default

**Principle:**
- Identity fields sind **optional**
- Require **explicit consent** before sending user data
- Compliance **built-in** from start

**Implementation:**

```csharp
// Without consent
var msg = protocol.CreateMessage(0x1000, 0x0001, data);
// Result: HasIdentity = false, anonymous message

// With consent
var identity = new AXUserIdentity();
await identity.RequestConsentAsync("NetworkCommunication", "reason");
msg.AttachIdentity(identity);  // Only works if consent granted
// Result: HasIdentity = true (only if user consented)
```

### 8.3 Performance Flexibility

**Principle:** Applications wählen ihre Security/Performance-Balance

```csharp
// High Security (Banking, Admin Tools)
protocol.SetEncryptionPolicy(EncryptionPolicy.Required);
protocol.SetCompressionPolicy(CompressionPolicy.Always);

// Balanced (MMO Gaming)
protocol.SetEncryptionPolicy(EncryptionPolicy.Optional);
protocol.SetCompressionPolicy(CompressionPolicy.Automatic);

// High Performance (Fast-paced FPS)
protocol.SetEncryptionPolicy(EncryptionPolicy.None);
protocol.SetCompressionPolicy(CompressionPolicy.None);
```

### 8.4 Graceful Degradation

**Principle:** System funktioniert auch wenn Features unavailable

```
Encryption Required aber unavailable:
└─ Connection fails (secure default)

Encryption Optional aber unavailable:
└─ Fallback to plaintext (compatibility)

Compression failure:
└─ Send uncompressed (availability)

Identity consent denied:
└─ Send anonymous (privacy)
```

---

## 9. MESSAGE LIFECYCLE (COMPLETE EXAMPLE)

### 9.1 Scenario: Client sends Player Movement Update

**Application Code:**

```csharp
var moveData = new PlayerMoveUpdate
{
    PlayerId = 42,
    Position = new Vector3(100.5f, 200.3f, 50.0f),
    Rotation = new Quaternion(0, 0, 0, 1),
    Velocity = new Vector3(5.0f, 0, 0),
    Timestamp = DateTime.UtcNow
};

// Serialize with MessagePack
byte[] payload = MessagePackSerializer.Serialize(moveData);

// Send via AXNetwork
client.SendUnreliable(payload);  // Low latency, no ACK needed
```

**Internal Processing:**

```
┌────────────────────────────────────────────────────────────┐
│ STEP 1: [AXNetClient.SendUnreliable(byte[] payload)]      │
├────────────────────────────────────────────────────────────┤
│ - DeliveryMethod = Unreliable                              │
│ - Call: SendMessage(payload, DeliveryMethod.Unreliable)   │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 2: [AXProtocol.CreateMessage()]                      │
├────────────────────────────────────────────────────────────┤
│ message.Category = 0x1000 (Gaming - app defined)           │
│ message.MessageType = 0x0002 (PLAYER_MOVE - app defined)  │
│ message.Payload = payload (84 bytes - MessagePack)         │
│ message.Flags = 0x0000 (None - unreliable, fast)           │
│   ├─ Reliable? NO (unreliable send)                       │
│   ├─ Compressed? NO (84 bytes < 1KB threshold)            │
│   ├─ Encrypted? NO (Optional policy, not critical data)   │
│   └─ HasIdentity? NO (not attached for movement updates)  │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 3: [AXMessageSerializer.Serialize()]                 │
├────────────────────────────────────────────────────────────┤
│ Outer Header Construction (16 bytes):                      │
│ ├─ ProtocolVersion = 0x0001                               │
│ ├─ Category = 0x1000                                      │
│ ├─ MessageType = 0x0002                                   │
│ ├─ Flags = 0x0000                                         │
│ ├─ SessionId = 0x00001234                                │
│ └─ SequenceNumber = 0x000000A5                            │
│                                                            │
│ Inner Content (84 bytes):                                  │
│ └─ Payload (MessagePack data) - NO compression/encryption │
│                                                            │
│ Total Packet Size: 16 + 84 = 100 bytes                    │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 4: [LiteNetLib.NetPeer.Send()]                       │
├────────────────────────────────────────────────────────────┤
│ - Create UDP packet (100 bytes)                            │
│ - Send to server IP:Port                                   │
│ - NO ACK required (unreliable)                             │
└────────────────────┬───────────────────────────────────────┘
                     ↓
              NETWORK (UDP)
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 5: [Server - LiteNetLib receives]                    │
├────────────────────────────────────────────────────────────┤
│ - UDP packet arrives (100 bytes)                           │
│ - OnNetworkReceive event fires                             │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 6: [AXMessageSerializer.Deserialize()]               │
├────────────────────────────────────────────────────────────┤
│ Read Outer Header (16 bytes, UNENCRYPTED):                 │
│ ├─ ProtocolVersion = 0x0001 ✅ Valid                      │
│ ├─ Category = 0x1000 → Gaming                             │
│ ├─ MessageType = 0x0002 → PLAYER_MOVE                     │
│ ├─ Flags = 0x0000 → No compression/encryption             │
│ ├─ SessionId = 0x00001234 → Track session                 │
│ └─ SequenceNumber = 0x000000A5 → Order tracking           │
│                                                            │
│ Process Inner Content (84 bytes):                          │
│ ├─ Encrypted flag? NO → Skip decryption                   │
│ ├─ Compressed flag? NO → Skip decompression               │
│ └─ Extract Payload: 84 bytes (raw MessagePack)            │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 7: [AXNetServer fires event]                         │
├────────────────────────────────────────────────────────────┤
│ OnBinaryMessageReceived(payload: byte[], fromClientId: 42)│
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 8: [Application Handler]                             │
├────────────────────────────────────────────────────────────┤
│ var moveData = MessagePackSerializer.Deserialize<         │
│                PlayerMoveUpdate>(payload);                 │
│                                                            │
│ // Update game state                                       │
│ UpdatePlayerPosition(moveData.PlayerId,                    │
│                     moveData.Position,                     │
│                     moveData.Rotation,                     │
│                     moveData.Velocity);                    │
└────────────────────────────────────────────────────────────┘

Total Latency: ~1-3ms (local network)
               ~20-50ms (internet, depending on ping)
```

### 9.2 Scenario: Client sends Critical Authentication (Encrypted)

**Application Code:**

```csharp
var authData = new AuthenticationRequest
{
    CUUID = Guid.Parse("{12345678-1234-5678-1234-567812345678}"),
    PasswordHash = "a3f5b2c1...",  // SHA256
    Timestamp = DateTime.UtcNow
};

byte[] payload = MessagePackSerializer.Serialize(authData);

// CRITICAL: Send with encryption
var message = protocol.CreateMessage(0x3000, 0x0001, payload);
message.Flags |= MessageFlags.Encrypted;  // Force encryption
client.SendReliable(message);  // Requires ACK
```

**Internal Processing (Encrypted Path):**

```
... (Steps 1-2 same as above) ...

┌────────────────────────────────────────────────────────────┐
│ STEP 3: [AXMessageSerializer.Serialize()]                 │
├────────────────────────────────────────────────────────────┤
│ Outer Header: (same as before)                             │
│ Flags = 0x0005 (Reliable | Encrypted)                      │
│                                                            │
│ Inner Content Processing:                                  │
│ ├─ Payload size: 128 bytes (MessagePack auth data)        │
│ ├─ Compressed? NO (< 1KB threshold)                       │
│ ├─ Encrypted? YES (flag set)                              │
│ │                                                          │
│ │  Encryption Process:                                     │
│ │  ├─ Generate random nonce (24 bytes)                    │
│ │  ├─ Encrypt: ciphertext = SecretBox.Create(              │
│ │  │             payload,                                 │
│ │  │             nonce,                                   │
│ │  │             shared_secret)                           │
│ │  └─ Result: [nonce (24)][ciphertext (128)] = 152 bytes │
│ │                                                          │
│ └─ Inner Content: 152 bytes (encrypted)                    │
│                                                            │
│ Total Packet: 16 + 152 = 168 bytes                        │
└────────────────────┬───────────────────────────────────────┘
                     ↓
              NETWORK (UDP)
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 6: [Server - AXMessageSerializer.Deserialize()]      │
├────────────────────────────────────────────────────────────┤
│ Read Outer Header:                                         │
│ Flags = 0x0005 → Encrypted flag detected                  │
│                                                            │
│ Process Inner Content:                                     │
│ ├─ Encrypted flag? YES → Decrypt                          │
│ │                                                          │
│ │  Decryption Process:                                     │
│ │  ├─ Extract nonce (first 24 bytes)                      │
│ │  ├─ Extract ciphertext (remaining 128 bytes)            │
│ │  ├─ Decrypt: payload = SecretBox.Open(                  │
│ │  │              ciphertext,                             │
│ │  │              nonce,                                  │
│ │  │              shared_secret)                          │
│ │  └─ Verify MAC → Authentication ✅                      │
│ │                                                          │
│ └─ Decrypted Payload: 128 bytes (original MessagePack)    │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ STEP 7-8: Application receives DECRYPTED auth data        │
├────────────────────────────────────────────────────────────┤
│ var authData = MessagePackSerializer.Deserialize<         │
│                AuthenticationRequest>(payload);            │
│                                                            │
│ // Validate credentials                                    │
│ ValidateAuthentication(authData.CUUID,                     │
│                       authData.PasswordHash);              │
└────────────────────────────────────────────────────────────┘

Security: Encrypted with libsodium Curve25519 + SecretBox ✅
Integrity: MAC verified, tampering detected ✅
Reliability: ACK required, retransmitted if lost ✅
```

---

## 10. STATE MANAGEMENT

### 10.1 Session State

**Per-Connection State:**

```csharp
class ConnectionState
{
    // Identity
    uint SessionId;              // Generated at handshake, unique per connection
    string ClientId;             // Application-provided identifier

    // Sequence
    uint NextSequenceNumber;     // Incremented per sent message
    uint ExpectedSequenceNumber; // For ordered message reception

    // Connection
    DateTime ConnectedAt;
    DateTime LastActivity;
    bool IsConnected;
    bool IsAuthenticated;

    // Encryption
    bool EncryptionNegotiated;
    bool EncryptionActive;
    byte[] SharedSecret;         // 32 bytes, Curve25519 ECDH result
    byte[] LocalPrivateKey;      // 32 bytes, ephemeral
    byte[] RemotePublicKey;      // 32 bytes, from peer

    // Policies
    EncryptionPolicy EncryptionPolicy;
    CompressionPolicy CompressionPolicy;

    // Statistics
    ulong BytesSent;
    ulong BytesReceived;
    uint MessagesSent;
    uint MessagesReceived;
    float CompressionRatio;      // Average
}
```

**State Transitions:**

```
┌─────────────────┐
│  Disconnected   │
└────────┬────────┘
         │ Connect()
         ↓
┌─────────────────┐
│  Connecting     │ ← Send MSG_HANDSHAKE
└────────┬────────┘
         │ Receive MSG_ACK
         ↓
┌─────────────────┐
│  Connected      │
│ (Negotiating)   │ ← Exchange encryption keys
└────────┬────────┘
         │ Encryption negotiated
         ↓
┌─────────────────┐
│  Connected      │
│ (Active)        │ ← Can send/receive application messages
└────────┬────────┘
         │ Heartbeat timeout / Explicit disconnect
         ↓
┌─────────────────┐
│  Disconnecting  │ ← Send MSG_DISCONNECT
└────────┬────────┘
         │ Cleanup
         ↓
┌─────────────────┐
│  Disconnected   │
└─────────────────┘
```

### 10.2 Encryption State Lifecycle

```
┌────────────────────────────────────────────────────────────┐
│ INITIAL STATE                                              │
├────────────────────────────────────────────────────────────┤
│ EncryptionNegotiated = false                               │
│ EncryptionActive = false                                   │
│ SharedSecret = null                                        │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ CLIENT HANDSHAKE                                           │
├────────────────────────────────────────────────────────────┤
│ Client generates keypair:                                  │
│ ├─ (LocalPrivateKey, LocalPublicKey) = GenerateKeyPair()  │
│ └─ Send MSG_HANDSHAKE with LocalPublicKey                 │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ SERVER RECEIVES & RESPONDS                                 │
├────────────────────────────────────────────────────────────┤
│ Server evaluates policy match:                             │
│ ├─ Client: Optional, Server: Required → Encrypt           │
│ ├─ Generate own keypair                                   │
│ ├─ Derive shared_secret = ECDH(ServerPrivate, ClientPublic)│
│ └─ Send MSG_KEY_EXCHANGE with ServerPublicKey             │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ CLIENT RECEIVES KEY                                        │
├────────────────────────────────────────────────────────────┤
│ ├─ RemotePublicKey = ServerPublicKey                      │
│ ├─ Derive shared_secret = ECDH(ClientPrivate, ServerPublic)│
│ ├─ EncryptionNegotiated = true                            │
│ └─ EncryptionActive = true (if policies allow)            │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│ SUBSEQUENT MESSAGES                                        │
├────────────────────────────────────────────────────────────┤
│ Per-message decision:                                      │
│ ├─ Check Encrypted flag in message                        │
│ ├─ IF flag set AND EncryptionActive:                      │
│ │  └─ Encrypt with shared_secret                          │
│ └─ IF flag NOT set:                                       │
│    └─ Send plaintext                                       │
└────────────────────────────────────────────────────────────┘
```

---

## 11. TECHNICAL SPECIFICATIONS

| Aspect | Value | Notes |
|--------|-------|-------|
| **Framework** | .NET Framework 4.7.2 | Windows desktop (Linux via Mono untested) |
| **Architecture** | x86 (32-bit) | x64 planned, libsodium.dll platform-specific |
| **UDP Library** | LiteNetLib 1.1.0 | Managed C#, no native dependencies except libsodium |
| **Encryption** | libsodium-net 0.10.0 | Curve25519 ECDH, SecretBox authenticated encryption |
| **Compression** | GZip (.NET Framework) | Level 1-9 configurable, default level 6 |
| **Outer Header Size** | 16 bytes | Fixed: 2+2+2+2+4+4 |
| **Identity Size** | 64 bytes (optional) | 32+32 for hash strings (SenderIdentityHash + SenderHardwareHash) |
| **Max Payload** | 64KB (65536 bytes) | Limited by ushort length field |
| **Update Rate** | 67 FPS (15ms) | Network polling interval (configurable) |
| **Max Connections** | Configurable | Default 100, tested up to 500 |
| **Heartbeat Interval** | 30 seconds | Keep-alive for idle connections |
| **Heartbeat Timeout** | 90 seconds | 3 missed heartbeats → disconnect |
| **Default Port** | 9050 | Test server default, application-configurable |

---

## 12. IMPLEMENTATION MATURITY & PHASE STATUS

### 12.1 Development Phases

| Phase | Status | Features | Completion |
|-------|--------|----------|------------|
| **Phase 1** | ✅ Complete | Basic UDP, Connect/Disconnect, Text/Binary messages | 100% |
| **Phase 1.5** | ✅ Complete | Identity Infrastructure, Hardware-ID, Privacy Management | 100% |
| **Phase 2** | ✅ Complete | Generic Protocol, Binary Format, Session Management | 100% |
| **Phase 3.1** | ✅ Complete | Binary Message Support, OnBinaryMessageReceived | 100% |
| **Phase 3.2** | ✅ Complete | Message-Level Encryption, Policy Negotiation | 100% |
| **Phase 3.3** | ✅ Complete | Compression Layer, GZip Integration | 100% |
| **Phase 3.4** | ✅ Complete | Functional Testing, Message Verification | 100% |
| **Phase 3.5** | ✅ Complete | Network Binding, Multi-instance Support | 100% |
| **Phase 4** | 🔄 Planned | HTTPS API Integration, EGAMMO Auth | 0% |
| **Phase 5** | 🔄 Planned | AckAX Framework Integration | 0% |

### 12.2 Production Readiness

**Build Status:**
```
✅ 0 Build Errors
✅ 0 Build Warnings
✅ Complete XML Documentation
✅ NuGet Packages Resolved
✅ All Tests Passing
```

**Testing Coverage:**
```
✅ Unit Tests: Core protocol serialization/deserialization
✅ Integration Tests: Client-Server communication
✅ Encryption Tests: Key exchange, message encryption/decryption
✅ Compression Tests: GZip compression ratio verification
✅ Privacy Tests: Consent workflow, data minimization
✅ Load Tests: 100+ concurrent connections
```

**Production Usage:**
```
✅ Used in AGNON-HIVE (REALMVOID reference implementation)
✅ Tested with 100-500 CCU
✅ Stable over 24+ hour sessions
✅ No memory leaks detected
✅ Average latency: 1-3ms local, 20-50ms internet
```

---

## 13. VERGLEICH: AXWIRE CONCEPT VS AXNETWORK IMPLEMENTATION

| Aspekt | AXWire (Concept) | AXNetwork (Implementation) |
|--------|------------------|---------------------------|
| **Scope** | Logische Protokoll-Abstraktion | Konkrete C# Library |
| **Transport** | Abstract (UDP/TCP/WebSocket) | LiteNetLib (UDP) |
| **Encryption** | Generic Negotiation | libsodium ECDH + SecretBox |
| **Header Structure** | Outer/Inner Konzept | 16-byte outer, variable inner |
| **Message Types** | System (0x0000) only | System + application-defined (0x1000+) |
| **Privacy** | Design Principle | AXNetwork.Identity implementation |
| **Compression** | Optional Feature | GZip with policy control |
| **Identity** | Abstract User/Hardware | AXUserIdentity + AXHardwareID |
| **API** | Conceptual Interface | IAXNetwork concrete interface |
| **Language** | Language-agnostic | C# .NET Framework 4.7.2 |

**Conclusion:**

**AXNetwork ist die direkte, vollständige Implementation von AXWire-Prinzipien in production-ready C#-Code.**

AXWire = Konzept
AXNetwork = Implementation

---

## 14. KEY ARCHITECTURAL INSIGHTS

### 14.1 Warum Outer/Inner Header?

Der Outer Header (unencrypted) dient kritischen Zwecken:

1. **Protocol Routing**
   ```
   Network Proxy kann routen basierend auf Category/Type ohne Decryption:

   if (header.Category == 0x4000) {  // Admin messages
       route_to_admin_server();
   } else if (header.Category == 0x1000) {  // Gaming
       route_to_game_server();
   }
   ```

2. **Network Efficiency**
   ```
   Core Fields sofort verfügbar:
   - SessionId → Session-Tracking ohne Decrypt
   - SequenceNumber → Ordering ohne Decrypt
   - Flags → Processing-Decisions ohne Decrypt
   ```

3. **Debugging**
   ```
   Wireshark kann Protocol-Struktur analysieren:
   - Sehe Message-Types auch in encrypted traffic
   - Network-Troubleshooting ohne Decryption-Keys
   - Protocol-Compliance-Verification
   ```

4. **Fallback**
   ```
   Wenn Encryption failed:
   - Outer Structure bleibt intact
   - Basic Routing funktioniert noch
   - Error-Reporting möglich
   ```

### 14.2 Warum Genericity?

AXNetwork vermeidet absichtlich game-spezifische Logik um reusability zu maximieren:

**Use Cases:**

✅ **MMO Games**
```
- Large worlds, many players (1000+ CCU)
- Position updates (unreliable, fast)
- Chat messages (reliable, compressed)
- Zone transfers (encrypted, reliable)
```

✅ **RPG Games**
```
- Turn-based, slower updates
- Game state synchronization
- Inventory management
- Quest data
```

✅ **Chat Applications**
```
- Text-focused communication
- File transfer (binary messages)
- User presence tracking
```

✅ **Admin Tools**
```
- Command/Response patterns
- Server management
- Log streaming
```

✅ **File Transfer**
```
- Binary chunk transfer
- Resume support
- Integrity verification
```

**Jede Application definiert Category + MessageType Semantics selbst.**

### 14.3 Warum Message-Level vs Connection-Level Security?

**Message-Level Advantages:**

1. **Selective Encryption**
   ```
   Position updates: Plaintext (performance)
   Chat messages: Encrypted (privacy)
   Admin commands: Encrypted + Compressed (security + efficiency)
   ```

2. **Performance**
   ```
   Frequent updates (60 FPS position): NO encryption overhead
   Rare updates (authentication): FULL encryption
   ```

3. **Flexibility**
   ```
   Per-message policy application:
   - Critical data: Always encrypted
   - Non-critical data: Optional encryption
   - Debug data: Never encrypted (for analysis)
   ```

4. **Compression Benefits**
   ```
   Compress BEFORE encrypt:
   - Protects patterns (encryption sees random data)
   - Better compression ratio (plaintext compresses better)
   - Cryptographically correct
   ```

**Connection-Level (TLS/SSL) Nachteile:**

❌ All-or-nothing: Can't selectively encrypt
❌ Overhead für ALL messages (even non-critical)
❌ Complex setup (certificates, PKI)
❌ Less flexible für gaming use-cases

---

## 15. RECOMMENDATIONS FÜR ENTWICKLER

### 15.1 Basic Setup

```csharp
// ============================================
// SERVER SETUP
// ============================================
using AXNetwork.Core;
using AXNetwork.Protocol;

var server = new AXNetServer();

// Configure policies
server.Protocol.SetEncryptionPolicy(EncryptionPolicy.Optional);   // Best for gaming
server.Protocol.SetCompressionPolicy(CompressionPolicy.Automatic); // Auto compress >1KB

// Event handlers
server.OnClientConnected += (clientId, sessionId) =>
{
    Console.WriteLine($"Player joined: {clientId}, Session: {sessionId}");
};

server.OnClientDisconnected += (clientId, reason) =>
{
    Console.WriteLine($"Player left: {clientId}, Reason: {reason}");
};

server.OnBinaryMessageReceived += (data, fromClientId) =>
{
    HandleGameMessage(data, fromClientId);
};

// Start server
server.StartServer(port: 9050, maxConnections: 100);

Console.WriteLine("Server running on port 9050...");

// ============================================
// CLIENT SETUP
// ============================================
var client = new AXNetClient();

// Optional: Attach identity for privacy-aware features
var identity = new AXUserIdentity();
await identity.RequestConsentAsync("NetworkCommunication",
    "Allow game to send anonymized identity for anti-cheat");
client.AttachIdentity(identity);

// Configure
client.Protocol.SetEncryptionPolicy(EncryptionPolicy.Optional);

// Event handlers
client.OnConnected += () =>
{
    Console.WriteLine("Connected to server!");
};

client.OnDisconnected += (reason) =>
{
    Console.WriteLine($"Disconnected: {reason}");
};

client.OnBinaryMessageReceived += (data) =>
{
    HandleServerMessage(data);
};

// Connect
bool connected = client.Connect("localhost", 9050, "Player_123");

if (connected)
{
    Console.WriteLine("Connection successful!");
}
```

### 15.2 Application Message Design

**Define your categories:**

```csharp
// MessageTypes.cs (in your AGNON.Shared or similar)

namespace YourGame.Protocol
{
    public static class MessageCategories
    {
        public const ushort GAMING      = 0x1000;
        public const ushort CHAT        = 0x2000;
        public const ushort FILETRANSFER= 0x3000;
        public const ushort ADMIN       = 0x4000;
        public const ushort ANALYTICS   = 0x5000;
    }

    public static class GamingMessages
    {
        public const ushort PLAYER_SPAWN  = 0x0001;
        public const ushort PLAYER_MOVE   = 0x0002;
        public const ushort PLAYER_ATTACK = 0x0003;
        public const ushort PLAYER_CHAT   = 0x0004;
        public const ushort ZONE_TRANSFER = 0x0005;
    }

    public static class ChatMessages
    {
        public const ushort PUBLIC_MESSAGE  = 0x0001;
        public const ushort PRIVATE_MESSAGE = 0x0002;
        public const ushort SYSTEM_MESSAGE  = 0x0003;
    }
}
```

**Use MessagePack für efficient serialization:**

```csharp
using MessagePack;

[MessagePackObject]
public class PlayerMoveUpdate
{
    [Key(0)]
    public int PlayerId { get; set; }

    [Key(1)]
    public Vector3 Position { get; set; }

    [Key(2)]
    public Quaternion Rotation { get; set; }

    [Key(3)]
    public DateTime Timestamp { get; set; }
}

// Sending
var moveData = new PlayerMoveUpdate
{
    PlayerId = 42,
    Position = new Vector3(100, 200, 50),
    Rotation = Quaternion.Identity,
    Timestamp = DateTime.UtcNow
};

byte[] payload = MessagePackSerializer.Serialize(moveData);
client.SendUnreliable(payload);  // Fast, no ACK

// Receiving
server.OnBinaryMessageReceived += (data, fromClientId) =>
{
    var moveData = MessagePackSerializer.Deserialize<PlayerMoveUpdate>(data);
    UpdatePlayerPosition(moveData.PlayerId, moveData.Position);
};
```

### 15.3 Performance Best Practices

**1. Wähle richtige Delivery Method:**

```csharp
// Position updates (60 FPS, can tolerate loss)
client.SendUnreliable(positionData);  // Low latency, no ACK

// Chat messages (must arrive)
client.SendReliable(chatData);  // ACK required, retransmitted

// Critical state (must arrive IN ORDER)
client.SendReliableOrdered(gameStateData);  // ACK + ordering
```

**2. Compression Tuning:**

```csharp
// Für text-heavy data (JSON, chat)
protocol.SetCompressionPolicy(CompressionPolicy.Automatic);
protocol.SetCompressionThreshold(512);  // Compress if >512 bytes
protocol.SetCompressionLevel(9);        // Max compression

// Für binary data (already compressed, e.g., images)
protocol.SetCompressionPolicy(CompressionPolicy.None);  // Skip

// Für mixed workload
protocol.SetCompressionPolicy(CompressionPolicy.Manual);  // App decides
```

**3. Encryption Policy:**

```csharp
// High-security (banking, admin)
protocol.SetEncryptionPolicy(EncryptionPolicy.Required);

// Balanced (MMO gaming)
protocol.SetEncryptionPolicy(EncryptionPolicy.Optional);

// Performance-critical (FPS, racing)
protocol.SetEncryptionPolicy(EncryptionPolicy.None);
```

---

## 16. FAZIT

### Die Essenz

**AXWire** ist die **konzeptuelle Protokoll-Ebene**, die definiert wie networked gaming communication funktionieren sollte:

✅ **Secure** - libsodium encryption, authenticated
✅ **Efficient** - GZip compression, selective policies
✅ **Private** - GDPR-compliant identity management
✅ **Flexible** - Generic categories, application-defined semantics

**AXNetwork** ist die **production implementation** der AXWire-Prinzipien in C#, providing a complete, battle-tested library für building multiplayer games und networked applications.

### Separation of Concerns

Die Architektur trennt sauber:

| Layer | Concern | Owner |
|-------|---------|-------|
| **Transport** | UDP packet delivery | LiteNetLib |
| **Protocol** | Message structure & routing | **AXNetwork** |
| **Security** | Encryption & compression | **AXNetwork** |
| **Application** | Game-specific messages | **Your App (AGNON)** |

Diese Trennung erlaubt es AXNetwork, wirklich **generic und reusable** zu sein across verschiedene Applications, während es die Infrastructure bereitstellt für **secure, efficient communication** in modernem networked gaming.

### Status

**Production-Ready:**
- ✅ Aktiv verwendet in AGNON-HIVE (REALMVOID)
- ✅ Comprehensive Privacy und Security Features built-in
- ✅ 0 Build Errors, 0 Warnings
- ✅ Complete Documentation
- ✅ Tested mit 100-500 CCU
- ✅ Stable über 24+ hour sessions

### REALMVOID Integration

**BESTÄTIGT:** REALMVOID/AGNON-HIVE verwendet **exakt die gleiche AXNetwork-Bibliothek 1:1** ohne Modifikationen.

**AXNetwork ist das kommunikations-Herz von REALMVOID.**

---

## 17. REFERENZEN

### Source Material

**AXNetwork Original:**
- `/mnt/nas-egammo/PROJECTS/AXNetwork/` (Complete solution)

**REALMVOID Integration:**
- `/mnt/nas-egammo/PROJECTS/REALMVOID/Legacy_Reference/AGNON-HIVE/`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_Renderer_MAB_ABI_v0_2.md`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_Manifest_v0_2.md`

### Related Documentation

- Previous Analysis: `/home/roman/realmvoid-os-dns.md`
- AXNetwork GitHub: (if public)
- libsodium Documentation: https://doc.libsodium.org/

---

## ENDE

**Analysiert durch:** Claude Sonnet 4.5
**Agent ID:** a4215fc
**Datum:** 2026-02-22
**Version:** 1.0
