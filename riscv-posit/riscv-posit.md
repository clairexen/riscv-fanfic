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

The extension name "Xposit" shall be used for a core that supports all of the
instructions up to XLEN listed below, including the floating point convert
instructions when the core support F/D/Q, and the vector instructions when the
core supports the V extension.

The extension names "Xposit8", "Xposit16", "Xposit32", and "Xposit64" shall be
used for cores that support scalar posit instructions up to the specified size.
Such a core does not need to implement any posit-ieee floating point conversion
instructions or vector instructions, even if the core support F/D/Q and/or V.


Scalar Posit Extension
======================

The scalar posit extension uses the general purpose integer registers.

The posit instructions exist in .B, .H, .W, and .D variants, for 8-bit, 16-bit,
32-bit, and 64-bit posits. Posits that are smaller than XLEN are stored in the
LSB end of the register, sign-extended to XLEN. With the exception of PNAR,
the posit instructions ignore the upper bits of their inputs.

Posits compare like signed integers, so all integer branch instructions and compare
instructions work with posits as well, without the need for extra instructions.

Same for MIN/MAX (from B extension), negate, and abs().

The following new instructions are added to the OP major opcode:

	PADD.[BHWD]   rd, rs1, rs2
	PSUB.[BHWD]   rd, rs1, rs2
	PMUL.[BHWD]   rd, rs1, rs2
	PDIV.[BHWD]   rd, rs1, rs2
	PSQRT.[BHWD]  rd, rs1
	PSIGN.[BHWD]  rd, rs1
	PNAR.[BHWD]   rd, rs1

PSIGN returns the posit -1 if rs1 is negative, +1 if rs1 is positive, zero
if rs1 is zero, and NaR is rs1 is NaR.

PNAR returns the integer 1 is the posit argument is NaR, -1 if the posit
argument is not well formed (not properly sign extended), and 0 otherwise.

For "parsing" and generating posits we also add the following instructions:

	PEXP.[BHWD]  rd, rs1
	PFRC.[BHWD]  rd, rs1, rs2
	PMAK.[BHWD]  rd, rs1, rs2

PEXP returns the "effective exponent" encoded in the posit "R" and "E"
fields and implied by the posit size, and PFRAC returns the "effective fraction"
encoded in the posit "S" and "F" fields, when given PEXP(X) in rs2, so that,
ignoring rounting errors:

	X = exp2(PEXP(X)) * PFRAC(X, PEXP(X))

When PEXP(X) is given to PFRC in rs2, then PFRAC(X) is aligned to the MSB end
of the result, with the implicit 1 on position XLEN-2 and MSB (XLEN-1)
used as sign bit.

If another value is given to PFRC in rs2 then the result is shifted
accordingly, and shall satturate (not overflow) at XLEN_INT_MIN and XLEN_INT_MAX.

PMAK reverses the process, with "effective fraction" in rs1 and "effective
exponent" in rs2.

When called with rs2=zero, PFRC returns the nearest integer that best
approximates the posit, and PMAK simply converts an integer to a posit.

When called with rs1=1, PMAK returns the posit for exp2(rs2).

For zero and NaR we define:

- PEXP returns XLEN_INT_MIN if the argument is zero and XLEN_INT_MAX if the argument is NaR.
  
- PFRC returns zero if the argument is zero or NaR.
  
- PMAK returns NaR if rs1 is zero and rs2 is XLEN_INT_MAX, and rounds to the
  next posit according to the above formula for all other values of rs1 and rs2.

We also define conversion functions that convert between posit formats:

	PCVT.H.B rd, rs1   (convert from posit8 to posit16)
	PCVT.W.B rd, rs1   (convert from posit8 to posit32)
	PCVT.D.B rd, rs1   (convert from posit8 to posit64, RV64-only)

	PCVT.B.H rd, rs1   (convert from posit16 to posit8)
	PCVT.W.H rd, rs1   (convert from posit16 to posit32)
	PCVT.D.H rd, rs1   (convert from posit16 to posit64, RV64-only)

	PCVT.B.W rd, rs1   (convert from posit32 to posit8)
	PCVT.H.W rd, rs1   (convert from posit32 to posit16)
	PCVT.D.W rd, rs1   (convert from posit32 to posit64, RV64-only)

	PCVT.B.D rd, rs1   (convert from posit64 to posit8, RV64-only)
	PCVT.H.D rd, rs1   (convert from posit64 to posit16, RV64-only)
	PCVT.W.D rd, rs1   (convert from posit64 to posit32, RV64-only)

For architectures that support F/D/Q we also add conversion instructions:

	PCFF.[BHWD].[FDQ] rd, rs1   (convert from ieee float to posit)
	PCTF.[FDQ].[BHWD] rd, rs1   (convert from posit to ieee float)

PCFF converts an IEEE float (from a f0-f31 register) to a posit, and PCTF
converts a posit to an IEEE float.

In total, this are 4x6=24 R-type instructions, 4x4=16 unary compute instructions,
3x4=12 unary posit-posit convert instructions, and 2x3x4=24 unary posit-float
convertion instructions.

Thus the total required encoding space for this extension would be equivalent
to just under 26 R-type instructions.


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
	VPNAR.vv    vd, vs1, vm

The posit format is implicitly defined by the vector element size.

We also define the following OPIVX (vector-scalar) instructions:

	VPADD.vx    vd, vs1, rs2, vm
	VPSUB.vx    vd, vs1, rs2, vm
	VPRSUB.vx   vd, vs1, rs2, vm
	VPMUL.vx    vd, vs1, rs2, vm
	VPDIV.vx    vd, vs1, rs2, vm
	VPRDIV.vx   vd, vs1, rs2, vm

The scalar operand is always an XLEN posit, i.e. a posit64 on RV64 and
a posit32 on RV32.

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
posit standard, i.e. es=0 for posit8 (B), 1 for posit16 (H), 2 for posit32 (W),
and 3 for posit64 (D).
