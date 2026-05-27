# 12 — Auxiliary & Specialized Libraries

This chapter covers the remaining libraries that don't fit neatly into
the major categories — from demanding to file-checking, TableGen to
sandboxed IR.

---

## LLVMDemangle (`lib/Demangle/`)

**CMake target**: `LLVMDemangle`

**Symbol demangling** — converts compiler-mangled symbol names back to
human-readable form.  LLVM supports four mangling schemes:

| Scheme | Standard | Used By |
|--------|----------|---------|
| **Itanium C++ ABI** | Itanium C++ ABI specification | GCC, Clang on Linux/macOS/BSD |
| **Microsoft Visual C++** | MSVC name mangling | MSVC, Clang on Windows |
| **Rust** | Rust RFC 2603 (legacy) / v0 | Rust compiler |
| **D language** | D ABI specification | DMD, LDC |
| **Vector Function ABI** | `_ZGV` / `_ZGI` prefixes | OpenMP SIMD, GLIBC `simd` functions |

Demangling is used pervasively:
- `llvm-nm` demangles symbol names in its output
- `llvm-cxxfilt` is a standalone demangler
- Sanitizer error reports include demangled function names
- `llvm-symbolizer` demangles function names

**Compiler-science context**: Name mangling is a solution to the
**name-collision problem** in linkers: C++ overloading, templates, and
namespaces mean that the same human-readable name can refer to multiple
entities.  Mangled names embed the full qualified name, parameter types,
and template arguments in a compact ASCII encoding — essentially a
**string encoding of the compiler's type system**.  Demangling is the
inverse: parse the encoding and reconstruct the human-readable form.

---

## LLVMFileCheck (`lib/FileCheck/`)

**CMake target**: `LLVMFileCheck`

**FileCheck** is LLVM's **pattern-matching test harness** — a
mini-language for writing "check lines" in test files:

```llvm
; RUN: opt -passes=instcombine %s -S | FileCheck %s
; CHECK-LABEL: @fold_add_zero
; CHECK: ret i32 %x
; CHECK-NOT: add
```

FileCheck reads stdout, matches it against `CHECK` patterns, and reports
pass/fail.  It implements:

- **Literal matching** (`CHECK: ret i32`)
- **Regex matching** (`CHECK: ret i32 [[REGEX:.*]]`)
- **Variable capture and reference** (`CHECK: [[VAR:%[a-z]+]]` / `CHECK-SAME: [[VAR]]`)
- **Numeric expressions** (`CHECK: #[[#VAR+1]]`)
- **Line anchoring** (`CHECK`, `CHECK-NEXT`, `CHECK-SAME`, `CHECK-NOT`, `CHECK-COUNT`, `CHECK-DAG`)
- **Label scoping** (`CHECK-LABEL` re-anchors matching, preventing cross-test contamination)

FileCheck is the backbone of LLVM's testing strategy, used by virtually
every test in `llvm/test/` and `clang/test/`.  It embodies the
**literate testing** philosophy: the test file is both the test input
and the expected output specification.

The underlying `FileCheck.h` library can be embedded in other tools
(like `llvm-objdump`) for inline testing.

---

## LLVMTableGen (`lib/TableGen/`)

**CMake target**: `LLVMTableGen`

**TableGen** is a **domain-specific language** (DSL) and code-generation
tool for describing structured data and generating C++ code.  It is the
foundation of LLVM's target-description system.

### Why TableGen?

Without TableGen, each backend would need thousands of lines of repetitive
C++ to describe instructions, registers, calling conventions, and
scheduling models.  TableGen lets backend authors write
**declarative `.td` files**:

```tablegen
// In X86InstrArithmetic.td:
def ADD32rr : I<0x01, MRMDestReg, (outs GR32:$dst), (ins GR32:$src1, GR32:$src2),
                 "add{l}\t{$src2, $dst|$dst, $src2}",
                 [(set GR32:$dst, (add GR32:$src1, GR32:$src2))]>;
```

TableGen processes these files and generates:
- Instruction encoding/decoding tables
- Instruction selection pattern matchers
- Register class descriptions
- Calling convention lowering code
- Scheduling itineraries and resource tables
- Subtarget feature detection code

