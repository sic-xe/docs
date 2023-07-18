# SIC's Feature Set

## Specifications

- 8-bit bytes (1 byte = 8 bits)
- 3-byte words (3 bytes = 1 word)
- 32 kilobytes of memory (2<sup>15</sup> bytes)
- 24-bit unsigned integers with two's complement for negation (no floating points)
- 8-bit ASCII for characters
- Up to 265 "devices" to write bytes to or read bytes from

## Registers

SIC has 5 registers (listed as `mnemonic (number):`):

- `A`  (0): accumulator (used for arithmetic operations)
- `X`  (1): index (used for storing and calculating addresses)
- `L`  (2): linkage (used for jumping to memory addresses and return values)
- `PC` (8): program counter (used for holding the address of the next instruction)
- `SW` (9): status word (holds various info, e.g. flags)

Each register has its own special purpose and all 5 registers are 24 bits in
size (3 words).

### The `A` register

The `A` register is used when doing arithmetic operations, where its value is
modified (added to, subtracted from etc.) with the value from the address,
specified in the instruction.
The result is always stored back into register `A`.

### The `X` register

The `X` register is used for storing an offset, which is used when *indexed*
*addressing* is enabled (explained in the next section).

### The `L` register

The `L` register is used for storing the return address, when going into a
subroutine.

### The `PC` register

The `PC` register is used for storing the next instruction's address, so that
the microprocessor know where to find the instruction to execute.
The address stored in the register is implicitly (automatically) incremented
after executing each instruction.
The value of the `PC` register doesn't normally need to be set manually, so
SIC doesn't offer a separate instruction for it.

### The `SW` register

The `SW` register is split into multiple meaningful sections of bits, which
each hold useful information:

- `Bit 0`: Mode
    - If `mode = 0` then SIC is in user mode
    - If `mode = 1` then SIC is in supervising mode
- `Bit 1`: State
    - If `state = 0` then SIC is in running state
    - If `state = 1` then SIC is in idle mode
- `Bits 2-5`: PID (the process ID)
- `Bits 6-7`: CC (Condition Code - set by the compare operation and used in
conditional jumps)
- `Bits 8-11`: Interrupt mask
- `Bits 12-15`: Unused
- `Bits 16-23`: Interrupt Code (Interrupt Service Routine)

|   Bits  |   0   |   1   |   2-5  |   6-7  |      8-11      |  12-15 |      16-23     |
|:-------:|:-----:|:-----:|:------:|:------:|:--------------:|:------:|:--------------:|
|  Length | 1 bit | 1 bit | 4 bits | 2 bits |     4 bits     | 4 bits |     8 bits     |
| Purpose |  Mode | State |   PID  |   CC   | Interrupt mask | Unused | Interrupt Code |

**TODO: Figure out what separate sections do**

The value of the `SW` register doesn't normally need to be set manually, so
SIC doesn't offer a separate instruction for it.

## Instruction Format

SIC only supports a single instruction format (24-bit):

- 8 bits for opcode
- 1 bit for indexing flag (`x` bit)
- 15 bits for the address

|   Bits  |   0-7  |       8       |   9-23  |
|:-------:|:------:|:-------------:|:-------:|
|  Length | 8 bits |     1 bit     | 15 bits |
| Purpose | Opcode | Indexing flag | Address |

The microprocessor has to know where to get the value to use in an operation.
It can find the memory address of that value (the **target address** or TA) from
the address bits, but by combining the addressing bits with the indexing bit
value, it can do this in two ways:

- If `x = 0` then TA equals the address directly (*direct addressing mode*)
- If `x = 1` then TA equals the address plus the value of the `X` register
(*indexed addressing mode*)

| `x` bit |    Target address    | Addressing mode |
|:-------:|:--------------------:|:---------------:|
|    0    |    `TA = address`    |      Direct     |
|    1    | `TA = address + (X)` |     Indexed     |

*`(X)` means the value in register `X`.*
