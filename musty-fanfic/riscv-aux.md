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
reserved size. The reserved region size must be a power of two and the region
should be naturally aligned to its reserved size.

Xaux base address zero must never be allocated to a region.

Each region has an *active size*, which is zero if the region is *deactivated*.

The *active size* of a region must always be SMALLER than the reserved region size.

We add the following User CSRs to provide direct access the Xaus register file:

    CSR Name | Priv | Description
    ----------------------------------
    auxptr   |  URW | The current Xaux register file address
    auxnext  |  URO | AUXNEXT(auxptr)
    auxsize  |  URW | AUXGETSL(auxptr) or AUXSETSL(auxptr)
    auxdata  |  URW | AUXRD(auxptr++) or AUXWR(auxptr++)

`auxptr` is simply a pointer into the Xaux register file. The register is
WARL and legal values are zero or any address within an Xaux region.

The regions form a linked list in implementation-defined ordering. Setting
`auxptr` to zero and reading from `auxnext` returns the base address of
the first region, setting `auxptr` to any region and reading from `auxnext`
returns the base address for the next region, or zero when the end of the
list is reached. An implementation may skip over deactivated regions.

Writing/reading `auxdata` writes/reads the Xaux register at `auxptr` and
then increments `auxptr`.

Context switch
--------------

To save the auxilary state and disable all accelerators during context switch,
the OS kernel must run the following algorithm:

	write(AUXPTR)
	AUXPTR := 0
	AUXPTR := AUXNEXT
	while AUXPTR != 0 begin
		if AUXSIZE != 0 begin
			write(AUXPTR)
			write(AUXSIZE)
			for I := 0 .. AUXSIZE-1 begin
				write(AUXDATA)
			end
			AUXSIZE := 0
		end
		AUXPTR := AUXNEXT
	end

And restoring the state:

	p = read()
	while not EOF begin
		AUXPTR := read()
		AUXSIZE := read()
		for I := 0 .. AUXSIZE-1 begin
			AUXDATA := read()
		end
	end
	AUXPTR := p

Part 3: Time-sharing accelerators between cores
===============================================

TBD
