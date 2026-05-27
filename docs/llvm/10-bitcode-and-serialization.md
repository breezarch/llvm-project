# 10 — Bitcode, Serialization & Object Files

This chapter covers LLVM's **serialization** libraries — how LLVM IR
is stored on disk, how object files are parsed, and how binary formats
are interchanged with human-readable representations.

---

## LLVMBitcode (`lib/Bitcode/`)

**CMake target**: `LLVMBitcode`

### The LLVM Bitcode Format

LLVM Bitcode is a **binary serialization** of LLVM IR.  It is the
on-disk and network-transport representation used by:

- **LTO** (Link-Time Optimization): object files contain bitcode
  instead of (or alongside) machine code
- **ThinLTO**: summaries are serialized as bitcode
- **Clang modules**: precompiled headers and modules are bitcode
- **GPU offloading**: device code is embedded as bitcode in host objects
- **PCH**: Clang precompiled headers (though modern Clang uses
  a different format for actual PCH, module files use bitcode)

### Format Architecture

Bitcode has a **two-layer** design:

```
┌─────────────────────────────────┐
│  Bitcode Container               │
│  &#8226; Magic number: 0xDEC04342     │
│  &#8226; Wrapper for integrity,        │
│    producer identification        │
├─────────────────────────────────┤
│  Bitstream                        │
│  &#8226; Blocks and Records            │
│  &#8226; Abbreviation compression       │
│  &#8226; Variable-length integers        │
└─────────────────────────────────┘
```

The format is:
1. **Self-describing** — the schema (abbreviation table) is embedded in
   the stream, so a reader can parse it without external schema files
2. **Forward-compatible** — unknown blocks and records can be skipped
3. **Compact** — uses variable-length encoding (VBR — variable bit-rate)
   for integers, which are the most common data type in IR

### Reader & Writer

| Component | Class | Role |
|-----------|-------|------|
| **BitcodeReader** | `BitcodeReader` | Deserialize `.bc` file → in-memory `Module` |
| **BitcodeWriter** | `BitcodeWriter` | Serialize `Module` → `.bc` file |
| **BitstreamReader** | `BitstreamCursor` | Low-level bitstream parsing |
| **BitstreamWriter** | `BitstreamWriter` | Low-level bitstream emission |

The reader/writer are carefully designed for performance:
- The reader supports **lazy deserialization** — function bodies are
  only deserialized when the function is first accessed.  This is
  critical for ThinLTO (where a module may contain hundreds of
  functions, but only a few are imported)
- The writer supports **materialization** — serialize only the parts
  of the module that have been materialized

---

## LLVMBitstream (`lib/Bitstream/`)

**CMake target**: `LLVMBitstream`

The **low-level bitstream library** — a general-purpose binary serialization
format used by bitcode and by several other LLVM subsystems (Clang
serialized AST, PDB, remarks):

### Bitstream Concepts

| Concept | Description |
|---------|-------------|
| **Block** | A nested scope with an ID; can contain sub-blocks and records |
| **Record** | A sequence of integers identified by a code |
| **Abbreviation** (Abbrev) | A compression schema — maps record codes to specific encodings (fixed-width, VBR, char6, blob) |
| **BlockInfo** | Optional embedded schema describing which abbreviations apply to which block IDs |
| **EnterSubBlock** / **EndBlock** | Delimit nested blocks |

### Encoding

The bitstream uses a **variable-length encoding**:
- **VBR (Variable Bit-Rate)**: encode integers in chunks of N bits,
  with the top bit indicating "more chunks follow"
- **Fixed**: encode a value in exactly N bits
- **Char6**: encode 6-bit characters (for ASCII-like strings)
- **Blob**: raw byte sequence (for binary data like constant arrays)

The abbreviation mechanism allows the writer to define custom,
compact encodings for frequently-used record patterns.  This is
analogous to **Huffman coding** — common patterns get shorter
encodings — but with the crucial difference that the encoding
schemas (abbreviations) are **explicitly defined in the stream**
rather than being derived from symbol frequencies.

