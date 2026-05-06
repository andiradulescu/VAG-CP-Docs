# Lear D4 Gateway Firmware Analysis — CP Attack Surface Map

**Status**: Active research — firmware extracted and partially analyzed  
**Date**: 2026-05-06  
**Targets**: C7 A6 (4G8907468, rev 0113), D4 A8 (4H0907468F, rev 0213), B8/B9 A4/A5 (8W7907468B, rev 0086)

## Executive Summary

Three generations of Lear gateway firmware were extracted from ODIS FlashDaten FRF containers and analyzed for Component Protection (CP) implementation details. The C7 and D4 gateways share the same "LEAR D4 Gateway" codebase, identical AES-128 crypto tables, and a common CP architecture — the only named CP handler is `EV_GatewPkoUDS`, found in the C7 firmware. This document maps the attack surface for offline CP resolution.

## FRF Decryption

FRF files from ODIS FlashDaten use a rolling XOR cipher (not AES) with a 4095-byte key. Decrypted FRFs are standard ZIP archives containing ODX (XML) flash containers. The ODX files contain hex-encoded binary flash data with block address metadata.

**Decryption tool**: `bri3d/VW_Flash` — `frf/decryptfrf.py`

| File | Encrypted Size | ODX Size | Flash Blocks |
|------|---------------|----------|--------------|
| FL_4G8907468___0113_S.frf (C7) | 521 KB | 2.1 MB | 2 (64K + 960K) |
| FL_4H0907468F__0213_S.frf (D4) | 308 KB | 984 KB | 2 (2K + 472K) |
| FL_8W7907468B_0086___S.frf (B8) | 31.7 MB | 56.2 MB | 6 blocks |

## Firmware Architecture

### Platform Identification

Both C7 and D4 firmware contain the identity string **"LEAR D4 Gateway."** — confirming they share the same Lear codebase. The MCU is a Renesas/NEC V850ES (D70F3433) as confirmed by the interrupt vector table structure at offset 0x000000 (V850 `jr` instruction encoding).

### Flash Layout

**C7 (4G8907468, rev 0113)**:
- Block 0 (64 KB, source-address 0x03): Bootloader/vector region — starts with V850 vector table, `DEFE` padding on unused vectors
- Block 1 (983,040 bytes / 960 KB, source-address 0x01): Main application firmware

**D4 (4H0907468F, rev 0213)**:
- Block 0 (2,048 bytes, source-address 0x03): Small data/parameter block
- Block 1 (483,328 bytes / 472 KB, source-address 0x01): Main application firmware

### Compatible Part Numbers (from ODX EXPECTED-IDENT)

C7 firmware accepts: `4H0907468AA`, `4G0907468AA`, `4G1907468`  
D4 firmware accepts: `4H0907468C`, `4H0907468D`, `4H0907468E`

Part numbers embedded in C7 firmware data section: `4H0907468AK`, `4G5907468L`, `4G5907468M`, `4G1907468H`, `4G8907468L`, `4G8907468M`

