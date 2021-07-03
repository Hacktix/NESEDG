# Mapper 0 (NROM)

The mapper with the mapper number `$00` is often referred to as NROM and is the most basic of the available mappers.

## Read Operations

**TODO:** PRG-RAM @ $6000-$7FFF

### $8000-$FFFF - ROM

Either a 16KB or 32KB ROM can be directly mapped to the memory section $8000-$FFFF. In the case of a 16KB ROM, the range $C000-$FFFF acts as a memory mirror to $8000-$BFFF. Assuming PRG-ROM is kept in a separate array, the index of the read byte can be calculated using the following formula: `ADDR & (ROMSIZE - 1)`, where `ADDR` is the memory address being accessed and `ROMSIZE` is the size of PRG-ROM in bytes.

## Write Operations

**TODO:** PRG-RAM @ $6000-$7FFF

Aside from PRG-RAM, Mapper 0 supports no write operations. All writes should simply be ignored.