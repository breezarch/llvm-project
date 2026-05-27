# 05 — Target Backends

LLVM supports **25 target architectures**.  Each lives in its own directory
under `llvm/lib/Target/<Name>/` and produces a CMake target `LLVM<Name>`.

Every backend provides:
- **TargetMachine** — the entry point; creates the code generation pipeline
- **Instruction definitions** (`.td` files) — TableGen descriptions of the ISA
- **AsmParser** — parse text assembly
- **AsmPrinter** — print assembly from MachineInstr/MCInst
- **Disassembler** — decode bytes to MCInst
- **TargetLowering** — legalize types and operations for this target
- **TargetRegisterInfo** — register file description
- **TargetInstrInfo** — instruction properties (latency, side effects, etc.)
- **TargetFrameLowering** — stack frame layout, prolog/epilog
- **TargetSubtargetInfo** — CPU variants and features

---

## Tier 1 Backends (Production Quality)

These are the most heavily tested and maintained backends.

### X86 (`lib/Target/X86/`) — `LLVMX86`

**Covers**: x86-32 (i386), x86-64 (AMD64), MMX, SSE, AVX, AVX2, AVX-512,
FMA, BMI, AES-NI, SHA, SGX, CET, AMX (Advanced Matrix Extensions)

x86 is a **CISC** (Complex Instruction Set Computer) architecture with:
- Variable-length instructions (1–15 bytes)
- Register-memory operations (not strictly load-store)
- 8 GPRs (x86-32) / 16 GPRs (x86-64) + 8/16 XMM + 8/16 YMM + 32 ZMM
- Legacy encoding, VEX encoding, EVEX encoding (AVX-512)
- Complex addressing modes (base + index * scale + displacement)

**LLVM specifics**:
- The code generator handles x86's complex addressing modes via
  `X86DAGToDAGISel::Select()` which does custom C++ matching for LEA,
  addressing modes, etc.
- AVX-512 masking and broadcast are handled as target-specific DAG nodes
- The backend has extensive peephole optimization in `X86InstrInfo.cpp`

**Compiler-science context**: The x86 ISA is arguably the most complex
compiler target in existence (with the possible exception of IBM z/Architecture).
It is the canonical example of a **CISC** ISA — rich addressing modes,
micro-coded instructions, and a decode bottleneck that makes code size
critical for performance (because smaller code fits in the L1 instruction
cache).

### AArch64 (`lib/Target/AArch64/`) — `LLVMAArch64`

**Covers**: ARM 64-bit (A64 ISA), AArch32 compatibility, NEON SIMD, SVE,
SVE2, SME (Streaming Matrix Extensions), Pointer Authentication (PAC),
Branch Target Identification (BTI), Memory Tagging Extension (MTE)

AArch64 is a **RISC** (Reduced Instruction Set Computer) architecture:
- Fixed 32-bit instruction width
- 31 GPRs + 32 FPR/vector registers (128-bit, extendable via SVE)
- Load-store architecture (only loads/stores touch memory)
- Predicated execution (CSEL, etc.) but not full predication

**LLVM specifics**:
- AArch64 is the **reference backend** for GlobalISel — it was the first
  backend to fully support GISel at all optimization levels.
- SVE (Scalable Vector Extension) introduces **vector-length-agnostic**
  (VLA) programming — the vector width is unknown at compile time.
  This requires SVE-specific IR intrinsics and a special legalization
  strategy.
- PAC (Pointer Authentication) and BTI implement **control-flow integrity**
  in hardware — discussed further in [Ch. 12](./12-auxiliary-libraries.md).

**Theoretical background**: SVE's VLA model is inspired by **Cray vectors**
(a traditional vector supercomputer approach) rather than SIMD fixed-width
vectors.  The compiler must reason about **unknown vector lengths** using
predicates and `llvm.vscale` — an interesting challenge for dependence
analysis and cost modeling.

### ARM (`lib/Target/ARM/`) — `LLVMARM`

**Covers**: ARM32 (ARM, Thumb, Thumb-2), NEON SIMD, VFP, DSP extensions

ARM32 uses a hybrid ISA:
- ARM mode: fixed 32-bit instructions
- Thumb mode: mixed 16/32-bit instructions (higher code density)
- Conditional execution of most instructions (predication via condition codes)
- PC-relative addressing

**LLVM specifics**: The ARM backend handles **constant island placement** —
large constants that can't be embedded in instructions must be placed in
"constant pools" within PC-relative range.  ARM's predication model allows
if-conversion to be more aggressive than on most architectures.

