# bf16: a 16-bit brainfuck cpu

* <a href="#bf16-a-16-bit-brainfuck-cpu">bf16: a 16-bit brainfuck cpu</a>
* <a href="#introduction">Introduction</a>
* <a href="#the-instruction-set">The Instruction Set</a>
* <a href="#the-cpu">The CPU</a>
* <a href="#the-compiler">The Compiler</a>

## Introduction

The bf16 cpu was designed to run object code compiled from a brainfuck source file. Originally intended
to be 8-bit to be as small as possible, the design was changed to 16 bits so that more than 30,000 memory
addresses could be referenced, in line with the classic brainfuck implementation. 

## The Instruction Set

The instruction set includes the standard brainfuck instructions **> < + - , . [ ]** plus two meta
instructions **@ !**, as described below. This yields 10 total instructions, grouped into 5 pairs.

1\. Meta Instructions

* **@** tells the cpu to stop execution
* **!** is a no-op, essentially a wasted cpu cycle (sometimes useful for debugging)
* These instructions don't implement brainfuck commands, instead they are instructions to the cpu.

2\. Pointer Instructions

* **>** increments the data pointer by 1
* **<** decrements the data pointer by 1
* Pointer instructions take 1 cpu cycle

3\. Value Instructions

* **+** increments the value at the data pointer by 1
* **-** decrements the value at the data pointer by 1
* Value instructions take 1 cpu cycle

4\. IO Instructions

* **.** outputs the byte at the data pointer
* **,** inputs a byte to the data pointer
* IO instructions take 1 cpu cycle

5\. Loop Instructions

* These instructions are essentially jumps
* The jump destination is stored immediately after the loop instruction
* **[** sets the instruction pointer to the value after the loop instruction if the value at the data pointer is 0
* **]** sets the instruction pointer to the value after the loop instruction if the value at the data pointer isn't 0
* If a jump is required, loop instructions must read the next instruction pointer value and so take 2 cpu cycles
* If a jump is not required, loop instructions take 1 cpu cycle as with simple instructions.

Each pair of instructions is specified by a different bit in the object code (or the lack of bits, in the case
of meta instructions), while the low bit indicates which instruction of the pair to execute. This table, with
the specification bits highlighted, should help clarify:

<table>
  <tr>
    <th>Binary</th><th>Hex</th><th>Instruction</th><th>Type</th>
  </tr>
  <tr>
    <td>000<b>0000</b>0</td><td>0x00</td><td>@</td><td>M</td>
  </tr>
  <tr>
    <td>000<b>0000</b>1</td><td>0x01</td><td>!</td><td>M</td>
  </tr>
  <tr>
    <td>000000<b>1</b>0</td><td>0x02</td><td>&gt;</td><td>P</td>
  </tr>
  <tr>
    <td>000000<b>1</b>1</td><td>0x03</td><td>&lt;</td><td>P</td>
  </tr>
  <tr>
    <td>00000<b>1</b>00</td><td>0x04</td><td>+</td><td>V</td>
  </tr>
  <tr>
    <td>00000<b>1</b>01</td><td>0x05</td><td>-</td><td>V</td>
  </tr>
  <tr>
    <td>0000<b>1</b>000</td><td>0x08</td><td>.</td><td>IO</td>
  </tr>
  <tr>
    <td>0000<b>1</b>001</td><td>0x09</td><td>,</td><td>IO</td>
  </tr>
  <tr>
    <td>000<b>1</b>0000</td><td>0x10</td><td>[</td><td>L</td>
  </tr>
  <tr>
    <td>000<b>1</b>0001</td><td>0x11</td><td>]</td><td>L</td>
  </tr>
</table>

It is assumed that the program instructions will be loaded into a zeroed address space such that the cpu will
read 0x0 after the last instruction and stop execution. If this is not the case, implementations can add a 0x0
instruction after the last command at load time, or when object code is translated to a certain cpu implementation.
Compilers *should not* add a 0x0 instruction at compile time, so that multiple consecutive segments can be compiled
separately.

## The CPU

