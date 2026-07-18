# CROSS-REF — mycelium-std-testing

Mycelium-internal dependencies only (steer handoff §6.1; external crates stay in Cargo
metadata). Pinned revs are the fixed (buildable) tips recorded by the Phase-B wave;
content hash = git tree hash of the pinned rev.

| Interface consumed | Repo | Pinned rev | Content hash | Notes |
|---|---|---|---|---|
| mycelium-core | https://github.com/tzervas/mycelium-core | `46d2515cbd86d2ae4d1365f4adcd2796737e9f0b` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-core` (see monorepo `docs/api-index/INDEX.md#mycelium-core`) |
| mycelium-diag | https://github.com/tzervas/mycelium-runtime | `487b1e7049ff521b1a6fa33f376245089e7dc1e1` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-diag` (see monorepo `docs/api-index/INDEX.md#mycelium-diag`) |
| mycelium-proj | https://github.com/tzervas/mycelium-proj | `20b8a6d264ac728e81cfe8cd90cec8d2a91370be` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-proj` (see monorepo `docs/api-index/INDEX.md#mycelium-proj`) |
| mycelium-std-core | https://github.com/tzervas/mycelium-std-core | `376762cc17853e1582684ececf9e760426bcfb0c` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-std-core` (see monorepo `docs/api-index/INDEX.md#mycelium-std-core`) |

**Owning docs:** `docs/spec/stdlib/testing.md` (slice in this repo) · RFC-0016.
**Source provenance:** extracted from `tzervas/mycelium` archive `aad96b7a…`; fixed by
the course-correction Phase B (workspace root, git pins, toolchain + supply-chain
replicas, CI v2). Full program record: monorepo
`docs/planning/course-correction-2026-07-18/PROGRAM.md`.
