Proposal for RISC-V instruction formats >32 bit
===============================================

This is a proposal for RISC-V instruction formats for 48-bit instructions and larger.

Design goals:
- Provide a future-safe way of managing the encoding space
  - with plenty of reserved space so we never run out of space for instructions
  - but also means of providing very compact encodings for frequently used instructions
  - reserve encoding space in ways that don't assume what future demand will look like
- Uniform instruction formats that work with a wide range of instructions
  - so we can keep decoder logic simple even when adding loads of instructions over time

We define five instruction formats: "prefix", "load-immediate", "jump-and-link", "compressed-packed", and "packed".

     |              4                    |  3                   2        |          1                    |
     |7 6 5 4 3 2 1 0 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     ----------------------------------------------------------------------------------------------------|
    ...       funct7   |   rs2   |   rs1   |  f3 |    rd   |     opcode    | len | 00|page | 00|  11111  | prefix format
    ...                               immediate                          |f| len |ssp| rd' |spc|  11111  | load-immediate format
    ...                               immediate                          |f| len |ssp| rd  |spc|  11111  | jump-and-link format
    ...           immediate              |      funct9     | rs2'| f2| rs1'| len |ssp| rd' |spc|  11111  | compressed-packed format
    ...           immediate              |    funct7   |   rs2   |   rs1   | len |    rd   |spc|  11111  | packed format


For comparison, the standard 32-bit format:

                                         |  3                   2        |          1                    |
                                         |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
                                         |---------------------------------------------------------------|
                                         |    funct7   |   rs2   |   rs1   |  f3 |    rd   |    opcode   | 32-bit format


Organization of the encoding space
----------------------------------

The length field uses the following encoding. Reserved entries may be used
later to encode for larger instructions, or for allocating additional opcode
space for the instruction lengths that already have an encoding.

    |len|
    |---|
    |000| 48-bit instruction
    |001| 64-bit instruction
    |010| 80-bit instruction
    |011| 96-bit instruction
    |1--| reserved
    |111| reserved for custom extensions (op != 11)
    |111| reserved instructions >96-bit  (op == 11)

The encoding space under each "len" value is divided into 4 spaces (spc).
If the space is not used for packed format instructions it is further
subdivided into 4 subspaces (ssp) each.


Prefix instruction format
-------------------------

Subspace spc=00 ssp=00 is always used for prefix format instructions.

For 48-bit instructions, the prefix format simply provides a huge extra
encoding space for more instructions that look like regular 32-bit
instructions, just with a two-bytes prefix.

A prefix format encoding space is organized in 8 "pages", each containing 256
opcodes, each equivalant in encoding space to one major opcode in the 32-bit
format. (Of course, one could also simply use a page as 15-bit prefix into a
33-bit instruction of any arbitrary custom format.)

For instructions >48-bit there is simply an additional immediate at the
end of the instruction (or more funct7, if you prefer to see it that way).

page=111 shall always stay reserved for custom extensions.


Load-immediate, jump-and-link, compressed-packed, and packed formats
--------------------------------------------------------------------

The remainging subspaces of spc=00 can be used for load-immediate,
jump-and-link, and compressed-packed instructions.

A subspace used for load-immediate/jump-and-link can host up to two
instructions, selected by the "f" field in instr[15].

A subspace used for compressed-packed instructions could host up to 2048
instructions, selected by funct9 and f2, if none of those instructions
would want to use any of those bits as additional parameters.

A space used for packed instructions could host up to 128 instructions,
selected by funct7, if none of those instructions would want to use any of
those bits as additional parameters.

Space spc=11 is always used for packed format instructions. Spaces spc=01
and spc=10 are allocated as-needed and stay reserved for now.


Register encoding in load-immediate, jump-and-link, compressed-packed
---------------------------------------------------------------------

jump-and-link instructions use 3-bit rd:

    rd := rd[2:0]         (x0-x7, includes zero, ra, t0)

