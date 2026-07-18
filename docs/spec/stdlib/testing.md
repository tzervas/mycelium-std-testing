# Spec — `std.testing` (the repo's verification discipline as a library — a skipped check is reported, never a silent pass)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-testing` (M-534, #174, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.testing` · Ring 2 (RFC-0016 §4.2) · Tier B (RFC-0016 §4.4) |
| **Tracks** | `M-534` (#174) — the Phase-5 task this spec delivers |
| **Scope** | The repo's own verification discipline as a library: **property** tests (a bound for every guarantee), **golden / snapshot** tests, the **differential** harness (the M-151/M-210 interp↔AOT/native oracle pattern), and the explicit **skip / undetermined** reporter that makes a non-run check visible. |
| **Boundary** | Out of scope: the structured diagnostic *record* a failure carries — `std.diag` (M-510); `testing` *produces* failures as those records and adds the verdict/aggregation layer. The bound *kernels* a property checks against — `std.numerics` (M-512) / the capability crates; `testing` asserts a module's matrix, it does not define the bounds. Random input generation for properties shares the seeded-generator discipline of `std.rand` (M-531). The `just check` *runner/CI wiring* is the repo tooling, not this library. |
| **Depends on** | RFC-0016 §4.1 (the C1–C6 contract), §4.2 (ring layering), §4.4 (the `testing` row), §4.5 (the guarantee-matrix obligation every module discharges — what `testing` provides the harness *for*); the M-151 / M-210 interp↔AOT differential pattern (the oracle discipline; NFR-7); RFC-0013 (the diagnostic record a failure carries, via `diag`); RFC-0001 (the value model + guarantee lattice). |
| **Grounds on** | `std.diag` (M-510) — the structured failure record + trace; `std.core` (M-515) — `Option`/`Result`/error values; the `just check` local↔CI parity ethos (skips are explicit and explained); the differential-oracle precedent M-151/M-210 (Done). KC-3: above the kernel — adds no trusted code; the harness *checks* the trusted base, it does not enlarge it. |

---

## 1. Summary

`std.testing` is the Mycelium verification discipline expressed as a library — the language dogfooding its own honesty rule. It provides **property** tests (the "a bound for every guarantee" discipline: a guarantee tag is backed by a checked property, not prose), **golden / snapshot** tests, and the **differential** harness (run the same program through two backends — interpreter vs AOT/native — and require agreement, the M-151/M-210/NFR-7 oracle). Its **honesty crux** is C1/G2 turned on the test report itself: **a skipped or undetermined check is *reported*, never a silent pass** — a test that could not run (a missing oracle, an unmet precondition, an `ignore`) produces an explicit `Skipped{reason}` verdict that aggregates distinctly from `Pass`, so "green" can never silently include "did not actually check". Its second crux is C2/VR-5: a passing property establishes *only what it checked* — it backs an `Empirical` (or, with a checked theorem, supports a `Proven`) claim, and the harness never inflates a verdict. Ring 2, Tier B; it adds no trusted code (KC-3), consuming `std.diag` and `std.core`.

## 2. Scope & module boundary

- **In scope:**
  - **Property testing** — `for_all(gen, prop)`: a property over generated inputs, with shrinking to a minimal counterexample, a declared trial budget, and a *seed* so a failure is reproducible (the seeded-generator discipline, `std.rand`). A property that establishes "a bound for every guarantee" is the unit that backs a matrix row's `Empirical` tag.
  - **Golden / snapshot testing** — `golden(name, produce)`: compare a produced value against a content-addressed stored baseline; a mismatch is an explicit diff; a missing baseline is an explicit `Skipped`/`record-needed`, never an auto-accept.
  - **Differential / oracle testing** — `differential(input, lhs, rhs)`: run two implementations (the interp↔AOT/native pattern, M-151/M-210) and require observable agreement; a mismatch carries both outputs; an unavailable backend is an explicit `Skipped{reason}`, not a silent pass (NFR-7).
  - **The verdict + reporter** — the `Verdict` sum (`Pass`/`Fail{diagnostic}`/`Skipped{reason}`/`Undetermined{reason}`) and the aggregation that keeps `Skipped`/`Undetermined` counts *distinct* from `Pass` in the summary.
- **Out of scope (and who owns it):**
  - The structured diagnostic **record** (severity, source span, trace) a `Fail` carries — `std.diag` (M-510). `testing` constructs failures *as* `diag` records and owns only the verdict/aggregation layer above them.
  - The **bound kernels** a property checks (the ε/δ certificate, the RFC-0003 §4 matrix) — `std.numerics` (M-512) / the capability crates. `testing` *asserts* a module's guarantee matrix against its implementation; it does not define the guarantee.
  - **Random input generation** internals — the seeded generator is `std.rand` (M-531); `testing` consumes it (reproducible properties) and shares its no-silent-entropy discipline.
  - The **runner / CI wiring** (`just check`, the workflow) — repo tooling. This library is what such a runner *invokes*; it is not the runner.
- **Ring & layering:** Ring 2 (RFC-0016 §4.2). `testing` is **new library code written to the contract over Ring 0/1**: it composes `diag` records, `core` values, and the `rand` seeded generator. It is a **consumer** that verifies the trusted base; it enlarges no trusted base (KC-3) — indeed a test harness that could silently weaken what it checks would violate the honesty rule it exists to enforce.

## 3. Exported-op surface (design sketch)

A design sketch — enough to fix the surface and feed the guarantee matrix, not a committed grammar. Value-semantic: a test *run* is a pure function from `(case, seed)` to a `Verdict` (plus declared effects for golden IO); the harness never hides state.

```
// illustrative signatures (not a committed surface)

enum Verdict {
  Pass,
  Fail        { diagnostic: Diag },                 // a std.diag record: trace + reason + reproducing seed
  Skipped     { reason: SkipReason },               // REPORTED, aggregated distinctly from Pass (the crux)
  Undetermined{ reason: UndetReason },              // ran but could not decide (e.g. oracle unavailable)
}

// --- property: a bound for every guarantee; shrink to a minimal counterexample; reproducible by seed ---
fn for_all<T>(gen: Gen<T>, trials: Budget, prop: fn(T) -> Result<(), Diag>) -> Verdict
    // Fail carries the SHRUNK counterexample + the seed; a generator that cannot produce -> Skipped

// --- golden / snapshot: compare against a content-addressed baseline ---
fn golden<T>(name: GoldenId, produce: fn() -> T) -> Verdict ! io
    // mismatch -> Fail{diff}; MISSING baseline -> Skipped{NeedsRecord} (NEVER a silent auto-accept)

// --- differential / oracle: two backends must agree (interp <-> AOT/native; M-151/M-210; NFR-7) ---
fn differential<T: Eq>(input: Input, lhs: fn(Input)->T, rhs: fn(Input)->T) -> Verdict
    // disagreement -> Fail{both outputs}; a backend unavailable -> Skipped{BackendUnavailable}

// --- the honest aggregator: Skipped/Undetermined stay DISTINCT from Pass ---
fn summarize(vs: &[Verdict]) -> Summary   // { passed, failed, skipped, undetermined } — green != skipped
fn is_green(s: Summary) -> bool           // true ONLY if failed == 0 AND skipped/undetermined are surfaced

enum SkipReason  { Ignored, UnmetPrecondition, NeedsRecord, BackendUnavailable, ToolMissing }
enum UndetReason { OracleUnavailable, BudgetExhaustedInconclusive, NonDeterministicInput }
```

> **Note (the load-bearing design choice):** `Skipped` and `Undetermined` are *first-class verdict variants*, not a flavour of `Pass`. `is_green` cannot return a clean pass while hiding a skip — the summary always surfaces the skip/undetermined counts. This is the `just check` "skips are explicit and explained" ethos encoded in the type.

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops. To be encoded as a checked table and asserted in tests once code lands — and, fittingly, `testing` is the surface that does that asserting for every *other* module's matrix. **Tag legend:** the harness ops themselves carry no accuracy semantics — they are `Exact` *as mechanisms* (a verdict is an exact function of the run). What they must not do is inflate the *subject's* tag; that discipline is the justification below, not a tag on these rows.

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|
| `for_all` (property) | `Exact` (the verdict is exact; it *backs* an `Empirical` subject claim — never upgrades it) | `Fail{Diag}` / `Skipped{...}` as a verdict | none (pure; seeded) | yes (counterexample + seed) |
| `golden` (snapshot) | `Exact` (verdict) | `Fail{diff}` / `Skipped{NeedsRecord}` | `io` (read baseline) | yes (the diff + baseline hash) |
| `differential` (oracle) | `Exact` (verdict) | `Fail{lhs,rhs}` / `Skipped{BackendUnavailable}` | none / `io` per backend | yes (both outputs + input) |
| `summarize` | `Exact` (a total function over verdicts) | total | none | yes (the per-class counts) |
| `is_green` | `Exact` (green ⇔ no fail ∧ skips surfaced) | total | none | yes |

**Tag justification (VR-5 — downgrade rather than overclaim):**
- **The harness ops are `Exact` mechanisms** — a `Verdict` is an exact, deterministic function of the run (a seeded property is reproducible; a golden compare is exact; a differential is exact equality of observables). This `Exact` is about the *verdict mechanism*, **not** a claim about the subject under test.
- **The harness never inflates the subject's tag (the crux of C2).** A *passing* `for_all` establishes only that the property held over the *sampled* inputs at the given budget — it **backs an `Empirical` claim**, and reaches `Proven` *only* when paired with a cited theorem whose side-conditions are checked (the property witnesses the theorem's checkable consequence). The harness has no operation that turns "passed N trials" into `Proven`; that would be the exact VR-5 violation the module exists to prevent.
- **A non-decision is never a `Pass`.** `Skipped` (could not run) and `Undetermined` (ran, could not decide — e.g. the oracle was unavailable, or a budget was exhausted inconclusively) are distinct verdict classes; `summarize` keeps their counts separate and `is_green` surfaces them. "Green" therefore means *checked and passed*, never *did not check* (C1/G2).

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2).** The module's whole reason for being. A skipped or undetermined check is an explicit `Skipped{reason}` / `Undetermined{reason}` verdict that propagates into the summary distinct from `Pass`; a missing golden baseline is `Skipped{NeedsRecord}`, never a silent auto-accept; an unavailable differential backend is `Skipped{BackendUnavailable}`, never a silent pass; a property `Fail` carries its shrunk counterexample. No "green" is ever reported that silently omitted a check.
- **C2 — honest per-op tag (VR-5).** The harness asserts a module's guarantee tags and **never upgrades** them: a property pass backs `Empirical` (or supports `Proven` only alongside a checked theorem); there is no op that promotes trial-passing to `Proven`. The harness's own ops are `Exact` *as mechanisms*, separated from the subject's claim so neither inflates the other.
- **C3 — no black boxes / EXPLAIN (SC-3/G11).** Every non-`Pass` verdict is a reified, inspectable artifact: a `Fail` is a `std.diag` record with the trace, the reason, and the reproducing seed; a `golden` mismatch carries the diff and the baseline content hash; a `differential` mismatch carries both outputs and the input. A skip carries its *explained* reason. A failure can always be reproduced and explained — never an opaque red/green bit.
- **C4 — content-addressed, value-semantic (ADR-003 / RFC-0001).** Golden baselines are **content-addressed** (a baseline is identified by the hash of its value); a verdict is an immutable value; a seeded property run is a pure function of `(case, seed)`. The run metadata is `Meta`, not identity.
- **C5 — above the small kernel (KC-3).** `testing` adds no trusted code; it *checks* the trusted base by composing `diag` records, `core` values, and the `rand` seeded generator. A harness that could silently weaken what it checks would defeat its purpose — so the never-upgrade discipline (C2) is also its KC-3 posture.
- **C6 — declared, bounded effects (RFC-0014).** A property run declares its **trial budget** (bounded — a property cannot run unboundedly); a `golden` declares its baseline `io` read; a `differential` declares any per-backend `io`. Property generation uses the *seeded* (pure) `std.rand` surface, so a property is reproducible and pulls no undeclared entropy (RT3) — a non-deterministic input is itself reported as `Undetermined{NonDeterministicInput}`, never a flaky silent pass.

