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

For a complete and easy-to-understand summary of all instructions and how they operate, check out [this website.](http://obelisk.me.uk/6502/reference.html) It describes the effects each instruction has on the status register, as well as the amount of CPU cycles each instruction (depending on the addressing mode used) takes. Refer to the following sections for in-depth descriptions and diagrams of timings for various types of instructions. (This information is taken from [the 6502_cpu.txt document](http://nesdev.com/6502_cpu.txt))

### Terminology

Due to the property of each CPU cycle being either a read from or write to memory, "dummy operations" can occur while the CPU is processing a cycle where memory accesses aren't required. These are referred to as "dummy reads" and "dummy writes" and may be used by games and mappers for timing purposes.

Refer to the following image for reference on the meaning of the color-coded tiles in the following timing diagrams:

![timing_diagram_ref](./timing_diagram_ref.png)

### Quick Reference

- [Accumulator & Implied Addressing](#accumulator---implied-addressing)
- [Immediate Addressing](#immediate-addressing)
- [Absolute Addressing - Read Instructions](#absolute-addressing---read-instructions)
- [Absolute Addressing - Write Instructions](#absolute-addressing---write-instructions)
- [Absolute Addressing - Read-Modify-Write Instructions](#absolute-addressing---read-modify-write-instructions)
- [Zero Page Addressing - Read Instructions](#zero-page-addressing---read-instructions)
- [Zero Page Addressing - Write Instructions](#zero-page-addressing---write-instructions)
- [Zero Page Addressing - Read-Modify-Write Instructions](#zero-page-addressing---read-modify-write-instructions)
- [Zero Page Indexed Addressing - Read Instructions](#zero-page-indexed-addressing---read-instructions)
- [Zero Page Indexed Addressing - Write Instructions](#zero-page-indexed-addressing---write-instructions)
- [Zero Page Indexed Addressing - Read-Modify-Write Instructions](#zero-page-indexed-addressing---read-modify-write-instructions)

### Accumulator & Implied Addressing

This applies to the following instructions: `ASL, CLC, CLD, CLI, CLV, DEX, DEY, INX, INY, LSR, NOP, ROL, ROR, SEC, SED, SEI, TAX, TAY, TSX, TXA, TXS, TYA`

The Accumulator / Implied Addressing variants of the instructions listed above each take 2 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The operation is performed on the target register (`NOP` idles during this cycle). A dummy read occurs on the address stored in PC, but PC is not incremented.

**Timing Diagram:**

![timing_acc_impl](./timing_acc_impl.png)

### Immediate Addressing

This applies to the following instructions: `ADC, AND, CMP, CPX, CPY, EOR, LDA, LDX, LDY, ORA, SBC`

The Immediate Addressing variants of the instructions listed above each take 2 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The next byte after the opcode is read using the address stored in PC and PC is incremented by 1. The operation is performed on the target register during the same cycle.

**Timing Diagram:**

![timing_imm](./timing_imm.png)

### Absolute Addressing - Read Instructions

This applies to the following instructions: `LDA, LDX, LDY, EOR, AND, ORA, ADC, SBC, CMP, BIT, LAX, NOP`

The Absolute Addressing variants of the instructions listed above each take 4 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The low byte of the source address is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The high byte of the source address is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 4:** The operation is performed on the target register using the value read from the memory address fetched in the previous 2 cycles.

**Timing Diagram:**

![timing_abs_read](./timing_abs_read.png)

### Absolute Addressing - Write Instructions

This applies to the following instructions: `STA, STX, STY, SAX`

The Absolute Addressing variants of the instructions listed above each take 4 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The low byte of the source address is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The high byte of the source address is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 4:** The value is written to the memory address fetched in the previous 2 cycles.

**Timing Diagram:**

![timing_abs_write](./timing_abs_write.png)

### Absolute Addressing - Read-Modify-Write Instructions

This applies to the following instructions: `ASL, LSR, ROL, ROR, INC, DEC, SLO, SRE, RLA, RRA, ISB, DCP`

The Absolute Addressing variants of the instructions listed above each take 6 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The low byte of the source address is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The high byte of the source address is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 4:** The value at the memory address fetched in the previous 2 cycles is read and temporarily buffered.
* **Cycle 5:** The value fetched in the previous cycle is dummy-written to the memory address fetched in cycles 2 & 3. Afterwards, the operation is performed on the value and the result is buffered.
* **Cycle 6:** The result value is written to the memory address fetched in cycles 2 & 3.

**Timing Diagram:**

![timing_abs_rmw](./timing_abs_rmw.png)

### Zero Page Addressing - Read Instructions

This applies to the following instructions: `LDA, LDX, LDY, EOR, AND, ORA, ADC, SBC, CMP, BIT, LAX, NOP`

The Zero Page Addressing variants of the instructions listed above each take 3 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The byte after the opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The byte fetched in the previous cycle is used as the lower byte of the memory address while the high byte is set to `$00`. The operation is performed on the target register using the value read from this address.

**Timing Diagram:**

![timing_zpage_read](./timing_zpage_read.png)

### Zero Page Addressing - Write Instructions

This applies to the following instructions: `STA, STX, STY, SAX`

The Zero Page Addressing variants of the instructions listed above each take 3 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The byte after the opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The byte fetched in the previous cycle is used as the lower byte of the memory address while the high byte is set to `$00`. The value is then written to this address.

**Timing Diagram:**

![timing_zpage_write](./timing_zpage_write.png)

### Zero Page Addressing - Read-Modify-Write Instructions

This applies to the following instructions: `ASL, LSR, ROL, ROR, INC, DEC, SLO, SRE, RLA, RRA, ISB, DCP`

The Zero Page Addressing variants of the instructions listed above each take 5 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The byte after the opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The byte fetched in the previous cycle is used as the lower byte of the memory address while the high byte is set to `$00`. A value is read from this memory addressed and buffered.
* **Cycle 4:** The buffered value is dummy-written back to the address used in the previous cycle. Afterwards, the operation is performed on the value and the result is buffered.
* **Cycle 5:** The buffered result value is written to the address used in Cycle 3.

**Timing Diagram:**

![timing_zpage_rmw](./timing_zpage_rmw.png)

### Zero Page Indexed Addressing - Read Instructions

This applies to the following instructions: `LDA, LDX, LDY, EOR, AND, ORA, ADC, SBC, CMP, BIT, LAX, NOP`

The Zero Page Indexed Addressing variants of the instructions listed above each take 4 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The byte after the opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The value at the memory address fetched in the previous cycle (with the upper byte set to `$00`) is dummy-read. Afterwards, the value of the index register (X or Y, depending on the instruction) is added to the address and buffered. This is done within the boundaries of 8 bit values, so the result is effectively ANDed with `$FF`.
* **Cycle 4:** The operation is performed on the target register using the value read from the memory address determined in the previous cycle.

**Timing Diagram:**

![timing_zpage_indexed_read](./timing_zpage_indexed_read.png)

(`I` in the diagram above represents the value of the index register used to offset the given address)

### Zero Page Indexed Addressing - Write Instructions

This applies to the following instructions: `STA, STX, STY, SAX`

The Zero Page Indexed Addressing variants of the instructions listed above each take 4 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The byte after the opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The value at the memory address fetched in the previous cycle (with the upper byte set to `$00`) is dummy-read. Afterwards, the value of the index register (X or Y, depending on the instruction) is added to the address and buffered. This is done within the boundaries of 8 bit values, so the result is effectively ANDed with `$FF`.
* **Cycle 4:** The value is written to the memory address determined in the previous cycle.

**Timing Diagram:**

![timing_zpage_indexed_write](./timing_zpage_indexed_write.png)

(`I` in the diagram above represents the value of the index register used to offset the given address)

### Zero Page Indexed Addressing - Read-Modify-Write Instructions

This applies to the following instructions: `ASL, LSR, ROL, ROR, INC, DEC, SLO, SRE, RLA, RRA, ISB, DCP`

The Zero Page Addressing variants of the instructions listed above each take 6 CPU cycles:

* **Cycle 1:** The opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 2:** The byte after the opcode is fetched from the address stored in PC and PC is incremented by 1.
* **Cycle 3:** The value at the memory address fetched in the previous cycle (with the upper byte set to `$00`) is dummy-read. Afterwards, the value of the index register (X or Y, depending on the instruction) is added to the address and buffered. This is done within the boundaries of 8 bit values, so the result is effectively ANDed with `$FF`.
* **Cycle 4:** A value is read from the memory address determined in the previous cycle and buffered.
* **Cycle 5:** The buffered value is dummy-written to the memory address determined in cycle 3. Afterwards, the operation is performed on the buffered value and the result is once again buffered.
* **Cycle 6:** The buffered result value is written to the memory address determined in cycle 3.

**Timing Diagram:**

![timing_zpage_indexed_rmw](./timing_zpage_indexed_rmw.png)

(`I` in the diagram above represents the value of the index register used to offset the given address)

**Note:** This section is currently incomplete. Further documentation will be added soon™.