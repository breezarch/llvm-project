# 11 — Tools Reference

LLVM ships with **100+ command-line tools** under `llvm/tools/`.  This chapter
catalogs them grouped by function.  Each tool is a thin driver that
composes LLVM libraries — typically a few hundred to a few thousand lines
of C++ that set up the right library configuration and then delegate.

---

## Category 1: Core Compilation & IR Processing

| Tool | Purpose |
|------|---------|
| **`opt`** | The **modular optimizer** — reads LLVM IR (`.ll` or `.bc`), runs a specified sequence of passes, and writes the result. ``opt`` is the primary testing/debugging tool for optimization passes. Supports both the legacy pass manager (``-instcombine``, ``-gvn``) and the new pass manager (``-passes="instcombine,gvn"``). |
| **`llc`** | The **LLVM static compiler** — takes LLVM IR and generates assembly (`.s`) or object (`.o`) files. Communicates with the target backend through ``TargetMachine``.  ``llc`` is the command-line equivalent of the codegen half of Clang. |
| **`lli`** | The **LLVM interpreter / JIT executor** — directly executes LLVM IR (bitcode or assembly) using either the interpreter or a JIT compiler. Supports both MCJIT (legacy) and Orc (modern, via `--jit-kind=orc`). |
| **`llubi`** | **LLVM UB-aware interpreter** — interprets LLVM IR with undefined-behavior detection, catching UB at execution time rather than relying on compile-time diagnostics. |
| **`llvm-as`** | **LLVM assembler** — converts textual LLVM IR (`.ll`) to bitcode (`.bc`).  Essentially a thin wrapper around ``LLVMAsmParser`` + ``LLVMBitcode``. |
| **`llvm-dis`** | **LLVM disassembler** — converts bitcode (`.bc`) to textual LLVM IR (`.ll`).  A thin wrapper around `LLVMBitcode` + `LLVMIRPrinter`. |
| **`llvm-link`** | **LLVM IR linker** — links two or more bitcode/IR files into a single bitcode file.  Delegates to `LLVMLinker::linkModules()`. |
| **`llvm-extract`** | Extracts a single function (or global) from a bitcode file into a new bitcode file.  Useful for isolating test cases. |
| **`llvm-split`** | Splits a bitcode file with multiple functions into separate bitcode files (one per function).  The inverse of `llvm-link`. |
| **`llvm-diff`** | **Semantic diff** for LLVM IR — determines if two modules are structurally equivalent (ignoring names, comparing instructions/operands). |
| **`llvm-stress`** | Randomly generates "interesting" LLVM IR — used for fuzz-testing the optimizer and code generator. |
| **`llvm-isel-fuzzer`** | Fuzzes the instruction selector — generates random IR, runs instruction selection, and checks for crashes. |
| **`llvm-opt-fuzzer`** | Fuzzes the optimizer — generates random IR sequences and runs optimization passes on them. |
| **`llvm-cat`** | Concatenates bitcode files (like Unix ``cat`` but for bitcode). |
| **`llvm-modextract`** | Extracts a specific embedded module from a multi-module bitcode file. |
| **`verify-uselistorder`** | Verifies that use-list order is preserved across bitcode round-trips (critical for deterministic compilation). |

---

## Category 2: Object File Utilities (Binary-Level)

These tools are analogs of the GNU Binutils (`nm`, `objdump`, `ar`, etc.)
but using LLVM's libraries instead of BFD/libiberty.

