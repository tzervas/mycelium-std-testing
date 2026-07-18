# CROSS-REF — mycelium-std-testing

Mycelium-internal dependencies only (steer handoff §6.1; external crates stay in Cargo
metadata). Pinned revs are the fixed (buildable) tips recorded by the Phase-B wave;
content hash = git tree hash of the pinned rev.

| Interface consumed | Repo | Pinned rev | Content hash | Notes |
|---|---|---|---|---|
| mycelium-core | https://github.com/tzervas/mycelium-core | `781d3fcceba82acfe6b0eb46650513bd78a2416b` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-core` (see monorepo `docs/api-index/INDEX.md#mycelium-core`) |
| mycelium-diag | https://github.com/tzervas/mycelium-runtime | `ab9cee665b620ed80ab74ea61ea639817dc49077` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-diag` (see monorepo `docs/api-index/INDEX.md#mycelium-diag`) |
| mycelium-proj | https://github.com/tzervas/mycelium-proj | `cfc934b32c0b7b57becbb7ddfe3c516712c6ec0e` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-proj` (see monorepo `docs/api-index/INDEX.md#mycelium-proj`) |
| mycelium-std-core | https://github.com/tzervas/mycelium-std-core | `580b64316774e22f0b7d5d495ca1d9b9d6536a60` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-std-core` (see monorepo `docs/api-index/INDEX.md#mycelium-std-core`) |

**Owning docs:** `docs/spec/stdlib/testing.md` (slice in this repo) · RFC-0016.
**Source provenance:** extracted from `tzervas/mycelium` archive `aad96b7a…`; fixed by
the course-correction Phase B (workspace root, git pins, toolchain + supply-chain
replicas, CI v2). Full program record: monorepo
`docs/planning/course-correction-2026-07-18/PROGRAM.md`.