![bf16 cpu diagram](https://raw.github.com/briandef/bf16/master/images/bf16.cpu.png "images/bf16.cpu.png")

The bf16 cpu was designed in <a href="http://ozark.hendrix.edu/~burch/logisim/">Logisim</a>, a neat tool
for experimenting with digital components and circuits. It primarily consists of an instruction pointer IP,
instructions ROM, data pointer DP, and data RAM, plus all the components to run instructions. It faithfully
implements the above instruction set.

*Before you read on, you may wish to open bf16.circ in Logisim or view images/bf16.cpu.png*

The first meta instruction, stop execution **@**, is implemented by disabling IP updates when all the bits in
the instruction are 0 (or, conversely, only enabling IP updates when at least one bit in the instruction is set).
The no-op instruction **!** has no specific implementation beyond ensuring that no other instructions are
executed or processed this clock cycle (the second part of a two-cycle instruction may occur, however.)

Pointer instructions **> <** are implemented with a mux using the instruction bits to select either a 1 or a -1,
running that value through an adder with the current value of DP, and storing the result in DP.

Value instructions **+ -** are implemented by using the bits in the instruction to construct either a 1 or a -1,
running that value through an adder with the current value of the data at DP, and storing that value in RAM at DP.

Input/output instructions **. ,** use AND gates to determine which instruction is active. If output is selected,
the TTY reads an ascii value from RAM at DP and prints it. If input is selected, a byte flows from either the
stdin ROM or the keyboard components (depending on the stdin mux) through to RAM while the instruction bits are
used to enable RAM updates.

Loop instructions **[ ]** are the most complicated. Four separate AND gates (two for each instruction) look at
whether the value currently at RAM is 0. This activates either the branching OR gate or the incrementing OR gate.
The incrementing gate activates a mux which selects a 2 to add to IP instead of the normal 1, and execution
continues as normal. The branching gate activates a Q flip-flop which interferes with processing the next
instruction. On the next clock cycle, the flip-flop activates a mux which sends the no-op value of 0x1 to be
processed by the CPU. At the same time, the value of IP is set to the current instruction value. Execution
then continues as normal.

All this is easier to process after seeing it running in Logisim compared to just reading...

Other features of the cpu include a counter **#Instr** which counts how many cpu cycles have been processed
and a pin to choose whether stdin comes from the keyboard component or the ROM component, as mentioned above.

## The Compiler

The bf16-compiler was written in python. It uses an (initally empty) dictionary to keep track the instruction
object code and a stack to keep track of loops. A **for** loop steps over every character in the source code and
records the object code for the current instruction in the dictionary. If the instruction is a simple one-cycle
command **@ ! > < + - . ,** the compiler moves on to the next character.

Loop instructions are followed by the IP to jump to if needed. When processing opening loop instructions  **[**,
this location is not known until the matching end loop instruction **]** is found. Therefore, the compiler just
pushes the current location onto the stack and moves on. When the match is found, the compiler pops the most
recent IP location off the stack and uses that value to backfill the dictionary with the now known memory location.
It also uses the popped value to calculate the correct memory value for the current loop and puts it into
the dictionary.

If the compiler finds an end loop and the stack is empty, it errors with a message about unmatched loops.
Likewise if the stack isn't empty after code processing is complete.

## Compiling and Running Brainfuck Code

As mentioned above, the bf16-compiler turns brainfuck source code into object code for the cpu. The output can
be directly loaded onto a the instructions ROM in the circuit. However, Logisim requires a header on ROM images,
so after compiling there's an additional step. The `tools/make-rom` script adds the header to the ROM image so
that it will load properly in Logisim.

To compile the `helloword.bf` source code in the **examples** directory first run `bf16-compiler`, passing in
the code over stdin and saving stdout as the object code:

`<examples/helloworld.bf ./bf16-compiler > examples/helloworld.bf16`

Then make the rom image with `tools/make-rom`:

`<examples/helloworld.bf16 tools/make-rom > examples/helloworld.rom`

This is all done in one easy step with `tee`:

`<examples/hellworld.bf ./bf16-compiler | tee examples/helloworld.bf16 | tools/make-rom > examples/hellworld.rom`

At this point `hellworld.rom` can be loaded in Logisim by right clicking the instructions ROM and choosing
'Load Image'.

## Why?!

I designed this cpu just to learn and have something fun to play with, then discovered that it makes a halfway
decent brainfuck debugger, since you can see instructions, memory, input, and output all at once. 
