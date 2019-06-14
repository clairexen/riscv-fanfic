RISC-V Auxiliary State Proposal
===============================

**DISCLAIMER: This proposal is meant as a basis for further discussions about
this topic area. It will be subject to change.**

This document is structured in two part:

Part 1 is a problem statement, explaining certain RISC-V extensions would
greatly benefit from adding (sometime large) additional state to the ISA, and
why it would be a good idea to manage this additional state via a uniform
extension-agnostic interface.

Part2 is my proposed solution to the problem described in part 1.

Part 3 discusses a possible standard hardware interface that could be used to
create portable accelerator IPs that works with many different RISC-V cores.


Part 1: Auxiliary functions and auxiliary state
===============================================

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


Part 2: The RISC-V Auxiliary State Proposal (Xaux)
==================================================

This chapter first defines a base Xaux extension, and then provides examples
for how the motivating examples above could be implemented with RISC-V
extensions based on Xaux. Those additional extensions are only described
for illustrative purposes. They are not meant to be actual ISA extension
proposals as-is.

Xaux is an extension enabling other extensions. We use the term *extension*
when referring to the extensions enabled by Xaux, and we use *Xaux* when
referring to the Xaux extension itself. We further use the term
*extension-defined* for details that must be defined by those extensions.

Each region has a variable *state length* (SL), that can be read with AUXGETSL
and updated with AUXSETSL. Saving/restoring SL and the first SL elements of a
region must be sufficient to completely restore the state of a region. Reading
and writing the first SL elements of a region must be free of side-effects.

The base Xaux extension
-----------------------

The Xaux extension defines a new addres space, separate from memory addresses,
CSRs, and ISA registers. This address space addresses individual XLEN-sized
words, similar to the CSR space. We call this the *Xaux register file*.

An extension must define one or more *Xaux regions*, i.e. a base address and a
size. The region size should be a power of two and the region should be
naturally aligned to its size.

The AUXGETSL, AUXSETSL, and AUXNEXT instructions below operate on regions
and the region base address is used to specify a region.

We allocate one minor opcode and add the following instructions:

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    | 000 |  0000 |  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXGETSL
    | 000 |  0001 |  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXNEXT
    | off |  0010 |  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXRD
    | off |  0100 |  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXRD.S
    | off |  0101 |  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXRD.D
    | off |  0110 |  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXRD.Q
    | off |  0111 |  00000  |   rs1   |  f3 |    rd   |    opcode   | AUXRD.V
    |---------------------------------------------------------------|
    | 000 |  1000 |   rs2   |   rs1   |  f3 |  00000  |    opcode   | AUXSETSL
    | off |  1010 |   rs2   |   rs1   |  f3 |  00000  |    opcode   | AUXWR
    | off |  1100 |   rs2   |   rs1   |  f3 |  00000  |    opcode   | AUXWR.S
    | off |  1101 |   rs2   |   rs1   |  f3 |  00000  |    opcode   | AUXWR.D
    | off |  1110 |   rs2   |   rs1   |  f3 |  00000  |    opcode   | AUXWR.Q
    | off |  1111 |   rs2   |   rs1   |  f3 |  00000  |    opcode   | AUXWR.V

The instructions with an offset (off) field operate on the Xaux address
`rs1+off`. The remaining instructions operate on `rs1`. The offset is unsigned.

The offset helps eliminating ADDI instructions in code that copies multiple
words between GPRs and Xaux registers, or unrolled code that copies data
between memory and Xaux registers.

AUXRD and AUXWR read/write the Xaux register at `rd1+off`.

On RV32:
- AUXWR.S and AUXWR.S behave like AUXRD and AUXWR, but rs2/rd is a floating point
  register.
- AUXRD.D and AUXWR.D address two consequtive, naturally aligned Xaux registers.
- AUXRD.Q and AUXWR.Q address four consequtive, naturally aligned Xaux registers.

On RV64:
- AUXWR.S, AUXWR.S, AUXRD.D, and AUXWR.D behave like AUXRD and AUXWR, but rs2/rd
  is a floating point register.
- AUXRD.Q and AUXWR.Q address two consequtive, naturally aligned Xaux registers.

Finally, AUXRD.V and AUXWR.V read/write AVL words from/to a vector register
rs2/rd.  If `SEW/EDIV > XLEN` then the instructions addresses blocks of
consecutive, naturally aligned Xaux registers.


Part 3: The RISC-V Auxiliary Function Interface
===============================================

TBD
