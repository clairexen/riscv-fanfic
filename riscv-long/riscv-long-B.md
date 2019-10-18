Proposal for RISC-V instruction formats >32 bit
===============================================

This is a proposal for RISC-V instruction formats for 48-bit instructions and larger.

Design goals:
- Provide a future-safe way of managing the encoding space
  - with plenty of reserved space so we never run out of space for instructions
  - but also means of providing reasonably compact encodings for frequently used instructions
- Uniform instruction formats that work with a wide range of instructions
  - so we can keep decoder logic simple even when adding loads of instructions over time

We define two instruction formats: "prefix", and "immediate".

     |              4                    |  3                   2        |          1                    |
     |7 6 5 4 3 2 1 0 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
     ----------------------------------------------------------------------------------------------------|
    ...       funct7   |   rs2   |   rs1   |  f3 |    rd   |    opcode   |  len  |   page  | 11|  11111  | prefix format
    ...                               immediate                          |  opc  |    rd   |len|  11111  | immediate format

The prefix format is simply a 16-bit prefix for a regular formatted instruction. It uses a 4-bit field to encode the instruction length. The instruction is `16*len+48` bits long, including the prefix. Thus, len=0 encodes for a 48-bit instruction and len=13 encodes for a 256-bit instruction. len=14 and len=15 are reserved for future standard and custom extensions respectively. The page field selects the opcode page, with pages 16-31 reserved for custom extensions.

The immediate format has only a 2-bit length field, encoding for instructions that are 48-bit (00), 64-bit (01), or 80-bit (10) in size. The 4-bit opc (opcode) field encodes for the following operations.

    | opc|
    |----|
    |0000| load integer immediate, zero extended
    |0001| load integer immediate, ones extended
    |1000| load floating point immediate
    |1001| jump and link
    |-01-| reserved
    |-1--| custom

Note that the opc field is formatted in a way that would allow for future instructions that are formatted similar to the "extended" instruction format proposed in [riscv-long-A.md](riscv-long-A.md).

In prefix format instructions, the lower bits of the opcode field may be used for additional immediate bits, effectively allocating naturally aligned consecutive opcodes to the same instruction.

----

```
==============================================================================

                                   APPENDIX

     Everything below is additional remarks and not part of the proposal

==============================================================================
```


Appendix I: (Un)frequently Asked Questions
==========================================

Q: How many opcodes does the 48-bit prefix-format (len=0) provide?

A: The equivalent of 2048 major opcodes (or 16384 minor opcodes) for standard extensions, and another 2048 major opcodes for custom extensions.


Appendix II: Example Instructions
=================================

The following sections describe how the above formats could be used, using some
concrete examples. Again, everything below is just an example, not part of the
proposal.


Load-large-immediate and long JALR
----------------------------------

The above instruction format is set up to support efficient encodings for
load-immediate and jump-and-link instructions.

The load-integer-immedate instructions come in ones-extended and zero-extended
forms for when the immediate is smaller than XLEN.

The load-float-immediate instructions are only valid for 32-bit and 64-bit immediates.

The jump-and-link instructions use the LSB bit of the immediate as sign bit. When `IALIGN=32`
then then the instruction is only valid if `imm[1]` is zero.


Overflow-Checked Arithmetic
---------------------------

The J Extension proposal discusses overflow-checked arithmetic instructions. One possible semantic for those would be
an instruction with an additional immediate that, when an overflow is detected, writes PC to rd and jumps to PC+imm.

For example, as 48-bit instruction with a 12-bit immediate we could fit 32 overflow-checked arithmetic instructions
in one opcode page:

    |              4                |  3                   2        |          1                    |
    |7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    ------------------------------------------------------------------------------------------------|
    |     imm7    |   rs2   |   rs1   | im3 |    rd   |  opcode |im2|0| 000 |   page  | 11|  11111  | Overflow-checked OP


"V" Vector Extension Ops
------------------------

Based on the current "V" vector extension proposal (https://riscv.github.io/documents/riscv-v-spec/),
something like the following 64-bit instruction could implement OP-V instructions with overrides
for the most relevant CSRs. This format would allocate an entire opcode page for V extension instructions.

    |      6                   5    |              4                |  3                   2        |          1                    |
    |3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8|7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2|1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |-------------------------------------------------------------------------------------------------------------------------------|
    |        VTYPE        |  RM | VM|   RSM   | F2|   rs2   |   rs1   |  F3 |    rd   |      F7     |  0001 |   page  | 11|  11111  | OP-V

F2 is and additional field for addressing an even wider range of operations.

F3 contains funct3, i.e. select OPIVV, OPFVV, OPMVV, OPIVI, OPIVX, OPFVF, or OPMVX.

F7 contains funct6, i.e. select the operation. Note that this field is 7 bits
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
