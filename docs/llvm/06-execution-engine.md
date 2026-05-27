# 06 — Execution Engine

This chapter covers LLVM's **JIT compilation** and **execution** libraries.
These allow LLVM IR to be compiled to native code at runtime and executed
directly — the foundation for dynamic languages, REPLs, and online code
generation.

---

## The Execution Engine Architecture

```
LLVM IR (Module)
        │
        ▼
┌──────────────────────────────────────────────────────┐
│  ExecutionEngine (abstract base)                      │
│  &#8226; Manages module ownership                         │
│  &#8226; Provides getPointerToFunction(), runFunction()     │
│  &#8226; Handles global variable mapping                   │
│  &#8226; Supports lazy compilation (stub-based)            │
├──────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │ Interpreter  │  │   MCJIT     │  │     Orc      │ │
│  │ (walk IR)   │  │ (MC-layer   │  │ (modern JIT  │ │
│  │              │  │  JIT)       │  │  API)        │ │
│  └─────────────┘  └─────────────┘  └──────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## LLVMExecutionEngine (`lib/ExecutionEngine/`)

**CMake target**: `LLVMExecutionEngine`

The abstract base class and common infrastructure for execution engines.

### Core Responsibilities

- **Module management**: owns one or more `Module`s; handles global
  variable mapping (linking JIT globals to host addresses)
- **Function lookup**: `getPointerToFunction()` — returns a function
  pointer callable from C/C++
- **Lazy compilation**: `getPointerToFunctionOrStub()` — if the function
  hasn't been compiled yet, returns a stub that triggers compilation
  on first call
- **Global mapping**: `addGlobalMapping()` — tell the JIT that global
  `@foo` lives at address `0x1234`
- **Exception handling**: `registerEHFrames()` — register DWARF/PDB
  unwind tables for C++ exception support in JIT'd code

### Use Cases

JIT compilation is used for:
- **Dynamic language VMs** (JavaScript, Python, Ruby, Lua)
- **Database query compilation** (e.g., PostgreSQL JIT with LLVM)
- **GPU kernel compilation** (OpenCL/CUDA JIT)
- **REPL environments** (Clang-Repl, Julia REPL)
- **Expression evaluation** in debuggers (lldb expression evaluator)
- **Offline partial evaluation** (specializing code for known inputs)

---

## Interpreter (`lib/ExecutionEngine/Interpreter/`)

**The simplest execution model.**  Instead of compiling to machine code,
the interpreter **walks the IR** and performs each operation:

- Each `Instruction` is dispatched to a C++ handler (e.g., `visitAdd()`,
  `visitLoad()`, `visitCall()`)
- Operands are represented as `GenericValue` — a union of int/float/pointer
- Memory is emulated via a flat address space (no actual MMU)
- External functions can be registered and called via FFI

The interpreter is primarily useful for:
- **Testing** (verifying that optimizations don't change program behavior)
- **Bootstrapping** (running optimization passes during LLVM's own build)
- **Education** (an interpreter is easier to understand than a JIT)

**Theoretical context**: Interpretation is the dual of compilation —
where a compiler translates a program to machine code, an interpreter
walks the AST/IR and performs the computation directly.  The tradeoffs
(no compilation latency vs. slower steady-state performance) are well
studied (Aho & Ullman, 1977).

---

## MCJIT (`lib/ExecutionEngine/MCJIT/`)

**The "MC-layer" JIT** — uses the full MC infrastructure (assembler,
object writer) to JIT-compile functions into native code.

MCJIT works by:
1. Taking a `Module`
2. Running the code generation pipeline on each function (instruction
   selection, register allocation, etc.)
3. Emitting the result as an in-memory object file
4. Loading the object file into the current process (via `RuntimeDyld`)

This means MCJIT produces **the same code quality** as the offline
compiler (`llc`) — all optimization passes, all codegen passes.
The cost is latency (full compilation per module).

**Key design**: MCJIT is **module-at-a-time** — you must finalize a
module before calling any functions in it.  This is a limitation compared
to Orc (see below).

---

## RuntimeDyld (`lib/ExecutionEngine/RuntimeDyld/`)

**CMake target**: `LLVMRuntimeDyld`

A minimal in-memory linker:
- Takes the object-file image produced by MCJIT
- Performs **relocations** (fixes up addresses based on runtime layout)
- Resolves **external symbols** (linking JIT'd code against host symbols)
- Manages **memory permissions** (makes code pages executable via `mmap`
  or `VirtualAlloc`)
- Supports lazy compilation via **stub generation**

RuntimeDyld handles all five binary formats (ELF, MachO, COFF, XCOFF,
GOFF relocations).  This is essentially a **linker-as-a-library**.

---

## Orc (`lib/ExecutionEngine/Orc/`)

**CMake target**: `LLVMOrcJIT`

**Orc** (On-Request Compilation) is the modern JIT API — a complete
rethink of LLVM's JIT infrastructure, providing finer granularity,
better concurrency, and more flexibility than MCJIT.

### Orc Architecture

Orc is organized around **layers** — composable abstractions that form
a compilation pipeline:

```
┌───────────────────────────────────────────────┐
│  JITDylib — a "dynamic library" in the JIT    │
│  (holds symbols, like a .so or .dylib)        │
├───────────────────────────────────────────────┤
│  MaterializationUnit — a thing that can        │
│  produce symbol definitions (lazy or eager)    │
├───────────────────────────────────────────────┤
│  IRLayer — compiles IR → executable code       │
│  ┌─────────────────────────────────────────┐  │
│  │ IRCompileLayer: Module → MCJIT/llc      │  │
│  │ IRTransformLayer: Module → transform →  │  │
│  │                    compile              │  │
│  │ CompileOnDemandLayer: lazy per-function  │  │
│  │ ObjectLinkingLayer: object file → linked │  │
│  └─────────────────────────────────────────┘  │
├───────────────────────────────────────────────┤
│  ExecutionSession — owns all JITDylibs,        │
│  provides lookup, manages resource trackers    │
├───────────────────────────────────────────────┤
│  JITLink — native in-memory linker (replaces   │
│  RuntimeDyld for Orc-based JITs)               │
└───────────────────────────────────────────────┘
```

### Key Orc Concepts

| Concept | Description |
|---------|-------------|
| **JITDylib** | Analogous to a shared library; has a name, a set of symbols, and can be linked against other JITDylibs |
| **MaterializationUnit** | Promise of symbol definitions; can be compiled lazily |
| **SymbolStringPtr** | A pooled interned string — efficient symbol names |
| **ResourceTracker** | Tracks memory/code allocated by a materialization; enables removing code at runtime |
| **ExecutionSession** | Owns the JIT "universe" — dispatches lookups, coordinates compilation |
| **JITLink** | The native-code linker for Orc; supports ELF, MachO, COFF (via plugins) |

### JITLink (`lib/ExecutionEngine/JITLink/`)

**CMake target**: `LLVMJITLink`

JITLink is Orc's companion in-memory linker — a more modern, extensible
replacement for `RuntimeDyld`:

- **Plugin architecture**: custom passes can be injected into the linking
  pipeline (e.g., GOT/PLT optimization, compact unwinding)
- **Relaxation support**: like MC's relaxation, but for linked code
  (e.g., stub generation when branch targets are out of range)
- **EH-frame registration**: DWARF unwinding for JIT'd code

### Advantages Over MCJIT

| Feature | MCJIT | Orc |
|---------|-------|-----|
| Compilation granularity | Module-at-a-time | Function-at-a-time (lazy) |
| Code removal | Not supported | Yes (via ResourceTracker) |
| Concurrent compilation | No | Yes (concurrent compilation with lazy resolution) |
| Re-optimization | No (finalized) | Yes (re-compile with higher opt level) |
| Customizability | Limited | High (layered architecture) |
| Partitioning | Huge monolithic | Fine-grained (each JITDylib can be separately managed) |

### LLLazyJIT

Orc's **lazy compilation** feature works via **compile callbacks**:
1. Each function initially has a **stub** (a small trampoline)
2. When the function is first called, the stub invokes the compiler
3. The compiler produces the actual function body
4. The stub is **replaced** with the compiled code (self-modifying or
   via indirection)
5. Subsequent calls go directly to the compiled code

This is the same mechanism used by the JVM HotSpot compiler, V8's
Ignition/TurboFan, and LuaJIT's tracing JIT.

---

## Profiling & Debugging Support for JITs

### PerfJITEvents (`lib/ExecutionEngine/PerfJITEvents/`)

Emits JIT code-location information to Linux `perf` — enables `perf
record` to symbolicate JIT'd functions.  Essential for profiling
JIT-compiled workloads.

### IntelJITEvents (`lib/ExecutionEngine/IntelJITEvents/`)

Intel VTune integration for JIT'd code — sends function load/unload
events to the VTune profiler.

### IntelJITProfiling (`lib/ExecutionEngine/IntelJITProfiling/`)

Legacy Intel JIT profiling API integration.

### OProfileJIT (`lib/ExecutionEngine/OProfileJIT/`)

OProfile integration for JIT'd code — sends symbol information to the
OProfile daemon.

---

## Compiler-Science Context

### JIT Compilation Theory

JIT compilation sits at the intersection of **compiler construction** and
**runtime systems**.  The classic reference is Aycock (2003) "A Brief
History of Just-In-Time" (ACM Computing Surveys), which traces the idea
from Lisp (McCarthy, 1960) through Smalltalk (Deutsch & Schiffman, 1984)
to Java HotSpot.

The fundamental JIT tradeoff is **compilation time vs. code quality**:

- **Warm-up**: time spent compiling before peak performance is reached
- **Peak performance**: how fast the JIT'd code runs at steady state
- **Code bloat**: memory consumed by compiled code, profiling metadata,
  and intermediate representations

Orc's layered architecture allows different strategies:
- **Compile quick, run slow** (e.g., -O0 for cold functions)
- **Compile slow, run fast** (e.g., -O2 for hot functions, identified
  by profiling)
- **Tiered compilation** (multiple layers of increasing optimization,
  with on-stack replacement between tiers)

### Relation to Tracing JITs

LLVM's JIT is a **method-based JIT** (compile whole functions at a time).
This contrasts with **tracing JITs** (trace-based, like LuaJIT and early
TraceMonkey/Firefox) that compile linear execution traces through multiple
functions.

Method-based JITs are simpler to implement (no trace selection, no
trace linking) and integrate better with existing AOT compilation
infrastructure.  The downside is that they may compile code that is
never executed (dead code, cold paths), wasting compilation time.

### Memory Management for JITs

A critical JIT implementation concern is **garbage collection of
compiled code**.  When a JIT'd function becomes unreachable (e.g., the
containing module is reloaded or the class is redefined), the
executable memory must be freed.  Orc's `ResourceTracker` provides this
via reference-counted materialization units — when a `ResourceTracker`
is destroyed, all associated code and data is deallocated.

This connects to the broader problem of **code cache management** in
dynamic language runtimes (e.g., Java's CodeCache, V8's compilation cache).