compressed-packed instructions and load-immediate instructions use 3-bit rd',
rs1', rs2' fields, using the same encoding as compressed instructions:

    rx := 8 + rx'[2:0]    (x8-x15, i.e s0-s1, a0-a5)


----

```
==============================================================================

                                   APPENDIX

     Everything below is additional remarks and not part of the proposal

==============================================================================
```


Appendix I: (Un)frequently Asked Questions
==========================================

Q: Why have both the packed and the prefix format? Wouldn't "packed" be
sufficient? Some of the immediate bits in the packed format could be used to
distinguish instructions, solving the issue of "packed" providing only limited
encoding space for instructions.

A: In the prefix format page, opcode, and funct3 are all within the 32-bit of
the instruction word. Thus, assuming funct7 only contains additional arguments
for the instruction, a prefix-format instruction can be decoded by looking only
at the first 32-bit of the instruction. If we'd use the packed format only
and distinguish instructions using immediate bits, then the decoder would need
to look beyond the first 32-bits to decode an instruction.

Q: How to decide if an instruction should be using prefix format or packed
format?

A: A packed format instruction should be (1) fairly frequent, so that it pays of
to have a 16-bit shorter instruction word and (2) should only occupy one funct7
value (i.e. only use the immediate field for additional parameters), to ensure
there's enough space left for other packed instructions. Everything else should
use the prefix format.

Q: How many opcodes does the standard 48-bit prefix-format (len=00, spc=00, ssp=00) provide?

A: The equivalent of 2048 major opcodes, or 16384 minor opcodes.

Q: How many opcodes does the standard 48-bit packed-format (len=000, spc=11) provide?

A: The equivalent of only 1 minor opcode, but with an added 16-bit immediate.


Appendix II: Example Instructions
=================================

The following sections describe how the above formats could be used, using some
concrete examples. Again, everything below is just an example, not part of the
proposal.


Load-large-immediate and long JALR
----------------------------------

The above instruction format is set up to support efficient encodings for
load-immediate and jump-and-link instructions.

             |          1                    |
      9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     ----------------------------------------|
    ... imm  |E| len | 01| rd' | 00|  11111  | LLI.{32,48,64,80}
    ... imm  |E| len | 10| rd  | 00|  11111  | LJAL.{32,48,64,80}
    ... imm  |0| len | 11| rd' | 00|  11111  | LFI.{S,D}

LLI/LJAL extend their immediate with E to XLEN. Therefore the 48-bit LJAL.32
instruction can jump +/- 4GB.

LFI.S is a 48-bit instruction that loads an IEEE float32 immediate. If `FLEN>32`
then the immediate is NaN-boxed before storing it in the `f*` register rd.

Similarly, LFI.D is an 80-bit instruction that loads an IEEE float64 immediate.

LJAL instructions are only valid if `imm[0]` is zero. (`imm[1:0]` when `IALIGN=32`.)

(LLI = load large immediate, LJAL = long jump and link, LFI = load float immediate)


Bitfield extract and place
--------------------------

In the RISC-V Bit Manipulation Extension draft proposes two 64-bit instructions
with the following semantic (bfxp = bitfield extract and place):

    uint_xlen_t bfxp(uint_xlen_t rs1, uint_xlen_t rs2,
                    unsigned src_off, unsigned src_len, unsigned dst_off, unsigned dst_len)
    {
            assert(src_off < XLEN && src_len < XLEN && dst_off < XLEN && dst_len < XLEN);

            uint_xlen_t src_mask = rol(slo(0, src_len), src_off);
            uint_xlen_t dst_mask = rol(slo(0, dst_len), dst_off);
            if (dst_len == 0) dst_mask = ~(uint_xlen_t)0;

            uint_xlen_t value = ror((rs1 & src_mask), src_off);

            // sign-extend
            value = sra(sll(value, XLEN-src_len), XLEN-src_len);
            if (src_len == 0) value = ~(uint_xlen_t)0;

            return (rs2 & ~dst_mask) | (rol(value, dst_off) & dst_mask);
    }

    uint_xlen_t bfxpu(uint_xlen_t rs1, uint_xlen_t rs2,
                    unsigned src_off, unsigned src_len, unsigned dst_off, unsigned dst_len)
    {
            assert(src_off < XLEN && src_len < XLEN && dst_off < XLEN && dst_len < XLEN);

            uint_xlen_t src_mask = rol(slo(0, src_len), src_off);
            uint_xlen_t dst_mask = rol(slo(0, dst_len), dst_off);
            if (dst_len == 0) dst_mask = ~(uint_xlen_t)0;

            uint_xlen_t value = ror((rs1 & src_mask), src_off);

            return (rs2 & ~dst_mask) | (rol(value, dst_off) & dst_mask);
    }

