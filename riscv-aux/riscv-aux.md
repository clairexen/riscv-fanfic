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

Part 3 describes an S-mode extension for managing shared hardware resources.


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

We define the following instructions:

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    | off | --- |0|   rs2   |   rs1   | --- |  00000  | ----------- | AUXWR
    | 011 | --- |1|   rs2   |   rs1   | --- |  00000  | ----------- | AUXSETSL
    |---------------------------------------------------------------|
    | off | --- |0|  00000  |   rs1   | --- |    rd   | ----------- | AUXRD
    | 001 | --- |1|  00000  |   rs1   | --- |    rd   | ----------- | AUXGETSL
    | 010 | --- |1|  00000  |   rs1   | --- |    rd   | ----------- | AUXNEXT
    |---------------------------------------------------------------|
    | 000 | --- |0|   rs2   |   rs1   | --- |  00000  | ----------- | AUXWR.V
    | 000 | --- |0|  00000  |   rs1   | --- |    rd   | ----------- | AUXRD.V

The instructions with an offset (off) field operate on the Xaux address
`rs1+off`. The remaining instructions operate on `rs1`. The offset is unsigned.

The offset helps eliminating ADDI instructions in code that copies multiple
words between GPRs and Xaux registers, or unrolled code that copies data
between memory and Xaux registers.

AUXRD and AUXWR read/write the Xaux register at `rd1+off`.

AUXRD.V and AUXWR.V read/write AVL words from/to a vector register
rs2/rd.  If `SEW/EDIV > XLEN` then the instructions addresses blocks of
consecutive, naturally aligned Xaux registers.

Context switch
--------------

To save the auxilary state and disable all accelerators during context switch,
the OS kernel must run the following algorithm:

	PTR := AUXNEXT(0)
	while PTR != 0 begin
		LEN := AUXGETSL(PTR)
		if LEN != 0 begin
			write(PTR)
			write(LEN)
			for I := 0 .. LEN-1 begin
				write(AUXRD(PTR+I))
			end
			AUXSETSL(PTR, 0)
		end
		PTR := AUXNEXT(PTR)
	end

And restoring the state:

	while not EOF begin
		PTR := read()
		LEN := read()
		AUXSETSL(PTR, LEN)
		for I := 0 .. LEN-1 begin
			AUXWR(PTR+I, read())
		end
	end

Alternative CSR-based ISA
-------------------------

As an alternative to the `AUX*` instructions above, a CSR-based interface such
as the following could be used. It has the disadvantage of adding additional
state (the `auxptr` CSR), but the advantage or lower footprint in the ISA
space.

We add the following User CSRs:

    CSR Name | Priv | Description
    ----------------------------------
    auxptr   |  URW | The current Xaux register file address
    auxnext  |  URO | AUXNEXT(auxptr)
    auxsize  |  URW | AUXGETSL(auxptr) or AUXSETSL(auxptr)
    auxdata  |  URW | AUXRD(auxptr++) or AUXWR(auxptr++)

Part 3: Time-sharing accelerators between cores
===============================================

TBD
