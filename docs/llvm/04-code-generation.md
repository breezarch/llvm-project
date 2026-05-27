# 04 — Code Generation

This chapter covers the **back end** of LLVM: converting optimized IR into
machine code.  This is dominated by five major libraries: `LLVMCodeGen`,
`LLVMMC`, `LLVMMCA`, `LLVMCodeGenTypes`, and the 25 individual target
backends (covered in [Chapter 5](./05-target-backends.md)).

---

## The Code Generation Pipeline

```
LLVM IR (optimized, SSA form)
        │
        ▼
┌─────────────────────────────────────────────┐
│  Instruction Selection                       │
│  ┌─────────────────────────────────────┐    │
│  │ SelectionDAG (legacy)  or  GlobalISel│    │
│  │ IR → target-specific DAG/MI         │    │
│  └─────────────────────────────────────┘    │
│  Result: MachineInstr in MachineFunction     │
├─────────────────────────────────────────────┤
│  Machine IR (MIR) passes                     │
│  &#8226; Register Allocation (greedy/basic/fast)    │
│  &#8226; Instruction Scheduling (pre/post RA)        │
│  &#8226; Prolog/Epilog Insertion                     │
│  &#8226; Branch Folding, Tail Duplication            │
│  &#8226; Machine Block Placement                      │
├─────────────────────────────────────────────┤
│  MC Layer                                    │
│  &#8226; MachineInstr → MCInst                        │
│  &#8226; Object-file emission (ELF/MachO/COFF/...)   │
│  &#8226; Assembly printing                            │
└─────────────────────────────────────────────┘
        │
        ▼
   Object File (.o) or Assembly (.s)
```

This pipeline embodies the classic **three-phase backend** from retargetable
compiler research (Davidson & Fraser, 1984; Aho & Johnson, 1976):
1. **Instruction selection** — map IR to target instructions
2. **Register allocation** — map virtual registers to physical registers
3. **Instruction scheduling** — reorder instructions for pipeline efficiency

---

## LLVMCodeGen (`lib/CodeGen/`)

**CMake target**: `LLVMCodeGen`

The core code generation library — the largest and most complex library
in LLVM.  It provides the target-independent framework for:

### Machine IR (MIR)

The code generator's internal representation:

| Class | Description | IR Analogue |
|-------|-------------|-------------|
| `MachineFunction` | A function at the machine level | `Function` |
| `MachineBasicBlock` | A basic block of `MachineInstr`s | `BasicBlock` |
| `MachineInstr` | A single machine instruction | `Instruction` |
| `MachineOperand` | An operand (register, immediate, memory, ...) | `Use` / `Value` |
| `MachineRegisterInfo` | Virtual-register definitions | — (registers are explicit) |

MIR is **not in SSA form** after register allocation — virtual registers
are replaced with physical registers.  However, before register allocation,
MIR maintains SSA form (including φ-nodes as `PHI` MachineInstrs).

The MIR format is human-readable and can be round-tripped through files
(`.mir`) for testing.  `MIRParser` handles deserialization.

### Instruction Selection

LLVM supports **two** instruction selection frameworks:

#### SelectionDAG (Classic)

The original instruction selector, based on **DAG covering**:

1. **DAG construction**: IR instructions in a basic block are converted to
   a `SelectionDAG` — a directed acyclic graph of `SDNode`s.
2. **Legalization**: Types and operations are legalized to what the target
   supports (split `i64` on a 32-bit target; promote `i8` to `i32`).
3. **DAGCombine**: Target-independent and target-specific peephole
   optimizations on the DAG (equivalent to InstCombine at the DAG level).
4. **Instruction Selection**: The DAG is covered with target instructions
   using pattern matching (TableGen-generated matchers).
5. **Scheduling**: The linearized DAG order determines instruction order;
   may be re-ordered for pipeline efficiency.

**Theoretical basis**: DAG covering for instruction selection was formalized
by Aho & Johnson (1976) with their **optimal tree-pattern matching**
algorithm (dynamic programming, O(n)).  The extension to DAGs (handling
common subexpressions) is NP-complete in general, but heuristic approaches
(Pelegrí-Llopart & Graham, 1988) work well in practice.

#### GlobalISel (Modern)

A newer instruction selection framework designed for:
- **Gradual lowering** (IR → generic MIR → target-specific MIR)
- **Better compile-time** (no DAG construction/destruction per block)
- **Easier debugging** (MIR is the single IR through the whole pipeline)

Phases:
1. **IRTranslator**: IR → generic MIR (target-independent opcodes like `G_ADD`)
2. **Legalizer**: Legalize types (narrow, widen, split) and operations
   (libcall, scalarize, lower)
3. **RegBankSelect**: Assign generic virtual registers to register banks
   (GPR, FPR, vector)
4. **InstructionSelect**: Generic MIR → target-specific MIR via
   TableGen-generated pattern matching