## 6. Grounding

- The "a bound for every guarantee" property discipline + the never-upgrade rule: **RFC-0016** §4.5 (every module ships a guarantee matrix asserted in tests, never prose only); **VR-5** (honest tags; downgrade, never upgrade); **RFC-0001** §4.3 (the `Exact ⊐ Proven ⊐ Empirical ⊐ Declared` lattice — what a passing property may and may not establish).
- The differential / oracle pattern: the landed **M-151 / M-210** interp↔AOT differential precedent and **NFR-7** (the cross-backend equivalence obligation) — the basis for `differential` and its `Skipped{BackendUnavailable}` honesty.
- The skip-is-reported ethos: the **`just check`** local↔CI parity discipline ("checks skip gracefully … never hand off a red `just check` without explaining the skip", CONTRIBUTING.md / CLAUDE.md), lifted from the build tooling to a library verdict; **G2** (never-silent).
- The failure record + EXPLAIN: **RFC-0013** (the structured diagnostic record a `Fail` carries, via `std.diag` M-510); **SC-3/G11** (EXPLAIN). The seeded-generator reproducibility: **RT3** + **`std.rand`** (M-531). The per-op contract + ring/tier: **RFC-0016** §4.1/§4.2/§4.4. The value model + identity: **RFC-0001**, **ADR-003**, **KC-3**.

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) Scope ratification (M-501).** The exported harness surface (whether fuzzing/mutation testing, benchmarking, or a `Proven`-witness mode ship in v0) is **ratified at M-501 with RFC-0016**, not here; the §3 sketch is illustrative. — *Disposition: FLAGGED; align with the ratified module set. Ties to RFC-0016 §8-Q1.*
- **(Q2) The `diag` failure-record seam (M-510).** A `Fail{diagnostic}` is a `std.diag` record; the exact `Diag` constructor a property/golden/differential failure builds (the trace shape, the counterexample embedding, the reproducing-seed field) is co-designed with `diag` and should be reconciled when both ratify. — *Disposition: FLAGGED; `testing` consumes `diag`'s record, does not redefine it; reconcile the failure-record shape with M-510.*
- **(Q3) The differential oracle's agreement bar (NFR-7 / §8-Q5).** What two backends must match — bit-for-bit observable results, or results *and* guarantee-tags + EXPLAIN — is the same migration-differential bar the `swap`/`self-hosting-readiness` specs FLAG (RFC-0016 §8-Q5). `differential` must adopt whatever bar ratifies, not pick one. — *Disposition: FLAGGED; ties to RFC-0016 §8-Q5 and the M-151/M-210 differential definition.*
- **(Q4) When a property legitimately witnesses `Proven`.** A property pass backs `Empirical`; pairing it with a cited theorem (the property as the theorem's checkable consequence) is the only route to supporting `Proven`. Whether `std.testing` exposes a typed "theorem-witness" mode (vs leaving `Proven` entirely to the numerics/proof layer) is open. — *Disposition: FLAGGED; default to `Empirical`-backing only; never let trial-passing alone reach `Proven` (VR-5).*
- **(Q5) Ergonomics of the verdict-carrying surface (tension A).** Always returning a `Verdict` (with explicit `Skipped`/`Undetermined`) vs an implicit-but-inspectable assertion macro is the RFC-0016 §8-Q3 ergonomics-vs-contract tension, instantiated for the test harness — but here the explicit-verdict side is load-bearing for the honesty crux. — *Disposition: FLAGGED; default to the explicit `Verdict`; ties to RFC-0016 §8-Q3.*

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.testing` (M-534, #174) module spec under RFC-0016 (Draft): Ring 2 / Tier B — the repo's verification discipline as a library (property / golden / differential), the language dogfooding its own honesty rule. Fixes the scope + boundary (failures are `std.diag` M-510 records; the bound kernels are `numerics` M-512 / the capability crates; reproducible inputs share the seeded `rand` M-531 discipline; the `just check` runner is repo tooling, not this library), the exported-op surface sketch (`for_all` with shrinking + seed; `golden` content-addressed baselines; `differential` interp↔AOT/native oracle; the `Verdict` sum + honest aggregator), and — the load-bearing deliverable — the per-op **guarantee matrix**: the harness ops are `Exact` *mechanisms* that **back an `Empirical` subject claim and never inflate it to `Proven`** (VR-5), and — the **honesty crux** — a skipped or undetermined check is a first-class `Skipped`/`Undetermined` verdict aggregated *distinctly* from `Pass`, so "green" can never silently include "did not check" (C1/G2, the `just check` ethos). §4.1 conformance (C1–C6) stated concretely; grounding traces to RFC-0016 §4.1/§4.4/§4.5, VR-5, NFR-7 + the M-151/M-210 differential precedent, the `just check` discipline, RFC-0013 (via `diag`), RT3 (via `rand`), RFC-0001/ADR-003/KC-3. Five questions FLAGGED (scope ratification at M-501; the `diag` failure-record seam; the differential agreement bar §8-Q5; when a property may witness `Proven`; the verdict-surface ergonomics §8-Q3). No code; no kernel change (KC-3). Append-only.
- **2026-06-19 — FLAG-DIAG / §7-Q2 (testing↔diag seam) RESOLVED.** With `std.diag` (M-510) landed,
  `FailRecord` now **delegates** to the canonical record via `FailRecord::to_diag()` →
  `mycelium_diag::Diag` (description→message; op context + reproducing seed + trial → EXPLAIN notes,
  G11), keeping the testing-specific seed/trial reproduction metadata. Code + test landed; spec
  unchanged otherwise. Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
