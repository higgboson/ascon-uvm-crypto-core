# Ascon hardware core (NIST SP 800-232), verified with UVM, as a RISC-V SoC peripheral

A synthesizable Verilog implementation of the whole Ascon family that NIST standardised for
small devices in August 2025: authenticated encryption (**Ascon-AEAD128**, encrypt and
decrypt), the hash **Ascon-Hash256**, and the extendable-output functions **Ascon-XOF128**
and **Ascon-CXOF128**, in one core with one datapath.

It is checked three ways: against every official test vector, by a UVM environment that
compares it with the Ascon team's own C code through DPI-C, and by planting bugs to prove
the testbench catches them. It also runs as a memory-mapped peripheral of the
[RV32IM SoC](https://github.com/higgboson/riscv-rv32im-soc), driven from C.

New to Ascon or to this code? Start with [`docs/learning_guide.md`](docs/learning_guide.md).

---

# ✅ VERIFICATION SIGN-OFF

## 🎯 100% COVERAGE: EVERY METRIC CLOSED IN CADENCE IMC

<p align="center"><img src="docs/img/coverage_before_after.png" width="100%" alt="Coverage closure in Cadence IMC: 96.51% before, 100% after"></p>

| Metric | Before exclusions | **After exclusions** |
|---|---|---|
| **Overall (items covered)** | 10,792 / 11,182 (96.51%) | **10,792 / 10,792 (100%)** |
| Block | 99.11% | **100%** |
| Toggle | 97.74% | **100%** |
| Expression · FSM | 100% | **100%** |
| Assertions (16 SVA + 9 cover properties) | 100% | **100%** |
| Covergroups (6 groups, 347 bins) | 100% | **100%** |

### 🔍 HOW IT REACHED 100%: NOTHING WAS HIDDEN

Every one of the 390 excluded items was checked by hand in the source view and is either
a bit that is constant by design or code that cannot run in this build. The exclusions are
saved in a refinement file (`verif/uvm/sim/ascon_exclusions.vRefine`), so anyone can load
them and see exactly what was excluded. **Full step-by-step walkthrough with 7 annotated
screenshots: [docs/coverage/README.md](docs/coverage/README.md).**

**1. Toggle holes: padding bits that are constant by design**

<p align="center"><img src="docs/img/toggle_padv_big.png" width="100%" alt="padv toggle bits before and after exclusion"></p>

**2. Block holes: branches unreachable for the RPC = 1 build**

<p align="center"><img src="docs/img/block_holes_big.png" width="100%" alt="block holes explained"></p>

## 🟢 UVM REGRESSION: 9 / 9 TESTS PASS ON CADENCE XCELIUM

<p align="center"><img src="docs/img/regression_9_of_9.png" width="100%" alt="Xcelium regression: 9 of 9 tests passed, 0 failed"></p>

## 🐞 THE TESTBENCH CATCHES REAL BUGS: 6 / 6 PLANTED BUGS CAUGHT

The bug shown below leaves every output correct, so only an assertion can catch it:

<p align="center"><img src="docs/img/bug_caught_by_sva.png" width="100%" alt="Planted bug no_wipe caught by SVA a_wiped"></p>

---

## Results

| | |
|---|---|
| Official NIST test vectors | **4,228 of 4,228 pass** (1,089 AEAD, each run as encrypt, decrypt and tampered decrypt; 3,139 hash / XOF / CXOF), for 1, 2 and 4 rounds per cycle, on Verilator and Icarus Verilog |
| UVM regression (Xcelium and Verilator) | **9 of 9 tests pass** for each of the three core variants; 4,228 random and corner operations per regression, each checked against the official C code: 0 wrong results, 0 wrong cycle counts, 0 assertion failures |
| Coverage (Xcelium 24.09 + IMC, 9-test regression) | **100%** on every metric: block, expression, toggle, FSM, assertion (25/25), covergroup (347/347); overall 10792 / 10792 after documented exclusions of unreachable code and constant bits ([walkthrough](docs/coverage/README.md)) |
| Planted bugs | **6 of 6 caught**, one of them only by an assertion (the results stay correct) |
| On the RISC-V SoC | encryption **31x**, hashing **105x** faster than the official optimised C on the same CPU |
| Size (Yosys, Artix-7) | core about 2,300 LUTs and 760 flip-flops at 1 round per cycle |

What has not been done yet is listed under [Status](#status).

## The core (`rtl/ascon_core.v`)

```
                key, nonce, mode, start
                         |
   din (valid/ready) --> XOR --> [ 320-bit state: x0 x1 x2 x3 x4 ] --> dout (valid/ready)
     up to 16 bytes       ^                     |                        ciphertext /
     + bytes + last       |      RPC x ascon_round (1, 2 or 4)           plaintext / hash
                          +---------------------+
                                                |
                                           tag, tag_ok, done
```

- **Streaming**: data goes in and out as beats of up to 16 bytes with valid / ready and a
  `last` flag, so the core never needs the total length and can sit behind a bus, a FIFO or
  a DMA engine.
- **One knob, `RPC`** (rounds per clock cycle): 1, 2 or 4 copies of the round between the
  state registers.

| RPC | Cycles per 16-byte AEAD block | 1024-byte encryption | LUTs (core) | LUTs (with register wrapper) |
|---|---|---|---|---|
| 1 | 9 | 603 cycles | 2,338 | 3,767 |
| 2 | 5 | 335 cycles | 3,790 | 5,473 |
| 4 | 3 | 201 cycles | 5,806 | 7,105 |

LUT counts are Yosys estimates for a Xilinx 7-series FPGA (`syn/synth_fpga.sh`); flip-flops
are about 764 for the core and 1,316 with the wrapper for every RPC.

- **Security choices**
  - Constant time: the cycle count depends only on the lengths, never on key or data. The
    scoreboard checks it for every operation.
  - Decrypt gives a yes / no answer only. The tag it computed never leaves the core, so
    it cannot be used to forge a message.
  - State and key copy are cleared when an operation ends.
  - No lookup tables: the S-box is plain AND / XOR logic.

  Not covered: power or electromagnetic side channels and fault injection (no masking, no
  redundancy).

## Verification

| Layer | Where | What it does |
|---|---|---|
| Known answers | `verif/tb_kat.sv` | every vector of the four official files, with the beats cut differently and both handshakes stalled at random from test to test |
| Registers | `verif/tb_mmio.sv` | the same vectors through `ascon_mmio`, driven like the C driver, plus: the key cannot be read back, a decrypt's tag reads zero, soft reset really stops an operation |
| UVM | `verif/uvm/` | constrained-random operations; scoreboard through DPI-C to the official C code; cycle-exact timing check; 16 assertions; functional coverage of operation shapes; 8 tests |
| Planted bugs | `verif/inject_bug.py` | 6 realistic bugs, each must make the regression fail |

Planted bugs, `ascon_full_test`, 1,115 operations checked:

| Bug | Caught by |
|---|---|
| padding byte `0x80` instead of `0x01` (the older Ascon v1.2 rule) | scoreboard, 972 wrong results |
| 6 rounds between data blocks instead of 8 | scoreboard, 686 wrong results and 568 wrong cycle counts |
| decrypt absorbs plaintext instead of ciphertext | scoreboard, 329 wrong results |
| domain separation forgotten when there is no associated data | scoreboard, 62 wrong results (only the empty-AD operations) |
| decrypt shows the tag it computed | assertion (413) and scoreboard (413) |
| state and key not wiped after an operation | assertion only (2,412); every result is still correct |

Details, the coverage model and how to run on Xcelium or Verilator:
[`verif/uvm/README.md`](verif/uvm/README.md).

## On the RV32IM SoC

`rtl/ascon_mmio.v` puts the core at `0x3000_0000` on the SoC's bus; `soc/sw/common/ascon.h`
is the C driver. `soc/sw/programs/ascon` encrypts and hashes 1024 bytes in software (the
Ascon team's 32-bit optimised C) and with the core, and compares the results.

| 1024 bytes, RTL simulation, RPC = 1 | Software | Hardware | Speedup |
|---|---|---|---|
| Ascon-AEAD128 encrypt | 91,257 cycles | 2,947 cycles | **31.0x** |
| Ascon-Hash256 | 217,021 cycles | 2,064 cycles | **105.1x** |

Encryption is limited by the bus: a block costs 46 cycles and the core needs 9 of them, the
rest is the CPU moving words. Going to 2 or 4 rounds per cycle does not change the
encryption time at all. A DMA engine is the next step. How to add it to the SoC, the
register map and the driver: [`soc/README.md`](soc/README.md).

## Quick start

Free tools only (Verilator 5 or Icarus Verilog, gcc, Python 3):

```sh
cd verif
make all                 # all test vectors for RPC = 1, 2, 4; register wrapper; C reference
make kat RPC=4           # one of them
make kat SIM=iverilog    # with Icarus Verilog
make lint
```

UVM on Xcelium:

```sh
cd verif/uvm/sim
make regress             # then: make cov_merge cov_report
```

UVM on Verilator: see [`verif/uvm/README.md`](verif/uvm/README.md).

Prove the testbench works:

```sh
cd verif
python3 inject_bug.py list
python3 inject_bug.py no_wipe      # plant a bug, run any UVM test: it must fail
python3 inject_bug.py restore
```

## Repository layout

```
rtl/ascon_round.v     one round of the permutation (pure logic)
rtl/ascon_core.v      the core: all five modes, streaming interface, RPC parameter
rtl/ascon_mmio.v      register wrapper for the RV32IM SoC bus
kat/                  official known-answer test files (SP 800-232)
ref/                  reference model: official C code (ref/ascon-c, unmodified) + DPI wrapper
verif/                Makefile, known-answer testbench, register testbench, inject_bug.py
verif/uvm/            UVM environment (env_lib/, env/, test/, sim/) for Xcelium and Verilator
soc/                  SoC patch, C driver, software-vs-hardware program
syn/                  Yosys resource estimate for Xilinx 7-series
asic/ascon_mmio/      LibreLane (SkyWater 130 nm) configuration
docs/                 learning guide
```

## Status

Done and measured: everything in the Results table.

Not done yet:

- [x] UVM regression on Xcelium 24.09: 9 of 9 pass; IMC coverage collected
- [x] Coverage closure: every hole explained, unreachable ones excluded with reasons ([docs/coverage](docs/coverage/README.md))
- [ ] Vivado timing for RPC = 1, 2, 4 and a run on the Basys 3 board
- [ ] RTL to GDSII with LibreLane (`asic/ascon_mmio`, configuration only so far)
- [ ] CXOF128 in the C driver (the core and the register wrapper support it)

Ideas after that: a DMA engine, formal proofs of the security assertions, secure boot of
the SoC with Ascon-Hash256, a masked S-box against power analysis.

## Third-party code and references

- `ref/ascon-c/`, `soc/sw/programs/ascon/ascon-c/` and `kat/` come from the Ascon team's
  reference package, <https://github.com/ascon/ascon-c> (CC0), commit `446347f`, unmodified.
- NIST SP 800-232, *Ascon-Based Lightweight Cryptography Standards for Constrained Devices*
  (August 2025): <https://csrc.nist.gov/pubs/sp/800/232/final>
- Ascon: <https://ascon.isec.tugraz.at>
