# SIC's Feature Set

## Specifications

- 8-bit bytes (1 byte = 8 bits)
- 3-byte words (3 bytes = 1 word)
- 32 kilobytes of memory (2^15 bytes)
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

Each register has its own special purpose.

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

## Instruction Set

SIC only knows how to do basic operations, such as working with registers,
integer arithmetic (all in register `A`), comparing register `A` to a word in
memory, conditional jumping, and calling and returning from a subroutine using
register `L`.

### Register instructions

#### Load

- `LDA m`: Load a word from memory at address `m` into register `A` (`A <- (m..m+2)`)
- `LDCH m`: Load a byte (character) from memory at address `m` into the lowest
byte of register `A` (`A.low <- (m)`)
- `LDL m`: Load a word from memory at address `m` into register `L` (`L <- (m..m+2)`)
- `LDX m`: Load a word from memory at address `m` into register `X` (`X <- (m..m+2)`)

*Warning: `LDCH` only sets the lowest byte of register `A`, it leaves the other
bits unchanged, so you have to be careful when reading the value of register `A`
after using `LDCH`.*

#### Store

- `STA m`: Store a word from register `A` into memory at address `m` (`m..m+2 <- (A)`)
- `STCH m`: Store the lowest byte (character) from register `A` into memory at
address `m` (`m <- (A.low)`)
- `STL m`: Store a word from register `L` into memory at address `m` (`m..m+2 <- (L)`)
- `STSW m`: Store a word from register `SW` into memory at address `m` (`m..m+2 <- (SW)`)
- `STX m`: Store a word from register `X` into memory at address `m` (`m..m+2 <- (X)`)

*`(m..m+2)` means the value in memory from address (byte) `m` to `m+2`,*
*which is 3 bytes, i.e. a word.*

### Arithmetic instructions

All arithmetic instructions use the register `A` and store the result back into
it.

- `ADD m`: Sum the value in register `A` with a word from memory at address `m`
and store the result back into register `A` (`A <- (A) + (m..m+2)`)
- `DIV m`: Divide the value in register `A` with a word from memory at address
`m` and store the result back into register `A` (`A <- (A) / (m..m+2)`)
- `MUL m`: Multiply the value in register `A` with a word from memory at address
`m` and store the result back into register `A` (`A <- (A) * (m..m+2)`)
- `SUB m`: Subtract a word from memory at address `m` from the value in register
`A` and store the result back into register `A` (`A <- (A) - (m..m+2)`)

- `AND m`: Do a bitwise AND operation on the value of register `A` and a word
from memory at address `m`, and store the result into register `A`
(`A <- (A) & (m..m+2)`)
- `OR m`: Do a bitwise OR operation on the value of register `A` and a word from
memory at address `m`, and store the result into register `A`
(`A <- (A) | (m..m+2)`)

### Comparison instructions

- `COMP m`: Compare the value of register `A` with a word in memory at address
`m` and set the value of the `CC` bits according to the result
(`CC <- (A) : (m..m+2)`)
- `TIX m`: Increment the value of register `X` by 1, store it back into the
register and compare the value of the register to a word from memory at address
`m` (`X <- (X) + 1; (X) : (m..m+2)`)

There are three possible outcomes of the comparison:

- If `(A/X) = (m..m+2)` then `CC` is (`=`)
- If `(A/X) > (m..m+2)` then `CC` is (`>`)
- If `(A/X) < (m..m+2)` then `CC` is (`<`)

The actual underlying value for each of the results isn't mentioned in the SIC
specification, so it's up to each simulator to implement it and then consistently
use it (e.g `00` for `=`, `01` for `>` and `10` for `<`).

The `TIX` instruction is commonly used when counting or doing the assembly
version of a for loop.

### Jump instructions

All jump instructions set the value of the `PC` register to the address to jump to.

- `J m`: Unconditionally jump to address `m` (`PC <- m`)
- `JEQ m`: Jump to address `m` if `CC` is `=` (`PC <- m if CC is =`)
- `JGT m`: Jump to address `m` if `CC` is `>` (`PC <- m if CC is >`)
- `JLT m`: Jump to address `m` if `CC` is `<` (`PC <- m if CC is <`)

### Subroutines

- `JSUB m`: Store the value of the `PC` register to the `L` register and then
jump to address `m` (set the value of the `PC` register to the address `m` -
`L <- (PC); PC <- m`).
- `RSUB`: Set the value of the `PC` register to the value in register `L`, i.e.
restore the previous value and thus jump back to the parent routine at the
place you left off (`PC <- (L)`).

**TODO: Check if PC is incremented before it is stored in JSUB, since we want**
**to go to the next command, not the same one**

### Input and Output

SIC supports writing to and reading from external devices (usually text files
and/or standard input/output in simulator implementations).
Each device has an 8-bit ID (so there can be at most 256 devices), which has to
be stored somewhere in memory.

- `RD m`: Read a byte (character) from the device with ID written at address
`m` and store it into the lowest byte of register `A` (`A.low <- dev[(m)]`)
- `TD m`: Test device with ID written at address `m` (`CC <- test(dev[(m)])`)
    - If device is ready to send or receive bytes then `CC` is set to `<`
    - If device is not ready (is busy) then `CC` is set to `=`
- `WD m`: Write a byte (character) from register `A` to the device with ID
written at address `m` (`dev[(m)] <- A.low`)

*`dev[(m)]` means device, whose ID is stored in memory at the address `m`.*
