# LLVM — Internal Architecture & Component Reference

This directory contains a comprehensive study of the LLVM project's internal
structure (the `llvm/` directory only — not Clang, MLIR, LLD, or other sibling
projects).  The documentation is written for a newcomer who wants to understand:

1. **What each CMake target / library does** — its core functionality,
   the compiler-technique or research problem it solves, and why it exists.
2. **How the pieces fit together** — the software architecture, the layer
   model, and the data-flow between components.
3. **The compiler-science context** — connections to computational theory
   (automata, graph theory, formal semantics), classic compiler papers
   (Dragon Book, Muchnick, Cooper & Torczon), and modern research (PGO,
   ThinLTO, ML-driven heuristics, fuzzing).

## Document Index

| File | Covers |
|------|--------|
| [`01-architecture-overview.md`](./01-architecture-overview.md) | Layered architecture, data-flow graph of the compilation pipeline, dependency graph of libraries |
| [`02-core-libraries.md`](./02-core-libraries.md) | `LLVMCore` (IR), `LLVMSupport`, `ADT`, `BinaryFormat`, `TargetParser`, `AsmParser`, `IRReader`/`IRPrinter` |
| [`03-analysis-and-optimization.md`](./03-analysis-and-optimization.md) | `LLVMAnalysis`, `LLVMTransform*` (Scalar, InstCombine, Vectorize, IPO, ObjCARC, Coroutines, CFGuard, HipStdPar, AggressiveInstCombine, Utils, Instrumentation), `LLVMPasses` (PassBuilder, pipelines) |
| [`04-code-generation.md`](./04-code-generation.md) | `LLVMCodeGen` (SelectionDAG, GlobalISel, MIRParser, AsmPrinter, LiveDebugValues), `LLVMMC`, `LLVMMCA`, `LLVMCodeGenTypes` |
| [`05-target-backends.md`](./05-target-backends.md) | All 25 backends: X86, AArch64, ARM, RISCV, AMDGPU, NVPTX, WebAssembly, PowerPC, SystemZ, Mips, LoongArch, SPIRV, DirectX, Hexagon, BPF, Sparc, AVR, MSP430, M68k, CSKY, ARC, Lanai, VE, XCore, Xtensa |
| [`06-execution-engine.md`](./06-execution-engine.md) | `LLVMExecutionEngine`, `Orc`, `MCJIT`, `JITLink`, `RuntimeDyld`, Interpreter, `IntelJITEvents`, `PerfJITEvents`, `OProfileJIT` |
| [`07-linking-and-lto.md`](./07-linking-and-lto.md) | `LLVMLinker`, `LLVMLTO`, `LLVMDTLTO`, `llvm-lto`, `llvm-lto2`, ThinLTO, FullLTO |
| [`08-debug-info.md`](./08-debug-info.md) | `LLVMDebugInfoDWARF`, `PDB`, `GSYM`, `BTF`, `CodeView`, `MSF`, `Symbolize`, `LogicalView`, `DWARFLinker`, `DWARFCFIChecker`, `DWP`, `Debuginfod` |
| [`09-profiling-and-instrumentation.md`](./09-profiling-and-instrumentation.md) | `LLVMProfileData` (Coverage, PGO, MemProf), `LLVMXRay`, `LLVMFuzzer`/`FuzzMutate`, Sanitizers coverage |
| [`10-bitcode-and-serialization.md`](./10-bitcode-and-serialization.md) | `LLVMBitcode`, `LLVMBitstream`, `LLVMObject`, `LLVMObjectYAML`, `LLVMRemarks`, `LLVMTextAPI` |
| [`11-tools-reference.md`](./11-tools-reference.md) | Complete catalog of every tool under `llvm/tools/`, grouped by category |
| [`12-auxiliary-libraries.md`](./12-auxiliary-libraries.md) | `LLVMDemangle`, `LLVMFileCheck`, `LLVMTableGen`, `LLVMCAS`, `LLVMSandboxIR`, `LLVMTelemetry`, `LLVMLineEditor`, `LLVMTesting`, `LLVMFrontend*`, `LLVMInterfaceStub`, `LLVMOption`, `LLVMObjCopy`, `LLVMWindowsDriver`, `LLVMWindowsManifest`, `LLVMHTTP`, `LLVMExtensions`, `LLVMPlugins`, `LLVMToolDrivers`, `LLVMABI` |

## Quick Orientation

If this is your first time reading the LLVM source tree, these are the most
"load-bearing" directories inside `llvm/`:

```
llvm/
├── include/llvm/     # Public C++ headers (one subdirectory per library)
├── lib/              # Library implementations (each dir → one CMake target)
│   ├── IR/             LLVMCore — the heart: Value, Instruction, Function, Module, etc.
│   ├── Support/        LLVMSupport — OS abstraction, I/O, threading, hashing
│   ├── Analysis/       LLVMAnalysis — alias analysis, dependence analysis, etc.
│   ├── Transforms/     LLVMTransform* — 12 optimization libraries
│   ├── CodeGen/        LLVMCodeGen — machine-level code generation
│   ├── MC/             LLVMMC — assembler, disassembler, object-file emission
│   ├── Target/         All 25 hardware/VM backends
│   ├── Passes/         LLVMPasses — new-PM pass builder & standard pipelines
│   ├── ExecutionEngine/ LLVMExecutionEngine, Orc, MCJIT, JITLink, etc.
│   ├── DebugInfo/      DWARF, PDB, GSYM, BTF, Symbolize, etc.
│   └── ...
├── tools/            # 80+ command-line tools (opt, llc, lli, llvm-mc, etc.)
├── unittests/        # GoogleTest-based unit tests (mirrors lib/ structure)
└── utils/            # Build infrastructure, scripts, utilities
```

## Building Blocks: The CMake Target Naming Convention

Almost every directory under `lib/` corresponds to a single `add_llvm_component_library`
call.  The target names follow a pattern:

- **Core libraries**: `LLVMCore`, `LLVMSupport`, `LLVMMC`, etc.
- **Transform libraries**: `LLVMInstCombine`, `LLVMScalarOpts`, `LLVMVectorize`, etc.
  (They all link into `LLVMTransform*` but are individual shared/static libraries.)
- **Target backends**: `LLVMX86`, `LLVMAArch64`, `LLVMAMDGPU`, etc.
- **Tools**: each tool produces an executable, e.g. `opt`, `llc`, `lli`.

You can see the full dependency DAG by looking at the `LINK_COMPONENTS` clause in
each `CMakeLists.txt` inside `lib/*/`.
