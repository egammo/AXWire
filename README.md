# AXWire

> Transport-agnostic binary messaging protocol for real-time distributed systems.

AXWire is the communication layer of the [REALMVOID](https://github.com/egammo/realmvoid) ecosystem. It defines how game worlds, clients, and infrastructure services talk to each other — regardless of programming language or runtime.

**Status:** Draft Standard v1.0
**Language-agnostic** — implement in Go, C#, Rust, JavaScript, anything.

## Protocol at a Glance

```
[ ProtocolVersion | Category | MessageType | Flags | SessionID | SequenceNumber | Payload ]
```

16-byte fixed header + variable payload.

| Feature | Detail |
|---|---|
| Transport | UDP / TCP / WebSocket |
| Encryption | XChaCha20-Poly1305 |
| Key Exchange | Curve25519 |
| Compression | GZIP (optional, per-message) |
| Delivery | Reliable / Ordered / Unreliable (per-message flag) |

## Specification

Full spec: [`docs/AXWIRE_SPEC_v1_0.md`](docs/AXWIRE_SPEC_v1_0.md)

| Document | Content |
|---|---|
| [`docs/AXWIRE_SPEC_v1_0.md`](docs/AXWIRE_SPEC_v1_0.md) | Full protocol specification |
| [`docs/axnetwork-axwire-analysis.md`](docs/axnetwork-axwire-analysis.md) | Evolution from AXNetwork to AXWire |
| [`docs/AGNON-HIVE_Tree.md`](docs/AGNON-HIVE_Tree.md) | Legacy C# reference implementation tree |

## Ecosystem

| Repo | Role |
|---|---|
| [egammo/realmvoid](https://github.com/egammo/realmvoid) | Architecture papers (Codex) |
| **egammo/axwire** ← *you are here* | Protocol specification |
| [egammo/realmvoid-os](https://github.com/egammo/realmvoid-os) | Reference infrastructure (Go) |
| [egammo/agnon-city](https://github.com/egammo/agnon-city) | First game world |

## License

MIT — implement freely.

---

*Built by [EGAMMO.gaming](https://egammo.eu)*