### TableGen Architecture

```
┌────────────────────────────────────────────────┐
│  .td files (declarative descriptions)           │
│  &#8226; class, def, defm, let, foreach, multiclass  │
├────────────────────────────────────────────────┤
│  TableGen Backend (C++ backend in utils/)       │
│  &#8226; Parse .td → RecordKeeper (in-memory DB)     │
│  &#8226; Walk records, check constraints               │
│  &#8226; Emit C++ / C-header code                      │
├────────────────────────────────────────────────┤
│  Generated C++ code (linked into LLVM)          │
│  &#8226; InstrInfo tables, ISel matchers, etc.         │
└────────────────────────────────────────────────┘
```

### Standard TableGen Backends

| Backend | Generates |
|---------|-----------|
| `-gen-instr-info` | `*GenInstrInfo.inc` — instruction descriptions |
| `-gen-register-info` | `*GenRegisterInfo.inc` — register classes, sub/super-reg relationships |
| `-gen-dag-isel` | `*GenDAGISel.inc` — SelectionDAG pattern matchers |
| `-gen-global-isel` | `*GenGlobalISel.inc` — GlobalISel pattern matchers |
| `-gen-asm-writer` | `*GenAsmWriter.inc` — assembly printing |
| `-gen-asm-matcher` | `*GenAsmMatcher.inc` — assembly parsing |
| `-gen-disassembler` | `*GenDisassemblerTables.inc` — instruction decoding |
| `-gen-callingconv` | `*GenCallingConv.inc` — calling convention tables |
| `-gen-subtarget` | `*GenSubtargetInfo.inc` — CPU feature tables |
| `-gen-searchable-tables` | `*GenSearchableTables.inc` — int-to-enum lookup tables |

### TableGen Subsystems

| Component | Description |
|-----------|-------------|
| `TGLexer` / `TGParser` | Parses TableGen source (recursive descent) |
| `RecordKeeper` / `Record` | In-memory representation of all `.td` definitions |
| `Init` (IntInit, StringInit, DagInit, ...) | Value types in TableGen |
| `SetTheory` | Set-manipulation language embedded in TableGen |
| `AsmMatcherEmitter` | Generates assembly-instruction matchers |
| `CodeGenDAGPatterns` | DAG pattern matching infrastructure |
| `CodeGenSchedule` | Scheduling model analysis |
| `CodeGenTarget` | High-level view of a target from its TableGen records |

**Compiler-science context**: TableGen is a **staged DSL** — `.td` files are
compiled to C++ at build time, and the C++ is then compiled into LLVM.
This is an instance of the broader concept of **multi-stage programming**
(Taha & Sheard, 1997) or **generative programming** (Czarnecki & Eisenecker,
2000).  The `.td` DSL is declarative (what), while the generated C++ is
imperative (how).  This separation enables:

- **Static verification** (TableGen can check constraints across all
  instructions before any C++ is compiled)
- **Cross-cutting analyses** (e.g., compute the scheduling model for an
  instruction set from the .td files)
- **Multiple outputs from one source** (one instruction definition
  generates code for the assembler, disassembler, and code generator)

---

## LLVMSandboxIR (`lib/SandboxIR/`)

**CMake target**: `LLVMSandboxIR`

**SandboxIR** is a **transactional, constrained view** of LLVM IR for
use in environments that need safe, auditable IR manipulation:

- **SQLite-style transactions**: all modifications (create instruction,
  replace operand, delete instruction, ...) are recorded in a
  transaction log.  They can be committed (irreversible) or rolled back.
- **Sandboxed context**: a `SandboxIR::Context` owns a set of IR objects
  that exist independently of the main LLVM IR — they are "sandboxed"
  from the real IR until explicitly materialized.
