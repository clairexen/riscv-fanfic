RISC-V Proposal for Stateful Auxiliary Functions
================================================

**DISCLAIMER: This proposal is meant as a basis for further discussion about
this topic area. It will be subject to change.**

This document is structured in two part:

Part 1 is a problem statement, explaining why I think RISC-V would benefit from
an ISA extension that makes it easy to add "auxiliary state" to RISC-V as part
of (custom) extension, and manage that state in an extension-agnostic way.

Part2 is my proposed solution to the problem described in part 1.


Part 1: Why some auxiliary functions require auxiliary state
============================================================

In this document, I am using the term "auxiliary functions" to refer to any
instructions an ISA extension may add, that implements a pure function that
only uses the values from it's source registers (and parameters included in the
instruction word) as inputs, and only writes the destination register as
output.

Im using the term "stateful auxiliary functions" for added instructions that
also read/write some additional state that has been added to the ISA alongside
the new instructions.

This proposal is all about how to manage such an added state in a uniform
and extension-agnostic manner.

Motivating example: SHA-3
-------------------------

TBD

Motivating example: Floating Point Quire
----------------------------------------

TBD

Motivating example: Large Bit Matrix
------------------------------------

TBD

Motivating example: eFPGA-based Accelerators
--------------------------------------------

TBD

Why not use "V" extension vectors?
----------------------------------

TBD

Why not use memory-mapped accelerators?
---------------------------------------

TBD

Requirements for ISA-extension for managing auxiliary state
-----------------------------------------------------------

TBD


Part 2: The RISC-V Auxiliary Stateful Function Proposal (Xaux)
==============================================================

This chapter first defines a base Xaux extension, and then provides examples
for how the motivating examples above could be implemented in additional
extensions based on Xaus. Those additional extensions are only described
for illustrative purposes. They are not meant to be actual ISA extension
proposals as-is.

The base Xaux extension
-----------------------

We allocate one minor opcode and add the following instructions:

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |   00000 | 00|   rs2   |   rs1   |  f3 |    rd   |    opcode   | AUXSLN
    |   00001 | 00|  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXGLN
    |   00010 | 00|  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXNXT
    |  offset | 01|   rs2   |   rs1   |  f3 |    rd   |    opcode   | AUXWR
    |  offset | 10|  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXRD
    |  offset | 11|   rs2   |   rs1   |  f3 |    rd   |    opcode   | AUXFUN

An implementation that implements Xaux, but not a single extension using
Xaux, would simply write zero to rd for all those instructions.

All above instructions accept an "auxiliary state address" in `rs1`, or
`rs1+offset` for the instructions with offset field. An implementation may
use only an implementation-defined number of LSB bits of that address and
ignore all upper bits.

An ISA extension adding auxiliary stateful functions may must define one
or more *auxiliary state regions*, that consist of a *base address*
and a *region size*. Regions must be non-overlapping. The zero address
must be guaranteed to never be part of a region.

AUXSLN, with a base address in rs1 and a requested effective region length in
rs2, will *enable* the region if rs2 is nonzero, and *disable* the region if
rs2 is zero. The actual effective region length is returned in rd. The
requested effective region length maps to actual effective region length using
a function-defined method, that's a pure function of the requested length
and implementation-defined constants.

Decreasing the actual effective region length will effectively zero-out the
disabled words, and they will stay zeroed-out until the effective region length
is increased again.

An extension may provide "alternative views" to their auxiliary state at
addresses beyond the effective region length.

AUXGLN, with a base address in rs1, returns the current effective region length.

Executing AUXSLN or AUXGLN on an address that is not a base address has
function-defined behavior.

AUXNXT with a base address is rs1 will return the base address for the next region.
When rs1 has a zero value then the first base address is returned. When rs1 contains
the address of the last base address then zero is returned. This mechanism does not
need to return regions order of increasing base address.

AUXWR and AUXRD write-to and read-from the address specified in rs1+offset
respectively.

AUXFUN have function-defined behavior.

Using any of those functions on an address outside of a region will simply
write zero to rd.

Using AUXNXT with an address that is not a base address will also simply write
zero to rd.

Context save/restore
--------------------

Saving the current context performs two sweeps using AUXNXT. The first
one figures out the size of memory require to hold the state (ret=a0):

      LI a0, 1
      AUXNXT a1, zero
      BEQZ a1, done
    loop:
      AUGGLN a2, a1
      ADDI a0, a0, 2
      ADD a0, a0, a3
      AUXNXT a1, a1
      BNEZ a1, loop
    done:

The second sweep saves the state (dst=a0, on RV32) and disables all
aux regions:

    TBD

And finally restoring the state (src=a0):

    TBD

The offset parameter to AUXWR and AUXRD helps increase the performance when
unrolling those loops.
