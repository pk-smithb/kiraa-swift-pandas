# SwiftPandas — SPM consumability + a proposal for an external performance harness

**Status:** analysis + proposal
**Audience:** SwiftPandas maintainers (kiraa-ai) and the hypermedia-tech fork
**Author:** hypermedia-tech (Bruno)

---

## 1. The problem, in one sentence

The package cannot be added to another Swift project the normal way — by URL + version
("Add Package Dependency…" in Xcode, or a Swift Playground) — because its product
targets carry `unsafeFlags`, and SwiftPM refuses to resolve a **version-pinned**
dependency whose product graph uses unsafe build flags.

The user-visible symptom is the resolver bailing out:

```
could not resolve package graph, cannot continue
```

…or, with full output:

```
error: the target 'SwiftPandas'  in product 'SwiftPandas' contains unsafe build flags
error: the target 'CUltraJSON'   in product 'SwiftPandas' contains unsafe build flags
error: the target 'CKHash'       in product 'SwiftPandas' contains unsafe build flags
error: the target 'CSkipList'    in product 'SwiftPandas' contains unsafe build flags
```

## 2. Root cause

`Package.swift` forces an optimization level on the four targets that make up the
shipped library product:

```swift
// three vendored C targets
cSettings: [ .headerSearchPath("include"), .unsafeFlags(["-O3"]) ]
// the Swift library target
swiftSettings: [ .define("ACCELERATE_AVAILABLE", …), .unsafeFlags(["-O"]) ]
```

SwiftPM has a hard rule: **a product whose target graph contains `unsafeFlags`
may not be consumed as a versioned source-control dependency.** The intent is
supply-chain safety — a downstream package shouldn't silently inherit arbitrary
compiler flags it can't see when it pins `from: "x.y.z"`.

### Why it slipped through

The rule is enforced *only* for version-pinned consumption. It is **not** enforced for:

- the root package building itself (`swift build`, `swift test`)
- a `path:` dependency
- a `branch:` / `revision:` dependency
- **binary mode** (`SWIFTPANDAS_USE_BINARY=1`) — that branch builds a plain
  `.binaryTarget` with no unsafe flags, so it resolves fine

Every way the library is exercised in-house dodges the check. The single route that
trips it — *add by URL and pick a version* — is the one real consumers actually use,
and it was never tested.

### The docs document the opposite of reality

`README.md` (the "Note on Swift Package Registry") states:

> SwiftPandas uses `unsafeFlags(["-O3"])` … which prevents publishing the source
> package to the official Swift Package **Registry**. Both git-URL source pinning and
> the binary-target XCFramework distribution shown above are **fully supported and
> unaffected.**

This is the crux of the misunderstanding. `unsafeFlags` does *not* merely block the
(rarely used) Swift Package *Registry* — it blocks **git-URL version pinning**, which
is the primary distribution path. The note promises the broken path works.

## 3. Why the flags were there at all (and why removing them is safe)

The flags exist to make **benchmarks representative in debug builds**. SwiftPM
optimization defaults are config-driven:

| Build config | Swift | C |
|---|---|---|
| Debug (`swift build`) | `-Onone` | `-O0` |
| Release (`swift build -c release`) | `-O` | `-O2` |

Forcing `-O3`/`-O` regardless of config means `swift test` / a debug `swift run`
report release-like numbers without anyone remembering `-c release`. That is a
**measurement convenience baked into the build graph** — and it is exactly what
poisons distribution.

Removing the flags costs nothing that matters:

- **Correctness:** identical results at any optimization level.
- **Release performance / the 120× claim:** unaffected — a consumer's Release build
  already gets `-O`/`-O2` automatically.
- **Only loss:** SwiftPandas runs at debug speed in *debug* builds — like every other
  Swift library. Benchmarks must be run with `-c release` (which is the correct way
  to benchmark anyway).

**Minimum fix already validated:** delete `unsafeFlags` from the three C targets and
the `SwiftPandas` target. A versioned consumer then resolves and reaches
`Build complete!` — confirmed against a tagged clone.

## 4. The proposal — move performance into an external release-mode harness

The deeper fix is to stop coupling a *measurement* concern to the *build graph*.
This mirrors the k6 model we use for Tanuki: performance is asserted by an external
harness driving a realistic build, not by reaching into the product's compiler
settings.

### 4.1 Separation of concerns