### RISCV (`lib/Target/RISCV/`) — `LLVMRISCV`

**Covers**: RV32I, RV64I, M/A/F/D/C (extensions), V (Vector 1.0),
Zb* (bit-manipulation), Zk* (crypto), Zfh (half-precision float), and many more

RISC-V is a **modular open ISA** — the base integer ISA is minimal, and
everything else is an optional extension.  This creates compiler challenges:
- Feature combinations are combinatorial (any extension can be present/absent)
- The ISA string (`rv64i2p1_m2p0_a2p1`) encodes exact versioning
- Code generation must degrade gracefully when an extension is unavailable
  (e.g., emit a sequence of shifts+adds when `M` (multiply) is absent)

**LLVM specifics**: RISCV was the **first backend** to extensively use
the **new pass manager** and **GlobalISel**.  The backend models the
**RISC-V psABI** (processor-specific ABI) including ILP32/LP64/LP64D data
models and the full calling-convention register assignment.

**Compiler-science context**: RISC-V's modularity makes it a favorite
target for academic compiler research — it's simple enough to teach in a
semester but featureful enough for production work.  The V extension
(1.0) is a modern vector ISA with support for SEW (selected element width)
and LMUL (length multiplier), requiring nuanced auto-vectorization logic.

### AMDGPU (`lib/Target/AMDGPU/`) — `LLVMAMDGPU`

**Covers**: GCN (Graphics Core Next) and RDNA architectures, used in AMD
GPUs and compute accelerators

AMDGPU is a **GPU architecture** — fundamentally different from CPUs:
- **SIMT** (Single Instruction, Multiple Thread) execution model
- Thousands of hardware threads (wavefronts of 32/64 lanes)
- Multiple address spaces: global, local (LDS/shared), private, constant,
  flat (generic)
- Explicit memory hierarchy (L1, L2, LDS, VRAM)
- No stack in the traditional sense (recursion is not practical)

**LLVM specifics**:
- The backend uses a distinct address-space model (`addrspace(1)` = global,
  `addrspace(3)` = LDS, etc.)
- `AMDGPUAttributor` deduces memory access properties (readnone, readonly)
  for kernels
- `AMDGPUPromoteAlloca` promotes stack allocations to LDS or vectors
- `AMDGPUAnnotateUniformValues` identifies uniform (wave-invariant) values

**Theoretical background**: GPU compilation is a fundamentally different
problem from CPU compilation.  The key paper is **Lindholm et al. (2008)**
on NVIDIA's PTX, which established the SIMT programming model.  LLVM's
AMDGPU backend must handle divergent control flow (lanes taking different
branches → execution masking), coalesced memory access patterns, and
barrier synchronization.

### NVPTX (`lib/Target/NVPTX/`) — `LLVMNVPTX`

**Covers**: NVIDIA PTX (Parallel Thread Execution) — the virtual ISA for
NVIDIA GPUs, compiled to SASS by the NVIDIA driver

PTX is a **virtual ISA** — the LLVM backend outputs PTX assembly text,
not binary machine code.  The NVIDIA driver's JIT compiler produces SASS
for the actual GPU.

- Similar to AMDGPU in the SIMT model but with NVIDIA-specific intrinsics
- Supports CUDA-specific features (threadIdx, blockIdx, shared memory,
  __syncthreads, warp-level primitives)
- The NVPTX backend is simpler than AMDGPU because it doesn't emit
  machine code — PTX is a relatively clean, text-based ISA

### WebAssembly (`lib/Target/WebAssembly/`) — `LLVMWebAssembly`

**Covers**: WebAssembly (wasm32, wasm64), SIMD128, exception handling,
reference types, relaxed SIMD, multivalue

WebAssembly is a **stack-based virtual ISA** (like JVM bytecode) but with
a structured control-flow model:
- No `goto` — structured control flow only (blocks, loops, if-then-else)
- Explicit local variables (not SSA at the bytecode level, though LLVM
  handles this in the backend lowering)
- No arbitrary indirect branches
- 4 value types: i32, i64, f32, f64 (plus v128 for SIMD)

**LLVM specifics**:
- `WebAssemblyFixIrreducibleControlFlow` — transforms irreducible CFGs
  (which Wasm cannot represent) into reducible form via node splitting
  (Janssen & Corporaal, 1997)
- `WebAssemblyCFGStackify` — converts CFG-based MachineInstr to
  structured control flow (block/loop/if)