With src\_off, src\_len, dst\_off, and dst\_len being 7-bit immediate arguments.
(For future-compatibility with RV128, all four arguments must be 7 bits wide.)

The following format simply uses one byte for each of the immediate arguments.

    |      6                   5    |              4                |  3                   2        |          1                    |
    |3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8|7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |-------------------------------------------------------------------------------------------------------------------------------|
    |0|   dst_len   |0|   dst_off   |0|   src_len   |0|   src_off   |    funct7   |   rs2   |   rs1   | len |    rd   |spc|  11111  | bfxp
    |0|   dst_len   |0|   dst_off   |0|   src_len   |0|   src_off   |    funct7   |   rs2   |   rs1   | len |    rd   |spc|  11111  | bfxpu


"V" Vector Extension Ops
------------------------

Based on the current "V" vector extension proposal (https://riscv.github.io/documents/riscv-v-spec/),
something like the following packed 64-bit instruction could implement OP-V instructions with overrides
for the most relevant CSRs:

    |      6                   5    |              4                |  3                   2        |          1                    |
    |3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8|7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |-------------------------------------------------------------------------------------------------------------------------------|
    |        VTYPE        |  RM |   RSM   | VM|       F8      |  F3 |    funct7   |   rs2   |   rs1   | len |    rd   |spc|  11111  | OP-V

The fields "op", "len", and "funct7" are fixed and select OP-V instructions.

F3 contains funct3, i.e. select OPIVV, OPFVV, OPMVV, OPIVI, OPIVX, OPFVF, or OPMVX.

F8 contains funct6, i.e. select the operation. Note that this field is 8 bits
long, allowing for additional instructions that would not fit into the 32-bit
base encoding.

VM is a two-bit variant of the base instruction "vm" field, supporting both
regular and complement masking.

RSM selects the vector register containing the mask. (This is always assumed
zero in the 32-bit base vector instruction encoding).

The 3-bit RM field contains "frm" for floating point instructions, and "vxrm"
and "vxsat" for fixed-point instructions.

VTYPE contains "vtype" (vediv, vsew, vlmul). This field is 11 bits wide to
support the same number of config bits as vsetvl{i}.

Note that with this format, AVL still needs to be set with vsetvl{i}.


Appendix III: Instructions >96-bits
===================================

The above encoding only encodes for instructions up to 96-bit, but reserves `len=111`
and `op=11` for longer instructions.

We can easily define a prefix format for larger instructions, with `instr[11:7]`
encoding for the length, where instruction length in `bits = 112 + 16*length`.

     |              4                    |  3                   2        |          1                    |
     |7 6 5 4 3 2 1 0 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     ----------------------------------------------------------------------------------------------------|
    ...       funct7   |   rs2   |   rs1   |  f3 |    rd   |     opcode    | 111 |  length | 11|  11111  |

If `length=31` is reserved for even longer instructions then this scheme would
encode for instructions of up to 592 bits length (74 bytes).

In this scheme there would be no packed format, load-immediate format, or jump-and-link format for
instructions >96-bits.
