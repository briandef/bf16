bf16: a 16-bit brainfuck cpu
====

The bf16 cpu was designed to run object code compiled from a brainfuck source file. Originally intended
to be 8-bit to be as small as possible, the design was changed to 16 bits so that more than 30,000 memory
addresses could be referenced, in line with the classic brainfuck implementation. 

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

* **[** sets the instruction pointer to the value after the loop instruction if the value at the data pointer is 0
* **]** sets the instruction pointer to the value after the loop instruction if the value at the data pointer is 0
* These instructions are essentially jumps
* If a jump is required, loop instructions must read the next instruction pointer value and so take 2 cpu cycles
* If a jump is not required, loop instructions take 1 cpu cycle as with simple instructions.

Each pair of instructions is specified by a different bit in the object code, while the low bit indicates which
instruction of the pair to execute. This table, with the specification bit highlighted, should help clarify:

<table>
  <tr>
    <th>Binary</th><th>Hex</th><th>Instruction</th><th>Type</th>
  </tr>
  <tr>
    <td>000000<b>0</b>0</td><td>0x00</td><td>@</td><td>M</td>
  </tr>
  <tr>
    <td>000000<b>0</b>1</td><td>0x01</td><td>!</td><td>M</td>
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
