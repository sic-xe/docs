# SIC/XE Instruction Set

## SIC Instructions

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

### Subroutine instructions

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

## SIC/XE-only instructions

**This chapter is a work in progress.**
