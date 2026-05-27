# 08 — Debug Information

This chapter covers LLVM's libraries for representing, reading, writing,
and manipulating **debug information** — the data that maps machine code
back to source code for debuggers, profilers, and crash reporters.

---

## Why Debug Info in a Compiler Backend?

Debug information is a **parallel description** of the program: while the
code sequence describes *what executes*, the debug info describes *what it
means* (source file, line number, variable name, type, scope).  The compiler
must maintain this mapping through every optimization and transformation.

LLVM represents debug info as **metadata** (`!dbg`, `!dilexical_block`, etc.)
attached to IR instructions.  This metadata flows through the optimizer
(transforms update it) and eventually lands in the MC layer, which emits
DWARF, CodeView, or other debug formats.

This design is inspired by the **origin tracking** concept from
Hennessy's "Symbolic Debugging of Optimized Code" (1982) and the SGI
MIPSPro compiler's DWARF generation (which pioneered tracking debug
info through optimizations).

---

## LLVMDebugInfoDWARF (`lib/DebugInfo/DWARF/`)

**CMake target**: `LLVMDebugInfoDWARF`

The **primary** debug-info library — reads and writes **DWARF**, the
dominant debugging format on Linux, macOS, and most embedded systems.

### DWARF Format

DWARF (Debugging With Attributed Record Format) is a structured binary
format organized as a tree of **Debug Information Entries** (DIEs), each
with a tag and a set of attributes.

Key sections:

| Section | Content |
|---------|---------|
| `.debug_info` | The DIE tree — compilation units, subprograms, variables, types |
| `.debug_abbrev` | Abbreviation tables (compact encoding of tag + attribute schemas) |
| `.debug_line` | Line number program (state machine mapping PC → file:line:column) |
| `.debug_str` | String table for debug strings |
| `.debug_str_offsets` | Index into the string table (DWARF 5) |
| `.debug_addr` | Address table (DWARF 5) |
| `.debug_ranges` / `.debug_rnglists` | Non-contiguous address ranges |
| `.debug_loc` / `.debug_loclists` | Location descriptions (where a variable lives) |
| `.debug_frame` / `.eh_frame` | Call frame information (unwinding for exceptions and backtraces) |
| `.debug_aranges` | Quick lookup: address → compilation unit |
| `.debug_names` / `.debug_pubnames` | Name index for faster lookup (DWARF 5) |
| `.debug_macro` | Macro definitions and invocations |
| `.debug_types` | Type units (DWARF 4) / `.debug_info` type units (DWARF 5) |

### Key Classes

| Class | Role |
|-------|------|
| `DWARFContext` | Manages a set of DWARF sections; entry point for all queries |
| `DWARFUnit` | A single compilation unit or type unit |
| `DWARFDie` | A single DIE — access attributes, traverse children |
| `DWARFDebugLine` | Line table parser — maps addresses to source locations |
| `DWARFFormValue` | A single attribute value (reference, string, address, block, ...) |
| `DWARFExpression` | A DWARF expression — bytecode for computing variable locations |
| `DWARFDebugFrame` | CFI (Call Frame Information) parser for unwinding |
| `DWARFAddressRange` | An address range from `.debug_aranges` or `.debug_ranges` |
| `DWARFAcceleratorTable` | `.apple_names`, `.debug_names` lookup tables |

### Location Descriptions & DWARF Expressions

A **DWARF expression** is a stack-based bytecode that describes where
to find a variable's value at a given program point.  Examples:
- `DW_OP_reg5 RDI` — the value is in register RDI
- `DW_OP_fbreg -8` — the value is at frame-base minus 8 (a stack slot)
- `DW_OP_breg6 RSI 0` — the value is at the address in RSI
- Complex expressions: `DW_OP_piece` for bitfield access, `DW_OP_stack_value` for computed values

LLVM's `DIExpression` in the IR is lowered to DWARF expressions by
`DwarfExpression.cpp`.

**Theoretical background**: DWARF expressions are a specialized **bytecode
interpreter** — a stack machine with a small instruction set designed
specifically for location computation.  This is an instance of **domain-
specific language** (DSL) design: the bytecode is expressive enough for
the problem but simple enough for a debugger to interpret efficiently.

---

## Other Debug Info Formats

### CodeView (`lib/DebugInfo/CodeView/`) — `LLVMDebugInfoCodeView`

Microsoft's **CodeView** debug format — used on Windows with MSVC and PDB:
- Type records (LF_*) — describe C++ types, classes, templates
- Symbol records (S_*) — describe functions, variables, scopes
- The library handles serialization/deserialization of all record types

### PDB (`lib/DebugInfo/PDB/`) — `LLVMDebugInfoPDB`

Microsoft's **Program Database** format — a multi-stream file format
that packages CodeView type records, symbol records, line tables, and
module info:

- `PDBFile` — reads the MSF (Multi-Stream Format) container
- `NativeSession` — provides debugger-like queries over the PDB
- Used by `llvm-pdbutil` for analysis and merging of PDB files

### MSF (`lib/DebugInfo/MSF/`) — `LLVMDebugInfoMSF`

