# The CPU

After getting a basic memory and ROM loading system into place, the next step is usually emulating the CPU. The following section is going to elaborate on the basic way a CPU operates, but if you've already written an emulator before, chances are you'll already know most of it, so you can safely skip it.

## Basic Operation Concept

The basic idea of a usual CPU structure can be summarized in three steps, commonly referred to as "Fetch - Decode - Execute". During the first step, "Fetch", the CPU reads a value from memory. This value is called an "Opcode" and is intended to tell the CPU what exactly it should be doing. Each opcode has an associated "instruction", such as loading values from memory, storing them in memory, adding values together, etc. Determining which instruction should be run based on the Opcode is the "Decode" step. And, finally, the decoded instruction is executed in the final step.

In order to keep track of where in memory the CPU should read opcodes from an internal register named "Program Counter" (usually abbreviated to PC) is used. It simply stores the memory address the next opcode should be read from, and is incremented on every read. Some instructions, such as jump instructions, can also directly write to this register, making the CPU "jump" between code segments that are being executed. (Hence the name of the instruction)

Additionally, most processors have the capability of keeping a Stack. In practice this is simply just another register, named "Stack Pointer" (usually abbreviated to SP), which keeps track of a memory address. When data is "pushed" (or "written") to the stack, it's written to the address the stack pointer is pointing to, followed by the stack pointer being decremented. Data can be "popped" (or "read") from the stack as well by doing the same thing in reverse - first increment the stack pointer, then read the value at the resulting address. This is most commonly used for "subroutines" - sections of code that are separate from the "main line" of code. For example, the main code section could call a subroutine - this pushes the address of where the call operation occurred to the stack. Next the CPU jumps to the subroutine and executes it until it hits a return instruction, which makes the CPU pop the address off the stack and jump back to it, so that the main code can resume execution.

## The NES CPU - An Overview

The processor used in the NES is a variant of the 6502 processor. Compared to CPUs like the Gameboy's SM83 it's quite compact in terms of supported instructions, however, each of these instructions has a large amount of different "addressing modes" (determining where in memory the value which an instruction should be executed on is read from).

Additionally, the CPU is in a slightly unusual relationship with memory, in that every single clock cycle either a read or a write operation is performed. This causes certain timing quirks with certain instructions, as memory may be read from or written to unexpectedly.

### Registers

The following table gives a short overview of the data registers present in the CPU:

| **Name**               | **Size**     | **Description**                                              |
| ---------------------- | ------------ | ------------------------------------------------------------ |
| Accumulator (`A`)      | 8 bit        | The Accumulator register is the main register for all instructions that involve arithmetic or bitwise operations. |
| Indexes (`X` & `Y`)    | 8 bit (each) | The X and Y registers can be used to offset memory accesses with certain addressing modes or simply act as additional data storage. |
| Status (`P`)           | 8 bit        | The Status Register contains bitflags which are set or reset depending on certain conditions when an instruction is executed (such as overflow on addition, etc.) |
| Program Counter (`PC`) | 16 bit       | The Program Counter keeps track of the memory address the next instruction is to be fetched from. |
| Stack Pointer (`S`)    | 8 bit        | The Stack Pointer keeps track of the lower boundary of the stack and is used by instructions that involve pushing data to or popping data from the stack. |

#### The Status Register

The following image is a visualization of the Status Register P and the names for each section of bits within it:

![statusreg](./statusreg.png)

* **[C] Carry:** This flag is usually used by arithmetic instructions to represent whether a carry resulted from the operation, as well as bit shift instructions.
* **[Z] Zero:** Almost all instructions set this flag if the result of the performed operation is zero, otherwise it is reset.
* **[I] Interrupt Disable:** If this flag is set, no CPU Interrupts (aside from NMIs) can occur. This is negligible for emulators early on in development.
* **[D] Decimal:** This flag is a leftover from the original 6502 CPU and has no effect, as the NES variant is lacking Decimal mode functionality.
* **[V] Overflow:** Set by arithmetic instruction if an overflow occurs or by the `BIT` instruction.
* **[N] Negative:** Almost all instructions set this flag to the same value as bit 7 of the result of the instruction.
* **[B] The B Flag:** The B flag is technically not part of the Status Register itself, but rather additional data which is added when the register is pushed to the stack. Bit 5 is always 1, the value of bit 4 depends on how the register was pushed. If it was pushed as the result of a `PHP` or `BRK` instruction, it is set to `1`, if it was pushed as the result of an interrupt or an NMI, it is set to `0`. Emulating the behaviour of the `PHP` instruction is recommended for early CPU development, as the `nestest` ROM commonly used for debugging depends on it.

## The Instruction Set

For a complete and easy-to-understand summary of all instructions and how they operate, check out [this website.](http://obelisk.me.uk/6502/reference.html) It describes the effects each instruction has on the status register, as well as the amount of CPU cycles each instruction (depending on the addressing mode used) takes.

**Note:** This section is currently under construction. Eventually, this will contain cycle-by-cycle descriptions of each instruction type / addressing mode combination. For a slightly less visually appealing description, refer to [the 6502_cpu.txt document.](http://nesdev.com/6502_cpu.txt)