| Concern | Today (wrong home) | Proposed home |
|---|---|---|
| Library correctness | unit tests | unit tests (unchanged) |
| Performance / 120× claim | `unsafeFlags` in `Package.swift` | dedicated **release-mode benchmark target** with threshold gates |
| Distribution | coupled to the above, broken | clean manifest, consumable by version |

### 4.2 What the target looks like in practice

A separate executable target that is **never part of the shipped library product**,
so it can carry whatever flags it likes without affecting consumers:

```swift
// In Package.swift — an executable, not a library product.
.executableTarget(
    name: "SwiftPandasBench",
    dependencies: ["SwiftPandas"],
    path: "benchmarks/Runner"
    // No unsafeFlags needed: it is invoked with `swift run -c release`,
    // which yields -O / -O2 for the whole graph for free.
)
```

Run it the way a CI perf job would:

```bash
swift run -c release SwiftPandasBench --suite groupby,merge,sort --json out.json
```

Because it builds in Release, the numbers are *more* faithful than forced-`-O3`
debug ever was — Release also enables cross-module optimization and LTO that the old
flags never delivered.

### 4.3 What the harness looks like in practice (the "k6" part)

The runner is a thin loop: for each named scenario, build the input, time the
operation, and assert against a budget. Pass/fail gates — not just printed numbers —
so a regression *fails the build*.

```
scenario            | op                       | budget   | gate
--------------------+--------------------------+----------+------
groupby_sum_1M      | df.groupBy(k).sum()      | < 25 ms  | hard
merge_inner_100K    | left.merge(right, .inner)| < 20 ms  | hard
sort_1M             | df.sort("col")           | < 120 ms | soft (warn)
vs_pandas_speedup   | ratio vs recorded baseline| >= 50x  | hard
```

- **Budgets** live in a checked-in JSON/YAML next to the runner, versioned with the
  code, so a threshold change is a reviewable diff.
- **Baselines** (e.g. the pandas comparison) are recorded artifacts the harness loads
  and asserts a ratio against — the same way a k6 test asserts p95 latency, not raw
  timings.
- **CI job** (separate from the unit-test job) runs `swift run -c release
  SwiftPandasBench`, parses the JSON, and fails if any hard gate is breached. This is
  strictly stronger than the old setup: the flags made debug *look* fast but asserted
  nothing; the harness *enforces* the 120× claim on every PR.

### 4.4 Net effect

- `Package.swift` is clean → installable by URL + version in Xcode and Swift
  Playgrounds, builds on Linux, registry-publishable.
- Performance is continuously *enforced* in Release, where it's real.
- The "120× faster" headline becomes a CI invariant instead of a README assertion.

## 5. Secondary issues worth fixing alongside (out of scope for the harness, noted for completeness)

1. **No usable semver tag.** The only tag is `0.0.1`, sitting on the `v0.6.2-beta`
   commit; there is no `0.6.x` release tag. `from: "0.6.0"` finds no version. Prerelease
   tags (`-beta`) are also skipped by normal version ranges. Cut a clean `0.6.2`-style tag.
2. **Binary mode is misconfigured.** `xcframeworkURL` points at
   `github.com/kiraa-ai/kiraa-swift-pandas` while the fork lives at
   `github.com/hypermedia-tech/kiraa-swift-pandas`, and it references a release
   (`v0.6.1-beta`) whose XCFramework asset doesn't exist (the docs admit the asset is
   only on `v0.5.0-beta`). `SWIFTPANDAS_USE_BINARY=1` cannot resolve as written.
3. **Manifest/doc drift.** Header docs say v0.6.1, the binary URL says v0.6.1-beta, the
   repo is at v0.6.2-beta; `Package.resolved` pins argument-parser 1.7.0 while resolution
   computes a newer version. Cosmetic, but worth a pass.
4. **The README "Note on Swift Package Registry" is actively misleading** and should be
   corrected or deleted (see §2).

## 6. TL;DR

- The package is unconsumable by version because of `unsafeFlags` on its product
  targets. Removing them is a safe, one-line-per-target fix (validated end-to-end).
- The flags were a stand-in for "benchmark in optimized mode." The right replacement is
  an **external, release-mode benchmark target + threshold harness** (k6-style), kept
  out of the shipped product graph entirely.
- Doing both gets you: a clean, installable package *and* a stronger, enforced
  performance guarantee than the flags ever provided.