| Tool | GNU Equivalent | Purpose |
|------|---------------|---------|
| **`llvm-nm`** | `nm` | Lists symbols from object files, archives, and bitcode files |
| **`llvm-objdump`** | `objdump` | Disassembles and dumps object files (headers, sections, relocations, symbols) |
| **`llvm-readobj`** / **`llvm-readelf`** | `readelf` | Detailed dump of object file structure in LLVM-specific or GNU-compatible formats |
| **`llvm-objcopy`** | `objcopy` | Copies, strips, and transforms object files (add/remove sections, rename symbols, ...) |
| **`llvm-size`** | `size` | Prints section sizes and total size of object/executable files |
| **`llvm-strings`** | `strings` | Extracts printable strings from binary files |
| **`llvm-ar`** | `ar` | Creates and manipulates Unix archives (`.a` files) |
| **`llvm-libtool-darwin`** | `libtool` (Apple) | Creates static libraries on Darwin/macOS |
| **`llvm-lipo`** | `lipo` (Apple) | Creates and manipulates fat (universal) binaries on macOS |
| **`llvm-ranlib`** | `ranlib` | Generates symbol index for archives (symlink to `llvm-ar`) |
| **`llvm-strip`** | `strip` | Removes symbols and debug info (symlink to `llvm-objcopy`) |
| **`llvm-addr2line`** | `addr2line` | Converts addresses to source locations (symlink to `llvm-symbolizer`) |
| **`obj2yaml`** | — | Converts binary object files (`.o`) to human-readable YAML for inspection and testing.  The inverse of ``yaml2obj``. |
| **`yaml2obj`** | — | Converts YAML descriptions to binary object files.  Used extensively in LLVM's test suite — tests describe expected object files in YAML rather than checking in opaque binaries. |

---

## Category 3: Assembly & Disassembly

| Tool | Purpose |
|------|---------|
| **`llvm-mc`** | **LLVM Machine Code** toolkit — assemble (`.s` → `.o`), disassemble (`.o` → `.s`), and analyze machine code. Also provides an interactive assembly playground. |
| **`llvm-mca`** | **LLVM Machine Code Analyzer** — simulate the pipeline of an assembly snippet and report throughput, latency, and resource usage. Uses `LLVMMCA` to model the CPU pipeline. |
| **`llvm-ml`** | **MASM-compatible assembler** — assembles Microsoft Macro Assembler (MASM) syntax for x86/x86-64 on Windows. |
| **`llvm-mc-assemble-fuzzer`** | Fuzzes the MC assembler — generates random assembly and tries to assemble it. |
| **`llvm-mc-disassemble-fuzzer`** | Fuzzes the MC disassembler — generates random bytes and tries to disassemble them. |

---

## Category 4: Debug Information

| Tool | Purpose |
|------|---------|
| **`llvm-dwarfdump`** | Pretty-prints DWARF debug info from object files (DIE tree, line tables, CFI, etc.) |
| **`llvm-symbolizer`** | Converts raw addresses to ``function_name + offset`` and ``file:line:col`` using debug info. Essential for sanitizer error reports. |
| **`llvm-debuginfo-analyzer`** | Analyzes and reports on debug-info quality (coverage of variables, types, scopes). Uses the `LogicalView` library. |
| **`llvm-pdbutil`** | PDB (Program Database) utility — dump, analyze, and merge Microsoft PDB files. |
| **`llvm-gsymutil`** | Converts DWARF → GSYM (Google Symbol Format) — compact debug info for production symbolization. |
| **`dsymutil`** | **DWARF Symbol Utility** — links DWARF from `.o` files into a `.dSYM` bundle for macOS. Uses `DWARFLinker`. |
| **`llvm-dwarfutil`** | Debug-info utility — optimize and transform DWARF (deduplicate, gc, etc.) |
| **`llvm-dwp`** | DWARF Package tool — merges multiple DWARF files into a split-DWARF package (`.dwp`). |
| **`llvm-debuginfod`** / **`llvm-debuginfod-find`** | Debug info over HTTP — server/client for the `debuginfod` protocol. |

---

## Category 5: Profiling & Coverage

| Tool | Purpose |
|------|---------|
| **`llvm-profdata`** | **Profile data tool** — merges, indexes, and inspects profile data (`.profraw` → `.profdata`). |
| **`llvm-profgen`** | **Profile generation** — converts hardware performance-counter samples (from ``perf``) into LLVM's profile format (AutoFDO). |
| **`llvm-cov`** | **Coverage tool** — generates HTML/console coverage reports from LLVM's source-based coverage. Combines coverage mapping (embedded in the binary) with profile data. |
| **`llvm-opt-report`** | Generates optimization reports (which loops were vectorized, which inlines happened, etc.) from optimization remarks. |
| **`llvm-remarkutil`** | Utility for processing optimization remark files (merge, annotate, filter). |
| **`sancov`** | **Sanitizer Coverage** — processes coverage data from sanitizer-instrumented binaries. |
| **`sanstats`** | **Sanitizer Statistics** — extracts runtime statistics from sanitizer-instrumented binaries. |
| **`opt-viewer`** | Python tool (not compiled C++) that generates an HTML report from optimization remarks — shows optimization decisions overlaid on source code. |
| **`llvm-ctxprof-util`** | **Context profiling utility** — converts context-sensitive profiling data. |
| **`llvm-xray`** | **XRay trace processing** — converts, analyzes, and visualizes XRay function-call traces. |

