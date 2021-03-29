Proposal for C2/C3/C4-type RISC-V Instruction Formats
=====================================================

This is a proposal for RISC-V 32-bit instruction formats using compressed rs1"/rs2"/rs3"/rd'
register addressing.

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |          imm12        |   rs1   |  f3 |    rd   |    opcode   | i-type
    |    funct7   |   rs2   |   rs1   |  f3 |    rd   |    opcode   | r-type
    |    rs3  | f2|   rs2   |   rs1   |  f3 |    rd   |    opcode   | r4-type
    |---------------------------------------------------------------|
    |          imm12        | f2| rs1"|  funct5 | rd" |    opcode   | c2-type
    |       imm9      | rs2"| f2| rs1"|  funct5 | rd" |    opcode   | c3-type
    | i2| rs3'|  imm4 | rs2"| f2| rs1"|  funct5 | rd" |    opcode   | c4-type
    |---------------------------------------------------------------|

With the rs1"/rs2"/rs3" encoding being identical to rs1'/rs2', except that 000
encodes for the zero register instead of x8 (s0/fp).

The proposed encoding space for instuctions of these types would be as follows:

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |          imm12        | f2| rs1"| 11|  f3 | rd" |  OP-IMM-32  | c2-type
    |        imm9     | rs2"| f2| rs1"| 10|  f3 | rd" |   MISC-MEM  | c3-type
    | i2| rs3"|  imm4 | rs2"| f2| rs1"| 11|  f3 | rd" |   MISC-MEM  | c4-type
    |---------------------------------------------------------------|

Using this encoding scheme we could easily reserve sufficient space for 32
instructions of each of the three c2/c3/c4-type instructions formats, with the
maximum 12/9/6-bit immediate. And of course more if not all immediate bits are
neded:

    | imm- |    number of insns    |
    | bits |   c2   |   c3  |  c4  |
    |------|-----------------------|
    |  12  |     32 |    -- |   -- |
    |  11  |     64 |    -- |   -- |
    |  10  |    128 |    -- |   -- |
    |   9  |    256 |    32 |   -- |
    |   8  |    512 |    64 |   -- |
    |   7  |   1024 |   128 |   -- |
    |   6  |   2048 |   256 |   32 |
    |   5  |   4096 |   512 |   64 |
    |   4  |   8192 |  1024 |  128 |
    |   3  |  16384 |  2048 |  256 |
    |   2  |  32768 |  4096 |  512 |
    |   1  |  65536 |  8192 | 1024 |
    |   0  | 131072 | 16384 | 2048 |
    |------|-----------------------|

The example instructions and bitmanip instructions below would take up a total of 9.4% of the c2-type encoding space, 15.6% of the c3-type encoding space, and 0.3% of the c4-type encoding space.


What problems does this solve?
------------------------------

**Problem:** There is a need for instructions that encode two source registers
plus an immediate. (See the work being done in the code size task group and the
bitmanip instructions below.)

**Problem:** There is a need for large encoding spaces for instructions that
have many instruction variants. (See the work bein done in the packed SIMD
group.)

**Problem:** There are useful immediate instructions that would be easy to
justify in terms of hardware cost, but not in terms of encoding space.
(Reverse-subtract and multiply-immdiate would be obvious examples.)

**Problem:** There are obvious instructions with three surce operands that
would be useful, but adding a third source operand results in instructions that
take up huge portions of encoding space, and means that a third read port must
be added to the register file. (See ternary instructions in bitmanip draft
spec.)

**Solution:** By using three bit encodings into a subset of the register file
(x0,x9..x15 for source registers and x8..x15 for the destination register) we
save encoding space: 4 bit for c2-type instructions, 6 bit for c3-type
instructions, and 8 bit (!) for c4-type instructions.

