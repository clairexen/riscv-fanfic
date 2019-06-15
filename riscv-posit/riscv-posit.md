Proposal for a RISC-V Posit Extension
=====================================

**DISCLAIMERS**

**This proposal is meant as a basis for further discussion about adding posit
support to RISC-V. It will be subject to change.**

**There is a separate RISC-V posit proposal on posithub.org. This is completely
independent from the work in this document.**


Introduction
============

Posits are an alternative floating point format that provides a few bits more
precision than IEEE float, at least for fairly normalized values.

- https://posithub.org/docs/BeatingFloatingPoint.pdf
- https://posithub.org/docs/posit_standard.pdf
- https://www.youtube.com/watch?v=N05yYbUZMSQ

This proposal coveres 3 different types of instructions:

- Instructions for working with posits in base-ISA integer registers
- Instructions for working with vectors of posits, using V-extension vectors
- Instructions for managing a quire and performing posit multiply-accumulate.


Scalar Posit Extension
======================

Posits compare like integers, so all integer branch instructions and compare
instructions work with posits as well, without the need for extra instructions.

Same for MIN/MAX (from B extension), negate, and abs().

The following new instructions are added to the OP major opcode:

	PADD       rd, rs1, rs2
	PSUB       rd, rs1, rs2
	PMUL       rd, rs1, rs2
	PDIV       rd, rs1, rs2
	PSQRT      rd, rs1
	PSIGN      rd, rs1

  PSIGN returns the posit -1 if rs1 is negative, +1 if rs1 is positive, zero
  if rs1 is zero, and NaR is rs1 is NaR.

Testing if the posit in a0 is NaR (for triggering runtime exceptions) is
already a two-instruction sequence, so no special instruction is added for
this application:

	ADDI t0, a0, -1
	BLT a0, t0, error

For "parsing" and generating posits we also add the following instructions:

	PEXP  rd, rs1
	PFRC  rd, rs1, rs2
	PMAK  rd, rs1, rs2

  PEXP returns the "effective exponent" encoded in the posit "R" and "E"
  fields and implied by XLEN, and PFRAC returns the "effective fraction"
  encoded in the posit "S" and "F" fields, when given PEXP(X) in rs2, so that,
  ignoring rounting errors:

	X = exp2(PEXP(X)) * PFRAC(X, PEXP(X))

  When PEXP(X) is given to PFRC in rs2, then PFRAC(X) is aligned to the MSB end
  of the result, with the implicit 1 on position XLEN-2 and MSB (XLEN-1)
  used as sign bit.

  If another value is given to PFRC in rs2 then the result is shifted
  accordingly, and shall satturate (not overflow) at INT_MIN and INT_MAX.

  PMAK reverses the process, with "effective fraction" in rs1 and "effective
  exponent" in rs2.

When called with rs2=zero, PFRC returns the nearest integer that best
approximates the posit, and PMAK simply converts an integer to a posit.

For zero and NaR we define:

  PEXP returns INT_MIN if the argument is zero and INT_MAX if the argument
  is NaR.
  
  PFRC returns zero if the argument is zero or NaR.
  
  PMAK returns NaR if rs1 is zero and rs2 is INT_MAX, and rounds to the
  next posit according to the above formula for all other values of rs1 and rs2.

For posit arithmetic on 8-bit and 16-bit values we define conversion functions
that convert between an XLEN posit and a smaller posit:

	PCF8  rd, rs1
	PCF16 rd, rs1
	PCF32 rd, rs1  (RV64-only)

	PCT8  rd, rs1
	PCT16 rd, rs1
	PCT32 rd, rs1  (RV64-only)

The PCF* instructions convert from a smaller format to XLEN, and PCT* converts
to a smaller format.

The PCT* result is stored sign-extended in the destination register. This allows
for direct comparison of posits <XLEN using standard integer compare instructions.

The use of conversion functions best fits the common use-model for 8-bit and
16-bit arithmetic, where we usually want to use 8-bit and 16-bit formats for storing
values in main memory, but actual computations should be performed in larger formats to
minimize rounding errors.

For architectures that support F/D/Q we also add conversion instructions:

	PCFF.<fmt> rd, rs1
	PCTF.<fmt> rd, rs1

PCFF.<fmt> converts an IEEE float (from a f0-f31 register) to a posit, and
PCTF.<fmt> converts a posit to an IEEE float.

On RV64 we also add *W variants of all the instructions above that operate on
posit32 instead of posit64.


Vector Posit Extension
======================

For architectures that support the V (vector) extension we define the following
OPIVV (vector-vector) instructions:

	VPADD.vv    vd, vs1, vs2, vm
	VPSUB.vv    vd, vs1, vs2, vm
	VPMUL.vv    vd, vs1, vs2, vm
	VPDIV.vv    vd, vs1, vs2, vm
	VPSQRT.vv   vd, vs1, vm
	VPSIGN.vv   vd, vs1, vm

The posit format is implicitly defined by the vector element size.

We also define the following OPIVX (vector-scalar) instructions:

	VPADD.vx    vd, vs1, rs2, vm
	VPSUB.vx    vd, vs1, rs2, vm
	VPRSUB.vx   vd, vs1, rs2, vm
	VPMUL.vx    vd, vs1, rs2, vm
	VPDIV.vx    vd, vs1, rs2, vm
	VPRDIV.vx   vd, vs1, rs2, vm

The scalar operand is always an XLEN posit, i.e. a posit64 on RV64 and
a posit32 on RV32. (I.e. there are no *W versions of these instructions.)

The existing integer min/max reduction functions also work with posits. So
we just need to add ordered and unordered posit sum instructions:

	VPREDOSUM.vs  vd, vs2, vs1, vm
	VPREDOUM.vs   vd, vs2, vs1, vm

TODO: Widening and conversion instructions.


Posit Quire Extension
=====================

** This section will be added later. **


Notes
=====

A previous version of this document had instruction for differnt posit "es"
settings. This version just supports the "es" settings recommended in the
posit standard, i.e. es=0 for posit8, 1 for posit16, 2 for posit32, and 3
for posit64.
