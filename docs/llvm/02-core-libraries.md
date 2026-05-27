# 02 — Core Libraries

This chapter covers the **foundational libraries** upon which everything else
in LLVM is built.  These form the bottom and middle of the layered architecture.

---

## LLVMSupport (`lib/Support/`)

**CMake target**: `LLVMSupport`

The operating-system abstraction layer.  It provides a portable C++ runtime
on top of Windows / macOS / Linux / AIX / z/OS.

**Key facilities:**

| Subsystem | Headers | Provides |
|-----------|---------|----------|
| **Command-line parsing** | `CommandLine.h` | `cl::opt`, `cl::list`, `cl::alias` — global opt registration with automatic help text |
| **Error handling** | `Error.h`, `ErrorOr.h` | `llvm::Error` (monadic error type, inspired by Haskell's `Either`), `Expected<T>`, `ErrorOr<T>` |
| **Memory management** | `Allocator.h`, `BumpPtrAllocator.h` | Arena-based allocation; the `BumpPtrAllocator` is used throughout for fast, batch-free'd memory |
| **File I/O & Memory mapping** | `MemoryBuffer.h`, `raw_ostream.h` | `MemoryBuffer` (read-only file abstraction), `raw_fd_ostream`, `raw_string_ostream` |
| **Threading** | `Threading.h`, `ThreadPool.h` | Parallelism support; used by ThinLTO (parallel codegen), pass manager parallelization |
| **Hashing** | `xxhash.h`, `MD5.h`, `SHA256.h` | xxHash (fast, non-crypto), MD5, SHA-1, SHA-256 |
| **Path & filesystem** | `Path.h`, `FileSystem.h` | Portable path manipulation, `sys::fs` for directory walking, file status |
| **RTTI replacement** | `TypeName.h`, `Casting.h` | `isa<>`, `cast<>`, `dyn_cast<>` — LLVM's own RTTI system (discussed below) |
| **Debugging** | `Debug.h`, `Timer.h`, `Statistic.h` | DEBUG() macros, `NamedRegionTimer`, pass statistics |
| **Compression** | `Compression.h` | Zlib/Zstd compression (used by bitcode, debug info) |
| **Signals & crash reporting** | `Signals.h`, `CrashRecoveryContext.h` | Stack traces on crash, crash recovery for compiler-as-a-service |
| **Endianness** | `Endian.h` | `support::endian::read{16,32,64}` for cross-endian binary parsing |
| **Formatted output** | `Format.h` | `format()`, `utohexstr()` — type-safe printf-style formatting |
| **Caching** | `CachePruning.h` | LRU/LFU cache eviction for ThinLTO cache, CAS cache |
| **Regex** | `Regex.h` | POSIX regular expressions |
| **Virtual File System** | `VirtualFileSystem.h` | Overlay, redirecting, and in-memory VFS for reproducers, tools |
| **String Pooling** | `StringMap.h`, `StringSaver.h` | Interned strings (used everywhere for identifiers) |

**Compiler-science context**: The `CommandLine` library uses global registration
of options before `main()` — a pattern made possible in C++ through static
initialization.  The `Error` / `Expected` / `ErrorOr` framework is an algebraic
data type (sum type) idiom ported from functional languages (Haskell's `Either`,
OCaml's `result`, Rust's `Result`).  The RTTI replacement system (`isa<>`/`cast<>`)
is LLVM's most famous C++ pattern: it avoids the overhead and fragility of
`dynamic_cast` by using hand-rolled type IDs that are checked via pointer-comparison
against a `TypeID` constant — reminiscent of the **vtable pointer-comparison** trick
in the Smalltalk / Self virtual-machine literature.

---

## LLVM ADT (`include/llvm/ADT/`)

**Not a separate CMake target** — these are header-only types used by every
other LLVM library (they are installed alongside `LLVMSupport`).

**Key data structures:**

| Type | Header | Description |
|------|--------|-------------|
| `SmallVector<T, N>` | `SmallVector.h` | Inline storage for N elements; spills to heap when exceeded.  The single most-used container in LLVM. |
| `SmallString<N>` | `SmallString.h` | `SmallVector<char, N>` with string operations |
| `DenseMap<K, V>` | `DenseMap.h` | Open-addressing hash map (probing, not chaining); keys and values stored in parallel arrays |
| `SmallPtrSet<T>` | `SmallPtrSet.h` | Pointer-sized-element hash set with inline storage |
| `StringMap<T>` | `StringMap.h` | Hash map with interned string keys (no separate `std::string` allocation) |
| `StringRef` | `StringRef.h` | Borrowed string (pointer + length); avoids copies |
| `ArrayRef<T>` | `ArrayRef.h` | Borrowed array slice; LLVM's version of `std::span` |
| `Twine` | `Twine.h` | Lazy string concatenation; a rope-like structure avoiding intermediate allocations |
| `ilist<T>` | `ilist.h` | Intrusive linked list (the list pointer lives inside the element, not in a node wrapper) |
| `BitVector` | `BitVector.h` | Dense bit vector; used in data-flow analysis for gen/kill sets |
| `SparseBitVector` | `SparseBitVector.h` | Space-efficient bit vector for large-but-sparse sets |
| `SmallBitVector` | `SmallBitVector.h` | Tiny bit vector (≤ pointer size) with inline storage |
| `GraphTraits<T>` | `GraphTraits.h` | Template-based generic graph interface; allows any type to delegate child-iteration |
| `MapVector<K, V>` | `MapVector.h` | Hash map that preserves insertion order |
| `IntervalMap<K, V>` | `IntervalMap.h` | Immutable/CoW interval tree for register allocation (live intervals) |
| `PriorityWorkList` | `PriorityWorklist.h` | Worklist with deduplication and optional equality filter |
| `FoldingSet` | `FoldingSet.h` | Uniquing hash set (insert-or-get); used for CSE, node dedup in SelectionDAG |
| `APInt` | `APInt.h` | Arbitrary-precision integer (used for constant folding, data layout queries) |
| `APFloat` | `APFloat.h` | Arbitrary-precision (IEEE 754) float; supports all rounding modes |

**Compiler-science context**: The choice of data structures in a compiler
has enormous impact on performance.  `DenseMap` is an open-addressing hash
table (probing) — the literature (Cormen, Leiserson, Rivest) shows that
open addressing with a good hash function has better cache locality than
separate chaining, and LLVM's use of `SmallVector` for inline storage is
informed by **empirical studies of compiler heap allocation patterns**
(D. Knuth, "Empirical Study of FORTRAN Programs", 1971; Appel, 1989).

`ilist<T>` is motivated by the **use-def chain**: each `Value` needs a list
of `Use`s, and each `Use` is inside the *user* — so the list node *must*
be embedded inside the element rather than allocated separately.  This is
the classic **intrusive container** pattern that appears in many low-level
systems (Linux kernel `list_head`, BSD `LIST_ENTRY`).

`GraphTraits` enables generic graph algorithms (DFS, BFS, SCC, topological sort)
to operate on any structure — `BasicBlock*` with predecessors/successors,
`MachineBasicBlock*`, call graphs, dominator trees, etc. — via C++ template
specialization.  This is conceptually a **typeclass** (Haskell) or **concept**
(C++20) applied to compile-time polymorphism.

---

## LLVMCore (`lib/IR/`)

**CMake target**: `LLVMCore`

This is the **single most important** library in LLVM.  It defines the LLVM
Intermediate Representation — the language that flows between frontends,
optimizations, and backends.

### The Type System

LLVM IR is **strongly and statically typed**.  Types are first-class values
represented by `llvm::Type` subclasses:

| Type | Description |
|------|-------------|
| `IntegerType` | `i1`, `i8`, `i32`, `i64`, ... (arbitrary bit-width) |
| `FunctionType` | Return type + parameter types; first-class (you can have function pointers) |
| `PointerType` | Opaque pointer in modern LLVM (no element type); `ptr` |
| `ArrayType` | `[N x T]` — fixed-size array |
| `StructType` | `{T1, T2, ...}` — may be named, literal, packed, or opaque |
| `VectorType` | `<N x T>` — SIMD vector (fixed size); scalable via `<vscale x N x T>` |
| `TargetExtType` | Target-specific types (e.g., AMDGPU, DirectX) |

### The Value Hierarchy

Every entity in LLVM IR that can appear as an operand to an instruction
is a subclass of `llvm::Value`:

```
Value
├── Argument          (function parameters)
├── BasicBlock        (label type — can be a phi operand)
├── Constant
│   ├── ConstantInt, ConstantFP, ConstantPointerNull, UndefValue, PoisonValue
│   ├── ConstantDataArray, ConstantDataVector, ConstantAggregateZero
│   ├── ConstantExpr        (constant-folded expressions like getelementptr, bitcast)
│   ├── GlobalValue
│   │   ├── GlobalObject
│   │   │   ├── GlobalVariable     (global @var)
│   │   │   ├── Function           (the main unit of code)
│   │   │   └── GlobalAlias
│   │   └── GlobalIFunc            (ifunc — resolver-based aliasing)
│   ├── BlockAddress
│   ├── DSOLocalEquivalent
│   └── NoCFIValue
├── InlineAsm          (inline assembly string)
├── Instruction        (the workhorse — see below)
├── MetadataAsValue    (metadata used as a value operand)
└── Operator           (unary/binary/cmp/cast/getelementptr constant expressions)
```

### Instructions & Terminators

Every basic block ends with a **terminator** instruction; all other
instructions are **non-terminators**:

| Category | Instructions | Context |
|----------|-------------|---------|
| **Terminators** | `ret`, `br`, `switch`, `indirectbr`, `invoke`, `callbr`, `resume`, `catchswitch`, `catchret`, `cleanupret`, `unreachable` | Control flow |
| **Binary ops** | `add`, `fadd`, `sub`, `fsub`, `mul`, `fmul`, `udiv`, `sdiv`, `fdiv`, `urem`, `srem`, `frem`, `shl`, `lshr`, `ashr`, `and`, `or`, `xor` | Arithmetic and logic |
| **Bitwise ops** | `shl`, `lshr`, `ashr`, `and`, `or`, `xor` | Bit manipulation |
| **Memory ops** | `alloca`, `load`, `store`, `getelementptr`, `fence`, `cmpxchg`, `atomicrmw` | Stack, heap, atomic memory |
| **Conversion** | `trunc`, `zext`, `sext`, `fptrunc`, `fpext`, `fptoui`, `fptosi`, `uitofp`, `sitofp`, `ptrtoint`, `inttoptr`, `bitcast`, `addrspacecast` | Type conversions |
| **Other** | `icmp`, `fcmp`, `phi`, `select`, `freeze`, `call`, `va_arg`, `landingpad`, `catchpad`, `cleanuppad` | Comparisons, φ-nodes, calls, exceptions |

**Compiler-science context**: SSA form requires **φ (phi) nodes** at control-flow
merge points — each φ selects a value depending on which predecessor control
arrived from.  This comes directly from Cytron et al. (1991).  The
`getelementptr` (GEP) instruction is LLVM's abstraction for address arithmetic;
it decomposes pointer arithmetic into a statically analyzable form, enabling
alias analysis to reason about disjointness.

### The Module Structure

```
Module
 ├── Global Variables (global values)
 ├── Functions
 │   ├── Arguments
 │   └── BasicBlocks
 │       ├── PHINodes (at entry)
 │       └── Instructions
 │           └── (terminator at end)
 ├── Aliases
 ├── IFuncs
 ├── Named Metadata
 └── Comdat groups
```

A `Module` represents a translation unit.  The `Function` is the fundamental
unit of optimization (function passes).  `BasicBlock` is a straight-line
sequence of instructions terminated by a branch/return/etc. — exactly the
**basic block** concept from the Dragon Book (Aho, Ullman, 1977) and from
Frances Allen's seminal paper "Control Flow Analysis" (1970).

### The Use-Def Chain & SSA

Every `Value` carries a doubly-linked list of `Use` objects.  Each `Use` is
owned by the *user* (the instruction/constant that references the value).
This enables:

- **RAUW** (`replaceAllUsesWith`) — O(uses) replacement
- **Use-list order** — used by bitcode serialization for determinism
- **Value handles** — `WeakVH`, `WeakTrackingVH`, `AssertingVH`, `CallbackVH`
  that detect when a value is deleted or RAUW'd

### DataLayout

`DataLayout` encodes the target ABI:
- Pointer size, alignment, address spaces
- Endianness (little/big)
- Type sizes and alignments (integer, float, vector, aggregate)
- Mangling conventions

This is critical for constant folding: `getelementptr` needs to know the size
of each component type to compute byte offsets.

---

## LLVMBinaryFormat (`lib/BinaryFormat/`)

**CMake target**: `LLVMBinaryFormat`

Defines the shared **enumerations and constants** for the five binary file
formats that LLVM supports:

| Format | Headers | Used By |
|--------|---------|---------|
| **ELF** | `ELF.h` | Linux, BSD, Solaris, Fuchsia, many embedded |
| **MachO** | `MachO.h` | macOS, iOS, watchOS, tvOS, visionOS |
| **COFF** | `COFF.h` | Windows, UEFI |
| **XCOFF** | `XCOFF.h` | AIX |
| **GOFF** | `GOFF.h` | z/OS (IBM mainframe) |
| **Wasm** | `Wasm.h` | WebAssembly |
| **DXContainer** | `DXContainer.h` | DirectX shader container |
| **Minidump** | `Minidump.h` | Windows minidump (used by llvm-symbolizer) |
| **MsgPack** | `MsgPack.h` | MessagePack (used by some debug-info tools) |

Each header defines the magic numbers, section types, relocation types,
symbol flags, etc., as C++ `enum` values.  This is the lowest-friction way
for `LLVMMC` (which writes these formats) and `LLVMObject` (which reads
them) to share constants — every format constant lives exactly once in the
entire codebase.

**Compiler-science context**: Binary formats sit at the interface between the
compiler and the linker/loader.  Understanding them requires knowledge of
**linking theory** (Levine's *Linkers and Loaders*, 1999) and **object-file
format design** (the history of a.out → COFF → ELF → MachO reflects different
design philosophies around symbol resolution, relocation, and dynamic linking).

---

## LLVMTargetParser (`lib/TargetParser/`)

**CMake target**: `LLVMTargetParser`

Provides target-specific **string parsing and feature tables**:

| Subsystem | Description |
|-----------|-------------|
| `ARMTargetParser.h` | ARM CPU/arch/FPU/feature strings, NEON extensions |
| `AArch64TargetParser.h` | AArch64 architecture versions, extensions (SVE, SME, ...) |
| `RISCVTargetParser.h` | RISC-V ISA string parser (e.g., `rv64i2p1_m2p0_a2p1`) |
| `X86TargetParser.h` | x86 CPU vendor detection, feature strings |
| `HexagonTargetParser.h` | Hexagon feature strings |
| `LoongArchTargetParser.h` | LoongArch ISA feature parsing |
| `CSKYTargetParser.h` | CSKY architecture strings |
| `Targets/Builtins/*.def` | Builtin name tables (used by Clang's `__builtin_*`) |

This library sits *below* the actual target backends — it has no dependency
on `LLVMCodeGen` or `LLVMTarget`.  This separation allows tools like `llvm-nm`
and `llvm-objdump` to parse target triples and display architecture-specific
information without pulling in the entire code generator.

**Compiler-science context**: Target triples (`x86_64-unknown-linux-gnu`,
`aarch64-apple-darwin23.0`) are a compact encoding of the target ABI —
originating in GNU Autotools (config.guess / config.sub).

---

## LLVMAsmParser (`lib/AsmParser/`)

**CMake target**: `LLVMAsmParser`

Parses **textual LLVM IR** (`.ll` files) into an in-memory `Module`.
The parser is handwritten recursive-descent; the grammar is documented at
`llvm/docs/LangRef.rst`.  Key functions:

- `parseInstruction()` — recognizes opcodes and builds `Instruction` objects
- `parseType()` — parses LLVM's type syntax (e.g., `<4 x i32>`)
- `parseConstantValue()` — parses constant expressions and aggregates

The parser uses the `LLLexer` (defined in `AsmParser/`), which tokenizes the
LLVM IR text format.

---

## LLVMIRReader (`lib/IRReader/`)

**CMake target**: `LLVMIRReader`

Thin convenience library that wraps `LLVMAsmParser` and `LLVMBitcode`:
given a `MemoryBuffer`, it detects the format (text or bitcode) and builds
a `Module`.  Used by almost every LLVM tool that reads IR.

**Compiler-science context**: The ability to parse from both text and binary
is a classic compiler engineering lesson — text is for humans and rapid
prototyping; binary is for speed and compactness.  The LLVM IR format is
the **lingua franca** of the entire toolchain, analogous to Java bytecode
for the JVM ecosystem or CIL for .NET.

---

## LLVMIRPrinter (`lib/IRPrinter/`)

**CMake target**: `LLVMIRPrinter`

Thin library for printing textual LLVM IR — wraps `Module::print()`,
`Function::print()`, and the DOT-language printer for call graphs and CFG
visualization.

Notable: the `DOTGraphTraits` subsystem uses `GraphTraits` to make any
graph printable in Graphviz format — this is used heavily for debugging
passes (e.g., `opt -dot-cfg foo.ll`).