**Solution:** Limiting the ternary instructions to a quarter of the register
file means that only that portion of the register requires a third read port,
at least for microarchitectures that do not rename registers. (Furthermore, a
microarchitecture that can fuse a 32-bit instruction with a 16-bit instruction,
or two 16-bit instructions, a third read port would likely be needed on the
same quarter of the register file.

**Caveat:** Introducing instructions that can only access a fraction of the
register file makes register scheduling more difficult. But improving the
compilers capability to schedule registers around instructions that can only
access this portion of the register file will likely also benefit utilization
of compressed instructions.


Example Applications
--------------------

Instructions that effectively fuse an op-immediate instruction with a postfix or
prefix binary instruction, such as multiply-immediate-add:

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |        imm9     | rs2"| 00| rs1"| 10| 000 | rd" |   MISC-MEM  | MULIADD
    |        imm9     | rs2"| 01| rs1"| 10| 000 | rd" |   MISC-MEM  | MULIADDW
    |        imm9     | rs2"| 10| rs1"| 10| 000 | rd" |   MISC-MEM  | ADDIADD
    |        imm9     | rs2"| 11| rs1"| 10| 000 | rd" |   MISC-MEM  | ADDIADDW
    |---------------------------------------------------------------|

These 4 c3-type instructions would take up 12.5% of the c3-type encoding space.

Reverse-subtract-immediate:

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |          imm12        | 00| rs1"| 11| 000 | rd" |  OP-IMM-32  | RSUBI
    |---------------------------------------------------------------|

This c2-type instruction would take up 3.1% of the c2-type encoding space.

(No multiply-immediate will be needed if a 9-bit immediate is deemed sufficient,
because rs2"=000 woud effectively be a multiply-immediate instructions.)


Bitmanip Instuctions
====================

This section proposes replacemnts for the Zbt and Zbf ISA extensions in the
current BitManip spec.

Instead I propose new Zbt and Zbi extensions. With Zbt containing c4-type
terary instructions and c3-type versions of those instructions with an
immediate replacing the 3rd source argument. And with Zbi containing only
the c3-type instructions from Zbt, plus additional c3-type instructions.

**Zbt:** FSL FSR, SAP, CUT, MUX, MIX, FSLI, FSRI, SAPI, CUTI

**Zbi:** FSLI, FSRI, SAPI, CUTI, BFX, BFXU, BFP


Ternary Instructions
--------------------

Unlike the current Zbt draft spec, these ternary instructins use rs3" as the
control word. This way the immediate-version of the instructions are easier to
implement on architectures that do not support the versions with three source
arguments, as the non-immediate arguments are still rs1" and rs2", like with any
other instruction wth two source registers.

We define the following ternary operations:

**FSL rd, rs1, rs2, rs3**  
Funnel Shift Left. This shifts the bits in rs1 left, shifting in MSB bits from rs2, and rotating when the shift amount is greater than XLEN. With the shift amount in rs3.

**FSR rd, rs1, rs2, rs3**  
Funnel Shift Right. This shifts the bits in rs1 right, shifting in LSB bits from rs2, and rotating when the shift amount is greater than XLEN. With the shift amount in rs3.

**SAP rd, rs1, rs2, rs3**  
Shift And Place. This shifts the bits in rs1 left, replacing the vacancies in the LSB bits with the LSB bits from rs2, and rotating when the shift amount is greater than XLEN. That is, unlike FSL, the bits in rs2 do not move. SAP by half of XLEN is equivalent to PACK. SAP is equivaent to first shifting the 2nd operand left by XLEN-shamt, and then performing a FSL operation.

**CUT rd, rs1, rs2, rs3**  
Use the MSB bits from rs1 and the LSB bits from rs2, with rs3 specifying the first bit position in which to use bits from rs1, and rotating when the control word is greater than XLEN.

**MUX rd, rs1, rs2, rs3**  
Set rd to rs1 if rs3 is zero and to rs2 otherwise.

**MIX rd, rs1, rs2, rs3**  
Set each bit in rd to the value of the correspondig bit in rs1 one rs2, depending on the corresponding control bit in rs3.

"Rotating" in the descriptions of FSL FSR, SAP, and CUT above means that `shamt[0:log2(XLEN)-1]` acts in the obvious way, and `shamt[log2(XLEN)]` effectively swaps the first and second source argument, and all further bits in the shift amount are ignored. (*Note: Adding rotating and non-rotating versions of these instructions would be practically free. I just didn't want to make this proposal more complicated than it already is.*)

We further add immediate versions of FSL, FSR, SAP, ad CUT.

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    | 00| rs3"|  0000 | rs2"| 00| rs1"| 11| 001 | rd" |   MISC-MEM  | FSL
    | 00| rs3"|  0001 | rs2"| 00| rs1"| 11| 001 | rd" |   MISC-MEM  | FSR
    | 00| rs3"|  0010 | rs2"| 00| rs1"| 11| 001 | rd" |   MISC-MEM  | SAP
    | 00| rs3"|  0011 | rs2"| 00| rs1"| 11| 001 | rd" |   MISC-MEM  | CUT
    | 00| rs3"|  0100 | rs2"| 00| rs1"| 11| 001 | rd" |   MISC-MEM  | MUX
    | 00| rs3"|  0101 | rs2"| 00| rs1"| 11| 001 | rd" |   MISC-MEM  | MIX
    |---------------------------------------------------------------|
    |     imm7    | 00| rs2"| 00| rs1"| 10| 001 | rd" |   MISC-MEM  | FSLI
    |     imm7    | 01| rs2"| 00| rs1"| 10| 001 | rd" |   MISC-MEM  | FSRI
    |     imm7    | 10| rs2"| 00| rs1"| 10| 001 | rd" |   MISC-MEM  | SAPI
    |     imm7    | 11| rs2"| 00| rs1"| 10| 001 | rd" |   MISC-MEM  | CUTI
    |---------------------------------------------------------------|

Because of the rotating behavior of FSL/FSR/SAP/CUT the 8 LSB bits of the
control word would be significant on RV128. But only 7 bits are necessary for
the immediate version of the instruction, as the 8th bit can be implemented by
swapping the first and second source operands.

For the same reason the MSB of imm7 is always 0 on RV64 and the two MSB bits of
imm7 are always zero on RV32.

The above c4-type instructions take up 0.3% of the c4-type encoding space.

The above c3-type instructions take up 3.1% of the c3-type encoding space.


Bit-Field Instructions
----------------------

Bit-fields are contiguous regions of bits. We define four bit-field instructions: two for bit-field extract and two for bit-field place.

The bit-field extract instructions only replace two shifts, usually a 48-bit
sequence when compressed instructions are enabled, and a sequence that is easy
to fuse. But on small machines that can not fuse such sequences, an explicit
bit-field extract instruction can noticably improve perfomance, and in code with
lots of bit-field extracts it can noticably iprove code size.

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |   offset  |   length  | 00| rs1"| 11| 001 | rd" |  OP-IMM-32  | BFX
    |   offset  |   length  | 01| rs1"| 11| 001 | rd" |  OP-IMM-32  | BFXU
    |---------------------------------------------------------------|

With length=0 encoding for length=64. (Note that offset and length are only 6-bit wide, insufficient for full 128-bit bit-field extract on RV128.)

The BFX instruction sign-extends the extracted bit-field and the BFXU instruction zero-extends the extracted bit-field.

When called with offset=0 the BFX and BFXU instructions simply sign-extend or zero-extend the word in the LSB bits of the source operand.

The bit-field place instruction places the LSB bits of rs1 in a specified range in rs2. It is equivalent to `SAPI rd, rs1, rs2, offset; CUTI rd, rs2, rd, offset+length`.

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |  offset |  len  | rs2"| f2| rs1"| 10|  f3 | rd" |   MISC-MEM  | BFP
    |---------------------------------------------------------------|

With offset=0 encoding for offset=32 and len=0 encoding for len=16.

When an offset larger than 32 or a length larger than 16 is required, the operation must be implemented as two-instruction SAPI+CUTI sequence.

BFX and BFXU together take up 6.3% of the c2-type encoding space, and BFP takes up 3.1% of the c3-type encoding space.