GlobalISel is the future direction for most backends (AArch64 uses it
by default at `-O0`; AMDGPU for all levels).

### Register Allocation

LLVM has three register allocators:

| Allocator | Algorithm | Use Case |
|-----------|-----------|----------|
| **Greedy** | Linear scan + splitting + eviction (priority-based) | Default at `-O2`/`-O3` |
| **Basic** | Linear scan (Poletto & Sarkar, 1999) | Default at `-O0`/`-O1` |
| **Fast** | Linear scan, very fast, no splitting | PIC/JIT compilation |

The **greedy** allocator (Jakob Stoklund Olesen, 2011) is based on the
**linear scan** algorithm (Poletto & Sarkar, 1999) with extensions:
- **Live range splitting** (break a long live range at use points)
- **Spill weight** (cost model based on loop nesting depth)
- **Eviction** (choose which live range to spill when all regs are taken)
- **ML-based eviction** (TensorFlow model for eviction decisions, optional)

**Theoretical basis**: Register allocation is equivalent to **graph
coloring** (Chaitin et al., 1981) — nodes are live ranges, edges are
interference constraints, colors are physical registers.  Graph coloring
is NP-complete for K ≥ 3.  Linear scan (Poletto & Sarkar, 1999) is
O(n log n) and works well in practice because real programs have
structured live ranges.

### Other CodeGen Components

| Component | Description |
|-----------|-------------|
| **AsmPrinter** | Converts MachineInstr → MCInst and prints assembly text |
| **LiveDebugValues** | Propagates debug variable locations through the backend (InstrRef and VarLoc variants) |
| **MachineBlockPlacement** | Reorders basic blocks for optimal fall-through (Wu & Larus, 1994) |
| **MachineLICM** | Loop-invariant code motion at the machine level |
| **MachineCSE** | Common subexpression elimination for machine instructions |
| **MachineSink** | Sink instructions to their uses |
| **MachineCombiner** | Combine machine instruction patterns (e.g., `fmul+fadd → fma`) |
| **ShrinkWrap** | Move prolog/epilog into cold paths |
| **TailDuplication** | Duplicate blocks to eliminate branches |
| **BranchFolding** | Merge common tails, eliminate redundant branches |
| **IfConversion** | Convert short if-then-else to predicated instructions |
| **StackSlotColoring** | Merge non-overlapping stack slots to reduce frame size |
| **StackProtector** | Insert stack canaries (buffer overflow mitigation) |
| **ShadowStackGCLowering** | Lower GC roots via shadow stack |
| **RegAllocFast** | Very fast register allocation (JIT only) |
| **HardwareLoops** | Lower loops to hardware-loop instructions (Hexagon, ARC) |
| **CFIInstrInserter** | Insert `.cfi_*` directives for exception unwinding |
| **PatchableFunction** | Insert NOP sleds for runtime patching |
| **IndirectBrExpand** | Lower `indirectbr` for targets that don't support it |
| **InterleavedAccess** | Recognize interleaved loads/stores and use target-specific instructions |
| **ExpandLarge*Ops** | Lower large operations (memcpy, div, etc.) that exceed target limits |
| **SafeStack** | Dual-stack layout (safe stack for spills, unsafe for alloca) |

---

## LLVMCodeGenTypes (`lib/CodeGenTypes/`)

**CMake target**: `LLVMCodeGenTypes`

Provides C/C++ ABI type lowering — converting Clang/BE types into LLVM types
suitable for a particular target's calling convention.  Includes:

- Record layout (struct field offsets, alignments)
- Calling convention type coercion
- Vector type selection

---

## LLVMMC (`lib/MC/`)

**CMake target**: `LLVMMC`

The **Machine Code** layer — LLVM's assembler, disassembler, and object-file
emission library.

### MC Architecture

```
   Assembly source (.s)               MachineInstr
        │                                  │
        ▼                                  ▼
   MCParser (AsmParser)              MCInstLowering
   ┌──────────────────┐              ┌──────────────┐
   │ LLVMMCParser      │              │ (in CodeGen) │
   │ (separate target: │              └──────────────┘
   │  MCAsmParser*.cpp)│                     │
   └──────────────────┘                     ▼
        │                              MCInst
        ▼                                  │
      MCInst ◄─────────────────────────────┘
        │
        ├──▶ MCAsmStreamer  ──▶ Assembly (.s)
        │
        ├──▶ MCObjectStreamer  ──▶ Object file (.o)
        │    ├── MCELFStreamer        (ELF)
        │    ├── MCMachOStreamer      (MachO)
        │    ├── MCCOFFStreamer       (COFF)
        │    ├── MCWasmStreamer       (WebAssembly)
        │    └── MCGOFFStreamer       (GOFF / z/OS)
        │
        └──▶ MCNullStreamer  ──▶ (discard, for timing)
```

