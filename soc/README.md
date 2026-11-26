# The Ascon core in the RV32IM SoC

Adds the core to [riscv-rv32im-soc](https://github.com/higgboson/riscv-rv32im-soc) as a
memory-mapped peripheral at `0x3000_0000`, with a C driver and a program that compares it
with Ascon in software on the same CPU.

## Result (RTL simulation, 1024-byte message, `RPC = 1`)

| `sw/programs/ascon` | Software | Hardware | Speedup |
|---|---|---|---|
| Ascon-AEAD128 encrypt (32 bytes of associated data) | 91,257 cycles | 2,947 cycles | **31.0x** |
| Ascon-Hash256 | 217,021 cycles | 2,064 cycles | **105.1x** |

Ciphertext, tag and hash are identical in both, the hardware decrypts its own output, and
a ciphertext with one flipped bit is refused. The program also runs one official NIST test
vector for the cipher and one for the hash.

How to read these numbers:

- "Software" is the Ascon team's 32-bit optimised C (`opt32`), with the permutation as a
  loop. The unrolled build is faster but needs about 51 KB of code; the SoC has 16 KB.
- Encryption is limited by the bus, not by the core. A 16-byte block costs 46 cycles, of
  which the core uses 9. The rest is the CPU moving 4 words in and 4 words out and polling
  a status bit. With `RPC = 2` or `4` the encryption time does not change at all (still
  2,947 cycles); hashing improves to 1,672 cycles (130x). So `RPC = 1` is the right
  choice for this SoC, and a DMA engine is the next step, the same conclusion as for the
  matrix accelerator.

## Add it to the SoC

From a clean checkout of riscv-rv32im-soc (the patch was made against commit `f080adb`):

```sh
SOC=/path/to/riscv-rv32im-soc
mkdir -p $SOC/code/ascon
cp ../rtl/*.v           $SOC/code/ascon/
cp -r sw/*              $SOC/sw/
git -C $SOC apply $PWD/soc.patch

cd $SOC/sw
make PROG=ascon sim                      # add CROSS=riscv-none-elf- for the xPack toolchain
make PROG=ascon sim VFLAGS=-DASCON_RPC=4 # 4 rounds per cycle
```

If the patch does not apply because `soc.v` has changed, make the same edits by hand:

1. `code/soc.v`: decode `DataAdr[31:28] == 4'h3` as `is_ascon`, instantiate `ascon_mmio`
   like `matmul_accel` (same five ports), keep data memory from being written when
   `is_ascon`, and add `ascon_rdata` to the `ReadData` mux.
2. Every script that lists the RTL (`sw/Makefile`, `verif/run_*.sh`,
   `fpga/basys3/build.tcl`, `pl_riscv_cpu.qsf`): add `code/ascon/*.v`.

On the Basys 3, `ascon_mmio` with `RPC = 1` is about 3,800 LUTs by Yosys' estimate (the
board's FPGA has 20,800). Timing has not been checked in Vivado yet.

## Registers (`rtl/ascon_mmio.v`)

| Offset | Name | Access | Meaning |
|---|---|---|---|
| `0x00` | CTRL | write | bits 2:0 mode (0 encrypt, 1 decrypt, 2 Hash256, 3 XOF128, 4 CXOF128), bit 8 start, bit 31 reset the core |
| `0x00` | STATUS | read | bit 0 busy, bit 1 input beat still waiting, bit 2 output beat waiting, bit 3 done, bit 4 tag ok, bits 12:8 bytes in the output beat, bit 13 last output beat |
| `0x04` | OUT_LEN | write | XOF / CXOF output bytes |
| `0x10` | KEY[0..3] | write only | |
| `0x20` | NONCE[0..3] | write only | |
| `0x30` | DIN[0..3] | write only | data of the next input beat, first byte in the low byte of DIN[0] |
| `0x40` | DIN_CTRL | write | bits 4:0 bytes in the beat, bit 8 last beat of the stream. Sends the beat |
| `0x50` | DOUT[0..3] | read | the waiting output beat |
| `0x60` | DOUT_ACK | write | output beat read, give the next one |
| `0x70` | TAG[0..3] | read / write | read: the tag after an encrypt (zero after a decrypt). write: the tag a decrypt must check |

Reads have no side effects. The key cannot be read back, and a decrypt reports only
`tag ok`, never the tag it computed.

## Driver (`sw/common/ascon.h`)

```c
void ascon_hw_encrypt(uint8_t *c, uint8_t *tag, const uint8_t *m, uint32_t mlen,
                      const uint8_t *ad, uint32_t adlen,
                      const uint8_t *nonce, const uint8_t *key);
int  ascon_hw_decrypt(uint8_t *m, const uint8_t *c, uint32_t clen, const uint8_t *tag,
                      const uint8_t *ad, uint32_t adlen,
                      const uint8_t *nonce, const uint8_t *key);   // 0 = ok, -1 = tampered
void ascon_hw_hash256(uint8_t *out, const uint8_t *m, uint32_t mlen);
void ascon_hw_xof128(uint8_t *out, uint32_t outlen, const uint8_t *m, uint32_t mlen);
```

Buffers can have any address; word-aligned buffers take a faster path (whole words instead
of bytes). `ascon_hw_decrypt` wipes the plaintext buffer when the tag is wrong. The caller
must never reuse a nonce with the same key.

## Files

```
soc.patch                     changes to riscv-rv32im-soc (soc.v and the RTL file lists)
sw/common/ascon.h             the driver
sw/programs/ascon/main.c      known answers, software vs hardware, tamper test
sw/programs/ascon/sw_*.c      Ascon in software: wrappers around the official C code
sw/programs/ascon/ascon-c/    official Ascon C code (opt32), unmodified, CC0
```