- `WebAssemblyExceptionPrepare` — lowers LLVM exception handling to
  Wasm exception-handling instructions (tag-based)

**Theoretical background**: The transformation from arbitrary CFGs to
structured control flow is the **relooping** problem (Peterson, 1973;
Ammarguellat, 1992).  Normal-form reducibility requires eliminating
irreducible loops — a graph-theoretic problem related to **node splitting**
in flow-graph theory.

---

## Tier 2 & 3 Backends (Good Quality)

### PowerPC (`lib/Target/PowerPC/`) — `LLVMPowerPC`

IBM's POWER ISA (ppc32, ppc64, ppc64le).  Used on IBM POWER servers
and historically on Apple hardware (G3/G4/G5).

- Superscalar RISC with many special-purpose registers (count register,
  link register, condition registers)
- Supports both big-endian and little-endian (ppc64le is the modern default)
- Extensive use of TableGen for scheduling models (P7, P8, P9, P10)

### SystemZ (`lib/Target/SystemZ/`) — `LLVMSystemZ`

IBM Z (s390x) — the mainframe architecture.  CISC with very rich
addressing modes, hardware transactional memory, and decimal arithmetic.

- The Z architecture is notable for its **assembler-level compatibility**
  going back to System/360 (1964) — arguably the longest-running ISA family
- Requires special handling for the **PC-relative addressing** constraints
  and the **conditional branch ranges**

### Mips (`lib/Target/Mips/`) — `LLVMMips`

Classic RISC architecture (MIPS I through MIPS64, MSA SIMD, microMIPS,
MIPS16e).  Used extensively in embedded systems and historically in
SGI workstations.  The MIPS ABI is a standard teaching example for
**delay slots** (the instruction after a branch is always executed)
and **load delay slots**.

### LoongArch (`lib/Target/LoongArch/`) — `LLVMLoongArch`

Chinese RISC architecture (LA32, LA64) with its own vector extension (LSX,
LASX).  Similar in spirit to MIPS and RISC-V but with distinct encoding
and ABI conventions.

### SPIR-V (`lib/Target/SPIRV/`) — `LLVMSPIRV`

Khronos's SPIR-V — the standard intermediate representation for Vulkan
and OpenCL GPU compute.  SPIR-V is a **binary IR** (not a machine ISA)
for shaders and compute kernels.  LLVM's SPIR-V backend emits `.spv`
files for Vulkan/OpenCL consumption.

### DirectX (`lib/Target/DirectX/`) — `LLVMDirectX`

Microsoft's DirectX Intermediate Language (DXIL) — LLVM-based shader
compilation target for Direct3D 12.  DXIL is a **restricted subset**
of LLVM IR with DirectX-specific intrinsics and metadata.

### Hexagon (`lib/Target/Hexagon/`) — `LLVMHexagon`

Qualcomm Hexagon DSP — a **VLIW** (Very Long Instruction Word)
architecture used in Snapdragon SoCs for audio, imaging, and ML
acceleration.

- VLIW requires the compiler to **schedule multiple instructions per
  cycle** into a fixed-width packet
- The backend uses a custom post-RA scheduler and hardware-loop
  recognition
- This is one of the few LLVM backends with a sophisticated software
  pipeliner (modulo scheduling)

### BPF (`lib/Target/BPF/`) — `LLVMBPF`

Berkeley Packet Filter — the Linux kernel's in-kernel virtual machine
for eBPF programs (networking, tracing, security).  eBPF has a
verifier-friendly design:
- No unbounded loops (the verifier requires provable termination)
- Limited instruction count per program
- 10 64-bit GPRs + a read-only frame pointer
- Memory access through typed helpers (maps, perf buffers)

### Sparc (`lib/Target/Sparc/`) — `LLVMSparc`

Sun/Oracle SPARC (v8, v9).  A classic RISC with **register windows**
(sliding window of registers across function calls), which makes it
unique among LLVM targets.  Used in Oracle SPARC servers and in
LEON3/4 radiation-hardened processors for aerospace.

---

## Tier 3 Backends (Experimental/Limited)

### AVR (`lib/Target/AVR/`) — `LLVMAVR`

Atmel AVR 8-bit microcontrollers (Arduino Uno, etc.).  An **8-bit**
target with only 32 registers and very limited RAM — constrains the
compiler to emit minimal code and use aggressive scalarization.

