# Roc Optimizable Abstract Representation (ROAR)

> [!WARNING]
> This assumes familiarity with concepts like the stack, registers

Roc Optimizable Abstract Representation (ROAR) is proposed part of the `dev` backend to be between the `mono` stage and the machine instructions ultimately produced by the compiler. Currently, the `dev` backend tends to overuse memory and the stack, when ultimatly, intermediate computations could be stored in registers and not moved into memory. 
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
### ROAR goals 

ROAR seeks to help make the `dev` backend simpler, faster, and easier to maintain. It does this through the creation of an "abstract representation", where instructions are modeled as abstract components that while having the speed and efficencay of assembly has the easy optimizability and maintailibity of higher level languages. In this section, some more specific goals are noted

#### Abstraction and simplificiation of data modeled in the `dev` backend
ROAR seeks to abstract data away from where it is stored. This is done both for the purpose of making it easier to maintain and allowing optimizations to more easily reason about the data. Related to this is the goal hardware register allocation, where because there will now be a system in charge of maintaining the usage of memory (and registers) registers may now be used safely, increasing performance 

#### Increase runtime speed without decreasing build speed 
ROAR seeks to make a better system that is used more and produces better results, so that even though the results are better than can be reached in equal or less time. This would include also perhaps making compilation multi-threaded  

### Reading this Proposal 

> [!NOTE]
> Some confusion about this proposal might be resolved here! 

A *word*, as used here, is a 64 bit (8 byte) unit of information, as opposed to other word sizes such as 16 and 32 bit (among others). Unless otherwise specified, all values referenced are one word wide, and are unless stated otherwise unsigned 
Here, certain things will be written in a psuedo "ROAR assembly". "ROAR assembly", however, does not exist, and is merely used as a notational convention to represent the values being represented without specifiying an implementation and without taking attetion away from the concepts said. 

### Things not Covered 
There are a number of things that will *not* be covered in this proposal, and are noted here so that ROAR may be structed as to permit these in the future:
- Inlining of function calls 
- Semantic optimizations (that is, optimizations that rather than just dealing with memory also deal with specific instructions for instance [strength reduction](https://en.wikipedia.org/wiki/Strength_reduction))
## ROAR Storage and Values

ROAR instructions consist of three parts, the operation itself, `add`, `sub`, `mov`, among many others, the source operand(s), that is the arguements passed to the function, and the destination operand, where the value is being stored. 
In this manual, if a operation is in the form `D <- Op S [...]`, then `Op` is the operand, `S` (and the rest written `...`) is the source, and `D` is the result operand, so, for instance, `%2 <- add %0 %1` is saying set registers 2 to the sum of registers 1 and 0, there are some operations that don't have source and/or destination operand(s), however these will be explictly noted. 
> [!NOTE]
> Yes, this notation is *not* any conventional assembly. It's use, as will be elaborated, is to highlight the value being changed

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

### The Null Register
All operations in ROAR must take a destination register. The reasoning behind this is that it makes it easier to optimize if all instructions uniformly modify one thing, and that thing is easily accessible to Roc. However, certain operations either sometimes or always don't produce either in a specific case or always a specific value. For instance, if one wants to compare two integers, you want to set *flags* as if subtraction is happening but not actually produce any value. This is where the null register, here written `_`, comes in. It is used when one wants to either throw out the result of an operation, or the operation dosen't produce anything meaniful in the first place, so, for instance, `_ <- sub %0 %1` could be thought of as `cmp %0 %1`. Apart from just flag-setting instructions, certain instructions should *always* output to the null register, some of them are:
- `ret`
- `jmp` and it's derivatives 
- `store` 
### Structure Registers (Very much in progress)
While most things can be represented by Abstract Registers, some things cannot. One of the most important things that cannot be easily modeled by them are structures, those are data that does not fit into a small number of words. To create a structure register, one uses the instruction `%n <- create`, where `%n` stores a "refrence" to the register. Similar to refrences in high-level languages, rather than passing around the structure itself, we just pass a refrence stored in a given register to that structure. In addition, here sometimes `@n` will be used instead of `%n` for registers that store structure refrences for clarity  
There are three main instructions that use refrences to structure registers (apart from create), those being `load`, `store`, and `copy`. These are all just variations on the `mov` instruction, just specified to structs, as a quick list, where `A -> B` means "B being set to A":
- `load` is for `Structure -> Register`
- `store` is for `Register -> Structure`
- `copy` is for `Structure -> Structure`
Note that the forms of load and store are not just `%1 <- load @0` and `_ <- store @0 %1`, but rather `%1 <- load @0 %2` and `_ <- store @0 %2 %1`. `%2` is the offset and these instructions use indirect addressing (note that alignment may not be specified, the offset is always in bytes). The `copy` operation takes in total 6 arguemnts (although one of these is an unused null register) of the form `_ <- @0 %1 @2 %3 %4`, the offsets are covered before, and `%4` is a size arguement, basically saying "copy this many bytes from `@2` to `@0`

> [!NOTE]
> You may have noticed that the `store` and `copy` don't take a (useful) destination register, why is that? Because destination registers only serve to mark data that is being modified, and in the case of these instructions, the refrences themselves are not being modified, rather what's being modified are the things they are refereing to

## ROAR Instructions 
ROAR instructions were intended to be both 
- Abstract enough to be easily optimzable 
- But concrete enough they could quickly be converted to assembly 

### Function Calls 
ROAR is designed to work with descrete sets of instructions, usually individual functions. To call other functions in ROAR, the `call` instruction is used. The `call` instruction can take a variable number of arguements, but must always take at least one, that being the identifier of the function being called. It may then take one output register to store the return value in and some arguement registers that are then passed to the function. 
Note that functions in ROAR must *always* specify the number of arguements they take and where they should be stored, as well as specifying any of them that must be structural refrences. This is due to the fact that ROAR is intended to be hardware agnostic, and therefore must not specify any specific calling convention. 
All functions return a (potentially useless) value. These are in the function specifed as `ret X`, where X is the value being returned. Returns function similary to those in other languages, no further instructions are executed as control returns to the caller with the value. The value returned with `ret` is simply the output of `call`.

#### Function purity
One very important abstraction 

### Jump Instructions 
TODO

# TODO (This document)
- [ ] : Redo section on Abstract Stack
- [ ] : Finish section on Structure Registers 
- [x] : Work on outline 
- [x] : Work on goals
- [ ] : Word on packing
- [x] : Make something about the assembly just being a representation of the Rust implementation 
- [ ] : Talk about jumps and flags
- [ ] : Add section on actual optimizations done
- [ ] : Probaly some other stuff I forgot


[^1]: Not, in practice, actually infinite, instead represented by a integer, mostly likely an unsigned 32 bit integer. However, this *is* more than 4 billion registers, which should be more than enough 
[^2]: Technincally `jmp` and `call` themselves don't alter the registers, but they *do* cause them to be altered
[^3]: Because one major feature of ROAR is all things being assumed to be 64 bits, each of these *must* be aligned to the 8 byte mark, so the above is actually equivalent to `mov [%0+$97*8] %2`