- **Constraint checking**: operations can be validated (e.g., "does this
  transform preserve SSA?") before being applied.

**Use cases**:
- **ML-driven optimization**: an ML model proposes a sequence of
  transformations; SandboxIR lets the optimizer apply them in a sandbox,
  evaluate the result, and commit or rollback — without corrupting
  the real IR
- **Interactive compilers**: IDEs and REPLs can test transformations
  before applying them
- **Fuzzing harnesses**: random transformations can be applied in the
  sandbox and only committed if they pass verification

This connects to the concept of **software transactional memory** (STM)
(Shavit & Touitou, 1995) applied to compiler IR: all modifications are
atomic from the perspective of the real IR.

---

## LLVMCAS (`lib/CAS/`)

**CMake target**: `LLVMCAS`

**Content-Addressed Storage** — an experimental library for storing and
retrieving compiler artifacts by their cryptographic hash:

- **Immutable, deduplicated storage**: objects are stored keyed by
  `SHA256(content)`.  Identical content is stored once.
- **Merkle tree**: large objects are split into a tree of references
  (each node's hash includes its children's hashes).  This is the
  same structure as Git's object store.
- **Action caching**: an action (e.g., "compile `foo.c` with these flags")
  has a hash derived from all its inputs.  If the hash matches a previous
  run, the cached output can be used directly.

Use cases:
- **Build caching** (like `ccache` but using content-addressing for
  correctness)
- **Distributed builds** (CAS nodes can be fetched from a remote store)
- **Incremental compilation** (only recompile what changed, identified
  by hash)

CAS is related to the broader concept of **content-addressable memory**
and **Merkle DAGs** — the same data structure underlies Git, IPFS, Nix,
and Bazel's action cache.

---

## LLVMTelemetry (`lib/Telemetry/`)

**CMake target**: `LLVMTelemetry`

A new (WIP) library for structured event logging and performance
instrumentation within LLVM itself — not in compiled code, but in the
compiler process:

- Provides a vendor-neutral abstraction for telemetry events
- Integrates with build-system telemetry (e.g., build tracing)
- Collects timing, resource usage, and pass-execution data from LLVM
  tools

This is infrastructure for understanding and optimizing the compiler's
own performance — a recursive application of profiling techniques
(the profiler profiling itself).

---

## LLVMLineEditor (`lib/LineEditor/`)

**CMake target**: `LLVMLineEditor`

A minimal **line-editing library** (like GNU Readline or libedit) for
interactive CLI tools:
- Provides history (up/down arrow), editing, and tab-completion
- Used by `lli`'s interactive mode and by Clang-Repl
- Wraps either libedit (preferred) or a built-in minimal implementation

---

## LLVMTesting (`lib/Testing/`)

**Internally used only** — provides support utilities for LLVM's own
unit tests (GoogleTest helpers).  Contains facilities like `Annotations.h`
for marking positions in test IR strings.

---

## LLVMFrontend Components (`lib/Frontend/`)

The `Frontend/` directory contains several sub-libraries for generating
LLVM IR from language-level constructs:

### LLVMFrontendOpenMP (`lib/Frontend/OpenMP/`)

**CMake target**: `LLVMFrontendOpenMP`

Generates LLVM IR for OpenMP constructs (parallel regions, worksharing
loops, tasks, reductions, target offloading).  This is called by Clang's
codegen but lives in LLVM because other frontends (Flang, MLIR) also
generate OpenMP IR.

Key IR-generation pieces:
- `OMPIRBuilder` — constructs OpenMP runtime-call sequences in IR
- Outline parallel regions into separate functions
- Handle data sharing (shared/private/firstprivate/reduction)
- Target offloading: device-side kernel generation, host-side
  `__tgt_target` call insertion

### LLVMFrontendOpenACC (`lib/Frontend/OpenACC/`)

**CMake target**: `LLVMFrontendOpenACC`

Similar to OpenMP but for **OpenACC** (GPU offloading directive set
used primarily in HPC).

### LLVMFrontendOffloading (`lib/Frontend/Offloading/`)

**CMake target**: `LLVMFrontendOffloading`

Common offloading infrastructure:
- Offload binary bundling/unbundling
- Multi-target binary linking (host + GPU device code in the same
  executable)

### LLVMFrontendHLSL (`lib/Frontend/HLSL/`)

**CMake target**: `LLVMFrontendHLSL`

Generates LLVM IR for **HLSL** (High-Level Shader Language, used in
DirectX):
- HLSL-specific type lowering (vectors, matrices, buffers, samplers)
- HLSL intrinsic generation (transcendental functions, texture sampling)
- Resource binding metadata (registers, spaces)

### LLVMFrontendDirective (`lib/Frontend/Directive/`)

**CMake target**: `LLVMFrontendDirective`

Common infrastructure for compiler directives (pragmas, annotations):
- Directive parsing framework
- Language-agnostic directive representation

### LLVMFrontendAtomic (`lib/Frontend/Atomic/`)

**CMake target**: `LLVMFrontendAtomic`

Generates LLVM IR for **atomic operations** and **synchronization
constructs**:
- Atomic instruction lowering (seq_cst, acquire, release, relaxed)
- Fence generation
- Compare-and-swap (CAS) lowering for targets that don't support
  specific atomics

### LLVMFrontendDriver (`lib/Frontend/Driver/`)

**CMake target**: Not a separate library — utilities for compiler
driver tools.

---

## LLVMObjCopy (`lib/ObjCopy/`)

**CMake target**: `LLVMObjCopy`

The library behind `llvm-objcopy` and `llvm-strip`:
- Reads an object file, applies transformations, and writes the result
- Supports ELF, MachO, COFF, XCOFF, and Wasm
- Operations: add/remove/rename sections and symbols, strip debug info,
  change section flags/alignment, add/remove GNU-style notes, set symbol
  visibility, add-GNU-debuglink, extract-partition

**Compiler-science context**: Object-file manipulation is essential for
the build pipeline: strip binaries for production, split debug info into
separate files, merge or partition symbol tables.

---

## LLVMOption (`lib/Option/`)

**CMake target**: `LLVMOption`

A library for parsing **GCC/Clang-style command-line options**:
- `OptTable` — an indexed table of option definitions (prefix, name,
  help text, grouping, flags)
- Used by `llvm-objdump`, `llvm-readobj`, and other tools that want
  Clang-style `-option=value` / `--long-option=value` syntax

---

## LLVMWindowsDriver / LLVMWindowsManifest

### LLVMWindowsDriver (`lib/WindowsDriver/`)

**CMake target**: `LLVMWindowsDriver`

Parses Windows **driver command lines** — converts MSVC `cl.exe`-style
command lines and response files into options that Clang can understand.

### LLVMWindowsManifest (`lib/WindowsManifest/`)

**CMake target**: `LLVMWindowsManifest`

Parses and generates Windows **side-by-side assembly manifests** (XML
files that describe DLL dependencies and COM registration).  Used by
`llvm-mt` (Manifest Tool).

---

## LLVMHTTP (`lib/HTTP/`)

**CMake target**: `LLVMHTTP`

A minimal **HTTP client** library used by:
- `Debuginfod` (fetching debug info over HTTP)
- LLVM's build system (downloading LLVM release pages)

A lightweight wrapper around `libcurl`.

---

## LLVMExtensions (`lib/Extensions/`)

**CMake target**: `LLVMExtensions`

A **plugin-extension point** — allows external projects to install
additional LLVM extensions (extra passes, target backends) into LLVM's
build tree without modifying LLVM source code.

---

## LLVMPlugins (`lib/Plugins/`)

**CMake target**: `LLVMPlugins`

Library-side support for **dynamically loaded plugins**:
- Plugin discovery and loading infrastructure
- Works with `opt -load-pass-plugin=<plugin.so>`

---

## LLVMToolDrivers (`lib/ToolDrivers/`)

**CMake target**: `LLVMToolDrivers`

Shared infrastructure for **tool driver programs** — common code used
by multiple installed tools.  Includes shared logic for tasks like
symbol table reading, object-file dispatch, and error reporting.

---

## LLVMABI (`lib/ABI/`)

**CMake target**: `LLVMABI`

ABI (Application Binary Interface) breaking-change detection:
- Compares ABI dumps from two library versions
- Detects removed symbols, changed parameter types, etc.
- Used for LLVM's own ABI compatibility testing

---

## LLVMCGData (`lib/CGData/`)

**CMake target**: `LLVMCGData`

**CodeGenData** — a new infrastructure for collecting and consuming
code-generation profiling data (distinct from IR-level PGO):
- Tracks decisions made during codegen (register allocation, scheduling)
- Feeds back into the codegen pipeline for iterative optimization
- The counterpart to `llvm-cgdata` tool