### MSP430 (`lib/Target/MSP430/`) — `LLVMMSP430`

Texas Instruments MSP430 — a 16-bit ultra-low-power microcontroller.
Simple RISC with 16 registers.

### M68k (`lib/Target/M68k/`) — `LLVMM68k`

Motorola 68000 series — the CPU of the original Macintosh, Amiga, Atari ST,
and Sega Genesis.  CISC with 8 data + 8 address registers, complex
addressing modes.  Maintained primarily for retro-computing.

### CSKY (`lib/Target/CSKY/`) — `LLVMCSKY`

C-SKY 32-bit RISC CPUs — widely used in Chinese embedded/industrial
applications.

### ARC (`lib/Target/ARC/`) — `LLVMARC`

Synopsys ARC (Argonaut RISC Core) — a configurable RISC for embedded
applications.

### Lanai (`lib/Target/Lanai/`) — `LLVMLanai`

Google's internal network-processor ISA.  A 32-bit RISC with some
VLIW-like features.

### VE (`lib/Target/VE/`) — `LLVMVE`

NEC SX-Aurora Vector Engine — a modern **vector supercomputer**
architecture descended from the NEC SX series.  Features very long
vector registers (256 × 64-bit elements) and a related vector ISA.

### XCore (`lib/Target/XCore/`) — `LLVMXCore`

XMOS xCORE — a multi-threaded embedded processor with hardware threads,
channel-based communication, and event-driven I/O.

### Xtensa (`lib/Target/Xtensa/`) — `LLVMXtensa`

Tensilica/Cadence Xtensa — a **configurable processor** (you can add
custom instructions via Tensilica's tooling).  Used in ESP32 (Espressif)
microcontrollers.

---

## Target Organization Pattern

Every target backend follows the same layout:

```
lib/Target/<Name>/
├── CMakeLists.txt         — defines LLVM<Name> target with LINK_COMPONENTS
├── <Name>.td              — top-level TableGen file (includes all others)
├── <Name>InstrInfo.td     — instruction definitions
├── <Name>RegisterInfo.td  — register classes, sub-register relationships
├── <Name>Schedule*.td     — scheduling models (processor resources, itineraries)
├── <Name>CallingConv.td   — calling convention definitions
├── <Name>TargetMachine.h/cpp  — TargetMachine factory
├── <Name>ISelLowering.h/cpp   — TargetLowering implementation
├── <Name>InstrInfo.h/cpp      — TargetInstrInfo implementation
├── <Name>RegisterInfo.h/cpp   — TargetRegisterInfo implementation
├── <Name>FrameLowering.h/cpp  — TargetFrameLowering implementation
├── <Name>Subtarget.h/cpp      — TargetSubtargetInfo implementation
├── <Name>AsmPrinter.cpp       — assembly printing
├── <Name>MCAsmInfo.cpp        — assembly syntax info
├── <Name>MCCodeEmitter.cpp    — instruction encoding
├── <Name>MCTargetDesc.h/cpp   — MC-level target registration
├── <Name>AsmParser.cpp        — assembly parsing (optional)
├── <Name>Disassembler.cpp     — disassembly (optional)
├── InstPrinter/<Name>InstPrinter.h/cpp — formatted instruction printing
├── TargetInfo/<Name>TargetInfo.h/cpp  — target registration
├── GISel/<Name>*Lawyer*.cpp   — GlobalISel legalizer rules
└── MIRParser/<Name>MIRParser.cpp — MIR parsing (optional)
```

The separation between `<Name>ISelLowering` (the interface between IR and
the target) and `<Name>InstrInfo` (properties of individual instructions)
is a key architectural choice: it separates **type legalization** from
**instruction details**, allowing each to evolve independently.

## The `llvm/lib/Target/` Common Library

**CMake target**: `LLVMTarget`

The `lib/Target/` directory (not the backend subdirectories) provides the
common `TargetMachine` base class and the target registration mechanism.
Any backend links against `LLVMTarget`.

---

## Selecting a Target

You specify a target when building LLVM with:
```
-DLLVM_TARGETS_TO_BUILD="X86;AArch64;RISCV"
```

The `LLVM_DEFAULT_TARGET_TRIPLE` CMake variable sets the default
host target (`x86_64-unknown-linux-gnu`, `aarch64-apple-darwin`, etc.).

All the target-specific code is **conditionally compiled** — if you
omit AMDGPU from `LLVM_TARGETS_TO_BUILD`, no AMDGPU code appears
in the final binary.
