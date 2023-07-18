# SIC/XE's Feature Set

**This chapter is a work in progress.**

SIC/XE (SIC eXtended Edition) is an improved, but backwards-compatible version
of SIC.
Since most of the stuff is shared with SIC, this page lists just the differences
to not repeat itself too much.

## Specifications

- 1 megabyte of memory (2<sup>20</sup> bytes), compared to SIC's 32 kilobytes.
- 48-bit real numbers (floating points, "floats")

Integers are represented quite uniformly in computers, with with floats, which
have a changable accuracy, we have to know how they are stored.
SIC/XE splits the 48 bits into 3 sections:

- `Bit 0`: Sign (whether the number is positive or negative)
    - If `sign = 0` then the number is positive
    - If `sign = 1` then the number is negative
- `Bits 1-11`: Exponent (actual value is `exponent - 1024`)
    - The exponent can have values between 0 and 2047, so that means you can
    represent values from 2<sup>-1024</sup> to 2<sup>1023</sup>
- `Bits 12-47`: Mantissa (fraction)

|   Bits  |   0   |   1-11   |   12-47  |
|:-------:|:-----:|:--------:|:--------:|
|  Length | 1 bit |  11 bits |  36 bits |
| Purpose |  Sign | Exponent | Mantissa |

The actual value is calculated as [fraction] * 2<sup>([exponent] - 1024)</sup>.
The fraction is a number between 0 and 1, counted in binary (e.g. `0.1` in
binary is `1/2` in decimal, `0.01` is `1/4` and so on).

## Registers

SIC/XE introduces 4 new registers:

- `B` (3): base (used for addressing)
- `S` (4): general purpose
- `T` (5): general purpose
- `F` (6): floating point accumulator (used for arithmetic operations)

The first 3 registers are 24 bits in size, like other SIC registers, while the
`F` register is 48 bits in size (since floating point numbers on SIC/XE use
48 bits).

### The `B` register

TODO

### The `S` and `T` registers

The `S` and `T` registers are general purpose, meaning they don't have a
predefined purpose and can be used for anything.
Technically any of the registers can be used for anything, but since for example
the `A` and `F` registers are written to when running arithmetic instructions,
they are usually exclusively used for those.

### The `F` register

TODO

## Instruction formats

Since SIC/XE has a larger memory, the 15 bits used for the memory address in SIC
instructions isn't enough anymore, leading to new instruction formats.
SIC/XE also provides new instruction format that don't use memory at all,
which can be useful in some situations, as explained below.

SIC/XE supports 4 instruction formats and a backwards compatibility mode:
- Format 1: 8 bits for the opcode
- Format 2: 8 bits for the opcode, 4 bits for register 1 and 4 bits for register 2
- Format 3: 6 bits for the opcode, 6 bits for flags and 12 bits for the offset
- Format 4: 6 bits for the opcode, 6 bits for flags and 20 bits for the address

### Format 1

|   Bits  |   0-7  |
|:-------:|:------:|
|  Length | 8 bits |
| Purpose | Opcode |

Format 1 is used for some special operations, such as conversions between floats
and integers, halting the execution of the machine, normalizing values etc.
These commands don't need any extra variables, such as memory addresses or
register IDs, so the opcode alone is enough.

### Format 2

|   Bits  |   0-7  |    8-11    |    12-15   |
|:-------:|:------:|:----------:|:----------:|
|  Length | 8 bits |   4 bits   |   4 bits   |
| Purpose | Opcode | Register 1 | Register 2 |

Format 2 is used for register-only operations, such as arithmetic or shifting.
Although artithmetic can be done by getting a value from memory, using just
registers for the operation is much much faster.
On a hypothetical computer it doesn't really matter since it's emulated, but on
actual computers accessing memory is very expensive (time-wise) which is why
compilers optimize the use of registers as much as possible and is also why
cache exists (since accessing cache is still faster than accessing memory, but
slower than using just registers).

### Format 3

|   Bits  |   0-5  |   6   |   7   |   8   |   9   |   10  |   11  |  12-23  |
|:-------:|:------:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-------:|
|  Length | 6 bits | 1 bit | 1 bit | 1 bit | 1 bit | 1 bit | 1 bit | 12 bits |
| Purpose | Opcode |   n   |   i   |   x   |   b   |   p   |   e   |  Offset |

Format 3 is the most interesting, because based on the settings of the flags
it can either be used in SIC/XE-only mode, using advanced addressing modes and
offsets or it can be used in the backwards-compatible SIC mode, which means that
SIC/XE can run any SIC program, without it needing to be ported.

