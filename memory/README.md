[<< Back to Last Page](../)

# Memory

Before starting to emulate the CPU, we need some way of telling the CPU what it should be doing - that's where Memory comes in. While you will not be implementing every memory-related aspect right away, it's important to get a basic memory bus system implemented in order to test your CPU implementation easily.

## How does Memory work?

**Note:** If you're already familiar with the concept of memory addressing, memory buses, etc. you can probably skip this section without missing out on anything. If you aren't, stick around.

In hardware, memory is usually just a whole bunch of bytes, be it RAM, ROM or some external hardware connected to the CPU. The CPU needs a way of making it clear which byte in memory it's trying to access - that's where Memory Addresses come in. If you imagine memory as an array of bytes, a memory address is simply an index for that array. Address 0 accesses the first byte in the array, address 1 the second byte, and so on.

However, a lot of consoles have more than just a single memory chip. Most consoles have ROM, which is usually on the game cartridge, RAM for the game to use, which is inside the console itself, and some memory addresses are wired up to I/O ports of chips separate from the CPU, such as the PPU, the APU or the controller interface. This is where "memory mapping" comes into play - the process of assigning specific ranges of memory addresses to specific chips / ports. Which address ranges correspond to which sections of memory is usually summarized in a table known as a "memory map", which you'll find just below.

### Memory Mirrors

Due to this approach to managing memory, a quirk known as "Memory Mirrors" often occurs. In essence, memory mirroring means that two different memory addresses can refer to the same byte in memory. This is possible when the address range for a memory section is larger than the memory section itself. Let's take a look at an example:

You have a RAM chip capable of storing 128 bytes (`$00-$7F`), but the address range `$00-$FF` is mapped to this RAM chip. In this case, the most significant bit is simply ignored, as it physically cannot be connected to the address input of the RAM chip. Therefore, the address is bitwise ANDed with `$7F` to determine the effective address. Therefore, while the address `$7F` would access the last byte of memory on the chip, the address `$80` would access the first byte, as `$80 & $7F = $00`.

## The Memory Map

The NES uses 16-bit memory addresses, allowing for a range of `0x0000` - `0xFFFF`. However, a lot of address ranges go unused and act as mirrors. (Refer to the [Memory Mirrors](#memory-mirrors) section for a more in-depth explanation)

### Internal RAM ($0000 - $1FFF)

The address range from `$0000` to `$1FFF` is mapped to a 2KB internal RAM chip. Since 2KB would only require the range of `$0000`-`$07FF`, the ranges `$0800-0FFF`, `$1000-17FF` and `$1800-$1FFF` are all mirrors of this RAM chip. The initial state of this memory section is undefined, but the CPU is free to read from and write to any address in this range at any time. It's best represented as a simple byte array of size `$800`.

### PPU Registers ($2000-$3FFF)

The NES' Picture Processing Unit (PPU) has a total of 8 byte-wide registers, which can be accessed using this memory range. As these registers would only require the range of `$2000-$2007`, every 8-byte section between `$2008-$3FFF` is a mirror of these registers. For more details on these registers, refer to the PPU section of this documentation. While working on CPU emulation, writes to this memory region can be ignored and reads can be stubbed to zero.

### APU & I/O Registers ($4000-$4017)

The address range from `$4000` to `$4017` is mapped to various registers of the Audio Processing Unit (APU) as well as other I/O registers such as the controller interface. For more details on these registers, refer to the respective sections of this documentation. While working on CPU emulation, writes to this memory region can be ignored and reads can be stubbed to zero.

### APU Test Registers ($4018 - $401F)

The address range from `$4018` to `$401F` is mapped to special "APU Testing Registers", but these can be neglected for emulation. Writes to this memory region can be ignored and reads can be stubbed to zero.

### Cartridge Space ($4020-$FFFF)

This address range is mapped to the cartridge itself. How exactly it behaves is up to the Mapper the cartridge uses. For more details refer to the Mappers section of this documentation.

## How to implement Memory properly

While it may at first seem tempting to simply create a large byte array for the entire address range of `$0000-$FFFF`, that approach is most definitely going to complicate things further down the line. It's usually a better idea to keep different memory sections in separate arrays and variables and to use centralized functions for reading from and writing to memory, as this will allow you to implement quirks such as memory mirrors much more easily.

Additionally, you should be prepared to handle reads from and writes to the Cartridge Space region differently on a per-game basis, as different cartridges make this memory region behave differently.