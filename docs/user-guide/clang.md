# Clang Compiler — Complete Usage Guide

This guide documents every option available in Clang (version 23), organized by
functional category with practical examples. Based on `clang --help` output.

---

## Table of Contents

1. [Basic Compilation](#1-basic-compilation)
2. [Preprocessing](#2-preprocessing)
3. [Language Standards & Dialects](#3-language-standards--dialects)
4. [Optimization Control](#4-optimization-control)
5. [Code Generation](#5-code-generation)
6. [Debug Information](#6-debug-information)
7. [Warning & Diagnostic Control](#7-warning--diagnostic-control)
8. [Floating Point Control](#8-floating-point-control)
9. [Sanitizers](#9-sanitizers)
10. [Profiling (PGO, Coverage)](#10-profiling-pgo-coverage)
11. [Link-Time Optimization (LTO)](#11-link-time-optimization-lto)
12. [Modules & Precompiled Headers](#12-modules--precompiled-headers)
13. [GPU / Offloading (CUDA, HIP, OpenMP)](#13-gpu--offloading-cuda-hip-openmp)
14. [OpenCL Options](#14-opencl-options)
15. [OpenMP Options](#15-openmp-options)
16. [HLSL / DirectX / SPIR-V Options](#16-hlsl--directx--spirv-options)
17. [Objective-C / Objective-C++ Options](#17-objective-c--objective-c-options)
18. [SYCL Options](#18-sycl-options)
19. [Target-Specific Options](#19-target-specific-options)
20. [Include & Library Path Management](#20-include--library-path-management)
21. [Linking](#21-linking)
22. [Symbol Visibility & ABI](#22-symbol-visibility--abi)
23. [Security Hardening](#23-security-hardening)
24. [C++ Specific Options](#24-c-specific-options)
25. [Developer & Debugging Options](#25-developer--debugging-options)
26. [Miscellaneous](#26-miscellaneous)
27. [Pass-Through Options](#27-pass-through-options)

---

## 1. Basic Compilation

### Compile to Executable

```bash
# Compile a single source file to an executable named 'a.out'
clang hello.c

# Specify output file name
clang hello.c -o hello

# Compile multiple source files
clang main.c util.c -o program
```

### Compilation Stages

```bash
# Preprocess only (-E): output preprocessed source to stdout
clang -E hello.c

# Preprocess to file with macros defined
clang -E -dM hello.c           # print only macro definitions
clang -E -dD hello.c           # print macros in addition to normal output

# Compile only, no assembly (-S): produce .s assembly file
clang -S hello.c               # produces hello.s

# Compile + assemble, no link (-c): produce .o object file
clang -c hello.c               # produces hello.o

# Full compilation to executable (preprocess + compile + assemble + link)
clang hello.c -o hello
```

### Common Development Flags

```bash
# Check syntax only — don't generate code
clang -fsyntax-only hello.c

# Verbose: show all commands executed
clang -v hello.c -o hello

# Dry run: print commands without executing them
clang -### hello.c -o hello

# Show time for each compilation phase
clang -time hello.c -o hello
```

---

## 2. Preprocessing

### Controlling Preprocessor Output

```bash
# Preprocess to stdout, no line markers (clean output)
clang -E -P hello.c

# Include comments from source in preprocessed output
clang -E -C hello.c            # -C: include comments (but not from macros)
clang -E -CC hello.c           # -CC: include comments even from within macros

# Use #line directives in preprocessed output (useful for debugging)
clang -E -fuse-line-directives hello.c

# Minimize whitespace (normalize formatting, useful for diffing)
clang -E -fminimize-whitespace hello.c

# Emulate traditional C preprocessor behavior
clang -E -traditional-cpp hello.c
```

### Macro Definition & Undefinition

```bash
# Define a macro (-D)
clang -DDEBUG -DMAX_SIZE=1024 hello.c

# Undefine a macro (-U)
clang -U__STRICT_ANSI__ hello.c

# Undefine all system-defined macros (start with clean macro environment)
clang -E -undef hello.c
```

### Include Path Control

```bash
# Add directory to include search path (-I)
clang -I./include -I/usr/local/include main.c

# System include (suppresses warnings from these headers)
clang -isystem /opt/sdk/include main.c

# Include file BEFORE parsing (like #include at top of file)
clang -include prelude.h main.c

# Include macros only (no declarations) before parsing
clang -imacros config.h main.c

# Quote include path (for #include "..." search)
clang -iquote ./myheaders main.c

# Directory to search AFTER system paths
clang -idirafter /fallback/include main.c
```

### Dependency Generation

```bash
# Write Make-style dependency file
clang -M main.c                # writes to stdout, implies -E
clang -MD main.c               # writes to main.d, compiles normally
clang -MMD main.c              # like -MD but only user headers
clang -MF deps.mk main.c      # specify output file name

# Show included headers with nesting depth (-H)
clang -H main.c                # prints each #include with leading dots for depth
clang -H -fshow-skipped-includes main.c  # also show files skipped by include guards

# Create phony targets for each dependency (for Makefiles)
clang -MP -MMD main.c
```

### Header Include Control

```bash
# Disable standard include directories
clang -nostdinc hello.c        # disable all standard includes
clang -nostdlibinc hello.c     # disable standard library includes only
clang -nostdinc++ hello.c      # disable C++ standard library includes

# Keep system #includes in preprocessor output (don't expand them)
clang -E -fkeep-system-includes hello.c
```

---

## 3. Language Standards & Dialects

### C Standards

```bash
# Compile with a specific C standard
clang -std=c89 hello.c
clang -std=c99 hello.c
clang -std=c11 hello.c
clang -std=c17 hello.c
clang -std=c23 hello.c

# GNU dialect (includes GNU extensions)
clang -std=gnu11 hello.c
clang -std=gnu17 hello.c
```

### C++ Standards

```bash
# Compile with a specific C++ standard
clang -std=c++98 hello.cpp
clang -std=c++11 hello.cpp
clang -std=c++14 hello.cpp
clang -std=c++17 hello.cpp
clang -std=c++20 hello.cpp
clang -std=c++23 hello.cpp
clang -std=c++26 hello.cpp    # experimental C++26 support

# GNU C++ dialect
clang -std=gnu++17 hello.cpp
```

### Treating Source Files As Specific Languages

```bash
# Force language for subsequent input files (-x)
clang -x c header.h           # treat header.h as C
clang -x c++ source.c         # treat source.c as C++
clang -x assembler code.inc   # treat as assembly
clang -x none file.h          # reset to auto-detect by extension

# Treat sources as Objective-C / Objective-C++
clang -ObjC file.m            # Objective-C
clang -ObjC++ file.mm         # Objective-C++
```

### GNU Extensions

```bash
# Allow GNU keywords (typeof, __attribute__, etc.)
clang -fgnu-keywords hello.c

# Use GCC inline semantics (gnu89 style)
clang -fgnu89-inline hello.c

# Report GCC version for compatibility macros (default: 4.2.1)
clang -fgnuc-version=11
```

### Microsoft Compatibility

```bash
# Enable MSVC compatibility mode
clang -fms-compatibility hello.cpp

# MSVC-specific extensions
clang -fms-extensions hello.cpp

# Allow __declspec keyword
clang -fdeclspec hello.cpp

# MS-style anonymous structs
clang -fms-anonymous-structs hello.cpp

# Set MSVC version reported via _MSC_VER
clang -fmsc-version=1930 hello.cpp   # VS 2022

# Delayed template parsing (MSVC behavior)
clang -fdelayed-template-parsing hello.cpp
```

### Additional Language Features

```bash
# Enable C++20 Coroutines
clang -fcoroutines hello.cpp

# Enable fixed-point arithmetic types
clang -ffixed-point hello.c

# Enable the 'blocks' language feature (Apple extension)
clang -fblocks hello.c

# Allow '$' in identifiers
clang -fdollars-in-identifiers hello.c

# Disable digraphs (<:, :>, <% etc.)
clang -fno-digraphs hello.c

# Enable/disable trigraph processing (??= → #)
clang -ftrigraphs hello.c
clang -fno-trigraphs hello.c    # disable (modern default)

# Enable raw string literals
clang -fraw-string-literals hello.cpp
```

---

## 4. Optimization Control

### Optimization Levels

```bash
# Optimization levels
clang -O0 hello.c              # no optimization (default for debugging)
clang -O1 hello.c              # basic optimizations, fast compile
clang -O2 hello.c              # moderate optimizations (good default for production)
clang -O3 hello.c              # aggressive optimizations (includes -O2 + more)
clang -Os hello.c              # optimize for code size
clang -Oz hello.c              # optimize aggressively for code size

# Fast math + optimizations (non-conforming: may break IEEE correctness)
clang -Ofast hello.c           # equivalent to -O3 -ffast-math

# Disable all optimizations
clang -O0 hello.c
```

### Fine-grained Optimization Flags

```bash
# Enable function inlining
clang -finline-functions hello.c

# Inline functions marked with 'inline' keyword
clang -finline-hint-functions hello.c

# Limit inlining by stack size (suppress if callee exceeds threshold)
clang -finline-max-stacksize=128 hello.c

# Loop unrolling
clang -funroll-loops hello.c
clang -fno-unroll-loops hello.c

# Vectorization
clang -fvectorize hello.c                 # enable loop vectorization
clang -fslp-vectorize hello.c             # enable SLP (superword-level) vectorization
clang -fno-vectorize hello.c

# Set preferred vector width for auto-vectorization
clang -mprefer-vector-width=512 hello.c    # prefer 512-bit vectors

# Loop interchange (swap inner/outer loops for cache optimization)
clang -floop-interchange hello.c

# Link-time optimization
clang -flto hello.c              # full LTO
clang -flto=thin hello.c         # ThinLTO (better scalability)

# Devirtualization
clang -fdevirtualize-speculatively hello.cpp

# Dead store elimination
clang -flifetime-dse hello.c

# Merge identical constants
clang -fmerge-all-constants hello.c

# Jump tables for switch statements
clang -fjump-tables hello.c
clang -fno-jump-tables hello.c   # use if/else chains instead

# Tail call optimization
clang -foptimize-sibling-calls hello.c

# Place functions/data in separate sections (enables linker GC)
clang -ffunction-sections -fdata-sections hello.c

# Omit frame pointer (saves a register; may hinder debugging)
clang -fomit-frame-pointer hello.c

# Use the GlobalISel instruction selector
clang -fglobal-isel hello.c
```

### Size Optimization

```bash
# Optimize for size
clang -Os hello.c

# More aggressive size optimization
clang -Oz hello.c

# Merge identical functions
# (enabled automatically at higher optimization levels with -flto)
clang -fmerge-all-constants hello.c
```

### Optimization Remarks

```bash
# Show which optimizations were performed
clang -Rpass=.* hello.c

# Show missed optimization opportunities
clang -Rpass-missed=.* hello.c

# Show optimization analysis
clang -Rpass-analysis=.* hello.c

# Filter remarks by pass name pattern
clang -Rpass=inline hello.c              # show only inlining decisions
clang -Rpass-missed=loop.* hello.c       # show missed loop optimizations

# Save optimization remarks to a file
clang -fsave-optimization-record hello.c           # YAML format
clang -fsave-optimization-record=yaml hello.c
clang -foptimization-record-file=remarks.yml hello.c

# Filter which passes are included in the record
clang -foptimization-record-passes=inline hello.c
```

---

## 5. Code Generation

### Target Selection

```bash
# Compile for a specific target triple
clang --target=x86_64-unknown-linux-gnu hello.c
clang --target=aarch64-apple-darwin23 hello.c
clang --target=riscv64-unknown-elf hello.c

# Print the effective target triple for the host
clang -print-effective-triple

# Print all registered targets
clang -print-targets
```

### Architecture & CPU Selection

```bash
# Set target CPU
clang -march=armv8.5-a hello.c            # AArch64
clang -march=rv64gc hello.c               # RISC-V
clang -march=x86-64-v3 hello.c            # x86 (AVX2+FMA+BMI)
clang -march=native hello.c               # detect host CPU

# Set specific CPU model
clang -mcpu=cortex-a76 hello.c            # ARM Cortex-A76
clang -mcpu=neoverse-n1 hello.c           # ARM Neoverse N1
clang -mcpu=znver4 hello.c                # AMD Zen 4

# Tune for a specific CPU (without restricting instruction set)
clang -mtune=znver4 hello.c

# List supported CPUs for the target
clang -mcpu=help
```

### Target-Specific: x86 / x86_64

```bash
# Use specific instruction sets
clang -mavx2 hello.c             # enable AVX2
clang -mavx512f hello.c          # enable AVX-512 Foundation

# Set stack alignment
clang -mstack-alignment=16 hello.c

# Use Intel MCU ABI
clang -miamcu hello.c

# Enable SSE-to-AVX VEX prefix encoding
clang -msse2avx hello.c

# Skip RAX setup for variadic functions
clang -mskip-rax-setup hello.c
```

### Target-Specific: AArch64 / ARM

```bash
# Specify branch protection (PAC + BTI)
clang -mbranch-protection=standard hello.c   # PAC+BTI
clang -mbranch-protection=bti hello.c        # BTI only
clang -mbranch-protection=pac-ret hello.c    # PAC only

# SVE vector length
clang -msve-vector-bits=256 hello.c

# Pointer authentication
clang -fptrauth-returns hello.c
clang -fptrauth-calls hello.c

# Mark BTI property in assembly
clang -mmark-bti-property hello.c

# Function outlining
clang -moutline hello.c
clang -mno-outline hello.c
```

### Integrated Assembler

```bash
# Use the integrated assembler (default)
clang -fintegrated-as hello.c

# Use external assembler
clang -fno-integrated-as hello.c

# Pass arguments to assembler
clang -Wa,-march=armv8.5-a hello.c
clang -Wa,--fatal-warnings hello.c
clang -Xassembler -march=armv8.5-a hello.c
```

### LLVM IR Generation

```bash
# Emit LLVM IR (text)
clang -S -emit-llvm hello.c     # produces hello.ll

# Emit LLVM bitcode (binary)
clang -c -emit-llvm hello.c     # produces hello.bc

# Discard value names (smaller IR, faster processing)
clang -fdiscard-value-names -S -emit-llvm hello.c

# Embed LLVM bitcode in the object file
clang -fembed-bitcode hello.c
```

### Forward Arguments to LLVM

```bash
# Pass arguments directly to LLVM's backend option processing
clang -mllvm -enable-unsafe-fp-math hello.c
clang -mllvm -print-after-all hello.c
```

---

## 6. Debug Information

### Basic Debug Flags

```bash
# Generate debug information (default DWARF version for platform)
clang -g hello.c

# Generate minimal line tables only
clang -gline-tables-only hello.c

# Emit line number directives only (no full debug info)
clang -gline-directives-only hello.c

# Generate standalone debug info (complete type info)
clang -fstandalone-debug hello.c

# Limit debug info to reduce binary size
clang -fno-standalone-debug hello.c
```

### DWARF Versions

```bash
# Generate specific DWARF version
clang -gdwarf-3 hello.c
clang -gdwarf-4 hello.c
clang -gdwarf-5 hello.c         # default on modern platforms
clang -gdwarf-6 hello.c         # experimental

# Restrict to strict DWARF features of specified version
clang -gstrict-dwarf -gdwarf-4 hello.c
```

### DWARF Configuration

```bash
# Use 64-bit DWARF format (for very large debug info, >4GB)
clang -gdwarf64 hello.c

# Split DWARF (debug info in .dwo files, faster linking)
clang -gsplit-dwarf hello.c
clang -gsplit-dwarf=split hello.c
clang -gsplit-dwarf=single hello.c

# Split DWARF with inlining info for online symbolication
clang -fsplit-dwarf-inlining -gsplit-dwarf hello.c

# Compress DWARF sections
clang -gz=zlib hello.c          # zlib compression
clang -gz=zstd hello.c          # zstd compression

# Embed source text in DWARF (for reproducible debugging)
clang -gembed-source hello.c

# Emit macro debug information
clang -fdebug-macro hello.c

# Call site debug info for inlined calls
clang -gcall-site-info hello.c
clang -gno-call-site-info hello.c

# Attach linkage names to C++ ctor/dtor declarations in DWARF
clang -gstructor-decl-linkage-names hello.cpp

# Set default DWARF version
clang -fdebug-default-version=5 hello.c

# Enable "Key Instructions" for better debugging of optimized code
clang -gkey-instructions hello.c
```

### Debug Info Path Control

```bash
# Remap paths in debug info (for reproducible builds)
clang -fdebug-prefix-map=/build=/src hello.c

# Set compilation directory in debug info
clang -fdebug-compilation-dir=/app hello.c

# Record sysroot path in debug info
clang -fdebug-record-sysroot hello.c
```

### CodeView (Windows)

```bash
# Generate CodeView debug info (Windows/MSVC compatible)
clang -gcodeview hello.c

# Emit type record hashes
clang -gcodeview-ghash hello.c

# Embed compiler path and command line
clang -gcodeview-command-line hello.c
```

### Debug Info Module Support

```bash
# Use external references to Clang modules in debug info
clang -gmodules hello.c
```

### dSYM (macOS)

```bash
# Specify output directory for dSYM bundles
clang -dsym-dir /output/dsyms hello.c
```

---

## 7. Warning & Diagnostic Control

### Warning Basics

```bash
# Enable a specific warning
clang -Wall hello.c             # common warnings
clang -Wextra hello.c           # extra warnings
clang -Weverything hello.c      # ALL possible warnings (very noisy)

# Suppress all warnings
clang -w hello.c

# Treat warnings as errors
clang -Werror hello.c           # all warnings → errors
clang -Werror=return-type hello.c  # specific warning → error

# Pedantic warnings (ISO C/C++ compliance)
clang -pedantic hello.c
clang -pedantic-errors hello.c  # pedantic warnings are errors
```

### Diagnostic Formatting

```bash
# Colored diagnostics
clang -fcolor-diagnostics hello.c
clang -fno-color-diagnostics hello.c

# Control when colors are used
clang -fdiagnostics-color=always hello.c
clang -fdiagnostics-color=auto hello.c

# Print absolute paths in diagnostics
clang -fdiagnostics-absolute-paths hello.c

# Print source range information as numeric spans
clang -fdiagnostics-print-source-range-info hello.c

# Include fix-it hints in machine-parseable form
clang -fdiagnostics-parseable-fixits hello.c

# Show which flag controls each warning
clang -fdiagnostics-show-option hello.c

# Show template tree comparison for template mismatches
clang -fdiagnostics-show-template-tree hello.cpp

# Show include stacks in diagnostic notes
clang -fdiagnostics-show-note-include-stack hello.c

# Format messages to fit within N columns
clang -fmessage-length=80 hello.c

# Use ANSI escape codes for diagnostics
clang -fansi-escape-codes hello.c
```

### Controlling Diagnostic Information

```bash
# Show or hide source column numbers
clang -fshow-column hello.c
clang -fno-show-column hello.c

# Show or hide source location in diagnostics
clang -fno-show-source-location hello.c

# Show or hide line numbers in code snippets
clang -fno-diagnostics-show-line-numbers hello.c

# Suppress fix-it information
clang -fno-diagnostics-fixit-info hello.c

# Control the number of caret diagnostic lines
clang -fcaret-diagnostics-max-lines=10 hello.c

# Spell checking for unrecognized identifiers
clang -fspell-checking hello.c
clang -fno-spell-checking hello.c
clang -fspell-checking-limit=5 hello.c
```

### Template & Macro Backtrace Control

```bash
# Control template instantiation backtrace depth
clang -ftemplate-backtrace-limit=10 hello.cpp

# Control macro expansion backtrace depth
clang -fmacro-backtrace-limit=10 hello.c

# Control constexpr evaluation backtrace depth
clang -fconstexpr-backtrace-limit=10 hello.cpp

# Set maximum recursion depth for templates
clang -ftemplate-depth=1024 hello.cpp

# Set maximum depth for constexpr evaluation
clang -fconstexpr-depth=512 hello.cpp

# Set maximum number of constexpr evaluation steps
clang -fconstexpr-steps=1000000 hello.cpp
```

### Profile Hotness in Diagnostics

```bash
# Show profile hotness information in optimization remarks
clang -fdiagnostics-show-hotness hello.c

# Set hotness threshold for remarks
clang -fdiagnostics-hotness-threshold=1000 hello.c
clang -fdiagnostics-hotness-threshold=auto hello.c
```

### Warnings for Driver Arguments

```bash
# Don't warn about unused driver arguments (useful with build systems)
clang -Qunused-arguments hello.c

# Control unused-argument warnings for specific arguments
clang --start-no-unused-arguments -some-flag --end-no-unused-arguments hello.c
```

### Print Diagnostic Information

```bash
# Print all warning options
clang -print-diagnostic-options

# Print supported CPUs for the target
clang -print-supported-cpus

# Print enabled extensions for the target
clang -print-enabled-extensions

# Print supported -march extensions
clang -print-supported-extensions    # RISC-V, AArch64, ARM
```

---

## 8. Floating Point Control

### Floating Point Model

```bash
# Fast math: aggressive, possibly lossy, FP optimizations
clang -ffast-math hello.c

# Allow optimizations that ignore signed zeros
clang -fno-signed-zeros hello.c

# Allow optimizations assuming no NaNs or Infs
clang -ffinite-math-only hello.c

# Honor infinities (opposite: assume no Infs)
clang -fhonor-infinities hello.c

# Honor NaNs (opposite: assume no NaNs)
clang -fhonor-nans hello.c

# Unsafe math optimizations (may reduce precision)
clang -funsafe-math-optimizations hello.c

# Allow reciprocal (1/x) reassociation
clang -freciprocal-math hello.c

# Approximate math functions
clang -fapprox-func hello.c
```

### FP Contract (Fused Multiply-Add)

```bash
# Control formation of FMA operations
clang -ffp-contract=on hello.c            # FMA allowed per language rules
clang -ffp-contract=off hello.c           # no FMA
clang -ffp-contract=fast hello.c          # FMA everywhere (may change results)
clang -ffp-contract=fast-honor-pragmas hello.c
```

### FP Model Shorthand

```bash
# Predefined FP model (sets multiple flags)
clang -ffp-model=precise hello.c          # default: follows IEEE
clang -ffp-model=strict hello.c           # strict IEEE conformance
clang -ffp-model=fast hello.c             # aggressive, non-conforming
```

### FP Evaluation Behavior

```bash
# Set evaluation method for floating-point
clang -ffp-eval-method=source hello.c     # use types from source
clang -ffp-eval-method=double hello.c     # evaluate as double
clang -ffp-eval-method=extended hello.c   # use extended precision

# Exception behavior
clang -ffp-exception-behavior=ignore hello.c     # ignore exceptions
clang -ffp-exception-behavior=maytrap hello.c    # may trap on exceptions
clang -ffp-exception-behavior=strict hello.c     # strict: follow IEEE
```

### Math Errno & Honoring Parentheses

```bash
# Require math functions to set errno
clang -fmath-errno hello.c

# Don't set errno for math functions (enables more optimizations)
clang -fno-math-errno hello.c

# Honor parentheses in FP expressions (less optimization)
clang -fprotect-parens hello.c
```

### Vector Math Library

```bash
# Use vectorized math library
clang -fveclib=Accelerate hello.c     # macOS Accelerate
clang -fveclib=libmvec hello.c        # GNU libmvec (GLIBC 2.40+)
clang -fveclib=SVML hello.c           # Intel SVML
clang -fveclib=ArmPL hello.c          # Arm Performance Libraries
```

---

### Complex Arithmetic

```bash
# Basic algebraic expansions of complex operations
clang -fcx-limited-range hello.c

# Range reduction for complex operations (Fortran rules)
clang -fcx-fortran-rules hello.c

# Control complex multiplication/division methods
clang -fcomplex-arithmetic=improved hello.c
clang -fcomplex-arithmetic=promoted hello.c
clang -fcomplex-arithmetic=full hello.c
```

### Strict Float Cast Overflow

```bash
# Strict overflow behavior for float-to-int casts (default)
clang -fstrict-float-cast-overflow hello.c

# Relaxed overflow (match target native behavior)
clang -fno-strict-float-cast-overflow hello.c
```

---

## 9. Sanitizers

Sanitizers are runtime instrumentation tools that detect bugs
(memory errors, undefined behavior, data races, etc.) during program execution.

### Undefined Behavior Sanitizer (UBSan)

```bash
# Enable all common UBSan checks
clang -fsanitize=undefined hello.c

# Enable specific UBSan checks
clang -fsanitize=null hello.c             # null pointer dereference
clang -fsanitize=alignment hello.c        # misaligned pointer
clang -fsanitize=bounds hello.c           # array bounds
clang -fsanitize=integer hello.c          # integer overflow
clang -fsanitize=shift hello.c            # shift out of range
clang -fsanitize=return hello.c           # missing return
clang -fsanitize=float-divide-by-zero hello.c
clang -fsanitize=unsigned-integer-overflow hello.c

# UBSan with minimal runtime (useful for kernel/small environments)
clang -fsanitize=undefined -fsanitize-minimal-runtime hello.c

# Trap on UBSan error instead of printing diagnostic (for production)
clang -fsanitize-trap=undefined hello.c

# Use a specific trap function
clang -fsanitize-trap=undefined -ftrap-function=my_abort_handler hello.c

# Specific overflow patterns to suppress
clang -fsanitize-undefined-ignore-overflow-pattern=unsigned-saturated hello.c

# Strip path components from diagnostic output
clang -fsanitize-undefined-strip-path-components=2 hello.c
```

### Address Sanitizer (ASan)

```bash
# Enable AddressSanitizer (detects use-after-free, buffer overflow, leaks)
clang -fsanitize=address hello.c

# Use-after-scope detection
clang -fsanitize=address -fsanitize-address-use-after-scope hello.c

# Use-after-return detection mode
clang -fsanitize=address -fsanitize-address-use-after-return=runtime hello.c
clang -fsanitize=address -fsanitize-address-use-after-return=never hello.c

# Enable ODR violation detection (via indicator globals)
clang -fsanitize=address -fsanitize-address-use-odr-indicator hello.c

# Poison array cookies for custom operator new[]
clang -fsanitize=address -fsanitize-address-poison-custom-array-cookie hello.c

# Outline instrumentation (always generate function calls, not inline)
clang -fsanitize=address -fsanitize-address-outline-instrumentation hello.c

# Field padding level
clang -fsanitize-address-field-padding=1 hello.c
clang -fsanitize-address-field-padding=2 hello.c

# Enable linker dead stripping of ASan globals
clang -fsanitize-address-globals-dead-stripping hello.c

# Module destructor type for ASan
clang -fsanitize-address-destructor=global hello.c
clang -fsanitize-address-destructor=none hello.c
```

### Memory Sanitizer (MSan)

```bash
# Enable MemorySanitizer (detects uninitialized reads)
clang -fsanitize=memory hello.c

# Track origins of uninitialized values (for debugging)
clang -fsanitize=memory -fsanitize-memory-track-origins hello.c
clang -fsanitize=memory -fsanitize-memory-track-origins=1 hello.c

# Detect uninitialized parameters and return values
clang -fsanitize=memory -fsanitize-memory-param-retval hello.c

# Detect use-after-destroy
clang -fsanitize=memory -fsanitize-memory-use-after-dtor hello.c
```

### Thread Sanitizer (TSan)

```bash
# Enable ThreadSanitizer (detects data races)
clang -fsanitize=thread hello.c

# Control TSan instrumentation detail
clang -fsanitize=thread -fno-sanitize-thread-atomics hello.c
clang -fsanitize=thread -fno-sanitize-thread-func-entry-exit hello.c
clang -fsanitize=thread -fno-sanitize-thread-memory-access hello.c
```

### Hardware-Assisted Address Sanitizer (HWASan)

```bash
# Enable HWASan (uses hardware memory tagging, ARM AArch64)
clang -fsanitize=hwaddress hello.c

# Enable aliasing mode
clang -fsanitize=hwaddress -fsanitize-hwaddress-experimental-aliasing hello.c

# Select ABI
clang -fsanitize=hwaddress -fsanitize-hwaddress-abi=platform hello.c
```

### Kernel Control Flow Integrity (KCFI)

```bash
# Enable KCFI (fine-grained indirect call checking)
clang -fsanitize=kcfi hello.c

# Embed arity info in KCFI prefix
clang -fsanitize=kcfi -fsanitize-kcfi-arity hello.c

# Select hash algorithm for KCFI type IDs
clang -fsanitize=kcfi -fsanitize-kcfi-hash=xxHash64 hello.c
clang -fsanitize=kcfi -fsanitize-kcfi-hash=FNV-1a hello.c
```

### Control Flow Integrity (CFI)

```bash
# Enable CFI checks
clang -fsanitize=cfi hello.cpp

# Enable cross-DSO CFI checks
clang -fsanitize=cfi -fsanitize-cfi-cross-dso hello.cpp

# Canonical jump tables (make addresses canonical in symbol table)
clang -fsanitize-cfi-canonical-jump-tables hello.cpp

# CFI indirect call type generalization
clang -fsanitize-cfi-icall-generalize-pointers hello.cpp
clang -fsanitize-cfi-icall-experimental-normalize-integers hello.cpp
```

### SafeStack & CFI Combined

```bash
# Enable SafeStack (dual-stack: safe for spills, unsafe for allocas)
clang -fsanitize=safe-stack hello.c

# Combined: CFI + SafeStack + UBSan
clang -fsanitize=cfi,safe-stack,undefined hello.cpp
```

### Sanitizer Recovery & Trapping

```bash
# Don't abort on error: continue execution after reporting
clang -fsanitize-recover=address hello.c
clang -fsanitize-recover=undefined hello.c
clang -fsanitize-recover=all hello.c

# Trap instead of printing diagnostic (for environments without printf)
clang -fsanitize-trap=undefined hello.c
clang -fsanitize-trap=address hello.c

# Use infinite loop instead of trap instruction
clang -fsanitize-trap=undefined -fsanitize-trap-loop hello.c
```

### Sanitizer Ignorelists

```bash
# Path to ignorelist file (suppress instrumentation for specific functions/files)
clang -fsanitize=address -fsanitize-ignorelist=asan_ignore.txt hello.c

# System ignorelist (for system headers)
clang -fsanitize=address -fsanitize-system-ignorelist=system_ignore.txt hello.c

# Don't use ignorelist
clang -fsanitize=address -fno-sanitize-ignorelist hello.c
```

### Sanitizer Coverage

```bash
# Enable sanitizer coverage (for fuzzing)
clang -fsanitize-coverage=trace-pc-guard hello.c
clang -fsanitize-coverage=trace-cmp hello.c
clang -fsanitize-coverage=edge hello.c

# Allowlist / ignorelist for coverage
clang -fsanitize-coverage-allowlist=allow.txt hello.c
clang -fsanitize-coverage-ignorelist=block.txt hello.c

# Stack depth callback coverage
clang -fsanitize-coverage-stack-depth-callback-min=8 hello.c
```

### Sanitizer Metadata (Binary Analysis)

```bash
# Emit metadata for binary analysis tools
clang -fexperimental-sanitize-metadata=atomics hello.c
clang -fexperimental-sanitize-metadata=uar hello.c   # use-after-return

# Ignorelist for metadata
clang -fexperimental-sanitize-metadata-ignorelist=ignore.txt hello.c
```

### Sanitizer Statistics

```bash
# Enable sanitizer statistics gathering
clang -fsanitize=address -fsanitize-stats hello.c

# Disable stats
clang -fsanitize=address -fno-sanitize-stats hello.c
```

### Sanitizer Merge & ABI

```bash
# Merge sanitizer handler code (smaller binary)
clang -fsanitize=address -fsanitize-merge hello.c

# Use stable ABI for sanitizer runtime
clang -fsanitize=address -fsanitize-stable-abi hello.c
```

### Sanitizer Debug Info

```bash
# Annotate sanitizer instrumentation with debug info
clang -fsanitize=address -fsanitize-annotate-debug-info hello.c

# Set trap reason detail level
clang -fsanitize=address -fsanitize-debug-trap-reasons=detailed hello.c
clang -fsanitize=address -fsanitize-debug-trap-reasons=basic hello.c
clang -fsanitize=address -fsanitize-debug-trap-reasons=none hello.c
```

### Sanitizer Runtime Linking

```bash
# Static linking of sanitizer runtime
clang -fsanitize=address -static-libsan hello.c

# Dynamic linking of sanitizer runtime
clang -fsanitize=address -shared-libsan hello.c
```

### Sanitizer Hot/Cold Skipping

```bash
# Skip sanitization for hot code based on PGO profile
clang -fsanitize=address -fsanitize-skip-hot-cutoff=address=0.1 hello.c
clang -fsanitize=address -fsanitize-skip-hot-cutoff=all=0.2 hello.c
```

### AllocToken Sanitizer

```bash
# Enable AllocToken (detects mismatched alloc/dealloc)
clang -fsanitize=alloc-token hello.c

# Enable fast ABI mode
clang -fsanitize=alloc-token -fsanitize-alloc-token-fast-abi hello.c

# Enable extended coverage to custom allocation functions
clang -fsanitize=alloc-token -fsanitize-alloc-token-extended hello.c
```

### Numerical Stability Sanitizer

```bash
clang -fsanitize=numerical hello.c
```

### Realtime Sanitizer

```bash
clang -fsanitize=realtime hello.c
```

### Type Sanitizer (TySan)

```bash
# Outline instrumentation for type sanitizer
clang -fsanitize=type -fsanitize-type-outline-instrumentation hello.cpp
```

---

## 10. Profiling (PGO, Coverage)

### Instrumentation-Based PGO

```bash
# Phase 1: Generate instrumented build
clang -fprofile-instr-generate hello.c -o hello
./hello                         # runs and produces default.profraw

# Specify output directory for raw profiles
clang -fprofile-instr-generate=/tmp/profiles hello.c

# Specify exact output file
clang -fprofile-instr-generate=myapp.profraw hello.c

# Phase 2: Merge raw profiles into indexed profile
llvm-profdata merge -output=hello.profdata default.profraw

# Phase 3: Use the profile for optimization
clang -fprofile-instr-use=hello.profdata hello.c -o hello_opt

# Alternative: -fprofile-use (accepts directory or file)
clang -fprofile-use=/path/to/profiles hello.c
```

### Context-Sensitive PGO (CSPGO)

```bash
# Generate context-sensitive profiles
clang -fcs-profile-generate hello.c
./hello

# Use context-sensitive profiles
clang -fcs-profile-generate=/tmp/csprofiles hello.c
```

### Coverage

```bash
# Generate coverage mapping (instrumentation)
clang -fcoverage-mapping -fprofile-instr-generate hello.c
./hello
llvm-profdata merge -sparse default.profraw -o hello.profdata
llvm-cov show ./hello -instr-profile=hello.profdata

# MC/DC (Modified Condition/Decision Coverage) criteria
clang -fcoverage-mapping -fcoverage-mcdc -fprofile-instr-generate hello.c

# GCOV-style coverage
clang -fprofile-arcs -ftest-coverage hello.c
./hello
gcov hello.c
```

### Profile Configuration

```bash
# Filter which files to instrument (regex, semicolon-separated)
clang -fprofile-instr-generate -fprofile-filter-files="src/.*cpp" hello.c
clang -fprofile-instr-generate -fprofile-exclude-files="test/.*cpp" hello.c

# Use a profile list file (sanitizer special case list format)
clang -fprofile-instr-generate -fprofile-list=profile_list.txt hello.c

# Function groups: partition functions into N groups, instrument group i only
clang -fprofile-instr-generate -fprofile-function-groups=4 \
      -fprofile-selected-function-group=0 hello.c

# Atomic vs. non-atomic profile counter updates
clang -fprofile-instr-generate -fprofile-update=atomic hello.c
clang -fprofile-instr-generate -fprofile-update=prefer-atomic hello.c

# Remap symbols for profile matching
clang -fprofile-instr-use=profile.profdata \
      -fprofile-remapping-file=remap.txt hello.c

# 'Continuous' mode (profile instrumentation that can be sampled mid-run)
clang -fprofile-instr-generate -fprofile-continuous hello.c

# Cold function coverage (separate coverage for cold functions)
clang -fprofile-generate-cold-function-coverage hello.c
clang -fprofile-instr-generate -fcoverage-compilation-dir=/build/hello.c

# Temporal profile (collects temporal execution order info)
clang -ftemporal-profile hello.c
```

### Sample-Based PGO (AutoFDO)

```bash
# Use sample profile for optimization
clang -fprofile-sample-use=samples.afdo hello.c

# Mark that the sample profile is accurate
clang -fprofile-sample-accurate -fprofile-sample-use=samples.afdo hello.c

# Use profi to infer block/edge counts from samples
clang -fsample-profile-use-profi -fprofile-sample-use=samples.afdo hello.c

# Extra debug info for more accurate sample profiling
clang -fdebug-info-for-profiling hello.c

# Pseudo probes for sample profiling (lower overhead than debug info)
clang -fpseudo-probe-for-profiling hello.c
```

### Memory Profiling (MemProf)

```bash
# Enable heap memory profiling
clang -fmemory-profile hello.c

# Specify output directory for memory profile
clang -fmemory-profile=/tmp/memprof hello.c

# Use memory profile for optimization
clang -fmemory-profile-use=memprof.profdata hello.c
```

### CodeGen Data (CGData)

```bash
# Generate codegen data during compilation (outlining info, etc.)
clang -fcodegen-data-generate hello.c
clang -fcodegen-data-generate=/tmp/cgdata hello.c

# Use codegen data to optimize subsequent compilations
clang -fcodegen-data-use hello.c
clang -fcodegen-data-use=/tmp/cgdata hello.c
```

### XRay Instrumentation

```bash
# Enable XRay instrumentation (function call tracing)
clang -fxray-instrument hello.c

# Minimum function size threshold for instrumentation
clang -fxray-instrument -fxray-instruction-threshold=200 hello.c

# Ignore functions with loops below the threshold
clang -fxray-instrument -fxray-ignore-loops hello.c

# Select which instrumentation points to emit
clang -fxray-instrument -fxray-instrumentation-bundle=function-entry hello.c
clang -fxray-instrument -fxray-instrumentation-bundle=function-exit hello.c
clang -fxray-instrument -fxray-instrumentation-bundle=custom hello.c

# Always emit custom/typed events even if function is not instrumented
clang -fxray-always-emit-customevents hello.c
clang -fxray-always-emit-typedevents hello.c

# XRay function groups (instrument subset of functions)
clang -fxray-function-groups=4 -fxray-selected-function-group=0 hello.c

# Shared library XRay instrumentation
clang -fxray-shared -fxray-instrument hello.c

# Link XRay runtime library (default)
clang -fxray-instrument -fxray-link-deps hello.c

# Specify which XRay modes to link
clang -fxray-modes=xray-basic hello.c
clang -fxray-modes=xray-fdr hello.c

# Omit the XRay function index section
clang -fxray-instrument -fno-xray-function-index hello.c
```

---

## 11. Link-Time Optimization (LTO)

### Basic LTO

```bash
# Full LTO (monolithic, all modules merged into one)
clang -flto hello.c

# ThinLTO (scalable, per-module summaries + selective cross-module import)
clang -flto=thin hello.c

# Auto (let the toolchain decide)
clang -flto=auto hello.c

# Disable LTO
clang -fno-lto hello.c
```

### LTO Backend Configuration

```bash
# Control ThinLTO backend parallelism
clang -flto=thin -flto-jobs=8 hello.c

# Distribute ThinLTO backend compilations
clang -flto=thin -fthinlto-distributor=/path/to/distributor hello.c

# Number of partitions for parallel full LTO codegen (ld.lld only)
clang -flto=full -flto-partitions=4 hello.c

# Perform ThinLTO importing with a provided summary index file
clang -flto=thin -fthinlto-index=index.thinlto hello.c

# Write minimized bitcode for ThinLTO thin link
clang -flto=thin -fthin-link-bitcode=output.thinlto.bc hello.c
```

### LTO + PGO

```bash
# LTO with PGO profiling
clang -flto=thin -fprofile-instr-use=profile.profdata hello.c
```

### Fat LTO Objects

```bash
# Include both machine code and bitcode in object files
clang -ffat-lto-objects hello.c
clang -fno-fat-lto-objects hello.c

# Split LTO unit (for parallel native object compilation)
clang -fsplit-lto-unit hello.c
```

### Unified LTO Pipeline

```bash
# Use unified LTO pipeline (combines ThinLTO + FullLTO pipelines)
clang -funified-lto hello.c
clang -fno-unified-lto hello.c
```

### LTO-Specific Optimizations

```bash
# Whole-program vtable optimization (requires -flto)
clang -flto -fwhole-program-vtables hello.cpp

# Dead virtual function elimination (requires -flto=full)
clang -flto -fvirtual-function-elimination hello.cpp
```

---

## 12. Modules & Precompiled Headers

### C++20 Modules

```bash
# Build a named module
clang -std=c++20 -fmodule-name=mymod -c mymod.cppm -o mymod.pcm

# Use a module file
clang -std=c++20 -fmodule-file=mymod=mymod.pcm main.cpp

# Specify module cache path
clang -std=c++20 -fmodules-cache-path=/tmp/modcache main.cpp

# Build a Header Unit from a header
clang -std=c++20 -fmodule-header=user myheader.h
clang -std=c++20 -fmodule-header=system <system_header>

# Save intermediate module file results
clang -std=c++20 -fmodule-output main.cpp

# Prebuilt module path (look up implicit modules here)
clang -fprebuilt-module-path=/path/to/modules -fprebuilt-implicit-modules main.cpp

# Specify module user build path
clang -fmodules-user-build-path /tmp/modbuild main.cpp
```

### Clang Modules

```bash
# Enable Clang modules (not C++20 modules; uses module.modulemap)
clang -fmodules hello.c

# Enable modules for C++
clang -fcxx-modules hello.cpp

# Load a specific module map file
clang -fmodule-map-file=module.modulemap hello.c

# Module cache path
clang -fmodules-cache-path=/tmp/clang-module-cache hello.c

# Prune module cache: after 7 days of non-use
clang -fmodules-prune-after=604800 hello.c

# Prune interval: check every hour
clang -fmodules-prune-interval=3600 hello.c

# Require explicit module declarations (strict mode)
clang -fmodules-decluse hello.c
clang -fmodules-strict-decluse hello.c    # even stricter: all headers must be in modules

# Validate system headers when loading modules
clang -fmodules-validate-system-headers hello.c

# Ignore specific macros when building/loading modules
clang -fmodules-ignore-macro=DEBUG hello.c
clang -fmodules-ignore-macro=NDEBUG hello.c

# Embed all source files in the module
clang -fmodules-embed-all-files hello.c

# Disable diagnostic validation when loading modules
clang -fmodules-disable-diagnostic-validation hello.c

# Build a system module
clang -fsystem-module -fmodule-name=MySysModule hello.c

# Module-based API notes
clang -fapinotes-modules hello.c
```

### Precompiled Headers (PCH)

```bash
# Generate a PCH file
clang -x c-header prelude.h -o prelude.h.pch

# Use a PCH file
clang -include-pch prelude.h.pch main.c

# Build a relocatable PCH
clang -relocatable-pch -x c-header prelude.h -o prelude.h.pch

# Generate code from PCH (assumes explicit .o will be built)
clang -fpch-codegen -x c-header prelude.h -o prelude.h.pch

# Include debug info for types in the PCH object file
clang -fpch-debuginfo -x c-header prelude.h -o prelude.h.pch

# Instantiate templates while building PCH
clang -fpch-instantiate-templates -x c++-header prelude.h -o prelude.h.pch

# Validate PCH based on content hash (not just mtime)
clang -fpch-validate-input-files-content -include-pch prelude.h.pch main.c
```

---

## 13. GPU / Offloading (CUDA, HIP, OpenMP)

### CUDA

```bash
# Compile CUDA for both host and device (default)
clang --cuda-compile-host-device hello.cu

# Compile CUDA for device only
clang --cuda-device-only hello.cu

# Compile CUDA for host only (ignore device code)
clang --cuda-host-only hello.cu

# Specify CUDA installation path
clang --cuda-path=/usr/local/cuda hello.cu

# Ignore CUDA environment variables
clang --cuda-path-ignore-env hello.cu

# Specify GPU architecture
clang --cuda-gpu-arch=sm_80 hello.cu     # NVIDIA A100
clang --cuda-gpu-arch=sm_89 hello.cu     # RTX 4090

# Include PTX for specific architectures (for forward compatibility)
clang --cuda-include-ptx=sm_80 hello.cu
clang --cuda-include-ptx=all hello.cu

# Exclude PTX for specific architectures
clang --no-cuda-include-ptx=sm_50 hello.cu

# Disable CUDA version check (allow older CUDA for newer GPU arch)
clang --no-cuda-version-check hello.cu

# Path to ptxas assembler
clang --ptxas-path=/usr/local/cuda/bin/ptxas hello.cu

# Device debug without optimization
clang --cuda-noopt-device-debug hello.cu
```

### HIP (AMD GPU)

```bash
# Compile HIP source
clang -x hip hello.hip

# Specify ROCm path
clang --rocm-path=/opt/rocm hello.hip

# Specify HIP path
clang --hip-path=/opt/rocm/hip hello.hip

# Specify HIP version
clang --hip-version=5.5.0 hello.hip

# Emit relocatable device code (separate compilation mode)
clang -fhip-emit-relocatable hello.hip

# Don't use new HIP launch API
clang -fno-hip-new-launch-api hello.hip

# Preserve kernel argument names
clang -fhip-kernel-arg-name hello.hip

# GPU architecture for HIP
clang --offload-arch=gfx908 hello.hip      # AMD MI100
clang --offload-arch=gfx90a hello.hip      # AMD MI200

# Link HIP (bundle device binaries)
clang --hip-link hello.hip

# Don't link HIP runtime (use external runtime)
clang -no-hip-rt hello.hip

# HIP stdpar (parallel algorithm offloading)
clang --hipstdpar hello.cpp \
      --hipstdpar-path=/opt/rocm/lib \
      --hipstdpar-prim-path=/opt/rocm/rocprim \
      --hipstdpar-thrust-path=/opt/rocm/rocthrust

# Interpose alloc/dealloc with HIP managed memory
clang --hipstdpar --hipstdpar-interpose-alloc hello.cpp
```

### OpenMP Offloading

```bash
# Compile with OpenMP offloading to NVIDIA GPU
clang -fopenmp -fopenmp-targets=nvptx64-nvidia-cuda hello.c

# Compile with OpenMP offloading to AMD GPU
clang -fopenmp -fopenmp-targets=amdgcn-amd-amdhsa hello.c

# Multiple offloading targets
clang -fopenmp -fopenmp-targets=nvptx64-nvidia-cuda,amdgcn-amd-amdhsa hello.c

# Mandatory offloading (fail if device code can't be generated)
clang -fopenmp-offload-mandatory hello.c

# Device-side debugging
clang -fopenmp-target-debug hello.c

# JIT compilation for OpenMP offloading
clang -fopenmp-target-jit hello.c

# Offloading with LTO
clang -fopenmp -foffload-lto hello.c
```

### Portable Offloading (LLVM Offload Runtime)

```bash
# Use the new LLVM/Offload portable runtime
clang -foffload-via-llvm hello.c
clang -fno-offload-via-llvm hello.c
```

### General Offloading Options

```bash
# Specify offloading architectures (device targets)
clang --offload-arch=sm_80 hello.c           # CUDA
clang --offload-arch=gfx90a hello.c          # HIP
clang --offload-arch=native hello.c          # auto-detect installed GPUs

# Remove specific architectures from the target list
clang --no-offload-arch=sm_50 hello.c
clang --no-offload-arch=all hello.c

# Offloading include control
clang --offload-inc hello.c                  # add CUDA/HIP include paths (default)
clang --no-offload-inc hello.c               # don't add include paths

# Device library linking
clang --offloadlib hello.c                   # link device libraries
clang --no-offloadlib hello.c                # don't link device libraries

# Compress offload device binaries (HIP only)
clang --offload-compress hello.hip

# Offload compilation parallelism
clang --offload-jobs=4 hello.c

# Use new offloading linker
clang --offload-link hello.c

# New offloading driver
clang --offload-new-driver hello.c

# Embed offloading objects in host binary
clang -fembed-offload-object=mykernel.bc hello.c
```

### GPU-Specific Optimizations

```bash
# Generate relocatable device code
clang -fgpu-rdc hello.cu

# Approximate transcendental functions (GPU)
clang -fgpu-approx-transcendentals hello.cu

# Flush denormals to zero on GPU
clang -fgpu-flush-denormals-to-zero hello.cu

# Default stream behavior (CUDA/HIP)
clang -fgpu-default-stream=legacy hello.cu
clang -fgpu-default-stream=per-thread hello.cu

# Defer GPU diagnostics (suppress until instantiation)
clang -fgpu-defer-diag hello.cu

# Allow device-side init functions (HIP)
clang -fgpu-allow-device-init hello.hip

# CUDA short pointer mode (32-bit pointers for const/local/shared)
clang -fcuda-short-ptr hello.cu

# Uniform block size assumption for kernels (HIP/CUDA)
clang -foffload-uniform-block hello.cu

# Compilation unit ID for device-side statics
clang -cuid=my_id hello.cu
clang -fuse-cuid=hash hello.cu

# Link LLVM libc for GPUs
clang -gpulibc hello.cu

# Sanitize GPU device code
clang -fgpu-sanitize hello.cu
```

---

## 14. OpenCL Options

```bash
# Compile OpenCL kernel
clang -x cl hello.cl

# Set OpenCL language standard
clang -cl-std=CL1.2 hello.cl
clang -cl-std=CL2.0 hello.cl
clang -cl-std=CL3.0 hello.cl

# OpenCL math options
clang -cl-finite-math-only hello.cl
clang -cl-unsafe-math-optimizations hello.cl
clang -cl-fast-relaxed-math hello.cl
clang -cl-no-signed-zeros hello.cl
clang -cl-mad-enable hello.cl

# Denormals flushing
clang -cl-denorms-are-zero hello.cl

# FP32 correctly rounded divide/sqrt
clang -cl-fp32-correctly-rounded-divide-sqrt hello.cl

# Disable all optimizations (by default, OpenCL optimizations are on)
clang -cl-opt-disable hello.cl

# Generate kernel argument metadata
clang -cl-kernel-arg-info hello.cl

# Treat all double constants as single precision
clang -cl-single-precision-constant hello.cl

# Uniform work-group size assumption
clang -cl-uniform-work-group-size hello.cl

# Strict aliasing (OpenCL 1.0 compatibility)
clang -cl-strict-aliasing hello.cl

# Disable standard includes
clang -cl-no-stdinc hello.cl

# Enable/disable OpenCL extensions
clang -cl-ext=+cl_khr_fp64 hello.cl
clang -cl-ext=-cl_khr_fp16 hello.cl
```

---

## 15. OpenMP Options

```bash
# Enable OpenMP
clang -fopenmp hello.c

# Set OpenMP version
clang -fopenmp -fopenmp-version=45 hello.c   # OpenMP 4.5
clang -fopenmp -fopenmp-version=51 hello.c   # OpenMP 5.1 (default)

# Enable all Clang OpenMP extensions
clang -fopenmp-extensions hello.c

# SIMD-only (don't parallelize, just SIMD)
clang -fopenmp-simd hello.c

# Force unified shared memory behavior
clang -fopenmp-force-usm hello.c

# Link with static OpenMP runtime
clang -fopenmp -static-openmp hello.c

# OpenMP offloading targets
clang -fopenmp -fopenmp-targets=nvptx64-nvidia-cuda hello.c

# Pass arguments to OpenMP offloading toolchain
clang -Xopenmp-target=nvptx64-nvidia-cuda -march=sm_80 hello.c
clang -Xopenmp-target -O3 hello.c
```

---

## 16. HLSL / DirectX / SPIR-V Options

### HLSL (High-Level Shader Language)

```bash
# Compile HLSL shader
clang -x hlsl shader.hlsl

# Specify entry point
clang -hlsl-entry main shader.hlsl

# Enable strict availability diagnostics for HLSL built-in functions
clang -fhlsl-strict-availability shader.hlsl

# All resources bound (aggressive flattening - DXC compatibility)
clang /hlsl-all-resources-bound shader.hlsl

# Override root signature with a macro
clang -fdx-rootsignature-define=MY_RS shader.hlsl

# Set root signature version
clang -fdx-rootsignature-version=1.1 shader.hlsl
```

### SPIR-V

```bash
# Compile to SPIR-V
clang -target spirv-unknown-unknown shader.hlsl

# Set SPIR-V extensions availability
clang -fspv-extension=KHR shader.hlsl
clang -fspv-extension=DXC shader.hlsl

# Set SPIR-V target environment
clang -fspv-target-env=vulkan1.2 shader.hlsl
clang -fspv-target-env=opencl shader.hlsl
```

---

## 17. Objective-C / Objective-C++ Options

```bash
# Enable Automatic Reference Counting (ARC)
clang -fobjc-arc hello.m

# Enable ARC-style weak references
clang -fobjc-weak hello.m

# ARC with exception-safe retain/release
clang -fobjc-arc-exceptions hello.m

# Specify Objective-C runtime
clang -fobjc-runtime=macosx-10.15.0 hello.m
clang -fobjc-runtime=ios-14.0 hello.m
clang -fobjc-runtime=gnustep hello.m

# Use GNU Objective-C runtime
clang -fgnu-runtime hello.m

# Enable Objective-C exceptions
clang -fobjc-exceptions hello.m

# Disable direct methods for testing
clang -fobjc-disable-direct-methods-for-testing hello.m
```

---

## 18. SYCL Options

```bash
# Enable SYCL C++ extensions
clang -fsycl hello.cpp

# Compile for SYCL device only
clang -fsycl-device-only hello.cpp

# Compile for SYCL host only
clang -fsycl-host-only hello.cpp

# Set SYCL language standard
clang -sycl-std=2020 hello.cpp

# Don't link SYCL runtime
clang -nolibsycl hello.cpp
```

---

## 19. Target-Specific Options

For brevity, this section lists the most commonly used target-specific flags.
For exhaustive per-target register reservation options (e.g., `-ffixed-x0` through
`-ffixed-x31` for ARM, RISC-V, M68k, SPARC), see `clang --help` output directly.

### AArch64-Specific

```bash
clang -mbranch-protection=standard hello.c     # PAC+BTI
clang -msve-vector-bits=256 hello.c
clang -fptrauth-returns hello.c
clang -msign-return-address=all hello.c
clang -msign-return-address=non-leaf hello.c
clang -mno-bti-at-return-twice hello.c
clang -mlr-for-calls-only hello.c
clang -mfix-cortex-a53-835769 hello.c          # erratum workaround
clang -mfix-cortex-a53-843419 hello.c
clang -mtls-size=32 hello.c
clang -mgeneral-regs-only hello.c
clang -mharden-sls=all hello.c
```

### ARM-Specific

```bash
clang -mcmse hello.c                            # ARMv8-M security
clang -mexecute-only hello.c
clang -mrestrict-it hello.c                     # restrict IT block complexity
clang -mno-movt hello.c                         # no movt/movw pairs
clang -mfix-cortex-a57-aes-1742098 hello.c     # erratum workaround
clang -mfix-cortex-a72-aes-1655431 hello.c
clang -mfix-cmse-cve-2021-35465 hello.c
clang -fropi hello.c                            # read-only position independent
clang -frwpi hello.c                            # read-write position independent
clang -meabi=5 hello.c                          # ARM EABI version
```

### RISC-V-Specific

```bash
clang -march=rv64gc hello.c
clang -mabi=lp64 hello.c
clang -mrelax hello.c                           # linker relaxation (default)
clang -mno-relax hello.c
clang -menable-experimental-extensions hello.c
clang -msave-restore hello.c                    # use library save/restore
clang -mrvv-vector-bits=256 hello.c            # RVV vector register width
clang -mstrict-align hello.c
clang -mtls-dialect=trad hello.c
clang -mtls-dialect=desc hello.c
```

### x86-Specific

```bash
clang -miamcu hello.c                           # Intel MCU ABI
clang -mfunction-return=thunk-extern hello.c    # retpoline-like returns
clang -mindirect-branch-cs-prefix hello.c
clang -mskip-rax-setup hello.c
clang -mno-gather hello.c                       # disable gather instructions
clang -mno-scatter hello.c
clang -malign-branch=+jcc+fused+jmp hello.c
clang -malign-branch-boundary=32 hello.c
clang -mbranches-within-32B-boundaries hello.c
clang -mapx-features=+egpr hello.c             # Intel APX extensions
clang -mharden-sls=all hello.c
clang -mstackrealign hello.c
clang -mrtd hello.c                             # stdcall default calling convention
clang -mregparm=3 hello.c                       # pass 3 args in registers
clang -mssse3 hello.c -mavx hello.c -mavx2 hello.c -mavx512f hello.c
clang -msha hello.c -maes hello.c
```

### PowerPC-Specific

```bash
clang -mcrbits hello.c
clang -maltivec hello.c
clang -msvr4-struct-return hello.c              # PPC32
clang -maix-struct-return hello.c               # PPC32
clang -maix-use-ptrgl hello.c                   # AIX
clang -maix-small-local-dynamic-tls hello.c     # AIX
clang -maix-small-local-exec-tls hello.c        # AIX
clang -mtocdata hello.c
```

### AMDGPU-Specific

```bash
clang -mcpu=gfx90a hello.hip
clang -mcode-object-version=6 hello.hip
clang -mcumode hello.hip                        # CU execution mode
clang -mwavefrontsize64 hello.hip
clang -mtgsplit hello.hip
clang -mprintf-kind=buffered hello.hip
clang -mamdgpu-ieee hello.hip
clang -mamdgpu-precise-memory-op hello.hip
```

### WebAssembly-Specific

```bash
clang -target wasm32-unknown-unknown hello.c
clang -mexec-model=reactor hello.c
```

---

## 20. Include & Library Path Management

### Include Search Paths

```bash
# Standard include search (-I): prepended to search path
clang -I./headers -I/opt/third-party/include main.c

# System include (-isystem): suppresses warnings from these headers
clang -isystem /usr/local/include main.c

# Add after standard system paths (-idirafter): fallback search
clang -idirafter /fallback/include main.c

# Quote search (for #include "..." only)
clang -iquote ./project-headers main.c

# Framework search (macOS)
clang -F /Library/Frameworks main.c
clang -iframework /System/Library/Frameworks main.c

# Framework with sysroot (macOS)
clang -iframeworkwithsysroot /System/Library/Frameworks -isysroot /sdk main.c

# Search with prefix (-iwithprefix and -iwithprefixbefore)
clang -iprefix /usr -iwithprefixbefore include main.c   # adds /usr/include
clang -iprefix /usr -iwithprefix include main.c         # adds /usr/include to SYSTEM

# Search with sysroot (absolute paths relative to -isysroot)
clang -iwithsysroot /usr/include -isysroot /sdk main.c  # adds /sdk/usr/include

# C++ system include
clang -cxx-isystem /opt/custom-stdlib/include main.cpp
clang -stdlib++-isystem /opt/custom-stdlib/include main.cpp
```

### Library Search Paths

```bash
# Add directory to library search path (-L)
clang main.c -L/usr/local/lib -lfoo
```

### System Root

```bash
# Set system root (base path for system headers and libraries)
clang -isysroot /path/to/sdk main.c
```

### Standard Include Control

```bash
# Disable builtin includes (e.g., compiler-rt headers)
clang -nobuiltininc main.c

# Re-enable builtin includes after -nostdinc
clang -nostdinc -ibuiltininc main.c

# Disable all standard includes
clang -nostdinc main.c

# Disable standard library includes only
clang -nostdlibinc main.c

# Disable C++ standard library includes
clang -nostdinc++ main.cpp
```

### System Header Control

```bash
# Treat paths starting with prefix as system headers
clang --system-header-prefix=boost/ main.cpp

# Treat paths starting with prefix as NOT system headers
clang --no-system-header-prefix=boost/ main.cpp
```

---

## 21. Linking

### Basic Linking

```bash
# Link object files
clang a.o b.o c.o -o program

# Link against libraries
clang main.o -lm -lpthread -lz

# Specify linker script
clang -T link.ld main.o
```

### Passing Arguments to Linker

```bash
# Pass comma-separated arguments to the linker
clang -Wl,--gc-sections main.o
clang -Wl,-rpath,/usr/local/lib main.o
clang -Wl,-Map,output.map main.o

# Pass single argument to linker
clang -Xlinker --no-undefined main.o
```

### Linker Selection & Runtime

```bash
# Specify C++ standard library
clang -stdlib=libc++ main.cpp          # LLVM libc++
clang -stdlib=libstdc++ main.cpp       # GNU libstdc++

# Compile with pthread support
clang -pthread main.c

# Specify compiler runtime library
clang -rtlib=compiler-rt main.c

# Specify unwind library
clang -unwindlib=libunwind main.cpp

# Emit a static library
clang --emit-static-lib a.o b.o -o libmylib.a
```

### macOS-Specific Linking

```bash
# Link against Apple frameworks
clang main.m -framework Foundation -framework AppKit

# Set deployment target
clang -mmacos-version-min=11.0 main.c
clang -mios-version-min=14.0 main.c
clang -mtargetos=macos12.0 main.c

# Darwin target variant (for Catalyst / iOS-on-Mac etc.)
clang -darwin-target-variant x86_64-apple-ios14.0-macabi main.c
clang -darwin-target-variant-triple x86_64-apple-ios14.0-macabi main.c
```

### AIX-Specific

```bash
# Pass argument to the AIX linker
clang -b maxdata:0x80000000 main.c
```

---

## 22. Symbol Visibility & ABI

### Default Visibility

```bash
# Set default visibility for all global symbols
clang -fvisibility=hidden hello.c       # everything hidden by default
clang -fvisibility=default hello.c      # everything visible (default)

# Inlines get hidden visibility
clang -fvisibility-inlines-hidden hello.cpp

# Static variables in inline functions also get hidden
clang -fvisibility-inlines-hidden-static-local-var hello.cpp

# MS-compatible visibility (types default, functions hidden)
clang -fvisibility-ms-compat hello.cpp
```

### DLL Storage Class (Windows)

```bash
# Override visibility based on DLL storage class
clang -fvisibility-from-dllstorageclass hello.cpp

# Control visibility for dllexport definitions
clang -fvisibility-dllexport=Keep hello.cpp
clang -fvisibility-dllexport=Hidden hello.cpp

# Control visibility for dllimport extern declarations
clang -fvisibility-externs-dllimport=Keep hello.cpp

# Control visibility for definitions without DLL storage class
clang -fvisibility-nodllstorageclass=Keep hello.cpp

# Control visibility for externs without DLL storage class
clang -fvisibility-externs-nodllstorageclass=Keep hello.cpp
```

### C++ ABI

```bash
# Override C++ ABI
clang -fc++-abi=itanium hello.cpp       # Itanium ABI (Linux/macOS)
clang -fc++-abi=microsoft hello.cpp     # Microsoft ABI (Windows)

# Match Clang version ABI compatibility
clang -fclang-abi-compat=14 hello.cpp

# Enable C++17 aligned allocation
clang -faligned-allocation hello.cpp

# Enable C++14 sized deallocation
clang -fsized-deallocation hello.cpp
```

### Struct Return Conventions

```bash
# Return all structs on the stack (override default ABI)
clang -fpcc-struct-return hello.c

# Return small structs in registers (override default ABI)
clang -freg-struct-return hello.c
```

### Struct Layout

```bash
# Set maximum struct packing alignment
clang -fpack-struct=4 hello.c

# Randomize struct layout seed (security hardening)
clang -frandomize-layout-seed=42 hello.c
clang -frandomize-layout-seed-file=seed.txt hello.c
```

### Short Enums / WChar

```bash
# Smallest possible enum size
clang -fshort-enums hello.c

# Override wchar_t size
clang -fshort-wchar hello.c             # short unsigned int
clang -fno-short-wchar hello.c          # unsigned int
```

### Run-Time Type Info (RTTI)

```bash
# Enable RTTI (default)
# (RTTI enabled by default for C++)

# Disable RTTI
clang -fno-rtti hello.cpp

# Disable generation of RTTI data
clang -fno-rtti-data hello.cpp
```

### Exception Handling

```bash
# Enable C++ exceptions (default)
# (enabled by default for C++)

# Disable C++ exceptions
clang -fno-exceptions hello.cpp

# Use DWARF-style exception handling
clang -fdwarf-exceptions hello.cpp

# Use SEH-style exception handling (Windows)
clang -fseh-exceptions hello.cpp

# Use SjLj exception handling
clang -fsjlj-exceptions hello.cpp

# Use WebAssembly exception handling
clang -fwasm-exceptions hello.c

# Async exceptions
clang -fasync-exceptions hello.cpp

# Assume exception destructors are non-throwing
clang -fassume-nothrow-exception-dtor hello.cpp

# Ignore exception constructs (generate non-exception code)
clang -fignore-exceptions hello.cpp
```

### Stack Protector

```bash
# Enable stack protector (heuristic: functions with char arrays > 8 bytes)
clang -fstack-protector hello.c

# Stronger heuristic: functions with any arrays, any alloca calls
clang -fstack-protector-strong hello.c

# All functions
clang -fstack-protector-all hello.c

# Disable stack protector
clang -fno-stack-protector hello.c
```

### Stack Clash Protection

```bash
clang -fstack-clash-protection hello.c
clang -fno-stack-clash-protection hello.c
```

### Control Flow Guard (Windows)

```bash
clang -fcf-protection hello.c             # enable in full mode
clang -fcf-protection=full hello.c
clang -fcf-protection=branch hello.c
clang -fcf-protection=return hello.c

# Windows-specific CFG mechanism
clang -fwin-cfg-mechanism=default hello.c
```

### Register Reservation

(reserving registers for special use — OS kernels, hypervisors, GC)

```bash
# Reserve specific registers (architecture-specific)
clang -ffixed-x18 hello.c                # AArch64: platform register
clang -ffixed-r9 hello.c                 # ARM: platform register
clang -ffixed-r10 hello.c                # x86_64
clang -ffixed-edi hello.c                # x86
```

---

## 23. Security Hardening

### Pointer Authentication (AArch64)

```bash
# Authenticate return addresses
clang -fptrauth-returns hello.c

# Authenticate indirect calls
clang -fptrauth-calls hello.c

# Authenticate indirect goto targets
clang -fptrauth-indirect-gotos hello.c

# Sign init/fini array function pointers
clang -fptrauth-init-fini hello.c

# Authenticate GOT entries (ELF only)
clang -fptrauth-elf-got hello.c

# ObjC-specific pointer authentication
clang -fptrauth-objc-isa hello.c
clang -fptrauth-objc-interface-sel hello.c
clang -fptrauth-objc-class-ro hello.c

# VTable pointer authentication
clang -fptrauth-vtable-pointer-address-discrimination hello.cpp
clang -fptrauth-vtable-pointer-type-discrimination hello.cpp

# Block descriptor authentication
clang -fptrauth-block-descriptor-pointers hello.c

# Enable pointer auth intrinsics
clang -fptrauth-intrinsics hello.c

# Trap on authentication failure (debug mode)
clang -fptrauth-auth-traps hello.c
```

### Straight-Line Speculation (SLS) Hardening

```bash
clang -mharden-sls=all hello.c
clang -mharden-sls=return hello.c         # x86
clang -mharden-sls=retbr hello.c          # ARM/AArch64
clang -mharden-sls=blr hello.c            # ARM/AArch64
clang -mharden-sls=comdat hello.c         # ARM/AArch64
```

### LVI (Load Value Injection) Mitigations

```bash
clang -mlvi-hardening hello.c            # all LVI mitigations
clang -mlvi-cfi hello.c                  # control-flow only
```

### Speculative Execution Side Effect Suppression (SESES)

```bash
clang -mseses hello.c                     # includes LVI CFI mitigations
clang -mno-seses hello.c
```

### Branch Protection, Return Address Signing

```bash
# AArch64: combine PAC and BTI
clang -mbranch-protection=standard hello.c

# Return address signing scope
clang -msign-return-address=all hello.c
clang -msign-return-address=non-leaf hello.c
clang -msign-return-address=none hello.c
```

### Null Pointer Checks

```bash
# Assume null pointer dereference is undefined behavior (default)
clang -fdelete-null-pointer-checks hello.c

# Don't optimize based on null pointer UB assumption
clang -fno-delete-null-pointer-checks hello.c
```

### Integer Overflow

```bash
# Treat signed overflow as undefined behavior (default for optimization)
# (C standard: signed overflow is UB)

# Wrap signed overflow (two's complement behavior)
clang -fwrapv hello.c

# Wrap pointer overflow (treat pointer overflow as two's complement)
clang -fwrapv-pointer hello.c

# Trap on signed integer overflow (debugging)
clang -ftrapv hello.c

# Specify custom trap handler
clang -ftrapv -ftrapv-handler=my_overflow_handler hello.c
```

### Stack Safety & SafeStack

```bash
# SafeStack: split safe stack (spills) from unsafe stack (allocas)
clang -fsanitize=safe-stack hello.c

# Shadow stack for GC
clang -fsanitize=shadow-call-stack hello.c        # AArch64 only
```

### Forensics & Crash Diagnostics

```bash
# Enable crash diagnostic reporting (generate .sh + .i on crash)
clang -fcrash-diagnostics hello.c

# Set crash diagnostics level
clang -fcrash-diagnostics=compiler hello.c   # compiler crashes
clang -fcrash-diagnostics=all hello.c        # all crashes

# Specify crash report directory
clang -fcrash-diagnostics-dir=/tmp/clang-crashes hello.c

# Disable crash diagnostics
clang -fno-crash-diagnostics hello.c
```

---

## 24. C++ Specific Options

### C++ Standard Types & Features

```bash
# Enable char8_t (C++20)
clang -std=c++20 -fchar8_t hello.cpp

# Enable C++20 Coroutines
clang -std=c++20 -fcoroutines hello.cpp

# Prefer aligned allocation for coroutine frames
clang -fcoro-aligned-allocation hello.cpp

# Enable C++26 reflection
clang -freflection hello.cpp

# Enable fixed point types
clang -ffixed-point hello.cpp
```

### Constructor / Destructor

```bash
# Disable copy constructor elision (C++17 mandates elision in some cases)
clang -fno-elide-constructors hello.cpp

# Register global destructors with __cxa_atexit (default)
clang -fregister-global-dtors-with-atexit hello.cpp

# Don't use __cxa_atexit (use atexit instead)
clang -fno-use-cxa-atexit hello.cpp

# Thread-safe local static initialization (default in C++11+)
clang -fno-threadsafe-statics hello.cpp
```

### Virtual Table Optimization

```bash
# Assume unique vtables (enables devirtualization)
clang -fno-assume-unique-vtables hello.cpp

# Strict vtable pointer optimization
clang -fstrict-vtable-pointers hello.cpp

# Force emit vtables (improves devirtualization)
clang -fforce-emit-vtables hello.cpp

# Experimental relative C++ ABI vtables
clang -fexperimental-relative-c++-abi-vtables hello.cpp
```

### C++ `operator new`

```bash
# Don't assume operator new can't alias other pointers
clang -fno-assume-sane-operator-new hello.cpp

# operator new may return NULL (don't assume non-null)
clang -fcheck-new hello.cpp

# Largest alignment guaranteed by ::operator new
clang -fnew-alignment=32 hello.cpp

# Treat throwing new as infallible (returns_nonnull + throw())
clang -fnew-infallible hello.cpp
clang -fno-new-infallible hello.cpp

# Global new/delete visibility hidden
clang -fvisibility-global-new-delete-hidden hello.cpp
```

### Operator Names & Member Pointers

```bash
# Treat C++ operator keywords ('and', 'or', 'not') as synonyms
clang -foperator-names hello.cpp

# Disallow 'and', 'or', 'not' as operator synonyms
clang -fno-operator-names hello.cpp

# Require complete types for member pointers (MSVC ABI)
clang -fcomplete-member-pointers hello.cpp

# Max depth for operator-> chain resolution
clang -foperator-arrow-depth=10 hello.cpp
```

### Template Options

```bash
# Limit template instantiation depth
clang -ftemplate-depth=256 hello.cpp

# Limit template backtrace entries in diagnostics
clang -ftemplate-backtrace-limit=5 hello.cpp
```

### Access Control

```bash
# Disable C++ access control (public/private/protected)
clang -fno-access-control hello.cpp
```

### C++ Static Destructors

```bash
# Control static destructor registration
clang -fc++-static-destructors=none hello.cpp
clang -fc++-static-destructors=all hello.cpp
clang -fno-c++-static-destructors hello.cpp
```

### Enum Optimizations

```bash
# Strict enum value range assumption for optimization
clang -fstrict-enums hello.cpp
```

### Flexible Array Strictness

```bash
# Control strict flexible array member optimization
clang -fstrict-flex-arrays=0 hello.c       # least strict
clang -fstrict-flex-arrays=1 hello.c
clang -fstrict-flex-arrays=2 hello.c
clang -fstrict-flex-arrays=3 hello.c       # most strict
```

### Bool Optimization

```bash
# Optimize based on bool ∈ {0, 1} invariant
clang -fstrict-bool hello.cpp
clang -fno-strict-bool hello.cpp

# Specify bool conversion behavior
clang -fno-strict-bool=truncation hello.cpp
clang -fno-strict-bool=none hello.cpp
```

---

## 25. Developer & Debugging Options

### Compilation Timing

```bash
# Time each compilation phase
clang -ftime-report hello.c               # summary
clang -ftime-report=per-pass hello.c      # per-pass timing

# Generate chrome://tracing JSON trace
clang -ftime-trace hello.c                # output to <output>.json

# Specify output file for time trace
clang -ftime-trace=/tmp/trace.json hello.c

# Set time trace granularity (microseconds)
clang -ftime-trace-granularity=100 hello.c

# Verbose time trace (include file names — 2-3x larger output)
clang -ftime-trace-verbose hello.c
```

### LLVM Statistics

```bash
# Save LLVM statistics (pass counts, transforms, etc.)
clang -save-stats hello.c
clang -save-stats=/tmp/stats hello.c
```

### LLVM IR Verification

```bash
# Run the LLVM IR verifier after each pass
clang -fverify-intermediate-code hello.c

# Disable verification (faster compilation)
clang -fno-verify-intermediate-code hello.c
```

### Driver Debugging

```bash
# Dry run: print commands without executing
clang -### hello.c -O2 -Wall

# Only run driver, don't invoke cc1
clang -fdriver-only hello.c

# Verbose output (show every command)
clang -v hello.c

# Load pass plugin
clang -fpass-plugin=./MyPass.so hello.c

# Load general plugin
clang -fplugin=./MyPlugin.so hello.c

# Pass arguments to plugin
clang -fplugin=./MyPlugin.so -fplugin-arg-MyPlugin-option=value hello.c
```

### AST & PCH Verification

```bash
# Verify precompiled header freshness
clang -verify-pch prelude.h.pch

# Validate AST input files by content hash
clang -fvalidate-ast-input-files-content hello.c
```

### Module Debugging

```bash
# Show information about a module file
clang -module-file-info mymodule.pcm

# Directory to dump module dependencies
clang -module-dependency-dir /tmp/moddeps hello.cpp
```

### Incremental Processing

```bash
# Enable incremental processing extensions (global-scope statements)
clang -fincremental-extensions hello.cpp
```

### API Extraction

```bash
# Extract API information during compilation
clang -extract-api hello.h

# Emit symbol graphs
clang -emit-symbol-graph hello.h

# Generate extended module symbol graphs
clang --emit-extension-symbol-graphs hello.h

# Output directory for symbol graphs
clang --symbol-graph-dir=/tmp/graphs hello.h

# Ignore specific API symbols
clang --extract-api-ignores=ignores.txt hello.h
```

### Reproducer Generation

```bash
# Auto-generate reproducer on crash/error
clang -gen-reproducer=crash hello.c       # on crash (default)
clang -gen-reproducer=error hello.c       # on error
clang -gen-reproducer=always hello.c      # always
clang -gen-reproducer=off hello.c         # never

# Set reproducer output prefix
clang -dumpdir /tmp/repro hello.c         # auxiliary files go here
```

### Configuration File

```bash
# Specify a config file (clang.cfg format)
clang --config=my-clang.cfg hello.c

# Don't load default config files
clang --no-default-config hello.c
```

### Pass Plugin

```bash
# Load a pass plugin (new pass manager only)
clang -fpass-plugin=./MyCustomPass.so hello.c
```

### Specialized Option Printing

```bash
# Print the effective target triple
clang -print-effective-triple

# Print the resource directory (where compiler-rt, headers live)
clang -print-resource-dir

# Print search paths for libraries and programs
clang -print-search-dirs

# Print library path for libgcc / compiler-rt builtins
clang -print-libgcc-file-name

# Print C++ Standard Library module manifest path
clang -print-library-module-manifest-path

# Print full path to a program (e.g., ld, as)
clang -print-prog-name=ld

# Print the full library path for a library
clang -print-file-name=libc.a
```

---

## 26. Miscellaneous

### Environment & Freestanding

```bash
# Freestanding environment (no standard library assumed)
clang -ffreestanding hello.c

# Set Fuchsia API level
clang -ffuchsia-api-level=15 hello.c
```

### Thread Model

```bash
# Set thread model
clang -mthread-model posix hello.c
clang -mthread-model single hello.c
```

### Apple Kext (Kernel Extension)

```bash
# Use Apple kernel extension ABI
clang -fapple-kext hello.c
```

### App Extension Restriction

```bash
# Restrict to APIs available in App Extensions (iOS/macOS)
clang -fapplication-extension hello.m
```

### API Notes (Apple)

```bash
# Enable API notes
clang -fapinotes hello.m

# Specify Swift version for API notes filtering
clang -fapinotes-swift-version=5.0 hello.m

# Disable API notes
clang -fno-apinotes hello.m
```

### Address Significance Table

```bash
# Emit an address significance table (for linker GC of globals)
clang -faddrsig hello.c

# Don't emit address significance table
clang -fno-addrsig hello.c
```

### Common Block

```bash
# Place uninitialized globals in a common block (traditional C behavior)
clang -fcommon hello.c

# Compile common globals like normal definitions (stronger, modern default)
clang -fno-common hello.c
```

### String Literals

```bash
# Store string literals in writable data section (non-standard)
clang -fwritable-strings hello.c

# Pascal-style string literals ("\pHello")
clang -fpascal-strings hello.c
```

### K&R C Function Declarations

```bash
# Disable support for K&R-style function declarations
clang -fno-knr-functions hello.c
```

### Char Signedness

```bash
# char is signed (default on most platforms)
clang -fsigned-char hello.c

# char is unsigned
clang -fno-signed-char hello.c

# short wchar_t
clang -fshort-wchar hello.c
```

### Function Instrumentation

```bash
# Instrument function entry and exit (calls __cyg_profile_func_enter/exit)
clang -finstrument-functions hello.c

# Insert instrumentation calls after inlining
clang -finstrument-functions-after-inlining hello.c

# Instrument function entry only, no arguments to the call
clang -finstrument-function-entry-bare hello.c

# mcount-style instrumentation (gprof compatible)
clang -pg hello.c                      # with gmon prof
clang -p hello.c                       # with prof
```

### Frame & Unwind

```bash
# Always emit a debug frame section (.debug_frame)
clang -fforce-dwarf-frame hello.c

# Emit compact unwind for non-canonical entries (Darwin)
clang -femit-compact-unwind-non-canonical hello.c

# Control when DWARF unwind (EH frame) info is emitted
clang -femit-dwarf-unwind=always hello.c
clang -femit-dwarf-unwind=no-compact-unwind hello.c
```

### Stack Size Information

```bash
# Emit .stack_sizes section with function stack usage
clang -fstack-size-section hello.c

# Emit .su files with function stack usage information
clang -fstack-usage hello.c
```

### Code Alignment

```bash
# Align loops to a boundary
clang -falign-loops=16 hello.c

# Preferred function alignment
clang -fpreferred-function-alignment=64 hello.c

# Set maximum type alignment
clang -fmax-type-align=16 hello.c
```

### Basic Block Sections

```bash
# Place basic blocks in unique sections (ELF Only)
clang -fbasic-block-sections=all hello.c
clang -fbasic-block-sections=list=list.txt hello.c

# Emit basic block address map section
clang -fbasic-block-address-map hello.c

# Unique names for basic block sections
clang -funique-basic-block-section-names hello.c
```

### Split Machine Functions

```bash
# Enable splitting functions based on PGO (x86/AArch64 ELF)
clang -fsplit-machine-functions hello.c

# Disable function splitting
clang -fno-split-machine-functions hello.c
```

### Partition Static Data Sections

```bash
# Enable partition of static data sections (PGO+MemProf, x86/AArch64 ELF)
clang -fpartition-static-data-sections hello.c
```

### Emulated TLS

```bash
# Use emulated TLS (for targets without native TLS support)
clang -femulated-tls hello.c
```

### Large Code Model Support

```bash
# Use segmented stack (for large stack frames)
clang -fsplit-stack hello.c
```

### BSS & Zero-Initialized Data

```bash
# Don't place zero-initialized data in BSS
clang -fno-zero-initialized-in-bss hello.c
```

### Direct Access External Data

```bash
# Don't use GOT indirection for external data (direct access)
clang -fdirect-access-external-data hello.c

# Use GOT indirection
clang -fno-direct-access-external-data hello.c
```

### Unique Section Names

```bash
# Don't use unique names for text/data sections
clang -fno-unique-section-names hello.c

# Use separate unique sections for named sections (ELF Only)
clang -fseparate-named-sections hello.c
```

### PLT / GOT Control

```bash
# Use GOT indirection instead of PLT for external calls (x86 only)
clang -fno-plt hello.c
```

### Autolink Directives

```bash
# Disable generation of linker directives for automatic library linking
clang -fno-autolink hello.c
```

### Container Binary Version

```bash
# Set minimum binutils version for ELF features
clang -fbinutils-version=2.35 hello.c
```

### Temp Files

```bash
# Create output files directly (no temp file + rename)
clang -fno-temp-file hello.c
# Warning: may produce corrupted output if compiler crashes
```

### Save Temps

```bash
# Save intermediate files (.i, .bc, .s) in current directory
clang -save-temps hello.c

# Save temps in output directory
clang -save-temps=obj hello.c

# Save temps in current directory
clang -save-temps=cwd hello.c
```

### Disable Pragma Pack

```bash
# Apple gcc-compatible #pragma pack handling
clang -fapple-pragma-pack hello.c

# IBM XL #pragma pack handling
clang -fxl-pragma-pack hello.c
```

---

## 27. Pass-Through Options

### General Pass-Through

```bash
# Pass arguments to the Clang -cc1 invocation (the C++ frontend)
clang -Xclang -disable-llvm-passes hello.c
clang -Xclang -print-stats hello.c
clang -Xclang -fdump-record-layouts hello.cpp

# Pass arguments to clang -cc1as (integrated assembler)
clang -Xclangas -mllvm -print-after-all hello.c
```

### Tool-Specific Pass-Through

```bash
# Pass args to the assembler
clang -Wa,-march=armv8.5-a hello.c
clang -Xassembler --gdwarf-5 hello.c

# Pass args to the linker
clang -Wl,-rpath,/usr/local/lib main.o
clang -Xlinker --wrap=malloc main.o

# Pass args to the preprocessor
clang -Wp,-DDEBUG,-I/tmp hello.c
clang -Xpreprocessor -dM hello.c

# Pass args to the offload compilers/linkers
clang -Xoffload-compiler -O3 hello.c
clang -Xoffload-linker -rpath,/opt/rocm/lib hello.c
clang -Xoffload-compiler=nvptx64-nvidia-cuda -O3 hello.c
clang -Xoffload-linker=amdgcn-amd-amdhsa -L/path hello.c

# Pass args to the static analyzer
clang -Xanalyzer -analyzer-output=text hello.c

# Pass args to CUDA fatbinary
clang -Xcuda-fatbinary --compress-all hello.cu

# Pass args to ptxas
clang -Xcuda-ptxas -v hello.cu

# Pass args to architecture-specific compilation
clang -Xarch_x86_64 -mavx2 hello.c
clang -Xarch_device -O3 hello.cu
clang -Xarch_host -O0 hello.cu
```

### Pass z Arguments to Linker

```bash
clang -z defs main.o
clang -z relro main.o
clang -z now main.o
```

---

## Option Lookup Reference

For quick lookups, options are grouped by prefix:

| Prefix | Category |
|--------|----------|
| `-f` | Feature flags (sanitizers, optimizations, language options) |
| `-m` | Target/machine-specific options |
| `-g` | Debug information |
| `-W` | Warnings |
| `-O` | Optimization level |
| `-I` / `-L` / `-F` | Include/library/framework paths |
| `-D` / `-U` | Macro define/undefine |
| `-X` | Pass arguments to sub-tools |
| `-std=` | Language standard |
| `-stdlib=` / `-rtlib=` | Standard/compiler-rt library |
| `-target` / `--target=` | Target triple |
| `-print-*` | Print configuration info (targets, paths, CPUs) |
| `-cl-*` | OpenCL options |
| `-fopenmp*` / `-fopenacc*` | OpenMP / OpenACC |
| `--cuda*` / `-fgpu*` / `--hip*` | GPU offloading |
| `-fsanitize*` | Sanitizer instrumentation |
| `-fprofile*` / `-fcoverage*` | Profiling and coverage |
| `-fmodules*` / `-fmodule*` | Modules and precompiled headers |
| `-fxray*` | XRay instrumentation |

---

*Based on `clang-23 --help` output. For the most up-to-date information, run*
`clang --help` *on your system.*
