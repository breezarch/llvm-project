# 09 — Profiling & Instrumentation

This chapter covers LLVM's libraries for **profile-guided optimization**,
**runtime instrumentation**, **fuzzing**, and **performance analysis**.

---

## The Feedback-Directed Optimization Loop

PGO (Profile-Guided Optimization) is a two-phase process:

```
Phase 1: Instrumented Build                Phase 2: Optimized Build
───────────────────────────               ─────────────────────────
Source ──▶ Instrumented Binary              Same Source + Profile Data
              │                                         │
              ▼                                         ▼
         Run on training input              Compiler uses profile to:
              │                             · Inline hot functions
              ▼                             · Outline cold code
         Profile data (.profraw)            · Optimize branch layout
              │                             · Promote indirect calls
              ▼                             · Optimize loop trip counts
         .profraw → llvm-profdata → .profdata
```

This is also called **FDO** (Feedback-Directed Optimization) or **PGO**
(Profile-Guided Optimization).  The seminal papers are:
- **Pettis & Hansen (1990)**: "Profile Guided Code Positioning" — using
  execution counts to reorder basic blocks for better I-cache and branch
  prediction
- **Cohn & Lowney (1996)**: "Design and Analysis of Profile-Based
  Optimization" at DEC — the first production FDO system
- **Chen et al. (2016)**: "Context-sensitive PGO" (CSPGO) — using call
  contexts, not just function-level counts

---

## LLVMProfileData (`lib/ProfileData/`)

**CMake target**: `LLVMProfileData`

The core library for reading, writing, and manipulating profile data.

### Profile Data Pipeline

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ .profraw    │     │  .profdata   │     │  Compiler   │
│ (raw        │────▶│  (indexed,   │────▶│  consumes   │
│  counters)  │     │   merged)    │     │  profile)   │
└─────────────┘     └──────────────┘     └─────────────┘
     ▲                                         │
     │                                         ▼
   Runtime                           ┌─────────────────┐
   instrumentation                   │ Optimized binary │
                                     └─────────────────┘