The cross-platform part number overlap (`4H0907468` appearing in C7's accepted list) confirms these are the same gateway family with compatible firmware.

## AES-128 Implementation

### Table Locations

| Component | C7 Offset | D4 Offset | Size |
|-----------|-----------|-----------|------|
| Rcon table | 0x017FD8 | 0x009624 | 12 bytes |
| S-box | 0x017FE4 | 0x009630 | 256 bytes |
| Inverse S-box | 0x0180E4 | 0x009730 | 256 bytes |
| GF(2^8) xtime table | 0x0181E4 | 0x009830 | 256 bytes |

**The AES tables are byte-for-byte identical** between C7 and D4 (780/780 bytes match, 100%). This is the same AES library compiled into both firmwares.

### AES Table Structure

After the inverse S-box, both firmwares contain additional GF(2^8) lookup tables used for MixColumns optimization:
- Multiply-by-2 (xtime): `00 02 04 06 08 0a 0c 0e ...`
- Multiply-by-3 interleaved: `00 03 06 05 0c 0f 0a 09 ...` (visible as ASCII `" $ & ( * , . 0 2 ...` followed by `; 9 ? = 3 1 7 5 ...`)

This is a standard textbook AES-128 implementation using four 256-byte precomputed tables.

## CP Handler: `EV_GatewPkoUDS`

### Discovery

The string `EV_GatewPkoUDS` was found in the C7 firmware at offset **0x00F454**. This is the event handler for Component Protection ("Pko" = Protektion Komponente) over UDS protocol. This is the function name the gateway uses internally for CP operations.

### Surrounding Context (0x00F274–0x00F640)

| Offset | Content | Significance |
|--------|---------|-------------|
| 0x00F274 | `..\\sdg\\sdg_if.c` | SDG interface (Service Data Gateway?) |
| 0x00F2F0 | `..\\diag_adapter\\src\\diag_snapshot.c` | Diagnostic snapshot handler |
| 0x00F3BC | `002015` | Date stamp (2015) |
| 0x00F3F0 | `4H0907468AK` | D4 A8 gateway part number |
| 0x00F408–0x00F438 | `4G5907468L/M`, `4G1907468H`, `4G8907468L/M` | C7 A6/A7 part number variants |
| 0x00F444 | `J533--Gateway` | ECU address identifier |
| 0x00F454 | **`EV_GatewPkoUDS`** | **CP handler name** |
| 0x00F4C8+ | `..\\diag_adapter\\src\\diag_adapter.c` (×10) | UDS diagnostic adapter asserts |

The CP handler lives in the `diag_adapter` module alongside the main UDS service router. The repeated `diag_adapter.c` assert references suggest robust error handling in the UDS path.

### D4 Comparison

The D4 firmware has `EV_GatewUDS` (offset 0x003D78) — a generic UDS handler — but **no `Pko` suffix**. This could mean:
1. The D4 firmware version predates the CP handler being split out, or
2. CP is handled inline within the generic UDS handler, or
3. The D4 revision (0213) has a different code organization

Both firmwares share the AES implementation, so the CP crypto is definitely present in both.

## UDS Routine and DID References

### Routine ID 0x0226 (CP Routine)

Found in both firmwares in what appear to be UDS routine dispatch tables:
- **C7**: First occurrence at 0x001B8D in a structured table: `02 58 02 58 02 26 02 6e 01 17 01 f1 00 d8`
- **C7**: Repeated cluster at 0x002005–0x002013 — eight consecutive `02 26` entries (the CP routine registered 8 times, likely for different sub-functions)
- **D4**: At 0x00D107 and 0x00D28D in similar table structures

### DID 0x04A3 (Constellation Bitmap)

Found in C7 at multiple offsets (0x039989, 0x039FF7, 0x04C00F, 0x068491, 0x0684BB, 0x099AE1) — appears in code context (V850 instructions referencing this value), confirming the firmware reads/writes the constellation DID.

### DID 0x00BE (IKA Key Blob)

Extremely frequent in both firmwares — found at 100+ locations in C7, 70+ in D4. Many occurrences are in structured data tables (DID dispatch tables) and in code sections (V850 instructions operating on this DID). This confirms heavy usage of the IKA blob throughout the CP subsystem.

## B8/B9 Gateway (8W7907468B) — Cross-Reference

The B8 MQB gateway is a completely different architecture (much larger, 31MB, multi-core) but provides valuable CP string references:
- **"Transport Mode/Protection is enabled."** — confirms CP enforcement terminology
- **"No entry for key %s"** / **"Unknown key: %s"** — key management error messages
- EXI schemas referencing `authenticationMethods`, `fpinChallenge`, `factoryResetResponse` — SFD2/ZE-level crypto (not applicable to C7/D4 but shows architecture evolution)

## Source File Map

The C7 firmware contains 26 unique source file paths from compilation, providing a complete picture of the gateway software architecture:

- **UDS/Diagnostic**: `diag_adapter.c`, `diag_snapshot.c`, `sdg_if.c`
- **Bus Monitoring**: `busmon.c`, `flexrayBusMonitor.c`, `flexrayBusMonitorIdle.c`
- **LIN Bus**: `ZgwComLIN.cpp`, `LinSciH.cpp`, `LINMaster.cpp`, `LinBusId.cpp`, `LinSystem.cpp`, `LINScheduler.cpp`, `LinTpInterface.cpp`, `LINBitfieldFilter.cpp`, `FlashLinSlave.cpp`
- **FlexRay**: `flexray_lifecycle_adapter.c`, `fr_rx_queue.c`, `fr_state_machine.c`, `frReadFlexrayBranchData.c`, `isotp_fr.c`, `flexrayElmosTransceiver.c`
- **MOST**: `most_lear.c`
- **Infrastructure**: `NotifyMap.c`, `eci.c`, `history.c`, `assert_debug.c`

## Actionable Next Steps for Ghidra

### Loading the Binary

1. Create new Ghidra project, import `C7_4G8_FD_1DATA.bin`
2. Processor: **V850** (or V850ES if available)
3. Load address: **0x00000000** (standard internal flash base for D70F3433)
4. Mark block at 0x00000000 as V850 vector table (16-byte entries)
5. Disassemble from reset vector

### Priority Targets

1. **Find `EV_GatewPkoUDS` cross-references**: The string at 0xF454 will be referenced by initialization code that registers the CP handler. Follow that reference to the handler function pointer.

2. **AES caller identification**: Search for code that loads the address 0x017FE4 (S-box). In V850 this will be a `movhi`/`movea` pair loading `0x0001` / `0x7FE4`. The functions calling the AES routines are the CP challenge-response path.

3. **Routine 0x0226 dispatch**: The UDS routine table at ~0x001B8D maps routine IDs to function pointers. Decode the table format to find the handler address for 0x0226.

4. **DID 0x00BE handler**: Similarly, the DID dispatch table will map 0x00BE to a read/write handler function. This function manages the IKA blob storage.

5. **DID 0x04A3 handler**: The constellation bitmap read/write handler.

### What to Look For in the AES Caller

The CP challenge-response should follow this pattern:
1. Receive challenge from ODIS/tool (via UDS routine 0x0226)
2. Load IKA key material (from DID 0x00BE storage)
3. AES-128 encrypt or MAC the challenge using the IKA key
4. Compare result with expected response
5. Update constellation bitmap (DID 0x04A3) on success

The critical question: **Is the IKA key derived from a master key embedded in firmware, or stored per-module in NV memory?** The AES code callers will answer this.

## Key Insight: Shared Codebase

The 100% match on AES tables, the shared "LEAR D4 Gateway" identifier, and the cross-platform part number acceptance all confirm: **any CP bypass developed for one Lear D4 gateway variant should work across all C7 A6/A7, D4 A8, and related platforms.** The CP implementation is a shared library component, not per-platform custom code.

## Files

All extracted binaries are available for analysis:
- `C7_4G8_FD_0DATA.bin` — C7 bootloader/vector block (64 KB)
- `C7_4G8_FD_1DATA.bin` — C7 main firmware (960 KB) ← **primary analysis target**
- `D4_4H0_FD_0DATA.bin` — D4 small data block (2 KB)
- `D4_4H0_FD_1DATA.bin` — D4 main firmware (472 KB)
- `B8_8W7_FD_*.bin` — B8 MQB gateway blocks (cross-reference only)