### Key MC Classes

| Class | Role |
|-------|------|
| `MCInst` | A single machine instruction in MC form (opcode + operands) |
| `MCInstrInfo` | Instruction descriptions (TableGen-generated) |
| `MCRegisterInfo` | Register names, classes, sub-register relationships |
| `MCSubtargetInfo` | CPU features, scheduling model |
| `MCAsmInfo` | Target assembly syntax (directives, comment chars, alignment) |
| `MCAsmBackend` | Relaxable instruction fixups, object writer creation |
| `MCCodeEmitter` | Encode MCInst → bytes (target-specific) |
| `MCObjectWriter` | Write object file format (ELF/MachO/COFF/GOFF/Wasm) |
| `MCContext` | Manages uniqued MC-level objects (symbols, sections) |
| `MCAssembler` | The assembler core: layout, relaxation, symbol resolution |
| `MCAsmStreamer` | Prints assembly text from MCInst stream |
| `MCAsmParser` | Parses assembly text; uses target-specific AsmParser |
| `MCDisassembler` | Decodes bytes → MCInst (target-specific) |
| `MCExpr` | Symbolic expressions (symbol ± offset, relocations) |
| `MCSymbol` | A named or temporary symbol |
| `MCSection` | A section in the object file |
| `MCFragment` | A fragment of section data (instruction, data, alignment, ...) |
| `MCDwarf` | DWARF .debug_* section emission from CFI directives |

### Object File Formats

LLVMMC supports emitting five binary formats:

| Format | Streamer Class | Primary Platforms |
|--------|---------------|-------------------|
| **ELF** | `MCELFStreamer`, `MCELFObjectTargetWriter` | Linux, BSD, Fuchsia, embedded |
| **MachO** | `MCMachOStreamer` | macOS, iOS, Darwin derivatives |
| **COFF** | `MCCOFFStreamer` | Windows, UEFI |
| **GOFF** | `MCGOFFStreamer` | z/OS (IBM mainframe) |
| **Wasm** | `MCWasmStreamer` | WebAssembly |
| **XCOFF** | (in `MCAsmInfoXCOFF`, `XCOFFObjectWriter`) | AIX |
| **DXContainer** | `MCDXContainerStreamer`, `MCDXContainerWriter` | DirectX shader container |

### Assembler Relaxation

MC supports **instruction relaxation**: an instruction may be emitted at
one size and later relaxed (expanded or shrunk) when the final layout
is known.  Example: a branch on RISC-V may start as 2 bytes (compressed)
and expand to 4 bytes if the offset exceeds the range.  This is
implemented via `MCFragment` and a fix-point layout loop.

---

## LLVMMCA (`lib/MCA/`)

**CMake target**: `LLVMMCA`

The **Machine Code Analyzer** — a micro-architectural simulator for
predicting throughput and latency of a basic block on a given CPU.

### MCA Pipeline Model

```
Instructions  ──▶  DispatchStage  ──▶  Scheduler
                                        ├── ResourceManager (functional units)
                                        ├── RegisterFile (physical registers)
                                        └── RetireControlUnit (in-order retirement)
                           │
                           ▼
                       ExecuteStage (models pipeline execution)
                           │
                           ▼
                       RetireStage (in-order retirement)
```

MCA uses the **scheduling model** from the target's `.td` files (same
data that drives the instruction scheduler) to simulate:
- **Instruction dispatch** (width of the issue queue)
- **Resource contention** (execution ports, pipeline units)
- **Data hazards** (RAW, WAR, WAW — read-after-write, etc.)
- **Register pressure** (physical register file limits)
- **Retirement throughput** (reorder buffer capacity)

This is used by `llvm-mca` for performance analysis of assembly sequences.

**Theoretical basis**: MCA is a simplified **trace-driven pipeline
simulator** — it models the superscalar out-of-order execution engine of
a modern CPU.  The underlying theory is Hennessy & Patterson's pipeline
model (from *Computer Architecture: A Quantitative Approach*) with
reservation stations, reorder buffers, and physical register renaming.

---

## TableGen in CodeGen

The target `.td` files in each backend use the TableGen DSL to declaratively
specify:

- Instruction formats (encoding, operands, patterns)
- Register definitions (classes, aliases, sub-registers)
- Calling conventions
- Scheduling models (processor resources, itineraries)
- Subtarget features

TableGen processes these into C++ code that `LLVMCodeGen` and `LLVMMC`
consume.  This is LLVM's implementation of the **declarative machine
description** concept — instead of writing C++ for every instruction,
the backend author writes `.td` files and TableGen generates the
repetitive code.  This is inspired by the **CGSS/PO** (Code Generator
Specification System / Peephole Optimizer) work at Stanford
(Davidson & Fraser, 1984), and the **machine description** formalism
from GNU GCC's RTL machine descriptions.