```

| Format | Extension | Content | Tool |
|--------|-----------|---------|------|
| **Raw profile** | `.profraw` | Counters dumped by instrumented binary | Written by `__llvm_profile_write_buffer()` |
| **Indexed profile** | `.profdata` | Merged + indexed for compiler consumption | `llvm-profdata merge` |

### Key Classes

| Class | Role |
|-------|------|
| `InstrProfReader` | Read `.profdata` — provides function-level execution counts, branch weights |
| `InstrProfWriter` | Write `.profdata` — merges raw profiles |
| `InstrProfRecord` | Per-function counters — number of times each basic block executed |
| `InstrProfValueData` | Value-profile records — distribution of indirect call targets |
| `SampleProfileReader` | Read `.afdo` / `.samples` — sampled profiles (from `perf` or AutoFDO) |
| `MemProfReader` | Read heap memory profiles — memory allocation context data |
| `CtxProfReader/CtxProfWriter` | Context-sensitive profiling data |

### Coverage (`lib/ProfileData/Coverage/`)

Source-level code coverage:
- Reads GCOV-format coverage data (`.gcda` / `.gcno` files)
- Reads LLVM's own source-based coverage format
- Maps counter values → source file → line-level coverage

### Profile Types

| Type | Collection Method | Granularity | Precision |
|------|-------------------|-------------|-----------|
| **Instrumentation** | Counters inserted by compiler → counted at runtime | Basic block / edge | Exact |
| **Sampling** | Hardware performance counters (perf, AutoFDO) | Instruction pointer samples | Statistical |
| **Context-sensitive** | Counters tagged with call context | Function × call path | Exact |
| **Memory (MemProf)** | Heap allocation tracking → allocation context | Allocation call stack × hotness | Exact |
| **CSSPGO** | Context-Sensitive Sample-based PGO | Full call-context tree + samples | Statistical |

### Context-Sensitive PGO

CSPGO (introduced by Google in 2021) extends PGO with **call context**:
- Instead of "function `foo` was called 1000 times", it records
  "within call path `a→b→foo`, `foo` was called 300 times; within
  `c→d→foo`, 700 times"
- The optimizer can then differentiate: inline `foo` when called from
  `a→b` but not when called from `c→d` (if it's cold there)
- This requires tracking a **context identifier** at runtime (stored
  in a thread-local slot)

### MemProf

**Memory Profiling** (introduced by Google) tracks **heap allocation
contexts** — for each allocation, it records:
- The call stack that allocated it
- The size and lifetime of the allocation

The compiler can then:
- Clone functions based on allocation behavior (a function may be
  "cold" because it allocates long-lived memory)
- Guide memory layout (put allocations with similar lifetimes together)
- Drive memory-related optimizations

---

## LLVMXRay (`lib/XRay/`)

**CMake target**: `LLVMXRay`

**XRay** is a dynamic function-call tracing framework:
- The compiler inserts **no-op sleds** at function entry/exit
- At runtime, sleds are patched (binary rewriting) to call the tracing
  handler
- Tracing can be enabled/disabled at runtime with minimal overhead

### Use Cases

- **Latency analysis**: measure how long each function takes
- **Call-graph tracing**: reconstruct the dynamic call graph
- **Tail-latency analysis**: measure P99, P99.9 latencies per function
- **Flight-data recorder**: always-on trace buffer that can be dumped
  when a condition triggers

### Architecture

| Component | Description |
|-----------|-------------|
| `XRayRecord` | A single trace entry: timestamp, function ID, type (enter/exit/arg) |
| `XRayTrace` | A sequence of records (the `.xray` log file) |
| `XRayConvert` | Convert binary trace → YAML |
| `XRayAccount` | Reconstruct per-function call counts and cumulative latencies |
| `XRayGraph` | Build a dynamic call graph from the trace |
| `FDRRecords` / `FDRTraceWriter` | Flight Data Recorder mode — compressed binary trace format |

**Theoretical background**: XRay sled patching is a form of **dynamic
binary instrumentation** (DBI), similar in concept to Intel Pin, DynamoRIO,
or Valgrind — but integrated into the compiler rather than an external tool.

---

## LLVMFuzzer — libFuzzer (`lib/Fuzzer/`)

**CMake target**: `LLVMFuzzer` (though libFuzzer has its own build system)

**libFuzzer** is a library for **coverage-guided fuzz testing**:
1. The target function is compiled with sanitizer coverage instrumentation
2. libFuzzer generates random input bytes
3. It feeds inputs to the target function
4. It monitors coverage (which basic blocks executed)
5. If a new coverage pattern is seen, the input is added to the corpus
6. Inputs from the corpus are mutated (bit flips, insertions, deletions,
   splicing) to discover more coverage
7. When a crash (sanitizer failure) is found, libFuzzer minimizes the
   crashing input and reports it

**Key techniques**:
- **Corpus-based mutation** (the input corpus evolves over time)
- **Coverage feedback** (8-bit edge counters — same as AFL)
- **Value profiling** (compare operands of CMP instructions to guess
  magic constants)
- **Dictionary-based mutation** (user-provided tokens like `"<html>"`)

**Theoretical background**: Fuzz testing was pioneered by Miller, Fredriksen,
& So (1990) with their study on Unix utility reliability.  Coverage-guided
fuzzing (AFL-style) adds evolutionary algorithms to the mix — the corpus
evolves under a **fitness function** (novelty of coverage) with
**crossover** (splicing two inputs) and **mutation** (bit-level and
structure-aware mutations).

libFuzzer also pioneered **libFuzzer + SanitizerCoverage** — using the
compiler's own instrumentation for feedback, instead of QEMU-based
instrumentation (AFL) or hardware breakpoints.

---

## LLVMFuzzMutate (`lib/FuzzMutate/`)

**CMake target**: `LLVMFuzzMutate`

A library for **IR-level mutation** — takes a valid LLVM IR module and
randomly transforms it (insert instructions, delete instructions, replace
operands, shuffle basic blocks).  Used by:
- `llvm-opt-fuzzer` — randomly generates and mutates IR modules to fuzz
  the optimizer
- `llvm-isel-fuzzer` — fuzzes the instruction selector
- `llvm-mc-*-fuzzer` — fuzzes the assembler/disassembler

These fuzzers are part of LLVM's **quality-assurance strategy**: by
randomly generating/mutating IR and feeding it through the optimizer,
we can find assertion failures, miscompiles, and infinite loops in
the pass pipeline.

**Compiler-science context**: Random testing of compilers (also called
**fuzz testing** or **random differential testing**) has a long history:
- **Csmith** (Yang et al., 2011): random C program generator that found
  hundreds of bugs in GCC and LLVM
- **EMI** (Le et al., 2014): equivalence-modulo-inputs — mutate a program
  and check that the output is the same
- **Alive2** (Lopes et al., 2021): formal verification of InstCombine
  transformations using SMT solvers

LLVM's fuzzing infrastructure complements these approaches by testing
at the IR granularity rather than the source-code level.

---

## Sanitizers Integration

Sanitizers use `lib/Transforms/Instrumentation/` to insert runtime checks
into the IR.  The runtime libraries (AddressSanitizer, MemorySanitizer,
etc.) are in `compiler-rt/` (not in llvm/), but the instrumentation
passes live in LLVM proper:

| Pass | Instrumentation Inserted |
|------|-------------------------|
| **AddressSanitizer** | Red zones around stack/heap allocations, shadow memory checks |
| **MemorySanitizer** | Shadow bits for every byte (tracking initialization) |
| **ThreadSanitizer** | Happens-before relationship tracking |
| **HWAddressSanitizer** | Hardware-assisted (ARM TBI) memory tagging |
| **DataFlowSanitizer** | General taint tracking (user-defined labels) |
| **SanitizerCoverage** | Edge counters for fuzzing feedback |
| **BoundsChecking** | Array bounds checks via `__builtin_trap()` |

All sanitizers rely on the same infrastructure:
- **Shadow memory mapping** — part of the address space is reserved for
  metadata (e.g., ASan uses 1/8 of the virtual address space)
- **Unwinding** — when a sanitizer detects an error, it uses the
  symbolizer to produce a human-readable stack trace
- **Runtime callbacks** — instrumented code calls into the runtime
  library (`compiler-rt`) when a violation is detected