### Relation to Other Formats

LLVM's bitstream format was designed in 2004 by Chris Lattner as a
general-purpose binary format for compiler intermediate data.  It is:

- Simpler than **Protocol Buffers** (no schema compiler, no
  backward-compatibility rules built in)
- More structured than **MessagePack** or **CBOR** (which are
  value-oriented rather than block-oriented)
- Purpose-built for the access patterns of a compiler (lazy reading,
  large binary blobs for constant data)

---

## LLVMObject (`lib/Object/`)

**CMake target**: `LLVMObject`

The **object-file reading library** — reads and parses binary object files,
executables, and shared libraries in all supported formats.

### Modular Architecture

```
┌────────────────────────────────────────────────┐
│  ObjectFile (abstract base)                     │
│  &#8226; Sections, Symbols, Relocations              │
│  &#8226; Common API regardless of format               │
├────────────────────────────────────────────────┤
│  ┌───────┐ ┌───────┐ ┌──────┐ ┌─────┐ ┌─────┐│
│  │ ELF   │ │MachO  │ │COFF  │ │XCOFF│ │GOFF ││
│  │       │ │       │ │      │ │     │ │     ││
│  └───────┘ └───────┘ └──────┘ └─────┘ └─────┘│
│  ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │ DXContainer│ │ Minidump  │ │ OffloadBinary │  │
│  └──────────┘ └──────────┘ └───────────────┘  │
├────────────────────────────────────────────────┤
│  SymbolicFile — higher level (also handles     │
│  bitcode embedded in ELF/MachO/COFF)            │
└────────────────────────────────────────────────┘
```

### Key Classes

| Class | Role |
|-------|------|
| `ObjectFile` | Abstract base — sections, symbols, relocations |
| `ELFObjectFile` | ELF-specific reader; parses ELF header, program headers, section headers |
| `MachOObjectFile` | MachO-specific reader; fat binaries, dylib commands |
| `COFFObjectFile` | COFF/PE-specific reader; import/export tables, debug directories |
| `XCOFFObjectFile` | XCOFF-specific reader; loader sections, traceback tables |
| `GOFFObjectFile` | GOFF-specific reader; ESD records, RLD records |
| `WasmObjectFile` | WebAssembly object reader; LEB128 encoding, custom sections |
| `SymbolicFile` | Higher-level abstraction; can also represent bitcode (when bitcode is embedded in an object file) |
| `Archive` | Reads Unix `.a` archives (thin and regular) |
| `IRSymtab` | Reads the symbol table from bitcode files (without deserializing the full IR) |

### What You Can Query

For any object file, `LLVMObject` provides:
- **Sections**: name, address, size, alignment, flags (executable,
  writable, etc.), raw data
- **Symbols**: name, section, value, size, binding (local/global/weak),
  type (function/data), visibility
- **Relocations**: type, offset, symbol, addend
- **Dynamic information**: needed libraries, rpath, soname (ELF/MachO)

Used by: `llvm-nm`, `llvm-objdump`, `llvm-objcopy`, `llvm-readobj`,
`llvm-size`, `llvm-strings`, `llvm-symbolizer`, `llvm-cov`.

### Windows-Specific Object Formats

| Format | Library | Use Case |
|--------|---------|----------|
| **DXContainer** | `DXContainer.h` (Object) / `DXContainerInfo.h` (MC) | DirectX shader container |
| **Minidump** | `Minidump.h` (Object) | Windows minidump crash records |
| **Resource files** | `llvm/WindowsResource/` (headers) | `.res` files for Windows resource compilers |

---

## LLVMObjectYAML (`lib/ObjectYAML/`)

**CMake target**: `LLVMObjectYAML`

A **YAML serialization** of object files — round-trips between binary
object formats and human-readable YAML:

```
Binary (.o) ────▶ yaml2obj ────▶ YAML text (editable)
                                       │
                                       ▼
                                   obj2yaml
                                       │
                                       ▼
                                 Binary (.o)
```

This is used extensively in LLVM's **testing infrastructure**:
- Instead of checking in a binary `.o` file (which is opaque and
  platform-specific), tests describe the expected object file in YAML
- `yaml2obj` converts the YAML to a binary object file at test time
- `obj2yaml` converts binary to YAML for inspection

### Supported YAML Mappings

| Object Type | YAML Mapping | Description |
|-------------|-------------|-------------|
| ELF | `ELFYAML::Object` | Full ELF structure: header, sections, symbols, relocations |
| MachO | `MachOYAML::Object` | MachO structure with load commands |
| COFF | `COFFYAML::Object` | COFF header, sections, symbols, auxiliary records |
| XCOFF | `XCOFFYAML::Object` | XCOFF-specific records |
| GOFF | `GOFFYAML::Object` | GOFF ESD/RLD records |
| Wasm | `WasmYAML::Object` | WebAssembly sections |
| DXContainer | `DXContainerYAML::Object` | DirectX shader parts |
| Minidump | `MinidumpYAML::Object` | Streams and memory regions |
| Offload | `OffloadYAML::Object` | Offload binary format |

**Testing architecture insight**: YAML-based testing is a pattern borrowed
from **model-based testing** — the YAML description is a "model" of the
expected binary output, and the tool verifies that the compiler produces
this output.  This is much more maintainable than binary "golden files"
which are opaque and can't be code-reviewed.

---

## LLVMRemarks (`lib/Remarks/`)

**CMake target**: `LLVMRemarks`

**Optimization remarks** — a system for the compiler to emit structured
diagnostics about optimization decisions:

- "Inlined `foo` into `bar` (cost=5, threshold=10)"
- "Loop vectorized (width=4, interleaved=2)"
- "Missed optimization: could not prove noalias"

### Remarks Pipeline

```
Compiler passes  ──▶  RemarkStreamer  ──▶  Serialization
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                         YAML file       Bitstream file    stdout
                       (human-readable)  (compact binary)  (terminal)
```

### Key Uses

- **`.opt.yaml`**: Clang can emit remarks as YAML files
- **`llvm-remarkutil`**: Merges, filters, and analyzes remark files
- **`opt-viewer`**: Python tool that generates HTML reports showing
  optimization decisions overlaid on source code
- **Compiler fuzzing**: remarks are checked for consistency (e.g., an
  optimization that claims to have succeeded should change the IR)

---

## LLVMTextAPI (`lib/TextAPI/`)

**CMake target**: `LLVMTextAPI`

The **Text API** library — reads and writes **TBD (Text-Based Dylib)**
files, Apple's format for describing dynamic library interfaces:

- TBD files list the exported symbols of a `.dylib` without containing
  actual machine code
- Used by Apple's SDKs and the linker to resolve symbols against system
  libraries
- The library handles parsing TBD v1 through v4, symbol re-exports,
  platform/architecture variants

This is Apple-specific but important for Darwin/macOS/iOS toolchain
support.

---

## LLVMInterfaceStub (`lib/InterfaceStub/`)

**CMake target**: `LLVMInterfaceStub`

Similar to `LLVMTextAPI`, but for **ELF** platforms — reads and writes
**ELF stub files** (also known as **linker version scripts**):

- Defines the public interface of a shared library
- Used in Android and Fuchsia for stub library generation
- `llvm-ifs` (Interface Stub) is the command-line tool

### Compiler-Science Context

Interface stubs and TBD files solve a perennial problem in linker design:
**shared library interfaces**.  When you link against `libfoo.so`, you
only need its symbol table — not the actual code.  Interface stubs
extract and compress just the symbol information, enabling:
- **Faster linking** (no need to parse the full binary)
- **Smaller SDKs** (distribute only stubs, not full libraries)
- **ABI checking** (verify that a new library version is ABI-compatible)
