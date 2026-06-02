# LLVM Project — Source Code Statistics

Counted file types: `*.cpp`, `*.c`, `*.cc`, `*.cxx`, `*.h`, `*.hpp`, `*.hh`, `*.hxx`,
`*.inc`, `*.S`, `*.asm`, `*.td`, `*.mlir`, `*.py`, `*.def`.

Configuration and build files (CMakeLists.txt, `*.cmake`, `*.json`, `*.txt`, `*.md`,
etc.) are excluded. Only source code files are counted.

Date generated: 2026-05-27.

## By Subproject (sorted by lines of code)

| Subproject | Lines of Code | Source Files | Description |
|---|---|---|---|
| **llvm** | 3,438,611 | 9,660 | Core compiler infrastructure: IR, optimizer, codegen, MC, 25 targets, 100+ tools |
| **clang** | 3,227,155 | 24,911 | C/C++/Objective-C frontend: parser, semantic analysis, AST, codegen |
| **compiler-rt** | 620,551 | 4,212 | Compiler runtime libraries: sanitizers, builtins, profile, xray, fuzzer |
| **mlir** | 523,108 | 6,120 | Multi-Level Intermediate Representation framework |
| **clang-tools-extra** | 436,712 | 2,857 | Extra Clang tools: clang-tidy, clangd, clang-format, clang-apply-replacements |
| **polly** | 398,958 | 922 | Polyhedral loop optimization framework |
| **flang** | 369,738 | 954 | Fortran frontend (parser, semantics, lowering to MLIR) |
| **lldb** | 339,863 | 6,056 | LLVM debugger: expression evaluator, DWARF/PDB reader, scripting API |
| **third-party** | 243,635 | 667 | Third-party vendored libraries (benchmarks, etc.) |
| **libcxx** | 221,477 | 11,750 | C++ standard library implementation (libc++) |
| **openmp** | 170,747 | 670 | OpenMP runtime library |
| **lld** | 115,620 | 253 | LLVM linker: ELF, COFF, Mach-O, Wasm linkers |
| **bolt** | 107,119 | 326 | Binary Optimization and Layout Tool — post-link optimizer |
| **offload** | 74,991 | 636 | GPU offloading runtime (OpenMP device RTL, CUDA/HIP wrappers) |
| **libcxxabi** | 60,313 | 113 | C++ ABI library (exception handling, RTTI, dynamic_cast) |
| **libc** | 57,728 | 6,809 | C standard library implementation |
| **flang-rt** | 51,579 | 249 | Fortran runtime library |
| **libunwind** | 24,727 | 53 | Stack unwinding library |
| **cross-project-tests** | 18,758 | 244 | Cross-project integration tests |
| **libclc** | 18,767 | 481 | OpenCL C library implementation |
| **orc-rt** | 15,376 | 111 | ORC JIT runtime library |
| **libsycl** | 5,010 | 57 | SYCL runtime support library |
| **llvm-libgcc** | 129 | 1 | LLVM libgcc replacement (single source file) |
| **utils** | 593 | 5 | Miscellaneous utility scripts (source files only) |

## Grand Total

| Metric | Count |
|---|---|
| **Total lines of code** | **10,500,635** |
| **Total source files** | **77,128** |

## Top 5 by Size

| Rank | Subproject | Lines | % of Total |
|---|---|---|---|
| 1 | llvm | 3,438,611 | 32.8% |
| 2 | clang | 3,227,155 | 30.7% |
| 3 | compiler-rt | 620,551 | 5.9% |
| 4 | mlir | 523,108 | 5.0% |
| 5 | clang-tools-extra | 436,712 | 4.2% |

Together, llvm + clang account for ~63.5% of the entire codebase.
