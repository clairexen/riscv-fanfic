Proposal for RISC-V instruction formats >32 bit
===============================================

This is a proposal for RISC-V instruction formats for 48-bit instructions and larger.

Design goals:
- Provide a future-safe way of managing the encoding space
  - with plenty of reserved space so we never run out of space for instructions
  - but also means of providing reasonably compact encodings for frequently used instructions
- Uniform instruction formats that work with a wide range of instructions
  - so we can keep decoder logic simple even when adding loads of instructions over time

Note: For simplicity we always refer to additional instruction word bits as "immediate" when
describing instruction formats in this document. This should not be understood as dedicating
those extra bits for immediate values. The instructions define the semantic of those additional bits.

We define three instruction formats: "prefix", "extended", and "immediate".

     |              4                    |  3                   2        |          1                    |
     |7 6 5 4 3 2 1 0 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     ----------------------------------------------------------------------------------------------------|
    ...       funct7   |   rs2   |   rs1   |  f3 |    rd   |    opcode   | plen  |   page  | 11|  11111  | prefix format
    ...           immediate              |    funct7   |   rs2   |   rs1   | 000 |    rd   |len|  11111  | extended format
    ...                               immediate                          |  opc  |    rd   |len|  11111  | immediate format

The "prefix format" is simply a 16-bit prefix for a regular instruction, with an immediate appended when
the instruction is 64-bit or longer. It uses the 4-bit plen field to encode the instruction length. The
instruction is `16*plen+48` bits long, including the prefix. Thus, plen=0 encodes for an 48-bit instruction
and plen=13 encodes for a 256-bit instruction. plen=14 and plen=15 are reserved for custom extensions and
future standard extensions respectively.

The page field selects the opcode page, with pages 16-27 reserved for vendor extensions, and pages 28-31
reserved for custom extensions. If each vendor is assigned a unique page+opcode pair, then the vendor
extension space is large enough for 1536 vendors. If each vendor is assigned a unique page+opcode+f3
tuple then there is enough space for 12288 vendors.

The 48-bit "prefix format" can be thought of as just much more 32-bit encoding space. The longer prefix
formats simply add additional immediate bits.

The "extended format" is just the regular 32-bit instruction format with f3=000 and additional immediate bits
following after the 32-bit instruction word. funct7 encoding space for "extended format" instructions is precious.
These instructions should therefore never occupy more than one funct7 code point, and, if possible, should share
the same funct7 code point with other instructions from the same extension. `funct7=-----10` is used for R4-type
instructions and `funct7=-----11` is reserved for custom extensions.

The "extended format" and the "immediate format" contain a 2-bit length field, encoding for instructions that
are 48-bit (00), 64-bit (01), or 80-bit (10) in size, with 11 indicating a prefix format instruction.

    |len|
    |---|
    | 00| 48-bit extended/immediate format
    | 01| 64-bit extended/immediate format
    | 10| 80-bit extended/immediate format
    | 11| (16*plen+48)-bit prefix format

The "immediate format" is a truncated form of the "extended format", with only the rd field and a 4-bit opc
(opcode) field encoding for the operation.

    | opc|
    |----|
    |-000| extended format
    |0001| jump and link
    |1001| load immediate
    |-01-| reserved
    |-10-| reserved
    |-11-| custom

In prefix format instructions, the lower bits of the opcode field may be used for additional immediate bits,
effectively allocating naturally aligned consecutive opcodes to the same instruction.

----

```
==============================================================================

                                   APPENDIX

     Everything below is additional remarks and not part of the proposal

==============================================================================
```


Appendix I: (Un)frequently Asked Questions
==========================================

Q: How many opcodes does the 48-bit "prefix format" (plen=0) provide?

A: The equivalent of 2048 major opcodes (or 16384 minor opcodes) for standard extensions,
and another 2048 major opcodes for vendor extensions and custom extensions.

Q: How many opcodes does the 48-bit "extended format" (len=0) provide?

A: Only 128, if only funct7 is used to distinguish opcodes, fewer if `funct7=-----11`
is used for ternary instructions (see "r4 extended format" below). That's why it is
strongly encouraged to use just one funct7 codepoint for a whole family of instructions,
or use the "f8 extended format" (see below).

Q: When should an instruction use the "prefix format", when the "extended format"?

A: The "extended format" is the preferred way of encoding an instruction that
has a fixed length, is 80-bit or shorter, and can be encoded using only one
funct7 code point or a fraction of that, and is used frequently enough to
warrant the use of precious "extended format" encoding space. All other
instructions should use the "prefix format".


Appendix II: Example Instructions
=================================

The following sections describe how the above formats could be used, using some
concrete examples. Again, everything below is just an example, not part of the
proposal.


Load-large-immediate and long JALR
----------------------------------

The "immediate format" instruction format provides efficient encodings for
load-immediate and jump-and-link instructions.

The load-integer-immedate instruction sign-extends the immediate, if the
immediate is smaller than XLEN.

