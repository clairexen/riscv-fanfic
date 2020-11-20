Proposal for RISC-V RC and IC formats
=====================================

This is a proposal for RISC-V 32-bit instruction formats using compressed rs1'/rs2'/rd'
register addressing.

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |    funct7   |   rs2   |   rs1   |  f3 |    rd   |    opcode   | r-format
    |     imm9        | rs2'| f2| rs1'|  funct5 | rd' |    opcode   | rc-format
    |     imm12             | f2| rs1'|  funct5 | rd' |    opcode   | ic-format

If we for example dedicate funct3=11x in MISC-MEM to rc-format instructions,
then this would give us encoding space for 32 three-address instructions with
9-bit immediate.

If we dedicate one minor opcode in OP-IMM or OP-IMM-32 to the ic-format, then
this would give us encoding space for 16 two-address instructions with 12-bit
immediate.

Example Applications
--------------------

Instructions that effectively fuse an op-immediate instruction with a postfix or
prefix binary instruction, such as multiply-immediate-add.

Immediate instructions that are used frequently, but not frequently enough
to warrant a full minor opcode in OP-IMM/OP-IMM-32, such as multiply-immediate.

A bit-field-place instruction that could replace the current Zbf proposal.

Proposed encodings for preliminary tests within funct3=111 in MISC-MEM and OP-IMM-32:

    |  3                   2        |          1                    |
    |1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6|5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0|
    |---------------------------------------------------------------|
    |     imm12             | 00| rs1'|  11100  | rd' |   0011011   | MULI
    |     imm12             | 10| rs1'|  11100  | rd' |   0011011   | MULIW
    |---------------------------------------------------------------|
    |     imm9        | rs2'| 00| rs1'|  11100  | rd' |   0001111   | MULIADD
    |     imm9        | rs2'| 10| rs1'|  11100  | rd' |   0001111   | MULIADDW
    |---------------------------------------------------------------|
    |     imm9        | rs2'| 00| rs1'|  11101  | rd' |   0001111   | ADDIADD
    |     imm9        | rs2'| 10| rs1'|  11101  | rd' |   0001111   | ADDIADDW
    |---------------------------------------------------------------|
    |  offset |  len  | rs2'| 00| rs1'|  11110  | rd' |   0001111   | BFP
