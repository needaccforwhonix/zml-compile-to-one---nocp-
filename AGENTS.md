# AGENTS.md - ZML Agent Guide

This guide is for autonomous coding agents working in this repository.

## Scope and Environment

- Primary language: Zig (dev snapshot), built and tested with Bazel.
- Main subsystems: `zml/`, `stdx/`, `mlir/`, `pjrt/`, `platforms/`, `examples/`.
- Bazel version is pinned in `.bazelversion` (`8.5.1`).
- Build mode defaults to debug via `.bazelrc` (`--compilation_mode=dbg`).

## External Instruction Files

- Cursor rules: `.cursor/rules/**` and `.cursorrules` are not present.
- Copilot rules: `.github/copilot-instructions.md` is not present.
- If any of these files appear later, treat them as required constraints.

## Core Build Commands

```bash
# Build all targets
bazel build //...

# Build specific targets/packages
bazel build //zml
bazel build //examples/llama
bazel build //examples/mnist

# Alternate build configs
bazel build --config=debug //...
bazel build --config=alldebug //...
bazel build --config=release //...
bazel build --config=native //...
```

### Useful `.bazelrc` config meanings

- `--config=debug`: optimized backend, Zig frontend debug mode.
- `--config=release`: optimized (`-Ofast`, Zig `release_safe`).
- `--config=alldebug` / `--config=native` / `--config=silent`: debug-all, local CPU tuning, reduced UI noise.

## Test Commands

```bash
# Run everything
bazel test //...

# Common package tests
bazel test //zml:test
bazel test //stdx:test
bazel test //mlir:test
bazel test //zml/tokenizer:test

# Discover Zig test targets
bazel query 'kind(zig_test, //...)'
```

### Run a single test (inside a `zig_test` target)

`zml/test_runner.zig` supports substring filtering from positional args.
Pass filter with `--test_arg`:

```bash
# Runs tests whose name contains "normalizeL2"
bazel test //zml:test --test_arg=normalizeL2

# Example on another target
bazel test //stdx:test --test_arg=json
```

- Matching is substring-based on test function identifier; `.bazelrc` already sets `test --test_output=errors`.

## Lint / Formatting Commands

```bash
# Format Bazel/Starlark files
./tools/buildifier.sh

# Run ZLS through Bazel (editor and language tooling)
./tools/zls.sh
```

## Platform / Accelerator Flags

Append these to `bazel build|test|run` as needed:

```bash
--@zml//platforms:cuda=true
--@zml//platforms:rocm=true
--@zml//platforms:tpu=true
--@zml//platforms:neuron=true
--@zml//platforms:cpu=false
```

## Code Style (Repository Conventions)

### Imports and namespacing

- Prefer module imports, then namespaced access.
- Prefer: `const foo = @import("foo.zig"); foo.bar()`.
- Avoid: `const bar = @import("foo.zig").bar` as default style.
- Import specific types directly only when heavily used.
- Keep local imports relative within a package.

### Formatting and structure

- Follow Zig style and existing local patterns first.
- Prefer typed anonymous initialization where clear: `const x: Foo = .{ .bar = 1 };`.
- Keep functions focused; pair resource acquisition with nearby `defer`/`errdefer`.
- Add comments only for non-obvious invariants, ownership, or shape semantics.

### Types, naming, and tensor semantics

- Types/structs: `PascalCase`.
- Functions/vars/fields: Zig lower camel case.
- Keep names descriptive; avoid short opaque identifiers unless very local.
- Use semantic tensor tags (`.b`, `.s`, `.d`, `.h`, `.hd`, etc.) over raw index assumptions.

### Errors, assertions, and cleanup

- Prefer `try`-based propagation.
- Use `catch |err|` when you add useful recovery or context.
- Use `errdefer` on partial initialization paths.
- Prefer `stdx.debug.assert` over bare `std.debug.assert` in this codebase.
- Avoid panic-style control flow except for unrecoverable programmer errors.

### Logging and CLI entrypoints

- Use scoped logs (for example `std.log.scoped(.@"zml/io")`).
- CLI binaries commonly define `pub const std_options`.
- Argument parsing usually follows `stdx.flags` conventions.

## Testing Patterns

- Tests are usually inline Zig `test "..." {}` blocks near implementation.
- Common helpers: `zml.testing.env`, `zml.testing.expectClose`, `zml.testing.approxEq`.
- `zig_test` targets often use `zml/test_runner.zig` for progress/filter/leak checks.

## Critical Architecture Guardrails

- `Tensor` ops in `forward` build an MLIR graph; they do not execute immediately.
- Preserve distinctions:
  - `Shape` = metadata only
  - `Tensor` = compile-time graph value
  - `Buffer` = runtime device allocation
  - `Slice` = host-side data view
- `zml.Bufferized(T)` maps tensor-typed model structures to runtime buffer structures.
- Weight loading paths are centered on safetensors + TensorStore/VFS.