The jump-and-link instruction uses the LSB bit of the immediate as sign bit
for sign-extension when the immediate is smaller than XLEN. When `IALIGN=32`
then the instruction is only valid if `imm[1]` is zero.

     |  4      2        |          1                    |
     |7 6  ... 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     |--------------------------------------------------|
     |     immediate    |  0001 |    rd   | 00|  11111  | JAL.32
     |     immediate    |  1001 |    rd   | 00|  11111  | LI.32

     |  6      2        |          1                    |
     |3 2  ... 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     |--------------------------------------------------|
     |     immediate    |  0001 |    rd   | 00|  11111  | JAL.48
     |     immediate    |  1001 |    rd   | 00|  11111  | LI.48

     |  7      2        |          1                    |
     |9 8  ... 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     |--------------------------------------------------|
     |     immediate    |  0001 |    rd   | 00|  11111  | JAL.64
     |     immediate    |  1001 |    rd   | 00|  11111  | LI.64


Extended format encoding space and R4-type extended format instructions
-----------------------------------------------------------------------

To preserve encoding space, extended format instructions should use the "f8 extended format"
whenever there is sufficient space left in the immediate field.

Instructions that require a third source operand (aka R4-type instructions)
should use the the "r4 extended format":

     |              4                    |  3                   2        |          1                    |
     |7 6 5 4 3 2 1 0 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     ----------------------------------------------------------------------------------------------------|
    ...     immediate    |     funct8    |   0000000   |   rs2   |   rs1   | 000 |    rd   |len|  11111  | f8 extended format
    ...     immediate    |     funct8    |   rs3   | 10|   rs2   |   rs1   | 000 |    rd   |len|  11111  | r4 extended format

In both instruction formats `funct8=111-----` is reserved for custom extensions.


Overflow-Checked Arithmetic
---------------------------

The J Extension proposal discusses overflow-checked arithmetic instructions. One possible implementation would be
an instruction with an additional immediate that, when an overflow is detected, writes PC to rd and jumps to PC+imm.

Using a single funct7 code point in 48-bit "extended format":

    |              4                |  3                   2        |          1                    |
    |7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    ------------------------------------------------------------------------------------------------|
    |          imm          |   OP  |    funct7   |   rs2   |   rs1   | 000 |    rd   | 00|  11111  |

The 4-bit OP field would select the arithmetic operation.


Integer Packed SIMD Instructions
--------------------------------

Using a single funct7 code point in 48-bit "extended format":

    |              4                |  3                   2        |          1                    |
    |7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    ------------------------------------------------------------------------------------------------|
    |S| sfmt| dfmt|        OP       |    funct7   |   rs2   |   rs1   | 000 |    rd   | 00|  11111  |

Where OP selects an operation and [sd]fmt selects the format for rs1/rs2 (sfmt) and rd (dfmt).

    [sd]fmt | vector element size
    -----------------------------
      000   | bit        (1 bit)
      001   | pair       (2 bit)
      010   | nibble     (4 bit)
      011   | byte       (8 bit)
      100   | half-word (16 bit)
      101   | word      (32 bit)
      110   | dword     (64 bit)
      111   | qword    (128 bit)

S=1 selects signed operation and S=0 selects unsigned operation. This is used
both for sign/zero extension when converting vector sizes and for determining
overflow/saturation for overflow checks and saturated arithmetic.

If dfmt is smaller than sfmt then the operation is performed in sfmt and then
the results are down-converted to dfmt. The additional vector elements are
zero-initialized and values that do not fit dfmt are saturated according to the
S field.

If sfmt is smaller than dfmt then the source arguments are first up-converted
to dfmt, with sign/zero extension according to the S field. The excess source
vector elements are ignored.

Some example operations (OP is 9 bits wide, enough for a rich set of operations):

*ADD, SUB, MUL, DIV, REM, ...* the regular arithmetic operations.

*OADD, OSUB, OMUL, ...* set the output to 1 if the operation overflows, and
to zero otherwise.

*SADD, SSUB, SMUL, ...* like regular arithmetic but saturate instead of overflow.

*SLT, SLE, SEQ* set output to 1 if less-than, less-equal, or equal, and to
zero otherwise.

*ALT* copy even elements from rs1 and odd elements from rs2 into the output
vector.

*MIX* even output elements are taken from the lower half of rs1 and odd output
elements are taken from the lower half of rs2.

*MIXU* even output elements are taken from the upper half of rs1 and odd output
elements are taken from the upper half of rs2.

*PERM* the elements in rs2 are indices of elements in rs1 to be copied into the
output vector. out-of-bounds indices result in zero fields in the output.

*SEL* a non-zero field in rs2 returns the corresponding field from rs1. Zero-elements
in rs2 return zero fields.

*NSEL* a zero field in rs2 returns the corresponding field from rs1. Non-zero-elements
in rs2 return zero fields.


"V" Vector Extension Ops
------------------------

Based on the current "V" vector extension proposal (https://riscv.github.io/documents/riscv-v-spec/),
something like the following 64-bit instruction could implement OP-V instructions with static overrides
for the most relevant CSRs.

Using a single funct7 code point in 64-bit "extended format":

    |      6                   5    |              4                |  3                   2        |          1                    |
    |3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8|7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |-------------------------------------------------------------------------------------------------------------------------------|
    |        VTYPE        |  RM |   RSM   | VM|       F8      |  F3 |    funct7   |   rs2   |   rs1   | 000 |    rd   | 01|  11111  | OP-V

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