For a start, lets explain what the 6 flags mean:

- n: i**n**direct addressing
- i: **i**mmediate addressing
- x: inde**x**ed addressing
- b: **b**ase-relative addressing (relative to the base register `B`)
- p: **p**rogram counter-relative addressing (relative to the `PC` register)
- e: **e**xtended addressing

To accommodate a larger memory, there are 2 possibilities for adapting
instructions, either to extend the address field to 20 bits, or to use
relative addressing, meaning to only store an offset in the instruction and
then adding it to a value stored somewhere else (e.g. in a register).

Since there are 6 flags, there are quite a few addressing modes that SIC/XE
provides (luckily not all combinations are possible).
We'll go over them one by one, but first let's see a quick overview in the
tables below.

| b | p | Addressing mode |
|:-:|:-:|-----------------|
| 0 | 0 | Direct          |
| 0 | 1 | PC-relative     |
| 1 | 0 | Base-relative   |
| 1 | 1 | Disallowed      |

| n | i | Addressing mode |
|:-:|:-:|-----------------|
| 0 | 0 | SIC-format      |
| 0 | 1 | Immediate       |
| 1 | 0 | Indirect        |
| 1 | 1 | Direct          |

As we can see, we can use combinations of the `b` and `p` bits, and `n` and `i`
bits to set the addressing mode.
The `b` and `p` bits define how the target address is calculated, while the
`n` and `i` bits define how the target address is used in the instruction.

The two missing bits, `x` and `e` can be used in addition to the bits above,
but with a restriction that `x` (indexed addressing, as was with SIC) cannot
be used with immediate or indirect addressing, meaning that for `x` to be set
to 1, `n` and `i` both have to be either 0 or 1.
`e` can be used with any of the other bits and specifies whether to use the
extended mode (Format 4), where if `e = 0`, it means that the instruction is in
Format 3 and if `e = 1`, it means that the instruction is in Format 4 (extended
mode), which uses a longer address (explained below).

In the examples below, only the fields with fixed values matter for that
addressing mode, the other fields can be set as needed.

#### Base-relative addressing

| Opcode | n | i | x | b | p | e | Offset |
|:------:|:-:|:-:|:-:|:-:|:-:|:-:|:------:|
|        | 1 | 1 |   | 1 | 0 |   |        |

In base-relative addressing the target address is calculated as `TA = (B) + offset`,
meaning that the value in the base register is summed with the offset.
The offset is sometimes also called *displacement* and can be a value between
0 and 4095 inclusive.

#### PC-relative addressing

| Opcode | n | i | x | b | p | e | Offset |
|:------:|:-:|:-:|:-:|:-:|:-:|:-:|:------:|
|        | 1 | 1 |   | 0 | 1 |   |        |

In PC-relative addressing the target address is calculated as `TA = (PC) + offset`,
like before meaning that the value in the program counter register is summed with
the offset, but this time the offset is a value between -2048 and 2047 inclusive
(negative numbers are written in two's complement).

#### Direct addressing

| Opcode | n | i | x | b | p | e | Address |
|:------:|:-:|:-:|:-:|:-:|:-:|:-:|:-------:|
|        | 1 | 1 | X | 0 | 0 |   |         |

In direct addressing the target address is calculated based on the value of the
`x` bit (indexed addressing), as such:

- If `x = 0` then `TA = address`
- If `x = 1` then `TA = (X) + address`

#### Immediate addressing

| Opcode | n | i | x | b | p | e | Offset |
|:------:|:-:|:-:|:-:|:-:|:-:|:-:|:------:|
|        | 0 | 1 | 0 |   |   |   |        |

In immediate addressing the target address is used as the value (`value = TA`)

#### Indirect addressing

| Opcode | n | i | x | b | p | e | Offset |
|:------:|:-:|:-:|:-:|:-:|:-:|:-:|:------:|
|        | 1 | 0 | 0 |   |   |   |        |

In indirect addressing the value of the address at the target address is used
(`value = ((TA))`), meaning that first the value at the target address is checked
and then the value is retrieved from memory at that address.

#### Simple addressing

In simple addressing both `n` and `i` are either 0 or 1.

| Opcode | n | i | x | b | p | e | Offset |
|:------:|:-:|:-:|:-:|:-:|:-:|:-:|:------:|
|        | X | X |   |   |   |   |        |

When both are 0, the instruction is in the backwards-compatible SIC format
and the target address is based just on the `x` bit, while `b`, `p` and `e` are
used as part of the 15 bit address.

When both are 1, the value of the target address is used (`value = (TA)`).
