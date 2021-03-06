# REF

More details about Liveness analysis, please checkout [cranelift's annotation](https://github.com/CraneStation/cranelift/blob/8033deda3ac152d0b95bb0ad80b419625c3f0d58/lib/cretonne/src/regalloc/liveness.rs#L1-L176)...

# Liveness analysis for SSA values.

This module computes the live range of all the SSA values in a function and produces a
`LiveRange` instance for each.


## Liveness consumers

The primary consumer of the liveness analysis is the SSA coloring pass which goes through each
EBB and assigns a register to the defined values. This algorithm needs to maintain a set of the
curently live values as it is iterating down the instructions in the EBB. It asks the following
questions:

- What is the set of live values at the entry to the EBB?
- When moving past a use of a value, is that value still alive in the EBB, or was that the last
  use?
- When moving past a branch, which of the live values are still live below the branch?

The set of `LiveRange` instances can answer these questions through their `def_local_end` and
`livein_local_end` queries. The coloring algorithm visits EBBs in a topological order of the
dominator tree, so it can compute the set of live values at the begining of an EBB by starting
from the set of live values at the dominating branch instruction and filtering it with
`livein_local_end`. These sets do not need to be stored in the liveness analysis.

The secondary consumer of the liveness analysis is the spilling pass which needs to count the
number of live values at every program point and insert spill code until the number of
registers needed is small enough.


## Alternative algorithms

A number of different liveness analysis algorithms exist, so it is worthwhile to look at a few
alternatives.

### Dataflow equations

The classic *live variables analysis* that you will find in all compiler books from the
previous century does not depend on SSA form. It is typically implemented by iteratively
solving dataflow equations on bitvectors of variables. The result is **a live-out bitvector of
variables for every basic block** in the program.

This algorithm has some disadvantages that makes us look elsewhere:

- Quadratic memory use. We need a bit per variable per basic block in the function.
- Sparse representation. In practice, the majority of SSA values never leave their basic block,
  and those that do span basic blocks rarely span a large number of basic blocks. This makes
  the bitvectors quite sparse.
- Traditionally, the dataflow equations were solved for real program *variables* which does not
  include temporaries used in evaluating expressions. We have an SSA form program which blurs
  the distinction between temporaries and variables. This makes the quadratic memory problem
  worse because there are many more SSA values than there was variables in the original
  program, and we don't know a priori which SSA values leave their basic block.
- Missing last-use information. For values that are not live-out of a basic block, we would
  need to store information about the last use in the block somewhere. LLVM stores this
  information as a 'kill bit' on the last use in the IR. Maintaining these kill bits has been a
  source of problems for LLVM's register allocator.

Dataflow equations can detect when a variable is used uninitialized, and they can handle
multiple definitions of the same variable. We don't need this generality since we already have
a program in SSA form.

### LLVM's liveness analysis

LLVM's register allocator computes liveness per *virtual register*, where a virtual register is
a disjoint union of related SSA values that should be assigned to the same physical register.
It uses a compact data structure very similar to our `LiveRange`. The important difference is
that Cretonne's `LiveRange` only describes a single SSA value, while LLVM's `LiveInterval`
describes the live range of a virtual register *and* which one of the related SSA values is
live at any given program point.

LLVM computes the live range of each virtual register independently by using the use-def chains
that are baked into its IR. The algorithm for a single virtual register is:

1. Initialize the live range with a single-instruction snippet of liveness at each def, using
   the def-chain. This does not include any phi-values.
2. Go through the virtual register's use chain and perform the following steps at each use:
3. Perform an exhaustive depth-first traversal up the CFG from the use. Look for basic blocks
   that already contain some liveness and extend the last live SSA value in the block to be
   live-out. Also build a list of new basic blocks where the register needs to be live-in.
4. Iteratively propagate live-out SSA values to the new live-in blocks. This may require new
   PHI values to be created when different SSA values can reach the same block.

The iterative SSA form reconstruction can be skipped if the depth-first search only encountered
one SSA value.

This algorithm has some advantages compared to the dataflow equations:

- The live ranges of local virtual registers are computed very quickly without ever traversing
  the CFG. The memory needed to store these live ranges is independent of the number of basic
  blocks in the program.
- The time to compute the live range of a global virtual register is proportional to the number
  of basic blocks covered. Many virtual registers only cover a few blocks, even in very large
  functions.
- A single live range can be recomputed after making modifications to the IR. No global
  algorithm is necessary. This feature depends on having use-def chains for virtual registers
  which Cretonne doesn't.

Cretonne uses a very similar data structures and algorithms to LLVM, with the important
difference that live ranges are computed per SSA value instead of per virtual register, and the
uses in Cretonne IR refers to SSA values instead of virtual registers. This means that Cretonne
can skip the last step of reconstructing SSA form for the virtual register uses.

### Fast Liveness Checking for SSA-Form Programs

A liveness analysis that is often brought up in the context of SSA-based register allocation
was presented at CGO 2008:

> Boissinot, B., Hack, S., Grund, D., de Dinechin, B. D., & Rastello, F. (2008). *Fast Liveness
  Checking for SSA-Form Programs.* CGO.

This analysis uses a global precomputation that only depends on the CFG of the function. It
then allows liveness queries for any (value, program point) pair. Each query traverses the use
chain of the value and performs lookups in the precomputed bitvectors.

I did not seriously consider this analysis for Cretonne because:

- It depends critically on use chains which Cretonne doesn't have.
- Popular variables like the `this` pointer in a C++ method can have very large use chains.
  Traversing such a long use chain on every liveness lookup has the potential for some nasty
  quadratic behavior in unfortunate cases.
- It says "fast" in the title, but the paper only claims to be 16% faster than a dataflow based
  approach, which isn't that impressive.

Nevertheless, the property of only depending in the CFG structure is very useful. If Cretonne
gains use chains, this approach would be worth a proper evaluation.


## Cretonne's liveness analysis

The algorithm implemented in this module is similar to LLVM's with these differences:

- The `LiveRange` data structure describes the liveness of a single SSA value, not a virtual
  register.
- Instructions in Cretonne IR contains references to SSA values, not virtual registers.
- All live ranges are computed in one traversal of the program. Cretonne doesn't have use
  chains, so it is not possible to compute the live range for a single SSA value independently.

The liveness computation visits all instructions in the program. The order is not important for
the algorithm to be correct. At each instruction, the used values are examined.

- The first time a value is encountered, its live range is constructed as a dead live range
  containing only the defining program point.
- The local interval of the value's live range is extended so it reaches the use. This may
  require creating a new live-in local interval for the EBB.
- If the live range became live-in to the EBB, add the EBB to a work-list.
- While the work-list is non-empty pop a live-in EBB and repeat the two steps above, using each
  of the live-in EBB's CFG predecessor instructions as a 'use'.

The effect of this algorithm is to extend the live range of each to reach uses as they are
visited. No data about each value beyond the live range is needed between visiting uses, so
nothing is lost by computing the live range of all values simultaneously.

### Cache efficiency of Cretonne vs LLVM

Since LLVM computes the complete live range of a virtual register in one go, it can keep the
whole `LiveInterval` for the register in L1 cache. Since it is visiting the instructions in use
chain order, some cache thrashing can occur as a result of pulling instructions into cache
somewhat chaotically.

Cretonne uses a transposed algorithm, visiting instructions in order. This means that each
instruction is brought into cache only once, and it is likely that the other instructions on
the same cache line will be visited before the line is evicted.

Cretonne's problem is that the `LiveRange` structs are visited many times and not always
regularly. We should strive to make the `LiveRange` struct as small as possible such that
multiple related values can live on the same cache line.

- Local values should fit in a 16-byte `LiveRange` struct or smaller. The current
  implementation contains a 24-byte `Vec` object and a redundant `value` member pushing the
  size to 32 bytes.
- Related values should be stored on the same cache line. The current sparse set implementation
  does a decent job of that.
- For global values, the list of live-in intervals is very likely to fit on a single cache
  line. These lists are very likely ot be found in L2 cache at least.

There is some room for improvement.
