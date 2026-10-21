# HASKI_DOC

**Hybrid Architecture / Semantic Knowledge Interchange — Documentation**

This repository hosts proposed specifications and research documentation for Semantic Knowledge Interchange (SKI), supporting Semantic OS and distributed Semantic Memory interoperability.

## SKI specifications

| Version | Document | Status |
| --- | --- | --- |
| v0.1 | [Semantic Knowledge Interchange Specification (日本語)](docs/SKI_SPECIFICATION_v0.1_ja.md) | Independent Editor's Draft — 2026-10-18 |
| v0.2 | [Core Data Model, Lifecycle & Distributed Synchronization (日本語)](docs/SKI_v0.2_Core_Data_Model_Lifecycle_Distributed_Synchronization_ja.md) | Independent Editor's Draft — 2026-10-19 |
| v0.3 | [Natural-Language-Centric Semantic Exchange (日本語)](docs/SKI_v0.3_Natural_Language_Centric_Semantic_Exchange_ja.md) | Independent Editor's Draft — 2026-10-19 |
| v0.4 | [Distributed Semantic Identity & Replica Registry (日本語)](docs/SKI_v0.4_Distributed_Semantic_Identity_Replica_Registry_ja.md) | Proposal Draft — 2026-10-19 |
| v0.5 | [Natural-Language Dataset Roundtrip & Correlation (日本語)](docs/SKI_v0.5_Semantic_Dataset_Roundtrip_Interchange_ja.md) | Independent Editor's Draft — 2026-10-21 |

### Scope
- **SKI v0.1:** common knowledge assertions, context, provenance, epistemic assessment, revision, retraction, portability, and exchange profiles.
- **SKI v0.2:** SKI-Core/SDF, assertion lifecycle, independent Semantic Memory per VM, differential and idempotent synchronization, internalization tracking for `/sleep`, and TCP/TLS with optional RDMA transport.
- **SKI v0.3:** Exchange meaning via natural language and structured claims; each recipient generates local vectors as needed. Preserves v0.2 synchronization and lifecycle.
- **SKI v0.4 (proposal):** Bidirectional replica identification and origin lineage across multiple recipient Semantic Memories with per-VM local IDs, acknowledgements, and versioned replica registries.
- **SKI v0.5 (proposal):** SEMOS_S2N-derived Dataset A/B same-message-format roundtrip, receipt ACK separate from semantic replies, LLM identifiers, correlation, hashes, receiver-local re-semantification, and independent quality evaluation.

These specifications are **research proposals**, not standards approved by ISO, W3C, JIS or another standards body. Reference implementations, canonical JSON-LD contexts, SHACL validation and conformance suites remain future work.

## Related repository

- [SEMOS_SIM](https://github.com/digiponta/SEMOS_SIM) — Semantic OS simulator and source of the synchronized SKI Markdown drafts (branch `v0.7.0`).
- [SEMOS_S2N](https://github.com/digiponta/SEMOS_S2N) — local SKI natural-language A/B dataset exchange and roundtrip protocol prototype.

The companion Japanese DOCX manuscripts are not included in this repository yet. The Markdown editions here are the available public technical drafts.
