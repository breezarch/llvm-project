# 03 — Analysis & Optimization

This chapter covers the heart of the LLVM optimizer: **analysis**, which
answers questions about the program, and **transforms**, which rewrite the
IR to make it better.

---

## The Optimization Model

LLVM's optimizer is organized as a **pass pipeline** — a sequence of
individual passes that each do one thing well.  This architecture comes from
the Unix philosophy ("do one thing well") and from compiler textbooks
(Muchnick's *Advanced Compiler Design and Implementation*, 1997).

A pass can be:
- **Analysis**: readonly, computes facts (domtree, alias sets, loop info)
- **Transform**: mutates the IR (inlines, simplifies, vectorizes)
- **Utility**: provides helper functions used by other passes

Passes communicate through **analysis results**: a transform pass *requires*
certain analyses (declared via `getAnalysisUsage`), and the pass manager
ensures those analyses are computed and cached before the transform runs.

### The Two Pass Managers

LLVM has two pass managers:

| | **Legacy PM** | **New PM (NPM)** |
|---|---|---|
| Header | `llvm/IR/LegacyPassManager.h` | `llvm/Passes/PassBuilder.h` |
| Pass class | `FunctionPass`, `ModulePass`, etc. | `PassInfoMixin<Derived>` |
| Analysis caching | Manual (`addRequired<>`) | Automatic, lazy, dependency-aware |
| Nesting | Implicit (pass subclass) | Explicit (function/module/CGSCC adaptors) |
| Status | Deprecated but still used by some backends | Default for optimization pipelines |

The new PM is conceptually inspired by **dependency-directed computation**
(also known as incremental computation or self-adjusting computation, Acar
et al., 2006): an analysis is invalidated only when a transform modifies
the IR in a way that affects the analysis's inputs.

---

## LLVMAnalysis (`lib/Analysis/`)

**CMake target**: `LLVMAnalysis`

The analysis library provides **read-only queries** about the program.
Analyses are the "senses" of the optimizer — they look at the IR and
report facts.

### Alias Analysis

"Is this pointer allowed to point to that memory?"

| Analysis | Header | Approach |
|----------|--------|----------|
| `BasicAliasAnalysis` | `BasicAliasAnalysis.h` | Recursive analysis of GEP offsets, base-object tracking |
| `TypeBasedAliasAnalysis` | `TypeBasedAliasAnalysis.h` | TBAA metadata — type-based disambiguation (C/C++ strict aliasing) |
| `ScopedNoAliasAAResult` | `ScopedNoAliasAA.h` | Scope-based restrict/noalias annotations |
| `CFLAndersAA` | `CFLAndersAA.h` | Context-Free Language reachability (Andersen-style inclusion-based) |
| `CFLSteensAA` | `CFLSteensAA.h` | Context-Free Language reachability (Steensgaard-style unification-based) |
| `GlobalsModRef` | `GlobalsModRef.h` | Mod/Ref analysis for globals (interprocedural) |
| `ObjCARCAliasAnalysis` | `ObjCARCAliasAnalysis.h` | Objective-C ARC-specific alias knowledge |

**Theoretical background**: Pointer/alias analysis is one of the most
intensively studied areas in compiler research.  The two classic families are:

- **Andersen (1994)**: Inclusion-based — each pointer variable has a set of
  possible targets; constraints are solved iteratively.  O(n³) worst case
  but precise.
- **Steensgaard (1996)**: Unification-based — merges points-to sets
  aggressively; almost linear O(n α(n)) but imprecise.

LLVM's `CFLAndersAA` and `CFLSteensAA` implement these using **CFL
reachability** (Sridharan & Bodik, 2005), which reformulates points-to
as a graph reachability problem over a language of balanced parentheses.

The classic TBAA approach comes from **type-based disambiguation in
C/C++** — the strict aliasing rule (C99 §6.5/7) says that pointers of
incompatible types cannot alias.  LLVM encodes this as metadata that
alias analysis can query.

### Data-Flow Analyses

| Analysis | Header | Lattice / Domain | Use Cases |
|----------|--------|-------------------|-----------|
| `DominatorTree` | `Dominators.h` | Dominator relation on the CFG | Everything — SSA construction, loop detection, GVN |
| `PostDominatorTree` | `PostDominators.h` | Post-dominator relation | Control dependence, if-conversion |
| `LoopInfo` | `LoopInfo.h` | Nesting forest of natural loops | LICM, loop unrolling, loop vectorization |
| `ScalarEvolution` | `ScalarEvolution.h` | Affine expressions over induction variables | Loop optimization (trip counts, strides, dependence testing) |
| `LazyValueInfo` | `LazyValueInfo.h` | Value range (on-demand) | Jump threading, coroutine elision |
| `DemandedBits` | `DemandedBits.h` | Bit-level liveness | Bit-simplification, strength reduction |
| `AssumptionCache` | `AssumptionCache.h` | Cached `llvm.assume` intrinsics | Value-tracking based optimizations |
| `TargetLibraryInfo` | `TargetLibraryInfo.h` | Available libc/libm functions per target | Library call optimization, vectorization |
| `TargetTransformInfo` | `TargetTransformInfo.h` | Cost-model queries (how expensive is this op?) | Vectorization, inlining, loop unrolling |
| `BlockFrequencyInfo` | `BlockFrequencyInfo.h` | Estimated execution frequencies via branch probabilities | Code layout, PGO, inlining |
| `BranchProbabilityInfo` | `BranchProbabilityInfo.h` | Heuristic or PGO-derived branch probabilities | If-conversion, block placement |
| `ProfileSummaryInfo` | `ProfileSummaryInfo.h` | PGO profile summary (hot/cold thresholds) | Pipeline tuning |
| `MemorySSA` | `MemorySSA.h` | SSA for memory — defines memory-use / memory-def | Memory optimization (MemDep, DSE, MemCpyOpt) |
| `MemoryDependenceAnalysis` | `MemoryDependenceAnalysis.h` | Forward memory dependence queries | GVN, MemCpyOpt, DSE |
| `StackSafetyAnalysis` | `StackSafetyAnalysis.h` | Is this alloca always accessed in-bounds? | Stack safety, sanitizer optimizations |
| `IRSimilarityIdentifier` | `IRSimilarityIdentifier.h` | Detects similar IR subgraphs | Outlining, code-size reduction |
| `MLInlineAdvisor` | `MLInlineAdvisor.h` | ML-based inlining decisions | Reinforcement-learning-guided inlining |

### Dominator Tree & Loop Detection

These are the **most fundamental analyses** in the optimizer.

**Dominator tree**: Node d **dominates** node n if every path from the
entry to n passes through d.  LLVM uses the Lengauer-Tarjan (1979)
algorithm — O(n log n) worst case, O(n α(n)) in practice.  The dominator
tree is used by:
- SSA construction (φ-node placement at dominance frontiers)
- Control-dependence analysis
- Loop detection (natural loops defined by back edges where the header
  dominates the latch)
- Code motion (LICM sinks/hoists to/from loop headers)

**LoopInfo** detects **natural loops** from the back edges found during
a DFS of the CFG with edge classifications.  The nesting forest is used
for loop-level optimizations (loop unrolling, loop interchange, loop
fusion/fission, LICM).

### ScalarEvolution (SCEV)

SCEV is LLVM's workhorse for **loop analysis**.  It represents
induction-variable expressions in a symbolic form:

```
{base, +, step}<loop>         — addrec (add recurrence)
sext/zext(%expr)              — extension
%a + %b, %a * %b              — arithmetic
umin(%a, %b)                  — umin/umax/smin/smax
```

SCEV can:
- Compute trip counts of loops (when the condition is affine)
- Prove that two pointer accesses never overlap (dependence distance)
- Simplify bounds checks
- Enable aggressive loop transformations

**Theoretical background**: SCEV is based on **chains of recurrences**
(Zima, 1994; van Engelen, 2001), which generalize induction-variable
analysis from scalar to symbolic form.  The mathematical foundation
is in **finite differences** and **generating functions**.

### MemorySSA

`MemorySSA` (2015) is a modern rethinking of LLVM's memory representation.
It gives memory operations SSA form:
- `MemoryDef` — a store that defines a new memory version
- `MemoryUse` — a load that reads a memory version
- `MemoryPhi` — a φ at a CFG merge point
- `liveOnEntry` — the initial memory state

This allows optimizations like **MemorySSA-based DSE** and
**MemCpyOpt** to work on the same graph infrastructure as regular
SSA-based optimizations (GVN, PRE).

### ValueTracking

`ValueTracking.h` provides a library of **ad-hoc analysis queries**:
- `isKnownNonZero(V)` — can we prove V ≠ 0?
- `computeKnownBits(V)` — which bits are definitely 0 or 1?
- `isKnownNegative(V)` / `computeNumSignBits(V)` — sign-bit tracking
- `getUnderlyingObject(V)` — strip GEPs and casts to find the base object
- `isKnownNeverNaN(V)` / `cannotBeOrderedLessThanZero(V)` — FP analysis

These are used pervasively by `InstCombine` for peephole optimization.

---

## LLVMTransforms — The Optimization Pass Libraries

The `lib/Transforms/` directory is organized into 11 sub-libraries, each
containing a category of transforms:

### LLVMInstCombine (`lib/Transforms/InstCombine/`)

**The workhorse peephole optimizer.**  Handles algebraic simplification
of IR instructions: `x+0 → x`, `x*1 → x`, `and(x, -2) → clear low bit`,
etc.  The pass is contained in `InstructionCombining.cpp` (one very large
file per visitable instruction category).

**Theoretical basis**: InstCombine is essentially an **algebraic
simplification** engine using identities from integer arithmetic
(modulo 2^N), Boolean algebra, and IEEE 754 floating point.  It also
applies constant folding (compile-time evaluation of constant expressions)
and dead-code elimination as cleanup.

### LLVMScalarOpts (`lib/Transforms/Scalar/`)

Classic scalar optimizations:

| Pass | Description | Theory |
|------|-------------|--------|
| `GVN` (Global Value Numbering) | Eliminate redundant computations across basic blocks | Alpern, Wegman, Zadeck (1988) — partition-based value numbering |
| `LICM` (Loop Invariant Code Motion) | Hoist loop-invariant computations out of loops | Standard loop optimization from Allen & Cocke (1972) |
| `SCCP` (Sparse Conditional Constant Propagation) | Combine constant propagation with unreachable code elimination | Wegman & Zadeck (1991) — use SSA edges for sparse analysis |
| `ADCE` (Aggressive Dead Code Elimination) | Remove instructions whose results are never used | Mark-sweep liveness analysis on SSA |
| `BDCE` (Bit-Tracking Dead Code Elimination) | Remove dead bits within values | Demanded-bits analysis |
| `IndVarSimplify` | Simplify induction variables and eliminate redundant IVs | Loop strength reduction |
| `LoopStrengthReduce` | Replace multiplications with additions (IV strides) | Optimize addressing modes for the target |
| `LoopUnroll` / `LoopUnrollAndJam` | Unroll loop bodies to reduce branch overhead | Unroll factors from the cost model; unroll-and-jam for nested loops |
| `LoopFlatten` | Flatten nested loops into a single loop | Loop-nest optimization |
| `LoopIdiomRecognize` | Recognize memset/memcpy patterns in loops | Pattern-based loop idiom detection |
| `LoopFuse` / `LoopDistribute` | Fuse or split loops | Loop-nest optimization for locality / vectorization |
| `LoopInterchange` | Swap inner/outer loops | Locality optimization for cache behavior |
| `LoopDataPrefetch` | Insert prefetch instructions | Hardware prefetch optimization |
| `SROA` (Scalar Replacement of Aggregates) | Break structs/alloca into individual scalar values | SSA-based scalarization |
| `MemCpyOpt` | Optimize memcpy/memmove chains | Memory copy forwarding |
| `DSE` (Dead Store Elimination) | Remove stores that are never read | Memory-based liveness |
| `JumpThreading` | Duplicate blocks to eliminate branches with known outcomes | Path-sensitive CFG optimization |
| `CorrelatedValuePropagation` | Use branch conditions to simplify downstream code | Predicate-based simplification |
| `Float2Int` | Convert float ops to integer when safe | Type legalization |
| `GuardWidening` | Widen range checks to cover multiple checks | Range-based optimization |
| `InductiveRangeCheckElimination` | Eliminate range checks using inductive reasoning | Loop-specific optimization |

### LLVMVectorize (`lib/Transforms/Vectorize/`)

| Pass | Description | Theory |
|------|-------------|--------|
| `LoopVectorize` | Convert scalar loops to SIMD vector loops | Allen & Kennedy (1987) — dependence analysis; Nuzman et al. (2006) — outer-loop vectorization |
| `SLPVectorizer` (Superword-Level Parallelism) | Pack independent scalar operations into vector ops | Larsen & Amarasinghe (2000) — SLP concept from the SUIF compiler |
| `VectorCombine` | Fold extract/insert element chains into vector shuffles | Peephole vector optimization |
| `LoadStoreVectorizer` | Merge adjacent loads/stores into vector loads/stores | Merge consecutive memory accesses |

**Theoretical background**: Vectorization exploits **data-level parallelism**
(DLP) — the same operation applied to multiple data elements simultaneously.
SLP vectorization is of particular academic interest because Larsen &
Amarasinghe (2000) showed that even scalar code written without explicit
SIMD can be auto-vectorized by detecting isomorphic instruction sequences
at the basic-block level.

### LLVMIPO (`lib/Transforms/IPO/`)

**Inter-Procedural Optimization** — transforms that operate across function
boundaries:

| Pass | Description |
|------|-------------|
| `Inliner` | Inline function bodies into call sites (cost-model driven) |
| `GlobalOpt` | Optimize global variables (demote to local, constant-propagate) |
| `GlobalDCE` | Dead global elimination |
| `DeadArgumentElimination` | Remove unused function parameters and return values |
| `ArgumentPromotion` | Promote by-reference arguments to by-value |
| `FunctionAttrs` | Deduce function attributes (readnone, readonly, nounwind, ...) |
| `PruneEH` | Remove unused exception-handling landing pads |
| `StripSymbols` | Strip debug info and symbol names |
| `MergeFunctions` | Merge identical functions (semantic de-duplication) |
| `Internalize` | Mark all functions as internal (for LTO) |
| `WholeProgramDevirt` | Devirtualize virtual calls when the class hierarchy is known |
| `OpenMPOpt` | OpenMP-specific interprocedural optimization (barrier elimination, etc.) |
| `FunctionSpecialization` | Clone and specialize functions for constant argument patterns |
| `IROutliner` | Extract identical code sequences into functions (code-size reduction) |
| `PartialInlining` | Inline only the "hot path" of a function |
| `MemProfContextDisambiguation` | Clone functions based on memory allocation contexts (MemProf) |

**Theoretical background**: Interprocedural optimization is one of the
"hard problems" in compilation.  Call-graph SCC (Strongly Connected
Component) analysis is used to determine the order of processing
(bottom-up, from leaves of the call graph).  The inliner is the most
impactful IPO pass — it eliminates call overhead and enables
context-sensitive optimization.  The cost model (balancing code size
vs. performance) is informed by decades of research (Hall, 1991;
Ayers, Schooler & Gottlieb, 1997).

### LLVMAggressiveInstCombine (`lib/Transforms/AggressiveInstCombine/`)

More expensive pattern-matching combining than regular InstCombine:
- Combine consecutive operations (e.g., truncated math chains)
- Table-based optimization patterns
- Complex expression reassociation

### LLVMCoroutines (`lib/Transforms/Coroutines/`)

Transforms C++20 coroutines (or Swift async functions) from high-level
coroutine intrinsics (`llvm.coro.begin`, `llvm.coro.suspend`, etc.)
into normal functions with a state machine.

Transforms include:
- `CoroEarly` — lower to coroutine ABI
- `CoroSplit` — split into ramp function + resume/destroy functions
- `CoroElide` — elide heap allocation for coroutine frames when lifetime is known
- `CoroCleanup` — cleanup remaining coroutine intrinsics

### LLVMObjCARC (`lib/Transforms/ObjCARC/`)

Optimizes Objective-C Automatic Reference Counting (ARC):
- Removes redundant retain/release pairs
- Hoists retains out of loops
- Eliminates autorelease pool traffic

### LLVMCFGuard (`lib/Transforms/CFGuard/`)

Inserts **Control Flow Guard** (Windows CFG) and **Indirect Branch Tracking**
(Intel CET) checks — runtime verification that indirect calls/jumps
target valid entry points.  A security mitigation against ROP/JOP attacks.

### LLVMHipStdPar (`lib/Transforms/HipStdPar/`)

Transforms HIP (AMD GPU) standard-parallelism constructs — offloads
C++ standard algorithms to GPU kernels.

### LLVMInstrumentation (`lib/Transforms/Instrumentation/`)

Inserts runtime instrumentation into the IR:

| Instrumentation | Description |
|-----------------|-------------|
| `AddressSanitizer` | Out-of-bounds and use-after-free detection |
| `MemorySanitizer` | Uninitialized read detection |
| `ThreadSanitizer` | Data-race detection |
| `DataFlowSanitizer` | General dynamic taint tracking |
| `HWAddressSanitizer` | Hardware-assisted address sanitizer (ARM TBI/AArch64 MTE) |
| `BoundsChecking` | Array bounds checking |
| `GCOVProfiling` | GCOV-compatible coverage instrumentation |
| `InstrProfiling` | LLVM's own PGO instrumentation (counters) |
| `PGOInstrumentation` | Insert PGO counter updates (frontend-based) |
| `PGOMemOPSizeOpt` | Optimize memcpy/memset sizes with PGO data |
| `IndirectCallPromotion` | Promote indirect calls to direct calls based on PGO targets |
| `SanitizerCoverage` | Lightweight coverage for libFuzzer feedback |
| `KCFI` | Kernel Control Flow Integrity (fine-grained indirect-call checking) |
| `MemProfiler` | Heap memory profile instrumentation |
| `CtxProfiler` | Context-sensitive profiling |

### LLVMTransformUtils (`lib/Transforms/Utils/`)

Utility functions used by multiple transform passes:
- `CloneFunction`, `CloneModule`
- `InlineFunction` (the actual inlining mechanics)
- `LoopUtils` (loop simplify, loop rotation, LCSSA insertion)
- `SSAUpdater` (SSA repair after code duplication)
- `Local` (local dead-code elimination, constant folding helpers)
- `CodeExtractor` (extract a region into a new function)
- `UnrollLoop` (loop unrolling mechanics)
- `BasicBlockUtils` (split, merge, delete blocks)
- `CallPromotionUtils` (devirtualization)

---

## LLVMPasses (`lib/Passes/`)

**CMake target**: `LLVMPasses`

The **PassBuilder** — the central pass-pipeline construction library.
`PassBuilder` translates optimization levels (`-O0`, `-O1`, `-O2`, `-O3`,
`-Os`, `-Oz`) and customization flags into concrete pass sequences.

The standard `-O2` pipeline is roughly:

```
ModuleInlinerWrapperPass
  ├── Inliner (CGSCC pipeline, bottom-up)
  │   ├── DevirtSCCRepeatedPass
  │   ├── CGSCCToFunctionPassAdaptor (function passes on each callee)
  │   └── Inline pass (with ML or heuristic cost model)
  └── After-inlining cleanup

ModuleOptimizationPipeline (for functions not inlined)
  ├── GlobalOpt / GlobalDCE
  ├── OpenMPOpt
  ├── Function passes:
  │   ├── SROA, EarlyCSE, SimplifyCFG, InstCombine, ...
  │   └── Loop passes:
  │       ├── LoopRotate, LICM, LoopUnroll, LoopVectorize, ...
  │       └── SLPVectorize
  └── Late cleanup (InstCombine, SimplifyCFG, DSE, ADCE)
```

The `PassBuilder` also supports:
- **Extension points** — `registerPipelineEarlySimplificationEPCallback`,
  `registerOptimizerLastEPCallback`, etc., for plugins to insert passes.
- **AAPipeline** — alias analysis configuration (default: BasicAA + TBAA +
  ScopedNoAliasAA + GlobalsModRef).
- **LTO pipelines** — different pass ordering for FullLTO vs. ThinLTO.

`StandardInstrumentations.h` provides debugging instrumentation (print-before/after,
time-passes, verify-each, statistics).