**Multi-Stream Format** — the container format used by PDB.  An MSF file
contains multiple independent streams (like sections in an object file),
each addressed by a block allocation table (similar to a filesystem's FAT).

### GSYM (`lib/DebugInfo/GSYM/`) — `LLVMDebugInfoGSYM`

**GSYM** (Google Symbol) format — a compact, lookup-optimized debug info
format designed for **production symbolization**:
- Optimized for fast address → function name + source location lookup
- Much smaller than DWARF (1-5% of binary size vs. 10-50% for DWARF)
- Used by Android and Fuchsia for on-device crash symbolization
- `llvm-gsymutil` converts DWARF → GSYM

### BTF (`lib/DebugInfo/BTF/`) — `LLVMDebugInfoBTF`

**BPF Type Format** — a compact type information format for eBPF programs:
- Encodes types, functions, and line info in a format the Linux kernel
  verifier can understand
- Essential for **CO-RE** (Compile Once — Run Everywhere) BPF programs,
  where the program is compiled on one kernel version and runs on another
- LLVM generates BTF from debug info metadata when targeting BPF

### Symbolize (`lib/DebugInfo/Symbolize/`) — `LLVMDebugInfoSymbolize`

The **address symbolization engine** — converts raw addresses to
human-readable function name + source location + inline stack:
- Used by `llvm-symbolizer` (the command-line tool)
- Used by sanitizers (ASan, MSan, TSan) for error reports
- Used by LLVM's own crash handler for fatal error diagnostics
- Consumes DWARF, PDB, and (in some cases) symbol tables

### DWARFLinker (`lib/DWARFLinker/`)

**CMake target**: `LLVMDWARFLinker`

A library for **linking and optimizing DWARF** across multiple object files:
- Deduplicates type definitions (DWARF tends to duplicate type DIEs
  in every compilation unit)
- Patches cross-unit references (e.g., a DIE in one CU referencing a
  type in another)
- Used by `dsymutil` (macOS `dSYM` bundle creation) and `llvm-dwarfutil`

This is essentially a **DWARF-level linker** — analogous to how a regular
linker merges code sections, the DWARF linker merges debug info sections.
The deduplication uses **DIE hashing** (content-based dedup, similar to the
concept of content-addressable storage).

### DWARFCFIChecker (`lib/DWARFCFIChecker/`)

**CMake target**: `LLVMDWARFCFIChecker`

A **verification tool** for DWARF Call Frame Information (`.eh_frame`,
`.debug_frame`):
- Checks consistency of CFI programs
- Validates that unwinding information is complete and correct
- Used in LLVM's test suite to verify CFI emission

### DWP (`lib/DWP/`)

**CMake target**: `LLVMDWP`

The **DWARF Package** library — merges multiple DWARF files into a
single `.dwp` file (DWARF 5 split-DWARF package format).  This is the
counterpart to the `dwp` tool from the GNU binutils.

### Debuginfod (`lib/Debuginfod/`)

**CMake target**: `LLVMDebuginfod`

**Debug information over HTTP** — a client library for the
`debuginfod` protocol (https://sourceware.org/elfutils/Debuginfod.html):

- Given a **Build ID** (a hash identifying a specific binary), the
  client queries a debuginfod server to download debug info
- The server stores `.debug_info` sections, separate debug files, and
  source code archives
- Used by `llvm-debuginfod-find`, `llvm-symbolizer`, and LLDB

Eliminates the need to install debug info packages manually — the
debugger/analyzer fetches what it needs on demand.

### LogicalView (`lib/DebugInfo/LogicalView/`)

**CMake target**: `LLVMDebugInfoLogicalView`

A high-level, unified view across debug formats (DWARF, PDB, CodeView):
- Provides a common API for navigating the **logical structure** of
  debug information (scopes, types, symbols, line numbers)
- Traverses the debug info hierarchy and builds a single unified model
- Used by `llvm-debuginfo-analyzer` for debug-info quality analysis

---

## How Debug Info Survives Optimization

One of the hardest problems in compiler engineering is maintaining
debug info through optimization.  LLVM's model:

1. **IR-level debug info** (`DILocation`, `DILocalVariable`, `DISubprogram`,
   etc.) is attached to instructions as metadata
2. When a pass transforms IR, it must **update the debug info**:
   - `Instruction::setDebugLoc()` — update line/column after cloning
   - `DIBuilder` — create new debug metadata after transformations
   - `SalvageDebugInfo` — try to express the same variable in terms of
     the new code (e.g., if SROA decomposes a struct, describe each field
     as a separate fragment)
3. **Debugify** pass infrastructure: synthetic debug info for testing —
   injects debug info, runs an optimization pass, then checks how much
   was lost

This connects to the research area of **debugging optimized code**
(Zellweger, 1984; Adl-Tabatabai & Gross, 1996; Tichy, 2011).

---

## DWARF 5 & Debug Info Evolution

LLVM supports DWARF 2 through DWARF 5 (the current standard as of 2017).
Key innovations in DWARF 5:

- **Split DWARF**: debug info in a separate `.dwo` file (faster linking,
  smaller binary)
- **Debug fission**: the linker only sees skeleton units; the bulk of
  debug info stays in `.dwo` files, combined by the `dwp` tool
- **`.debug_names`**: a faster, more compact name index (replaces
  `.debug_pubnames`/`.debug_pubtypes`)
- **String offsets table**: separate string indexing for better compression
- **Range list and location list tables**: more compact encoding for
  non-contiguous ranges and locations
