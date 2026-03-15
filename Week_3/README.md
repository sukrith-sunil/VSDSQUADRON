# Caravel Integrated Test Results — VSDSquadron SoC Block-Level Verification

> **Week:** 3 — Block-Level Verification of VSDSquadron SoC  
> **Environment:** Caravel Harness + SKY130 Open-Source PDK  
> **Simulator:** Icarus Verilog (iverilog) + vvp runtime  
> **Firmware Compiler:** riscv64-unknown-elf-gcc (RV32I, bare-metal)  
> **Test Source:** `caravel_mgmt_soc_litex/verilog/dv/tests-caravel/`

---

## Table of Contents

1. [Caravel Integration Context](#1-caravel-integration-context)
2. [Test Environment Summary](#2-test-environment-summary)
3. [Caravel Integrated Test Results Table](#3-caravel-integrated-test-results-table)
4. [Per-Test Notes](#4-per-test-notes)
   - [user_pass_thru](#41-user_pass_thru)
   - [uart](#42-uart)
   - [sysctrl](#43-sysctrl)
   - [sram_exec](#44-sram_exec)
   - [spi_master](#45-spi_master)
   - [pullupdown](#46-pullupdown)
   - [pll](#47-pll)
   - [pass_thru_fix](#48-pass_thru_fix)
   - [mem](#49-mem)
   - [hkspi_power](#410-hkspi_power)
   - [gpio_mgmt](#411-gpio_mgmt)
   - [hkspi](#412-hkspi)
5. [Standalone vs. Caravel Comparison](#5-standalone-vs-caravel-comparison)
6. [Common Failure Modes (Caravel-Specific)](#6-common-failure-modes-caravel-specific)
7. [Execution Log](#7-execution-log)

---

## 1. Caravel Integration Context

### What is Caravel?

**Caravel** is an open-source SoC harness developed by Efabless that wraps the VSDSquadron user project area with:

- A **Management SoC** (PicoRV32-based) that initializes, configures, and monitors the user project
- A **Housekeeping SPI (HKSPI)** interface for external chip configuration
- **GPIO pad ring** — 38 configurable I/O pads that connect the chip's digital core to package pins
- **PLL** for on-chip clock generation
- **Power management** infrastructure

In the Caravel test environment, the simulation exercises the **entire chip boundary**, not just an isolated peripheral. Firmware executes on both the management SoC and the user project CPU (VexRiscv), and communication between them occurs through Caravel's internal bus architecture.

### Caravel Simulation Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      CARAVEL CHIP BOUNDARY                       │
│                                                                  │
│  ┌─────────────────┐         ┌──────────────────────────────┐   │
│  │  Management SoC │◄───────►│    User Project Area         │   │
│  │  (PicoRV32)     │  Logic  │    (VSDSquadron SoC)         │   │
│  │                 │  Analyzer│    VexRiscv + Peripherals    │   │
│  └────────┬────────┘  /Wishbo│                              │   │
│           │           ne     └──────────────┬───────────────┘   │
│           │                                 │                   │
│  ┌────────▼────────┐               ┌────────▼────────┐         │
│  │  Housekeeping   │               │   GPIO Pads      │         │
│  │  SPI (HKSPI)    │               │   (mprj_io[37:0])│         │
│  └────────┬────────┘               └────────┬────────┘         │
│           │                                 │                   │
└───────────┼─────────────────────────────────┼───────────────────┘
            │                                 │
     ┌──────▼──────────────────────────────────▼──────┐
     │              Verilog Testbench                  │
     │  - Drives SPI protocol to HKSPI                │
     │  - Monitors mprj_io signals                    │
     │  - Checks expected output patterns             │
     │  - Issues PASS/FAIL verdict                    │
     └──────────────────────────────────────────────────┘
```

### Key Difference from Standalone Tests

In Caravel tests, **two firmware programs** may be involved:
1. **Management SoC firmware** — configures GPIO pads, enables the user project, and communicates via the logic analyzer interface
2. **User project firmware** (VexRiscv) — executes the peripheral-specific test sequence

Both are compiled, loaded, and executed during the same simulation run.

---

## 2. Test Environment Summary

| Parameter | Value |
|---|---|
| **Operating System** | Ubuntu 22.04 LTS |
| **PDK** | SKY130 (sky130A) |
| **Caravel Root** | `$CARAVEL_ROOT` (must be set in environment) |
| **Simulator** | Icarus Verilog v11 + vvp |
| **RISC-V Compiler** | riscv64-unknown-elf-gcc, `-march=rv32i -mabi=ilp32` |
| **Test Directory** | `tests-caravel/` |
| **Simulation Includes** | Caravel harness RTL + PDK models + user project RTL |
| **Simulation Duration** | Typically longer than standalone (~10,000–100,000 ns) |

Each test was executed using:

```bash
make clean
make
```

---

## 3. Caravel Integrated Test Results Table

> **Note:** Fill in the **Status** column after executing each test.  
> Use `✅ PASS` or `❌ FAIL`. Add brief observations in the last column.

| # | Test Name | Block Under Test | Description | Simulation Environment | Status | Observations |
|---|---|---|---|---|---|---|
| 1 | `user_pass_thru` | User Project Passthrough | Verifies basic signal passthrough from management SoC GPIO to user project I/O pads | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., mprj_io signals mirror expected mgmt GPIO pattern* |
| 2 | `uart` | UART (Caravel-integrated) | Validates UART TX/RX within the Caravel environment — data driven via user project VexRiscv, observed on mprj_io pads | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., Serial stream on mprj_io[6] matches 0x55 pattern* |
| 3 | `sysctrl` | System Controller | Tests system-level control registers — clock gating, reset control, and power domain enable accessed via management SoC | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., System control register write/readback consistent* |
| 4 | `sram_exec` | SRAM Code Execution | Verifies that firmware can be copied from flash into SRAM and executed in-place by the VexRiscv CPU | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., Code relocated to SRAM, execution resumes correctly* |
| 5 | `spi_master` | SPI Master (Caravel-integrated) | Tests SPI master peripheral operation with user project VexRiscv inside the Caravel harness; signals visible on mprj_io pads | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., SPI SCK/MOSI visible on mprj_io; byte transfer OK* |
| 6 | `pullupdown` | GPIO Pull-up/Pull-down | Validates GPIO pad pull-up and pull-down termination configuration via housekeeping register writes | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., Pad voltage consistent with pull-up/pull-down config* |
| 7 | `pll` | On-Chip PLL | Verifies PLL lock sequence, output clock frequency, and clock selection mux control | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., PLL lock flag asserted; output clock period correct* |
| 8 | `pass_thru_fix` | Passthrough Fix Validation | Regression test confirming a known passthrough connectivity bug has been corrected | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., Previously failing passthrough path now PASS* |
| 9 | `mem` | Memory (Caravel-integrated) | Validates memory read/write operations through the full Caravel address decode path | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., Memory contents intact through power-up and access* |
| 10 | `hkspi_power` | Housekeeping SPI Power Sequencing | Tests power management register access via HKSPI — verifies correct power-up/power-down sequencing of domains | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., Power domain flags set correctly; no glitch on rails* |
| 11 | `gpio_mgmt` | GPIO Management (Caravel-integrated) | Validates GPIO management pad configuration and drive in the full Caravel environment — both management and user project GPIO interactions tested | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., GPIO direction and drive verified via testbench monitors* |
| 12 | `hkspi` | Housekeeping SPI Core | Verifies the full HKSPI protocol — register reads/writes via SPI frames to housekeeping registers | Caravel + iverilog + SKY130 | **PASS / FAIL** | *e.g., HKSPI read of chip ID returns expected value* |

---

## 4. Per-Test Notes

### 4.1 `user_pass_thru`

**Purpose:** Confirms that the fundamental connectivity between the management SoC and the user project I/O pads is intact. This is usually the first Caravel test to execute, as all other tests depend on this path being functional.

**Execution:**

```bash
cd user_pass_thru
make clean && make
```

**Expected output:**

```
User passthrough signal detected on mprj_io: OK
>>> user_pass_thru Test PASSED <<<
```

![Screenshot: user_pass_thru Caravel test terminal output](../screenshots/caravel_user_pass_thru.png)

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.2 `uart`

**Purpose:** Validates UART serial communication in the full Caravel chip context. The VexRiscv CPU sends a test byte sequence through the UART peripheral; the testbench observes the bit stream on the `mprj_io` GPIO pad and decodes it.

**Why it's harder than standalone:** The pad configuration must first be set up by the management SoC firmware before the user project can drive signals to the outside world.

**Execution:**

```bash
cd uart
make clean && make
```

**Expected output:**

```
Management SoC: GPIO pads configured for UART
VexRiscv UART TX: 0x41 ('A') - OK
Testbench: mprj_io[6] serial stream decoded: 0x41 - MATCH
>>> UART Caravel Test PASSED <<<
```

![Screenshot: Caravel UART test terminal output](../screenshots/caravel_uart_output.png)

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.3 `sysctrl`

**Purpose:** Tests the system controller — a set of Caravel housekeeping registers that govern clock enables, reset sequencing, and power domain switching. Critical for ensuring the chip powers up and initializes correctly.

**Execution:**

```bash
cd sysctrl
make clean && make
```

**Expected output:**

```
Sysctrl register write: OK
Clock gating enable: OK
Reset release sequence: OK
>>> System Control Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.4 `sram_exec`

**Purpose:** Verifies execute-in-place from SRAM. Firmware is first fetched from the simulated flash, a payload routine is copied into SRAM, and the VexRiscv CPU jumps to the SRAM address and executes it. This validates both the memory subsystem and the CPU's ability to branch to arbitrary addresses.

**Execution:**

```bash
cd sram_exec
make clean && make
```

**Expected output:**

```
SRAM copy: 512 bytes relocated - OK
Jump to SRAM execution: OK
SRAM payload executed: PASS flag set - OK
>>> SRAM Execution Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.5 `spi_master`

**Purpose:** Verifies the SPI master peripheral within the Caravel chip environment. GPIO pads are configured to expose SCK, MOSI, MISO, and CS_N externally, and the testbench acts as a SPI slave to verify the transaction.

**Execution:**

```bash
cd spi_master
make clean && make
```

**Expected output:**

```
SPI pads configured via management SoC: OK
SPI transaction: TX 0xDE, RX 0xDE (loopback) - OK
>>> SPI Master Caravel Test PASSED <<<
```

![Screenshot: SPI Master Caravel test terminal output](../screenshots/caravel_spi_master_output.png)

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.6 `pullupdown`

**Purpose:** Validates the GPIO pad pull-up and pull-down resistor configuration. Each Caravel GPIO pad has configurable internal weak pull resistors controlled via housekeeping registers. This test writes resistor configuration bits and verifies the pad behavior when driven by a high-impedance source.

**Execution:**

```bash
cd pullupdown
make clean && make
```

**Expected output:**

```
Pull-up configuration: pad reads HIGH in Hi-Z - OK
Pull-down configuration: pad reads LOW in Hi-Z - OK
>>> Pull-up/Pull-down Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.7 `pll`

**Purpose:** Tests the on-chip PLL — Caravel includes a digital PLL for clock multiplication. This test verifies that the PLL locks on the reference clock and that the selected output clock has the correct period as measured by the testbench.

**Execution:**

```bash
cd pll
make clean && make
```

**Expected output:**

```
PLL reference clock: detected OK
PLL lock: asserted at t=2400ns
PLL output clock period: 10ns (100 MHz) - MATCH
>>> PLL Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.8 `pass_thru_fix`

**Purpose:** A regression test confirming that a specific connectivity bug in the passthrough path (identified in an earlier revision) has been corrected. Verifies the fixed behavior is stable.

**Execution:**

```bash
cd pass_thru_fix
make clean && make
```

**Expected output:**

```
Passthrough signal verified on corrected path: OK
>>> pass_thru_fix Regression Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.9 `mem`

**Purpose:** Validates the full memory access path within the Caravel environment, including the address decode logic, SRAM timing, and Wishbone handshake behavior as seen from the management SoC.

**Execution:**

```bash
cd mem
make clean && make
```

**Expected output:**

```
Word write/read: OK
Byte-granular access: OK
Memory boundary check: OK
>>> Memory Caravel Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.10 `hkspi_power`

**Purpose:** Tests power domain control registers accessible via the Housekeeping SPI interface. Verifies that writing to power management registers correctly gates or enables specific functional blocks, and that power state flags are readable.

**Execution:**

```bash
cd hkspi_power
make clean && make
```

**Expected output:**

```
HKSPI power register write: OK
User project power-down: OK
User project power-up: OK
Power flags consistent: OK
>>> HKSPI Power Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.11 `gpio_mgmt`

**Purpose:** A comprehensive GPIO management test executed in the full Caravel context. Tests both the management GPIO (controlled by PicoRV32 firmware) and user project GPIO (controlled by VexRiscv), verifying that pad sharing and signal routing logic works correctly.

**Execution:**

```bash
cd gpio_mgmt
make clean && make
```

**Expected output:**

```
Management GPIO drive: OK
User project GPIO override: OK
GPIO direction switching: OK
>>> GPIO Management Caravel Test PASSED <<<
```

![Screenshot: GPIO Management Caravel test terminal output](../screenshots/caravel_gpio_mgmt_output.png)

**PASS/FAIL recorded:** `PASS / FAIL`

---

### 4.12 `hkspi`

**Purpose:** Verifies the full Housekeeping SPI protocol. The testbench drives a correctly framed SPI master transaction to the HKSPI peripheral. The test reads the Caravel chip ID register (a fixed-value register) and verifies the response matches the expected value.

**Execution:**

```bash
cd hkspi
make clean && make
```

**Expected output:**

```
HKSPI chip ID read: 0x10184 - MATCH (expected 0x10184)
HKSPI custom register write/read: OK
>>> Housekeeping SPI Test PASSED <<<
```

**PASS/FAIL recorded:** `PASS / FAIL`

---

## 5. Standalone vs. Caravel Comparison

| Aspect | Standalone Test | Caravel Integrated Test |
|---|---|---|
| **Test scope** | Single peripheral DUT | Full SoC inside Caravel harness |
| **RTL included** | DUT + testbench + VexRiscv | Caravel harness + management SoC + user project |
| **Firmware** | User project firmware only | Management SoC firmware + user project firmware |
| **Bus interactions** | Direct VexRiscv → Wishbone → DUT | Caravel management bus + user Wishbone + HKSPI |
| **GPIO** | Not exercised | Full pad ring configuration required |
| **Simulation time** | Fast (hundreds of ns) | Slow (thousands to hundreds of thousands of ns) |
| **Debug complexity** | Low — few signals to trace | High — hierarchically deep signal paths |
| **Failure isolation** | Easy — peripheral-specific | Hard — requires identifying whether fault is in harness, management SoC, or user peripheral |

---

## 6. Common Failure Modes (Caravel-Specific)

| Failure Mode | Likely Cause | Debugging Approach |
|---|---|---|
| `$CARAVEL_ROOT not set` | Environment variable not exported | `export CARAVEL_ROOT=/path/to/caravel` |
| `PDK_ROOT not set` | SKY130 PDK path missing | `export PDK_ROOT=/path/to/open_pdk` |
| Module not found — Caravel RTL | Caravel repo not cloned or path wrong | Verify `$CARAVEL_ROOT/rtl/*.v` files exist |
| Simulation stalls at PLL lock | PLL behavioral model not found / incorrect params | Check PDK model includes; verify PLL parameters |
| GPIO pads not driving expected values | Management SoC firmware didn't configure pads | Check management firmware GPIO config sequence |
| HKSPI test hangs | SPI CS_N or SCK not toggling — testbench timing issue | Inspect HKSPI clock and CS_N in GTKWave |
| Test FAIL with correct standalone result | Integration bus conflict or address decode error | Compare Wishbone addresses; check Caravel memory map |

---

## 7. Execution Log

> Replace entries below with actual terminal output from each test.

```
[user_pass_thru] make clean && make
>>> user_pass_thru Test PASSED <<<

[uart] make clean && make
>>> UART Caravel Test PASSED <<<

[sysctrl] make clean && make
>>> System Control Test PASSED <<<

[sram_exec] make clean && make
>>> SRAM Execution Test PASSED <<<

[spi_master] make clean && make
>>> SPI Master Caravel Test PASSED <<<

[pullupdown] make clean && make
>>> Pull-up/Pull-down Test PASSED <<<

[pll] make clean && make
>>> PLL Test PASSED <<<

[pass_thru_fix] make clean && make
>>> pass_thru_fix Regression Test PASSED <<<

[mem] make clean && make
>>> Memory Caravel Test PASSED <<<

[hkspi_power] make clean && make
>>> HKSPI Power Test PASSED <<<

[gpio_mgmt] make clean && make
>>> GPIO Management Caravel Test PASSED <<<

[hkspi] make clean && make
>>> Housekeeping SPI Test PASSED <<<
```

![Screenshot: Full Caravel test suite execution log — all 12 tests](../screenshots/caravel_all_results.png)

---

*Week 3 Caravel Results — VSD RTL-to-GDS SoC Implementation Program*  
*Author: Sukrith Sunil | M.Tech VLSI Design, Amrita Vishwa Vidyapeetham*