## Practical Agent Workflow

- Read the target file, nearby tests, and owning `BUILD.bazel`.
- Check whether behavior lives in compile-time graph construction or runtime execution.

- Run the narrowest relevant tests first (`--test_arg` when possible).
- Run broader package tests if shared code changed.
- Run `./tools/buildifier.sh` when any Bazel files changed.



## 🛡️ Strict Embedding Separation, Zero-Fallback Law & 24/7 Dual-GPU Invariant
- **Reference**: 

### 1. Das Absolute Fallback-Verbot (Zero-Fallback Law)
Unter keinen Umständen, zu keinem Zeitpunkt und aus keinem Grund darf ein Fallback zwischen verschiedenen Embedding-Modellen stattfinden.
* **Geltende Aktion:** Fällt ein Embedding-Modell aus oder ist überlastet, MUSS die Operation sofort hart fehlschlagen () oder die Payload transaktional in einer Queue (NATS/SQLite) verharren, bis das exakte Modell bereit ist.
* **Verboten:** Kein stiller oder dynamischer Modellwechsel (weder Jina -> Gemma noch umgekehrt).

### 2. Warum ein Embedding-Fallback mathematisch & informationstheoretisch unmöglich ist
* **Topologische Inkompatibilität heterogener Vektorräume (Non-Isomorphism):**
  Jedes Modell 	heta: \mathcal{X} 	o \mathbb{R}^D$ projiziert Text in eine spezifische, gelernte Riemannsche Mannigfaltigkeit. Jina v5 (=256$) und EmbeddingGemma (=768$) spannen zwei völlig inkompatible geometrische Räume auf. Die Basisvektoren der semantischen Achsen sind ohne explizite Procrustes-Transformation nicht ausgerichtet.
* **Kollaps der Kosinus-Ähnlichkeit ($	ext{sim} pprox 0$):**
  Wird eine Suchanfrage mit Modell $ berechnet ( = f_B(q)$), während der Dokumentenkorpus mit Modell $ indiziert wurde ( = f_A(d)$), verhält sich das Skalarprodukt mathematisch wie das zweier rein zufälliger Vektoren auf einer hochdimensionalen Einheitssphäre:
  86306\mathbb{E}[	ext{sim}(u, v)] = 0 \quad 	ext{mit Varianz} \quad \sigma^2 = rac{1}{D}86306
  Der Nearest-Neighbor-Algorithmus (HNSW/k-NN) liefert stochastisches Rauschen. Das RAG-System erhält völlig falsche oder irrelevante Kontexte.
* **Irreversible Index-Vergiftung (Index Poisoning):**
  Wird auch nur ein einziger Vektor von Modell $ als 'Fallback' in den Index von Modell $ geschrieben, verunreinigt er die Distanzgraphen und Clusterzentren dauerhaft.
* **Das Gesetz des Fail-Fast:**
  Ein Ausfall muss hart abbrechen ().

### 3. Duale 24/7 Erfassungspflicht (GPU-Only)
* **GPU-Only Mandat:** Es läuft absolut nichts auf der CPU — GPU ONLY (NVIDIA GB10 CUDA) für ausnahmslos jedes Embedding-Modell.
* **24/7 Parallelität:** Sowohl  (256D, ~4,1 GB VRAM) als auch  (768D, ~1,2 GB VRAM) laufen dauerhaft 24/7 im VRAM (Summe ~5,3 GB VRAM).
* **Duale Erfassung:** Jeder zu indizierende Text/Chunk wird immer von beiden Modellen parallel eingebettet und getrennt persistiert.

### 4. Idioten- & Failsafe-Sicherung auf Datenbankebene
* **SQLite Schema CHECK-Constraints:**
   und  erzwingen atomare Abbrüche auf Engine-Ebene bei Modell-Mismatches.
* **Qdrant Collection Constraints:**
  Strikte Trennung in separate Collections ( vs ) mit fixierter Vektordimension.


## ⚡ High-Quality Systems Programming Languages Priority (No-Python Policy)
- **Reference**: `/home/m1st/.agents/rules/RULE_High_Quality_Systems_Programming_Languages.md`
- **Rule**:
  1. **Bevorzugte Sprachen:** High Quality **Golang (Go), Rust, C++, Zig, PowerShell, C** sind IMMER und AUSNAHMSLOS die bevorzugten Programmiersprachen.
  2. **Kein Python:** Python ist für neue Daemons, Watcher, Automatisierungen, APIs, CLI-Tools und Dienste strikt untersagt (GIL-Bottlenecks, Dependency-Drift, Speicherineffizienz).
  3. **Natives Systems-Engineering:** Alle Hintergrunddienste, Caching-Ebenen und Task-Runner müssen als native, speichersichere und nebenläufige Binaries kompiliert werden.