---

## Category 6: Link-Time Optimization

| Tool | Purpose |
|------|---------|
| **`llvm-lto`** | **Full LTO** driver — runs the LTO pipeline; merges IR modules, runs whole-program optimization, and produces optimized objects. |
| **`llvm-lto2`** | Modern LTO testing tool — uses the resolution-based LTO API (newer than `llvm-lto`). |
| **`lto`** (shared library) | ``libLTO.so`` / ``libLTO.dylib`` — the shared library that linkers (``ld.lld``, GNU ``ld`` via plugin, Apple ``ld64``) load to perform LTO. |

---

## Category 7: Binary Utilities

| Tool | Purpose |
|------|---------|
| **`llvm-cvtres`** | Converts Microsoft resource files (`.res`) to COFF object files. Used for Windows resource compilation. |
| **`llvm-mt`** | **Manifest Tool** — merges and processes Windows side-by-side assembly manifests (`.manifest` files). |
| **`llvm-rc`** | Windows **Resource Compiler** — compiles `.rc` resource scripts into `.res` files. |
| **`llvm-cxxfilt`** | **C++ name demangler** — decodes mangled names (Itanium, Microsoft, Rust, D) to readable form. Delegates to ``LLVMDemangle``. |
| **`llvm-undname`** | **MSVC name demangler** — specifically for Microsoft Visual C++ mangled names. |
| **`llvm-cxxmap`** | Symbol mapping tool — creates a mapping from old symbol names to new symbol names (useful for ABI migration). |
| **`llvm-cxxdump`** | Dumps C++ ABI data structures from shared libraries (VTable layouts, RTTI data). |

---

## Category 8: GPU & Offloading

| Tool | Purpose |
|------|---------|
| **`llvm-ifs`** | **Interface Stub** tool — creates ELF interface stubs (stub libraries for Android/Fuchsia style shared libraries). |
| **`llvm-gpu-loader`** | GPU offloading loader — loads GPU kernels from the host binary at runtime. |
| **`dxil-dis`** | **DXIL Disassembler** — converts DXIL shaders (DirectX Intermediate Language) to human-readable LLVM IR. |
| **`llvm-offload-binary`** | Processes offload binary bundles — packs/unpacks device code embedded in host binaries. |
| **`llvm-offload-wrapper`** | Wraps offloaded device code into a host object file. |
| **`spirv-tools`** (wrapper) | Wraps SPIR-V tools (from the SPIRV-Tools project) for integration with LLVM's SPIR-V support. |

---

## Category 9: Development, Testing & Infrastructure

