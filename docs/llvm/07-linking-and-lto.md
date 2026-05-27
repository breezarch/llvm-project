# 07 — Linking & Link-Time Optimization

This chapter covers LLVM's **linking** and **Link-Time Optimization** (LTO)
libraries — the bridge between separate translation units and the
whole-program optimizer.

---

## The Linking Problem

Traditional compilation (C/C++) works one translation unit (`.c` file) at a
time:
1. Compile `a.c` → `a.o` (machine code, with unresolved external symbols)
2. Compile `b.c` → `b.o`
3. **Link** — resolve cross-references, produce `a.out`

The linker is a separate program (GNU ld, gold, lld, MSVC link, Apple ld64).
LLVM provides libraries that interface with this linking step.

---

## LLVMLinker (`lib/Linker/`)

**CMake target**: `LLVMLinker`

The **IR-level linker** — links two or more LLVM `Module`s into a single
`Module`.  This is the LLVM equivalent of `ld`, but operating on LLVM IR
rather than machine code.

### Core Operation

`Linker::linkModules(Dest, Src)` performs:
1. **Type unification**: ensure that `%struct.foo` in both modules
   refers to the same `StructType` (by-name matching)
2. **Global variable merging**: merge `@foo` definitions; reject
   conflicting definitions; link appending globals (e.g., `llvm.global_ctors`)
3. **Function merging**: merge declarations with definitions; handle
   weak linkage (`linkonce`, `weak`, `common`)
4. **Metadata linking**: merge debug info (e.g., compile units from
   different `.c` files)
5. **Comdat resolution**: select which copy of a COMDAT group to keep
   based on linkage rules
6. **Use-list transfer**: redirect references from the source module
   to the merged destination module

### Linkage Types

LLVM IR uses C/C++ linkage semantics (from the Itanium C++ ABI and
PE/COFF):

| Linkage | Meaning |
|---------|---------|
| `external` | Global symbol, visible to the linker |
| `internal` | File-local (like `static`) |
| `linkonce` | Merged with identical definitions; kept if referenced |
| `linkonce_odr` | `linkonce` + One Definition Rule — all definitions are identical |
| `weak` | Can be overridden by a strong definition |
| `weak_odr` | `weak` + One Definition Rule |
| `common` | Tentative definition (C `int x;` without initializer) |
| `appending` | Appended together (used for `llvm.global_ctors`) |
| `private` | Like internal, but never used in symbol tables |
| `available_externally` | Definition exists (for optimization) but another `.o` provides it |

### Module Linking for LTO

The linker is the entry point for LTO:
1. All IR modules are linked into a single merged module
2. Whole-program optimization passes run on the merged module
3. Each function is then compiled to machine code
4. The resulting object files are linked normally

---

## LLVMLTO (`lib/LTO/`)

**CMake target**: `LLVMLTO`

The **LTO interface library** — provides a C API (and C++ API) for
linkers to invoke LLVM's LTO pipeline.

### Full LTO

```
a.bc ─┐                          ┌──▶ a.o (merged, cross-module optimized)
b.bc ─┤─ linker ──▶ LLVMLTO ──┤──▶ b.o
c.bc ─┘                          └──▶ c.o
                    │
                    └──▶ Whole-program IR → optimize → codegen per-function
```

In **Full LTO**, all IR modules are merged into a single module, then
the optimizer runs on the merged whole-program IR.  This enables:
- **Cross-module inlining** — inline `foo.c` into `bar.c`
- **Whole-program devirtualization** — see all possible targets of a virtual call
- **Dead global elimination** — remove functions/data never referenced
- **Constant propagation across modules** — if all callers pass the same value
- **Interprocedural alias analysis**

The downside: Full LTO requires all IR in memory at once, which is
prohibitively expensive for large programs (like Chrome, LLVM itself).

### ThinLTO

```
a.bc ─┐                          ┌──▶ a.o (with cross-module import)
b.bc ─┤─ linker ──▶ ThinLTO ──┤──▶ b.o
c.bc ─┘                          └──▶ c.o
                    │
                    └──▶ Per-module summaries → selective cross-module import
```

**ThinLTO** (introduced in LLVM 3.9, ~2016) is a **scalable alternative**
to Full LTO:

1. **Summary phase**: each module is summarized into a compact "module
   summary" containing function signatures, callee lists, instruction
   counts, and PGO hotness
2. **Index phase**: the linker merges summaries and decides what to
   import where (based on call-graph analysis and PGO data)
3. **Import phase**: each module imports only the *functions it actually
   calls* (plus their transitive closure of call targets)
4. **Parallel compilation**: each module is compiled independently, with
   imported function bodies available for inlining and optimization

Key ThinLTO concepts:

| Concept | Description |
|---------|-------------|
| **Module summary index** | Compact per-function info: GUID, linkage, call graph edges, PGO hotness, CFI targets |
| **GUID** (Global Unique ID) | Hash of the mangled symbol name (xxHash64) |
| **Importing** | Pull function bodies from other modules into the current module for inlining |
| **Caching** | Compiled object files are cached (keyed by hash of module + imports) |
| **Distributed build** | The index can be created centrally, then modules are compiled in parallel on hundreds of machines |

**Performance**: ThinLTO achieves ~95% of Full LTO's performance with
orders-of-magnitude less memory and near-perfect parallelism.  It's
used in production by Google (Chrome, internal C++ services), Apple
(Xcode), and many others.

### ThinLTO vs. FullLTO Tradeoffs

| | Full LTO | ThinLTO |
|---|---|---|
| Memory | O(#modules) — all IR in memory | O(1) per compilation |
| Parallelism | Serial (monolithic module) | Fully parallel (each module independent) |
| Optimization quality | Slightly better (sees everything) | ~95% of FullLTO |
| Incremental | None (must redo from scratch) | Yes (only changed modules recompiled) |
| Caching | N/A | Yes (cached per-module objects) |

---

## LLVMDTLTO (`lib/DTLTO/`)

**CMake target**: `LLVMDTLTO`

**DTLTO** (Distributed ThinLTO) — support for offloading LTO preprocessing
in a distributed build system (like Bazel, Buck2, or Reclient):

- Creates a **pre-merge summary** before the final link
- Allows the build system to split LTO across a build cluster
- Integrates with the ThinLTO caching infrastructure

---

## LTO and PGO

LTO combined with PGO is particularly powerful:
- PGO provides **branch weights** and **hot/cold annotations** for each
  function and call site
- ThinLTO uses PGO data to decide **what to import**: only hot callees
  are imported
- PGO-driven **indirect-call promotion** is more effective with LTO
  because all callee candidates are visible

This combination is inspired by the **profile-guided optimization**
research from the 1990s (Pettis & Hansen, 1990; Cohn & Lowney, 1996)
and the **feedback-directed optimization** (FDO) in Java HotSpot.

## Compiler-Science Context

### The Separate Compilation Problem

The fundamental constraint is that C/C++ compilers historically compile
**one file at a time** — a design decision from the 1970s when RAM was
measured in kilobytes.  LTO is a workaround: defer optimization to
link time when all code is visible.

This is a special case of the more general problem of **staged
compilation** or **multi-phase optimization**, also seen in:
- **Whole-program optimization** in MLton (Standard ML)
- **JIT compilation with tiered optimization** (Java HotSpot, V8)
- **Ahead-of-time partial evaluation** (specializing code for known inputs)

### Call-Graph Analysis for LTO

ThinLTO's import decision is a **call-graph analysis** problem:
- Build the call graph (caller → callee edges)
- Annotate edges with PGO weights (hot/cold)
- Select a subset of edges to "promote" to cross-module imports
- Constraints: per-module size budget (importing a huge function is
  counterproductive), call depth limit

This can be modeled as a **profit/cost knapsack problem** on the call
graph — for each callee, decide whether to import it based on the
expected benefit (inlining opportunities, subsequent optimization)
versus the cost (code size increase, increased compile time).

### Incremental Compilation & Caching

ThinLTO's caching strategy connects to the broader problem of
**incremental computation** (Pugh & Teitelbaum, 1989; Acar, 2009).
By caching compiled objects keyed by the hash of:
- The module's own IR
- The set of imported functions and their summaries

...ThinLTO can skip re-compilation entirely when the module and its
imports haven't changed.  This makes ThinLTO viable for edit-compile-debug
cycles.
