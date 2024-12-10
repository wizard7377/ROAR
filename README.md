# Roc Optimizable Abstract Representation (ROAR)

> [!WARNING]
> This assumes familiarity with concepts like the stack, registers

Roc Optimizable Abstract Representation (ROAR) is proposed part of the `dev` backend to be between the `mono` bytecode and the machine instructions ultimately produced by the compiler. Currently, the `dev` backend tends to overuse memory and the stack, when ultimatly, intermediate computations could be stored in registers and not moved into memory. 
For example, the instructions:
```elm
x = 5
y = 3
z = x + y
z = 2 * z
```
might be translated into 
```gas
# .text
mov $5 X_LOC # x = 5
mov $3 Y_LOC # y = 3
mov [X_LOC] %rax # %0 = x
mov [Y_LOC] %rbx # %1 = y
add %rbx %rax # %0 = %0 + %1
mov %rax Z_LOC # Z = %0
mov [Z_LOC] %rax # %0 = Z
mul $2 %rax # %0 = %0 * 2
mov %rax Z_LOC # Z = %0
# ...
# .data
X_LOC: .byte
Y_LOC: .byte
Z_LOC: .byte
```
(Unless otherwise specified, AT&T arguement order is used (destination **last**), and GAS syntax used)
(This example is extremely prelimanry, I didn't check that this is what is actually generated, or even that this is valid assembly)
To instead
```gas
# .text
mov $5 X_LOC # x = 5
mov $3 Y_LOC # y = 3
mov [X_LOC] %rax # %0 = x
mov [Y_LOC] %rbx # %1 = y
add %rbx %rax # %0 = %0 + %1
mul $2 %rax # %0 = %0 * 2
mov %rax Z_LOC # Z = %0
# ...
# .data
X_LOC: .byte
Y_LOC: .byte
Z_LOC: .byte
```
## Outline

Ultimiltiy 
### ROAR Goals

## ROAR Storage and Values

ROAR instructions consist of three parts, the operation itself, `add`, `sub`, `mov`, among many others, the source operand(s), that is the arguements passed to the function, and the destination operand, where the value is being stored. 
In this manual, if a operation is in the form `Op S [...] D`, then `Op` is the operand, `S` (and the rest written `...`) is the source, and `D` is the result operand, so, for instance, `add %0 %1 %2` is saying set registers 2 to the sum of registers 1 and 0, there are some operations that don't have source and/or destination operand(s), however these will be explictly noted 

### Abstract Registers 
The most important feature of ROAR is the Abstract Register, which are used to:
- Store intermediate values
- Act as variables 
- Act as refrences 
- Pass arguements

Here, they are notated as a percent sign (`%`) followed by some number, corresponding to the register number. As opposed to computer registers, there are an infinite[^1] amount, so, for instance, `%432432` is a register, albiet one that will probably not be used
One very importatnt distinction between computer and ROAR registers, *ROAR registers represent presistent data*. This means, that, for example, even if there are 400 operations in between `x=4` and `x`'s next usage, `x` will still be stored in a register. Why the change? To make it easier to optimize code, because a distinction is not drawn between registers and small pieces of memory, we can decide whether to load something into memory or keep it in the computer registers by what is more efficient 
#### Types
To put it simply, Abstract Registers are completly untyped, one word wide storage spaces. That is to say, there is nothing in a register itself to say whether the bits (written in hex for brevity) `00 00 00 00 00 00 00 3E` represent the charecter `>`, the unsigned number 62, or the signed number +62. It is entierly the job of operations to say whether they are working on signed or unsigned arguements (with unsigned being assumed herein)

> [!NOTE]
> Words, as used here, the unit of 64 bit, 8 byte, or 16 nibbles (they're all the same size) 

### Abstract Stack (Needs to be redone)
While most operations only change information in the destination operand, there are two large exceptions, `jmp` and `call`[^2]. In ROAR, it is entierally up to the caller to save registers that will be operated on, that is to say, if `%3` is some value before a call, you should not expect it to be the same value after the call!
To solve this, one should use the *abstract stack* to "save" registers that one wants to persist across changes in code location
This is the only use of the abstract stack in ROAR, however. Call sites and returns are abstracted away in the instructions of `call` and `ret`, and arguements should be passed in registers, not on the stack

### Structure Registers (Very much in progress)
While most things can be represented by Abstract Registers, some things cannot. One of the most important things that cannot be easily modeled by them are structures, those are data that does not fit into a small number of words. To create a structure register, one uses the instruction `create A`, where `A` stores a "refrence" to the register. Similar to refrences in high-level languages, rather than passing around the structure itself, we just pass a refrence stored in a given register to that structure 
There are three main instructions that use refrences to structure registers (apart from create), those being `load`, `store`, and `copy`. These are all just variations on the `mov` instruction, just specified to structs, as a quick list, where `A -> B` means "B being set to A":
- `load` is for `Structure -> Register`
- `store` is for `Register -> Structure`
- `copy` is for `Structure -> Structure`

# TODO (This document)
- [ ] : Redo section on Abstract Stack
- [ ] : Finish section on Structure Registers 
- [ ] : Work on outline 
- [ ] : Work on goals
- [ ] : Make something about the assembly just being a representation of the Rust implementation 
- [ ] : Talk about jumps and flags
- [ ] : Add section on actual optimizations done
- [ ] : Probaly some other stuff I forgot


[^1]: Not, in practice, actually infinite, instead represented by a integer, mostly likely an unsigned 32 bit integer. However, this *is* more than 4 billion registers, which should be more than enough 
[^2]: Technincally `jmp` and `call` themselves don't alter the registers, but they *do* cause them to be altered
[^3]: Because one major feature of ROAR is all things being assumed to be 64 bits, each of these *must* be aligned to the 8 byte mark, so the above is actually equivalent to `mov [%0+$97*8] %2`
