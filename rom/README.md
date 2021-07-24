[<< Back to Last Page](../)

# ROMs & Mappers

After setting up a basic memory system it's time to get a ROM file loading system into place. Generally there's two "versions" of NES ROM files, namely iNES and NES 2.0, however, for a start, only supporting iNES ROMs is enough.

## PRG and CHR ROM

Many documentations (including this one in the following paragraphs) use the terms "PRG ROM" and "CHR ROM". These refer to the two different chips usually present on an NES cartridge. "PRG", standing for "Program", contains the actual game code and is directly accessible by the CPU, while "CHR", standing for "Character", is only directly accessible by the PPU and contains graphics data.

## The iNES Format

This next section will explain in detail how to decode iNES ROM files (while leaving out a few details unnecessary for emulators in an early state of development).

### The iNES Header

The first 16 bytes of every iNES ROM file are referred to as the "header", containing information relevant for interpreting the rest of the file correctly. These 16 bytes are themselves split up into multiple sections which will be explained in the following paragraphs.

#### Bytes 0-3 - Header Title

The "header title" consists of the 4 constant bytes `$4E $45 $53 $1A`, which are ASCII codes for "NES" followed by an end-of-file byte. The point of these bytes, as for any header title section, is to identify the file type and make sure that the file you're trying to open is actually an NES ROM. If these bytes don't match up, the user is most likely trying to open an invalid file as a ROM.

#### Byte 4 - PRG ROM Size

Byte 4 of the iNES header describes the size of the PRG ROM chip containing the game code in 16KB (16384 Bytes) units. For example, if this byte has the value `$02`, the ROM file contains 32768 bytes of PRG ROM data.

#### Byte 5 - CHR ROM Size

Byte 5 of the iNES header describes the size of the CHR ROM chip containing graphics data in 8KB (8192 Bytes) units. Unlike the PRG ROM Size byte, this byte may also be set to `$00`, in which case the cartridge is equipped with modifiable CHR RAM instead. However, this can be ignored for emulators early in development.

#### Byte 6 - Flags 6

Byte 6 of the iNES header is the first flags byte containing multiple bitflags. When starting off with a ROM loading system, most of these can be ignored however. The upper 4 bits of this byte represent the lower 4 bits of the mapper number which will be important later on. Checking bit 2 of this byte may also be required in very rare cases (depending on the ROM that's being loaded), as, if it is set, the file contains a 512 byte "Trainer", which can effectively be ignored for emulation, but must be skipped while loading the ROM file, as it offsets PRG and CHR data.

#### Byte 7 - Flags 7

The upper 4 bits of byte 7 of the iNES header contain the upper 4 bits of the mapper number. All other bit flags can be ignored by emulators early on in development.

#### Bytes 8-15 - Flags & Padding

While bytes 8-10 still contain flags, these are mostly irrelevant, especially when starting off with the implementation of a ROM loading system. The remaining bytes 11-15 are padding and can be fully ignored.

### ROM Data

Following the 16 byte header is the actual ROM data, split up into an optional 512 byte Trainer, PRG ROM and CHR ROM. It is useful to split these ROM segments up and to store them in separate arrays, as will be apparent later on when implementing the first Mapper. Refer to the following table for a quick summary of the ROM file structure:

| **Section Name** | **Section Size (Bytes)** | **Notes**                                                    |
| ---------------- | ------------------------ | ------------------------------------------------------------ |
| ROM Header       | 16                       | Always at the start of the file, always fixed size           |
| Trainer          | 512                      | Only present if ROM header bit is set, otherwise the Header is followed directly by PRG ROM Data |
| PRG ROM          | 16384 * (PRG ROM Size)   | Contains game code                                           |
| CHR ROM          | 16384 * (CHR ROM Size)   | Not present if CHR ROM Size field in ROM header is set to zero |

## Mappers

The term "mapper" generally refers to the hardware that's present on the cartridge itself, as it was common with older game consoles to add additional functionality (such as extended RAM, memory for savegames, etc.) to the cartridge instead of the console itself. The mapper number, which can be extracted from the Flags 6 and Flags 7 header fields of the file determines which mapper the currently loaded game uses. The mapper controls, among other things, the behaviour of read/write operations from/to the Cartridge Space address range (`$4020-$FFFF`). The first mapper that should be implemented by young emulators is [Mapper 0 / NROM](./mappers/00).