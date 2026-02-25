# AXWire Protocol Specification v1.0

**Status:** Draft Standard
**Date:** 2026-02-22
**Authors:** AXWire Working Group
**Abstract:** This document specifies AXWire v1.0, a transport-agnostic binary messaging protocol designed for real-time communication in distributed systems, with support for selective encryption, compression, and flexible delivery guarantees.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Protocol Overview](#2-protocol-overview)
3. [Terminology](#3-terminology)
4. [Wire Format](#4-wire-format)
5. [Header Fields](#5-header-fields)
6. [Flags Specification](#6-flags-specification)
7. [Message Types](#7-message-types)
8. [Encryption](#8-encryption)
9. [Compression](#9-compression)
10. [Handshake Procedure](#10-handshake-procedure)
11. [Delivery Methods](#11-delivery-methods)
12. [Policy Negotiation](#12-policy-negotiation)
13. [Extensibility](#13-extensibility)
14. [Security Considerations](#14-security-considerations)
15. [Implementation Requirements](#15-implementation-requirements)
16. [IANA Considerations](#16-iana-considerations)

---

## 1. Introduction

### 1.1 Purpose

AXWire is a generic binary messaging protocol designed to provide:
- Transport-agnostic message delivery (UDP, TCP, WebSocket, QUIC)
- Message-level selective encryption and compression
- Application-defined message categories and types
- Flexible delivery guarantees (unreliable, reliable, ordered, sequenced)
- Cross-platform compatibility via binary wire format

### 1.2 Design Principles

The protocol adheres to the following principles:

1. **Separation of Concerns**: Transport layer is independent of application logic
2. **Selective Security**: Encryption and compression applied per-message, not per-connection
3. **Genericity**: No application-specific semantics in the core protocol
4. **Extensibility**: Versioned headers and reserved flag bits for future features
5. **Privacy by Design**: Identity fields are optional and consent-based

### 1.3 Scope

This specification defines:
- Binary wire format for AXWire messages
- System message types (handshake, heartbeat, disconnect, etc.)
- Encryption and compression mechanisms
- Policy negotiation procedures

This specification does NOT define:
- Application-specific message categories (these are application-defined)
- Transport layer implementation details (UDP reliability, TCP framing, etc.)
- Serialization formats for application payloads (MessagePack, Protobuf, JSON, etc.)

---

## 2. Protocol Overview

### 2.1 Architecture

AXWire uses a layered architecture:

```
┌─────────────────────────────────────┐
│   Application Layer                 │  ← App-defined categories/types
├─────────────────────────────────────┤
│   AXWire Protocol Layer             │  ← This specification
│   - Message structure               │
│   - Encryption/Compression          │
│   - Policy negotiation              │
├─────────────────────────────────────┤
│   Transport Layer                   │  ← UDP, TCP, WebSocket, QUIC
│   - Packet delivery                 │
│   - Connection management           │
└─────────────────────────────────────┘
```

### 2.2 Message Structure

Each AXWire message consists of two parts:

1. **Outer Header** (16 bytes, ALWAYS unencrypted)
   - Contains routing information (category, type, session ID)
   - Allows network-layer inspection and routing without decryption

2. **Inner Content** (variable length, CONDITIONALLY encrypted)
   - Optional identity layer (64 bytes)
   - Application payload (0-65535 bytes)
   - May be compressed and/or encrypted based on flags

### 2.3 Rationale for Outer/Inner Design

The outer/inner header design provides:

- **Efficient Routing**: Network proxies can route messages based on category/type without decryption
- **Selective Security**: Only sensitive messages incur encryption overhead
- **Session Tracking**: Session IDs remain visible for connection state management
- **Protocol Visibility**: Network monitoring tools can inspect protocol structure

---

## 3. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### 3.1 Definitions

- **Message**: A single AXWire protocol data unit
- **Outer Header**: The unencrypted 16-byte header present in every message
- **Inner Content**: The optionally encrypted/compressed payload section
- **Category**: A 16-bit application-defined message grouping (e.g., 0x1000 for gaming)
- **MessageType**: A 16-bit application-defined message subtype within a category
- **Session**: A logical connection between two endpoints, identified by SessionID
- **Endpoint**: A sender or receiver of AXWire messages
- **Policy**: A set of rules governing encryption, compression, and delivery

---

## 4. Wire Format

### 4.1 Overall Structure

```
┌────────────────────────────────────────────────────────┐
│                   OUTER HEADER (16 bytes)              │
│                   ALWAYS UNENCRYPTED                   │
├────────────────────────────────────────────────────────┤
│                   INNER CONTENT (variable)             │
│                   CONDITIONALLY ENCRYPTED              │
│   ┌────────────────────────────────────────────────┐   │
│   │  Identity Layer (64 bytes, optional)           │   │
│   ├────────────────────────────────────────────────┤   │
│   │  Payload (0-65535 bytes)                       │   │
│   └────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

### 4.2 Byte Order

All multi-byte integer fields MUST be encoded in **little-endian** byte order.

**Example**: The 16-bit value `0x1234` MUST be encoded as bytes `[0x34, 0x12]`.

### 4.3 Outer Header Layout

The outer header is exactly 16 bytes and has the following structure:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        ProtocolVersion        |           Category            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         MessageType           |             Flags             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          SessionID                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       SequenceNumber                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 4.4 Inner Content Layout

#### 4.4.1 Without Identity Layer

```
┌────────────────────────────────────┐
│  Payload (0-65535 bytes)           │
└────────────────────────────────────┘
```

#### 4.4.2 With Identity Layer (HasIdentity flag set)

```
┌────────────────────────────────────┐
│  SenderIdentityHash (32 bytes)     │  ← SHA256 hash (hex string)
├────────────────────────────────────┤
│  SenderHardwareHash (32 bytes)     │  ← SHA256 hash (hex string)
├────────────────────────────────────┤
│  Payload (0-65535 bytes)           │
└────────────────────────────────────┘
```

### 4.5 Encryption/Compression Processing

When both Encrypted and Compressed flags are set, processing order MUST be:

**Sending:**
1. Compress payload (if Compressed flag set)
2. Construct inner content (identity + compressed payload)
3. Encrypt inner content (if Encrypted flag set)
4. Construct outer header
5. Transmit message

**Receiving:**
1. Receive message
2. Parse outer header
3. Decrypt inner content (if Encrypted flag set)
4. Decompress payload (if Compressed flag set)
5. Extract identity (if HasIdentity flag set)
6. Deliver payload to application

---

## 5. Header Fields

### 5.1 ProtocolVersion (16 bits)

**Purpose**: Indicates the AXWire protocol version.

**Format**: `0xMMNN` where `MM` is major version, `NN` is minor version.

**Values**:
- `0x0100` (256 decimal): AXWire v1.0 (this specification)

**Processing**:
- Receivers MUST check this field upon receiving a message
- If major version differs from implementation, receiver SHOULD reject message
- If minor version differs, receiver MAY attempt compatibility processing

### 5.2 Category (16 bits)

**Purpose**: Application-defined message grouping.

**Format**: Unsigned 16-bit integer.

**Reserved Values**:
- `0x0000`: System category (reserved by AXWire specification)
- `0x0001-0x0FFF`: Reserved for future AXWire extensions
- `0x1000-0xFFFF`: Application-defined categories

**Examples**:
- `0x1000`: Gaming messages
- `0x2000`: Chat messages
- `0x3000`: File transfer messages
- `0x4000`: Administration messages

### 5.3 MessageType (16 bits)

**Purpose**: Application-defined message subtype within a category.

**Format**: Unsigned 16-bit integer.

**Semantics**: Defined by application for each category.

**System Category Types** (Category `0x0000`):
- `0x0001`: MSG_HANDSHAKE
- `0x0002`: MSG_HEARTBEAT
- `0x0003`: MSG_DISCONNECT
- `0x0004`: MSG_ACK
- `0x0005`: MSG_ERROR
- `0x0006`: MSG_KEY_EXCHANGE
- `0x0007-0xFFFF`: Reserved for future system messages

### 5.4 Flags (16 bits)

**Purpose**: Bitfield indicating message properties and processing instructions.

**Format**: Bitfield (see Section 6 for detailed flag specifications).

### 5.5 SessionID (32 bits)

**Purpose**: Identifies the session/connection this message belongs to.

**Format**: Unsigned 32-bit integer.

**Generation**:
- Server SHOULD generate unique session IDs during handshake
- Session ID `0x00000000` is reserved for connectionless messages

**Lifetime**: Session ID SHOULD remain constant for the duration of a session.

### 5.6 SequenceNumber (32 bits)

**Purpose**: Message ordering and duplicate detection.

**Format**: Unsigned 32-bit integer.

**Processing**:
- Sender SHOULD increment sequence number for each message sent on a session
- Receiver MAY use sequence number for ordering when Sequenced flag is set
- Receiver MAY use sequence number for duplicate detection when Reliable flag is set

---

## 6. Flags Specification

### 6.1 Flag Bits

The Flags field is a 16-bit bitfield with the following definition:

```
Bit Position | Hex Mask | Name         | Description
─────────────┼──────────┼──────────────┼───────────────────────────
     0       | 0x0001   | Reliable     | Guaranteed delivery required
     1       | 0x0002   | Encrypted    | Inner content is encrypted
     2       | 0x0004   | Compressed   | Payload is compressed
     3       | 0x0008   | HasIdentity  | Identity layer present
     4       | 0x0010   | Sequenced    | Ordered delivery required
     5       | 0x0020   | Broadcast    | Message for all clients
     6       | 0x0040   | Ack          | Acknowledgment message
    7-15     | 0xFF80   | Reserved     | Reserved for future use
```

### 6.2 Flag Semantics

#### 6.2.1 Reliable (0x0001)

**Purpose**: Indicates that guaranteed delivery is required.

**Sender Behavior**:
- Message SHOULD be retransmitted if not acknowledged
- Transport layer SHOULD implement reliability mechanism

**Receiver Behavior**:
- Receiver SHOULD send acknowledgment
- Duplicate messages SHOULD be detected via SequenceNumber

#### 6.2.2 Encrypted (0x0002)

**Purpose**: Indicates that inner content is encrypted.

**Sender Behavior**:
- Sender MUST encrypt inner content before transmission
- Encryption algorithm MUST be agreed upon during handshake (see Section 8)

**Receiver Behavior**:
- Receiver MUST decrypt inner content before processing
- If decryption fails, message SHOULD be discarded

#### 6.2.3 Compressed (0x0004)

**Purpose**: Indicates that payload is compressed.

**Sender Behavior**:
- Sender MUST compress payload before encryption
- Compression algorithm SHOULD be GZIP (RFC 1952) unless negotiated otherwise

**Receiver Behavior**:
- Receiver MUST decompress payload after decryption
- If decompression fails, message SHOULD be discarded

#### 6.2.4 HasIdentity (0x0008)

**Purpose**: Indicates that identity layer is present in inner content.

**Sender Behavior**:
- Sender MUST include 64-byte identity layer before payload
- Identity SHOULD only be sent with user consent (privacy consideration)

**Receiver Behavior**:
- Receiver MUST parse identity layer (64 bytes) before payload
- Identity hashes MAY be used for authentication/authorization

#### 6.2.5 Sequenced (0x0010)

**Purpose**: Indicates that ordered delivery is required.

**Sender Behavior**:
- Sender MUST increment SequenceNumber monotonically

**Receiver Behavior**:
- Receiver MUST deliver messages in SequenceNumber order
- Out-of-order messages SHOULD be buffered until earlier messages arrive

#### 6.2.6 Broadcast (0x0020)

**Purpose**: Indicates message should be delivered to all clients.

**Semantics**: Application-defined broadcast scope.

#### 6.2.7 Ack (0x0040)

**Purpose**: Indicates this is an acknowledgment message.

**Usage**: Typically used with MSG_ACK system message type.

### 6.3 Flag Combinations

Certain flag combinations have specific semantics:

- **Reliable + Sequenced**: Guaranteed ordered delivery (like TCP)
- **Encrypted + Compressed**: Compress-then-encrypt (MUST process in this order)
- **Reliable + Ack**: Acknowledgment for reliable message

---

## 7. Message Types

### 7.1 System Messages (Category 0x0000)

The following message types are reserved for system use:

#### 7.1.1 MSG_HANDSHAKE (0x0001)

**Purpose**: Initial connection negotiation.

**Direction**: Client → Server

**Payload Structure**:
```
struct HandshakeRequest {
    uint8   encryption_policy;     // 0=None, 1=Optional, 2=Preferred, 3=Required
    uint8   compression_policy;    // 0=None, 1=Automatic, 2=Always, 3=Manual
    uint16  reserved;
    byte[]  client_public_key;     // Optional, 32 bytes if encryption != None
}
```

**Expected Response**: MSG_ACK with server parameters

#### 7.1.2 MSG_HEARTBEAT (0x0002)

**Purpose**: Keep-alive signal for idle connections.

**Direction**: Bidirectional

**Payload**: Empty (0 bytes)

**Recommended Interval**: 30 seconds

**Timeout Behavior**: Disconnect after 3 missed heartbeats (90 seconds)

#### 7.1.3 MSG_DISCONNECT (0x0003)

**Purpose**: Graceful disconnection notification.

**Direction**: Bidirectional

**Payload Structure**:
```
struct DisconnectRequest {
    uint8   reason;                // 0x00=User, 0x01=Shutdown, 0x02=Timeout, 0x03=Kicked
    byte[]  reason_message;        // UTF-8 string (optional)
}
```

#### 7.1.4 MSG_ACK (0x0004)

**Purpose**: Acknowledgment of received message.

**Direction**: Response to any message with Reliable flag

**Payload Structure**:
```
struct AckMessage {
    uint32  acked_sequence_number; // SequenceNumber of acknowledged message
    uint8   status;                // 0x00=Success, 0x01=Error
}
```

#### 7.1.5 MSG_ERROR (0x0005)

**Purpose**: Error condition reporting.

**Direction**: Bidirectional

**Payload Structure**:
```
struct ErrorMessage {
    uint16  error_code;            // Application-defined
    byte[]  error_message;         // UTF-8 string describing error
}
```

**Common Error Codes**:
- `0x0001`: Encryption policy mismatch
- `0x0002`: Invalid message format
- `0x0003`: Authentication failed
- `0x0004`: Session expired
- `0x0005`: Unsupported protocol version

#### 7.1.6 MSG_KEY_EXCHANGE (0x0006)

**Purpose**: Encryption key exchange.

**Direction**: Server → Client (response to handshake)

**Payload Structure**:
```
struct KeyExchangeMessage {
    byte[]  server_public_key;     // 32 bytes (Curve25519)
    uint8   encryption_enabled;    // 0x00=No, 0x01=Yes
}
```

### 7.2 Application-Defined Messages

Applications MUST define their own categories and message types starting from category `0x1000`.

**Example**:
```
Category 0x1000 - Gaming
  ├─ 0x0001: PLAYER_SPAWN
  ├─ 0x0002: PLAYER_MOVE
  ├─ 0x0003: PLAYER_ATTACK
  └─ ...

Category 0x2000 - Chat
  ├─ 0x0001: PUBLIC_MESSAGE
  ├─ 0x0002: PRIVATE_MESSAGE
  └─ ...
```

---

## 8. Encryption

### 8.1 Encryption Algorithm

Implementations SHOULD support **XChaCha20-Poly1305** (RFC 8439) for authenticated encryption.

**Parameters**:
- Key size: 256 bits (32 bytes)
- Nonce size: 192 bits (24 bytes)
- Authentication tag size: 128 bits (16 bytes)

### 8.2 Key Exchange

#### 8.2.1 Curve25519 ECDH

Implementations SHOULD use **Curve25519** (RFC 7748) for key exchange.

**Procedure**:

1. **Client generates ephemeral keypair**:
   ```
   (client_private_key, client_public_key) = Curve25519_GenerateKeypair()
   ```

2. **Client sends public key in MSG_HANDSHAKE**:
   ```
   Send: MSG_HANDSHAKE with client_public_key (32 bytes)
   ```

3. **Server generates ephemeral keypair**:
   ```
   (server_private_key, server_public_key) = Curve25519_GenerateKeypair()
   ```

4. **Server computes shared secret**:
   ```
   shared_secret = Curve25519_ScalarMult(server_private_key, client_public_key)
   ```

5. **Server sends public key in MSG_KEY_EXCHANGE**:
   ```
   Send: MSG_KEY_EXCHANGE with server_public_key (32 bytes)
   ```

6. **Client computes shared secret**:
   ```
   shared_secret = Curve25519_ScalarMult(client_private_key, server_public_key)
   ```

**Result**: Both endpoints have identical `shared_secret` (32 bytes).

### 8.3 Authenticated Encryption

#### 8.3.1 Encryption Process

For each message with Encrypted flag:

1. **Generate random nonce**:
   ```
   nonce = RandomBytes(24)  // XChaCha20 nonce
   ```

2. **Encrypt inner content**:
   ```
   ciphertext = XChaCha20Poly1305_Encrypt(
       plaintext = inner_content,
       nonce = nonce,
       key = shared_secret,
       additional_data = outer_header  // AEAD: outer header as associated data
   )
   ```

3. **Prepend nonce to ciphertext**:
   ```
   encrypted_inner = nonce || ciphertext
   ```

#### 8.3.2 Decryption Process

For each message with Encrypted flag:

1. **Extract nonce**:
   ```
   nonce = encrypted_inner[0:24]
   ciphertext = encrypted_inner[24:]
   ```

2. **Decrypt**:
   ```
   plaintext = XChaCha20Poly1305_Decrypt(
       ciphertext = ciphertext,
       nonce = nonce,
       key = shared_secret,
       additional_data = outer_header  // AEAD: must match encryption
   )
   ```

3. **Verify authentication tag**:
   - If verification fails, message MUST be discarded
   - Error SHOULD be logged for security monitoring

### 8.4 Security Properties

- **Confidentiality**: Inner content hidden from eavesdroppers
- **Integrity**: Authentication tag prevents tampering
- **Authenticity**: AEAD with outer header prevents header manipulation
- **Perfect Forward Secrecy**: Ephemeral keys, no key persistence

---

## 9. Compression

### 9.1 Compression Algorithm

Implementations SHOULD support **GZIP** (RFC 1952) compression.

**Recommended Parameters**:
- Compression level: 6 (balance between speed and ratio)
- Window size: 15 (default)

### 9.2 Compression Processing

#### 9.2.1 Compression Order

Compression MUST be applied BEFORE encryption:

```
Payload → Compress → Encrypt → Transmit
```

**Rationale**: Encrypted data has high entropy and is not compressible.

#### 9.2.2 Threshold-Based Compression

Implementations SHOULD NOT compress payloads smaller than a threshold (recommended: 1024 bytes).

**Reason**: GZIP header overhead (~18 bytes) makes small payloads larger.

#### 9.2.3 Compression Procedure

```
IF payload_size >= compression_threshold:
    compressed_payload = GZIP_Compress(payload, level=6)

    IF compressed_size < original_size:
        payload = compressed_payload
        flags |= Compressed
    ELSE:
        // Compression ineffective, send uncompressed
        flags &= ~Compressed
ELSE:
    // Too small, skip compression
    flags &= ~Compressed
```

### 9.3 Decompression Procedure

```
IF flags & Compressed:
    payload = GZIP_Decompress(payload)

    IF decompression_failed:
        Discard message
        Log error
```

---

## 10. Handshake Procedure

### 10.1 Handshake Flow

```
Client                                  Server
  |                                       |
  |  MSG_HANDSHAKE                        |
  |  (encryption_policy,                  |
  |   compression_policy,                 |
  |   client_public_key)                  |
  |-------------------------------------->|
  |                                       |
  |                                       | [Validate policies]
  |                                       | [Generate keypair]
  |                                       | [Compute shared_secret]
  |                                       | [Assign SessionID]
  |                                       |
  |                      MSG_KEY_EXCHANGE |
  |              (server_public_key,      |
  |               encryption_enabled)     |
  |<--------------------------------------|
  |                                       |
  | [Compute shared_secret]               |
  |                                       |
  |  MSG_ACK                              |
  |  (SessionID confirmed)                |
  |-------------------------------------->|
  |                                       |
  |        [Connection Established]       |
  |                                       |
```

### 10.2 Policy Negotiation

#### 10.2.1 Encryption Policy Matrix

```
Client Policy | Server Policy | Result
──────────────┼───────────────┼─────────────────────
None          | Any           | No Encryption
Optional      | None          | No Encryption
Optional      | Optional      | Encryption (if supported)
Optional      | Preferred     | Encryption
Optional      | Required      | Encryption
Preferred     | None          | No Encryption
Preferred     | Optional      | Encryption
Preferred     | Preferred     | Encryption
Preferred     | Required      | Encryption
Required      | None          | Connection FAILS
Required      | Optional      | Encryption
Required      | Preferred     | Encryption
Required      | Required      | Encryption
```

#### 10.2.2 Policy Validation

Server MUST validate client policies:

```
IF client_encryption_policy == Required AND server_encryption_policy == None:
    Send MSG_ERROR (code: 0x0001, "Encryption required but not supported")
    Close connection

IF policies_compatible:
    encryption_enabled = Determine_Encryption_Result(client_policy, server_policy)
    Send MSG_KEY_EXCHANGE with encryption_enabled
ELSE:
    Send MSG_ERROR
    Close connection
```

---

## 11. Delivery Methods

### 11.1 Unreliable Delivery

**Flags**: None (0x0000)

**Semantics**:
- No delivery guarantee ("fire-and-forget")
- No retransmission on loss
- No ordering guarantee

**Use Cases**: Position updates, audio packets, video frames

### 11.2 Reliable Delivery

**Flags**: Reliable (0x0001)

**Semantics**:
- Guaranteed delivery
- Sender retransmits if no ACK received within timeout
- Receiver sends MSG_ACK upon receipt

**Procedure**:

**Sender**:
```
Send message with Reliable flag
Start timeout timer (default: 5 seconds)

IF timeout AND retry_count < max_retries:
    Retransmit message
    retry_count++
    Restart timer
ELSE IF timeout AND retry_count >= max_retries:
    Mark message as failed
    Notify application
```

**Receiver**:
```
Receive message with Reliable flag
Process message
Send MSG_ACK with acked_sequence_number
```

### 11.3 Ordered Delivery

**Flags**: Reliable | Sequenced (0x0011)

**Semantics**:
- Guaranteed delivery
- Messages delivered in SequenceNumber order
- Out-of-order messages buffered until gap filled

**Receiver Procedure**:
```
expected_sequence = current_sequence + 1

IF message.SequenceNumber == expected_sequence:
    Deliver message
    current_sequence = message.SequenceNumber

    // Check buffer for next expected message
    WHILE buffer.contains(current_sequence + 1):
        Deliver buffer[current_sequence + 1]
        current_sequence++

ELSE IF message.SequenceNumber > expected_sequence:
    // Out of order, buffer
    buffer[message.SequenceNumber] = message

ELSE:
    // Duplicate or old message, discard
    Log warning
```

### 11.4 Broadcast Delivery

**Flags**: Broadcast (0x0020)

**Semantics**: Application-defined broadcast scope

**Typical Implementations**:
- Server sends message to all connected clients
- Server sends message to all clients in a "room" or "zone"

---

## 12. Policy Negotiation

### 12.1 Encryption Policy

```
enum EncryptionPolicy:
    None      = 0  // No encryption support
    Optional  = 1  // Encryption supported but not required
    Preferred = 2  // Encryption preferred, try encrypted first
    Required  = 3  // Encryption mandatory, fail if unavailable
```

### 12.2 Compression Policy

```
enum CompressionPolicy:
    None      = 0  // No compression
    Automatic = 1  // Compress if size >= threshold
    Always    = 2  // Always compress
    Manual    = 3  // Application decides via Compressed flag
```

### 12.3 Negotiation Rules

1. **Encryption**:
   - If either endpoint has policy `Required`, encryption MUST be used
   - If both have policy `None`, encryption MUST NOT be used
   - Otherwise, encryption SHOULD be used if both support it

2. **Compression**:
   - Policy is not negotiated; each endpoint compresses independently
   - Receiver MUST support decompression regardless of local policy

---

## 13. Extensibility

### 13.1 Protocol Versioning

**Major Version Changes**:
- Incompatible wire format changes
- Receivers MUST reject messages with different major version

**Minor Version Changes**:
- Backward-compatible additions (new flags, new message types)
- Receivers SHOULD accept newer minor versions

### 13.2 Reserved Flag Bits

Bits 7-15 (mask `0xFF80`) are reserved for future use.

**Implementation Requirements**:
- Senders MUST set reserved bits to 0
- Receivers MUST ignore reserved bits

### 13.3 Adding New System Messages

Future versions MAY add new system message types in category `0x0000`.

**Requirements**:
- New types MUST use previously unused MessageType values
- Backward compatibility SHOULD be maintained when possible

### 13.4 Custom Categories

Applications are free to define categories `0x1000-0xFFFF`.

**Best Practices**:
- Document category assignments
- Use ranges for related functionality (e.g., 0x1000-0x1FFF for core gameplay)
- Reserve high-value categories (0xF000-0xFFFF) for debugging/testing

---

## 14. Security Considerations

### 14.1 Encryption

**Threats Mitigated**:
- Eavesdropping: XChaCha20-Poly1305 provides confidentiality
- Tampering: AEAD authentication tag prevents modification
- Replay attacks: Random nonces prevent replay

**Limitations**:
- No endpoint authentication (identity verification is application responsibility)
- No protection against traffic analysis (message sizes visible)

### 14.2 Denial of Service

**Considerations**:
- Large payloads (up to 65535 bytes) can be used for amplification attacks
- Implementations SHOULD implement rate limiting
- Implementations SHOULD validate payload sizes before processing

**Recommendations**:
- Limit maximum message size based on application needs
- Implement connection-level rate limits
- Use transport-layer flood protection (e.g., UDP flood detection)

### 14.3 Privacy

**Identity Layer**:
- SenderIdentityHash and SenderHardwareHash are optional
- Implementations MUST NOT send identity without user consent
- Hashes SHOULD be used instead of plaintext identifiers

**Outer Header Visibility**:
- Category, MessageType, SessionID are always visible
- Applications SHOULD NOT encode sensitive information in these fields

### 14.4 Key Management

**Ephemeral Keys**:
- Private keys SHOULD NOT be persisted
- New keypair SHOULD be generated for each session

**Shared Secret Lifetime**:
- Shared secret SHOULD be cleared from memory after session ends
- Implementations MAY implement periodic rekeying for long-lived sessions

---

## 15. Implementation Requirements

### 15.1 Mandatory Features

Implementations MUST support:
- Outer header parsing (16-byte fixed format)
- Little-endian byte order
- System message types (MSG_HANDSHAKE, MSG_HEARTBEAT, MSG_DISCONNECT)
- Unreliable delivery
- Flags: Reliable, Encrypted, Compressed, HasIdentity

### 15.2 Recommended Features

Implementations SHOULD support:
- XChaCha20-Poly1305 encryption
- Curve25519 key exchange
- GZIP compression
- Ordered delivery (Sequenced flag)
- Policy negotiation

### 15.3 Optional Features

Implementations MAY support:
- Additional encryption algorithms (with negotiation)
- Additional compression algorithms (with negotiation)
- Broadcast delivery
- Custom transport layers (WebSocket, QUIC, etc.)

### 15.4 Interoperability Testing

Implementations SHOULD verify:
- Correct byte order (little-endian)
- Correct encryption (cross-implementation message exchange)
- Correct compression (compressed payload readable by other implementations)
- Policy negotiation (all policy combinations)

**Test Vectors**:

```
Test 1: Unencrypted, Uncompressed Message
  Outer Header:
    ProtocolVersion: 0x0100
    Category: 0x1000
    MessageType: 0x0042
    Flags: 0x0000
    SessionID: 0x00001234
    SequenceNumber: 0x00000001
  Payload: "Hello World" (UTF-8)

  Expected Bytes (hex):
  00 01 00 10 42 00 00 00 34 12 00 00 01 00 00 00
  48 65 6C 6C 6F 20 57 6F 72 6C 64

Test 2: Encrypted Message (simplified, no actual encryption)
  Outer Header:
    ProtocolVersion: 0x0100
    Category: 0x0000
    MessageType: 0x0001
    Flags: 0x0002  (Encrypted)
    SessionID: 0x00005678
    SequenceNumber: 0x00000002
  Inner Content: [24-byte nonce][ciphertext]

  (Full test vectors provided in separate test suite document)
```

---

## 16. IANA Considerations

### 16.1 Category Registry

This specification reserves category values `0x0000-0x0FFF` for system use.

**Requested Actions**:
- Establish AXWire Category Registry
- Reserve 0x0000 for system messages (this specification)
- Reserve 0x0001-0x0FFF for future IANA assignments

### 16.2 System Message Type Registry

This specification defines message types `0x0001-0x0006` in category `0x0000`.

**Requested Actions**:
- Establish AXWire System Message Type Registry
- Register types 0x0001-0x0006 as defined in Section 7.1
- Reserve 0x0007-0xFFFF for future assignments

### 16.3 Error Code Registry

This specification defines error codes `0x0001-0x0005`.

**Requested Actions**:
- Establish AXWire Error Code Registry
- Register codes 0x0001-0x0005 as defined in Section 7.1.5

---

## Appendix A: Design Rationale

### A.1 Why Outer/Inner Header?

Traditional fully-encrypted protocols face challenges:
- Cannot route without decryption (all metadata encrypted)
- Must decrypt every message even for routing decisions
- Session tracking requires decryption

The outer/inner design provides:
- **Efficient routing**: Proxies can route by category without decryption
- **Selective security**: Only sensitive messages incur encryption overhead
- **Visibility**: Network monitoring tools can inspect protocol structure
- **Session management**: SessionID visible for connection tracking

### A.2 Why Message-Level Encryption?

Connection-level encryption (TLS/DTLS) has limitations:
- All-or-nothing: Cannot selectively encrypt messages
- Overhead for all messages, even non-sensitive ones
- Complex setup (certificates, PKI)

Message-level encryption provides:
- **Flexibility**: Sensitive messages encrypted, others not
- **Performance**: Reduced CPU overhead for non-critical messages
- **Simplicity**: No certificate infrastructure required

### A.3 Why Compress-Then-Encrypt?

Encryption produces high-entropy output that is not compressible.

**Wrong Order** (Encrypt-Then-Compress):
```
Payload (10 KB) → Encrypt → Encrypted (10 KB) → Compress → Still 10 KB ❌
```

**Correct Order** (Compress-Then-Encrypt):
```
Payload (10 KB) → Compress → Compressed (2 KB) → Encrypt → Encrypted (2 KB) ✅
```

Result: 80% bandwidth savings with encryption.

---

## Appendix B: Implementation Pseudocode

### B.1 Message Serialization

```
function Serialize(message):
    buffer = ByteBuffer()

    // Outer Header (16 bytes, little-endian)
    buffer.WriteUInt16LE(message.ProtocolVersion)
    buffer.WriteUInt16LE(message.Category)
    buffer.WriteUInt16LE(message.MessageType)
    buffer.WriteUInt16LE(message.Flags)
    buffer.WriteUInt32LE(message.SessionID)
    buffer.WriteUInt32LE(message.SequenceNumber)

    // Inner Content
    inner_content = ByteBuffer()

    if message.Flags & HasIdentity:
        inner_content.WriteBytes(message.SenderIdentityHash)  // 32 bytes
        inner_content.WriteBytes(message.SenderHardwareHash)  // 32 bytes

    inner_content.WriteBytes(message.Payload)

    // Compression
    if message.Flags & Compressed:
        inner_content = GZIP_Compress(inner_content)

    // Encryption
    if message.Flags & Encrypted:
        nonce = RandomBytes(24)
        ciphertext = XChaCha20Poly1305_Encrypt(
            plaintext=inner_content,
            nonce=nonce,
            key=shared_secret,
            aad=buffer  // Outer header as additional data
        )
        inner_content = nonce || ciphertext

    buffer.WriteBytes(inner_content)
    return buffer.ToBytes()
```

### B.2 Message Deserialization

```
function Deserialize(bytes):
    buffer = ByteBuffer(bytes)
    message = AXMessage()

    // Outer Header (16 bytes, little-endian)
    message.ProtocolVersion = buffer.ReadUInt16LE()
    message.Category = buffer.ReadUInt16LE()
    message.MessageType = buffer.ReadUInt16LE()
    message.Flags = buffer.ReadUInt16LE()
    message.SessionID = buffer.ReadUInt32LE()
    message.SequenceNumber = buffer.ReadUInt32LE()

    // Validate protocol version
    if message.ProtocolVersion >> 8 != 0x01:  // Major version mismatch
        throw UnsupportedVersionError

    // Inner Content
    inner_content = buffer.ReadRemaining()

    // Decryption
    if message.Flags & Encrypted:
        nonce = inner_content[0:24]
        ciphertext = inner_content[24:]

        inner_content = XChaCha20Poly1305_Decrypt(
            ciphertext=ciphertext,
            nonce=nonce,
            key=shared_secret,
            aad=bytes[0:16]  // Outer header
        )

        if decryption_failed:
            throw DecryptionError

    // Decompression
    if message.Flags & Compressed:
        inner_content = GZIP_Decompress(inner_content)

        if decompression_failed:
            throw DecompressionError

    // Identity Layer
    if message.Flags & HasIdentity:
        message.SenderIdentityHash = inner_content[0:32]
        message.SenderHardwareHash = inner_content[32:64]
        message.Payload = inner_content[64:]
    else:
        message.Payload = inner_content

    return message
```

---

## Appendix C: References

### Normative References

- **[RFC2119]** Key words for use in RFCs to Indicate Requirement Levels
- **[RFC7748]** Elliptic Curves for Security (Curve25519)
- **[RFC8439]** ChaCha20 and Poly1305 for IETF Protocols
- **[RFC1952]** GZIP file format specification

### Informative References

- **[NACL]** Networking and Cryptography library
- **[LIBSODIUM]** Modern, easy-to-use software library for encryption

---

## Appendix D: Changelog

### Version 1.0 (2026-02-22)
- Initial specification
- Defined wire format, flags, system messages
- Specified encryption (XChaCha20-Poly1305) and compression (GZIP)
- Defined handshake and policy negotiation procedures

---

## Authors' Addresses

AXWire Working Group
Email: axwire-spec@example.com
Web: https://axwire.example.com

---

**END OF SPECIFICATION**
