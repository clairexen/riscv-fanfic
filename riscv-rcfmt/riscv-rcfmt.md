Proposal for RISC-V RC formats
==============================

This is a proposal for RISC-V 32-bit instruction formats using compressed rs1'/rs2'/rd'
register addressing.

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |    funct7   |   rs2   |   rs1   |  f3 |    rd   |    opcode   | r-format
    |     imm9        | rs2'| f2| rs1'|  funct5 | rd' |    opcode   | rc-format
    |     imm12             | f2| rs1'|  funct5 | rd' |    opcode   | ic-format

If we for example dedicate funct3=1xx in MISC-MEM to this format, then this
would give us encoding space for 64 three-address instructions with 9-bit
immediate.

If we dedicate one minor opcode in OP-IMM or OP-IMM-32 to this format, then
this would give us encoding space for 16 two-address instructions with 12-bit
immediate.

**Example Applications:**

Instructions with fused postfix-add-immediate, such as mul-addi or add-addi.

Immediate instructions that are used frequently, but not frequently enough
to warrant a full minor opcode in OP-IMM/OP-IMM-32, such as multiply-immediate.

A bit-field-place instruction that could replace the current Zbf proposal.