| Tool | Purpose |
|------|---------|
| **`llvm-config`** | Queries the LLVM build configuration — provides include paths, library paths, and library dependency lists (``--ldflags``, ``--libs``, ``--system-libs``). Essential for third-party code linking against LLVM. |
| **`llvm-bcanalyzer`** | **Bitcode analyzer** — prints statistics about a bitcode file (block/record sizes, compression ratios). |
| **`llvm-c-test`** | Tests the LLVM C API — verifies that the C bindings work correctly. |
| **`llvm-driver`** | Multiplexer tool — a single binary that dispatches to other LLVM tools based on the invocation name (like BusyBox). |
| **`llvm-shlib`** | Builds `libLLVM.so` — a single shared library containing all LLVM components. |
| **`remarks-shlib`** | Builds `libRemarks.so` — a standalone shared library for reading optimization remarks (useful for tools that only need remark parsing). |
| **`llvm-cfi-verify`** | Verifies Control Flow Integrity in compiled binaries — checks that indirect calls/jumps are properly guarded. |
| **`llvm-tli-checker`** | **Target Library Info checker** — verifies that a target library correctly implements the standard C library functions that LLVM assumes. |
| **`llvm-reduce`** | **Test-case reducer** (LLVM's `delta` or `creduce`) — automatically reduces large crashing IR test cases to minimal reproductions. Uses a hill-climbing algorithm to remove instructions/blocks/arguments. |
| **`reduce-chunk-list`** | Helper for `llvm-reduce` — manages the chunk-list reduction strategy. |
| **`llvm-cas`** / **`llvm-cas-fuzzer`** | CAS (Content-Addressed Storage) utilities — store and retrieve objects by content hash. |
| **`llvm-cgdata`** | **CodeGenData** tool — merges and inspects code-generation profiling data (new infrastructure). |
| **`llvm-readtapi`** | Reads and pretty-prints TBD (Text-Based Dylib) files. |
| **`llvm-sim`** | **Static Inliner Model** — evaluates ML-based inlining models against real programs. |

---

## Category 10: Fuzzing Harnesses

These are specialized fuzzers, each targeting a specific LLVM component:

| Tool | Target |
|------|--------|
| **`llvm-as-fuzzer`** | Bitcode writer (module → bitcode serialization) |
| **`llvm-dis-fuzzer`** | Bitcode reader (bitcode → module deserialization) |
| **`llvm-itanium-demangle-fuzzer`** | Itanium C++ ABI demangler |
| **`llvm-microsoft-demangle-fuzzer`** | Microsoft VC++ demangler |
| **`llvm-rust-demangle-fuzzer`** | Rust symbol demangler |
| **`llvm-dlang-demangle-fuzzer`** | D language demangler |
| **`llvm-special-case-list-fuzzer`** | Sanitizer special-case-list parser |
| **`llvm-yaml-parser-fuzzer`** | YAML parser (used by ObjectYAML) |
| **`llvm-yaml-numeric-parser-fuzzer`** | Numeric parsing in YAML |
| **`llvm-opt-fuzzer`** | Optimizer (random IR → pass pipeline) |
| **`llvm-isel-fuzzer`** | Instruction selector |
| **`llvm-mc-assemble-fuzzer`** | MC assembler |
| **`llvm-mc-disassemble-fuzzer`** | MC disassembler |
| **`vfabi-demangle-fuzzer`** | Vector Function ABI demangler |

---

## Category 11: Platform-Specific

### macOS / Darwin

| Tool | Purpose |
|------|---------|
| **`llvm-libtool-darwin`** | Darwin static library tool |
| **`llvm-lipo`** | Universal binary manipulation |
| **`dsymutil`** | dSYM bundle creation |

### Windows

| Tool | Purpose |
|------|---------|
| **`llvm-ml`** | MASM assembler |
| **`llvm-mt`** | Manifest tool |
| **`llvm-rc`** | Resource compiler |
| **`llvm-cvtres`** | Resource → COFF converter |
| **`llvm-undname`** | MSVC name demangler |

### Plugin Loader

| Tool | Purpose |
|------|---------|
| **`llvm-jitlistener`** | Tool for testing/benchmarking Intel JIT event listeners |
| **`gold`** | LLVM Gold linker plugin (`LLVMgold.so`) — enables LTO when using the GNU Gold linker.  Uses ``LLVMLTO`` to provide the ``onload`` plugin interface that Gold expects. |

### Developer Infrastructure

| Tool | Purpose |
|------|---------|
| **`llvm-config`** | Build configuration queries |
| **`xcode-toolchain`** | Creates Xcode toolchain bundles for macOS development |

### Exegesis

| Tool | Purpose |
|------|---------|
| **`llvm-exegesis`** | **Microbenchmark** tool — measures instruction latency, throughput, and port pressure on real hardware. Helps validate and improve the scheduling models used by the instruction scheduler and MCA. |

### Misc

| Tool | Purpose |
|------|---------|
| **`llvm-ir2vec`** | Machine learning utility — converts LLVM IR to vector representations for ML-based compiler research. |
| **`llvm-rtdyld`** | **Runtime Dynamic Linker** test tool — tests the RuntimeDyld library's relocation handling. |
| **`llvm-jitlink`** | Tests the JITLink library — loads and links an object file into memory and executes it. |
| **`llvm-yaml-parser-fuzzer`** | Tests YAML parsing used by ObjectYAML. |
