# Live Vehicle Test Results — 2026-05-06

**Vehicle**: 2013 Audi A6 C7 (4G0 chassis)  
**Gateway**: J533 — 4G0 907 468 AC, SW 0037, ASAM `EV_GatewPkoUDS` rev 001018  
**HVAC (installed)**: J255 basic — 4G0 820 043 AH, `EV_AirCondiBasisUDS` rev 002044  
**Seat Driver**: J136 — 4G8 959 760 A, KWP2000 sub-bus  
**Tools**: MongoosePro ISO2 (J2534), 32-bit Python ctypes direct DLL, VCDS 26.3.0  

## CP Status Confirmed

J255 reports **U1101 Component Protection: Active** (value 32 / 0x20).  
J533 constellation remains **CP-CLEARED** (`FDA1E80CFE62600D0000`) — unchanged after 3-4 months of basic J255 being installed, unchanged after ignition cycles.

**Key architectural insight**: CP enforcement is MODULE-SIDE, not gateway-side. J255 self-polices by checking its credentials against what J533 has enrolled. J533's constellation is a GEKO/ODIS bookkeeping record that doesn't update when an unenrolled module is present.

## J533 Full DID Dump

| DID | Name | Length | Value |
|-----|------|--------|-------|
| 0x04A3 | Constellation (coded) | 10B | `FD A1 E8 0C FE 62 60 0D 00 00` (CP-CLEARED) |
| 0x2A2A | Allocation table | 80B | `01 02 03 03 04 05 06 08 09 0E 10 11 13 8B 34 15 16 17 17 18 19 1B 1E 20 22 28 2E 30 36 3C...` |
| 0x2A26 | Present bitmap | 10B | `FD A1 E8 0C FE 62 60 0D 00 00` (matches constellation) |
| 0x2A27 | Sleep indication | 10B | `22 52 1A 73 41 3D FB F3 06 00` |
| 0x2A28 | DTC bitmap | 10B | `B1 A0 E8 08 D4 62 40 00 00 00` |
| 0x2A29 | DiagProt | 80B | `01 01 01 01 00 02 02 01 02 02 02 01 01 01 01 01 02 01 00 02 01 03 02 02 01 01 02...` |
| 0x2A2C | TP-Identifiers | 160B | `00 00 00 00 07 13 00 00 ... 07 57 07...` |
| 0x0438 | Theft keys stored | 10B | `05 81 48 08 22 40 00 0C 00 00` |
| 0x0439 | Auth currently incorrect | 10B | `01 00 00 00 00 00 00 00 00 00` |
| 0x043A | Auth formerly incorrect | 10B | `00 00 00 00 00 00 00 00 00 00` |
| 0x043C | Key corrections count | 1B | `00` |
| 0x043D | Key downloads count | 1B | `00` |
| 0x043E | Showroom mode | 1B | `FF` |
| 0xF187 | Part number | 11B | `4G0907468AC` |
| 0xF189 | SW version | 4B | `0037` |
| 0xF18C | Serial | 14B | `00000000837162` |
| 0xF19E | ASAM dataset | 15B | `EV_GatewPkoUDS\0` |
| 0xF1A2 | ASAM revision | 6B | `001018` |
| 0xF191 | HW number | 11B | `4G0907468AC` |
| 0x00BE | IKA Key | — | **NO RESPONSE** (J533 hangs — forwards to sub-bus) |
| 0x00BD | GKA Key | — | NRC 0x31 |
| 0xEA61-64 | CP activation | — | NRC 0x31 |

## J255 Test Results

| Test | Result |
|------|--------|
| Extended session (10 03) | ✅ Positive (50 03 00 32 01 F4) |
| Read DID 0xF18C (Serial) | ✅ `25021300111130` |
| Read DID 0xF187 (Part) | ✅ `4G0 820 043 AH` |
| Read DID 0x00BE (IKA) | ❌ NRC 0x31 (read not supported on basis variant) |
| **Write DID 0x00BE** | **⚠️ NRC 0x78 (pending) → NRC 0x31** (processed then rejected value) |
| Write zeros to 0x00BE | ❌ NRC 0x31 |
| Write 16-byte key only | ❌ NRC 0x31 (wrong length) |
| Programming session write | ❌ NRC 0x13 (different format expected) |
| **Security Access 0x03** | **✅ SEED = `BA 2F 9A 47` (4 bytes)** |

## CP Routine 0x0226 on J533

| Payload | Response |
|---------|----------|
| `31 01 02 26` (bare) | NRC 0x13 (wrongLength — needs 3 data bytes) |
| `31 03 02 26` (request results) | `71 03 02 26` (positive — routine was previously run) |
| `31 01 02 26 00 07 46` | NRC 0x31 (3-byte format correct, value out of range) |
| `31 01 02 26 00 08 00` | NRC 0x31 (same) |
| `31 01 02 26 00 00 08` | NO RESPONSE (J533 started routine, hung on sub-bus) |
| All other valid-length payloads | NO RESPONSE (J533 hangs trying to communicate with module) |

## Critical Discoveries

### 1. J255 Basic Variant Does NOT Expose DID 0x00BE for READ
The `EV_AirCondiBasisUDS` ASAM dataset does not implement ReadDataByIdentifier for DID 0x00BE. Our earlier MWB confirmation was from `BV_AirCondiUDS` / `EV_AirCondiComfoUDS` (the comfort variant). The basic variant stores CP data internally but doesn't expose it via UDS read.

### 2. J255 DOES Support WriteDataByIdentifier for DID 0x00BE
The write was accepted (NRC 0x78 pending received), processed for ~1 second, then rejected with NRC 0x31. This means:
- The UDS write service IS supported
- DID 0x00BE IS recognized for writes
- The BLOB VALUE was rejected (not the DID or format)
- The J136 blob (from Feb 2024) is NOT valid for J255 → blobs are likely module-serial-specific

### 3. SA2 Security Access is Available on J255
Level 0x03 returned a 4-byte seed (`BA 2F 9A 47`). Completing the SA2 challenge-response may change the write acceptance criteria. The SA2 bytecode for `EV_AirCondiBasisUDS` is needed.

### 4. J533 Constellation Does NOT Update on Module Swap
After 3-4 months with the wrong J255 installed, and after ignition power cycles, J533's constellation remains CP-CLEARED. The constellation is a static GEKO/ODIS record, not a dynamic enforcement bitmap.

### 5. Multi-Frame ISO-TP Works on Mongoose (Selectively)
DID 0x2A2A (80 bytes) reads successfully through the Mongoose. The issue with DID 0x00BE is NOT a general multi-frame problem — J533 specifically hangs when processing this DID because it attempts to read from sub-bus modules.

## Architecture Model (Refined)

```
GEKO Server (Wolfsburg)
    │
    ▼ (SVM token via ODIS online)
J533 Gateway ← "Bouncer with guest list"
    │ Stores: IKA blob per enrolled module serial
    │ Constellation: bookkeeping record (static)
    │
    ├──→ J255 Climatronic ← "Guest checking their own invitation"
    │     Receives challenge from J533
    │     Checks against internal CP state
    │     Sets U1101 if mismatch
    │     Self-restricts functionality
    │
    ├──→ J136 Memory Seat (KWP2000 sub-bus)
    │     Same self-policing model
    │
    └──→ J521 Memory Seat Passenger (KWP2000 sub-bus)
          Same self-policing model
```

## Next Steps

### Immediate (SA2 Path)
1. **Find SA2 bytecode** for `EV_AirCondiBasisUDS` — check bri3d/sa2_seed_key, ODIS engineering data, or extract from FlashDaten ODX
2. **Compute SA2 key** from seed `BA2F9A47` + bytecode
3. **Unlock J255** with `27 04` + computed key
4. **Retry write** to DID 0x00BE after SA2 unlock — the acceptance criteria may change with security unlocked
5. **Test with correct blob** — if the blob is serial-specific, we need to understand the derivation (VIN + serial → GEKO computation → IKA blob)

### Hardware Path (Parallel)
1. **Read 4-zone J255 EEPROM** — the enrolled unit has a valid IKA blob in its EEPROM. Dumping it reveals the blob format and the relationship between blob and serial.
2. **Compare blobs** — if 4-zone blob matches J136 blob → VIN-derived (universal). If different → serial-specific.
3. **EEPROM map** — identify the CP data region in J255's EEPROM for direct manipulation.

### Tools Built Today
- `D:\ECU FLASH\j533_read.py` — standalone J2534 DID reader (32-bit Python)
- `D:\ECU FLASH\py310x86\` — 32-bit Python 3.10 embeddable (for J2534 DLL compatibility)
- simos-suite Deep Diagnostic bug fix (commit `276f6bd`)
- simos-suite `quick_test()` + known blob constants (commit `e19751f`)

## SA2 Security Access — Deep Dive

### Bytecode Extraction
SA2 bytecode extracted from `FL_4G0820043HI_0088_S.frf` ODX:
- SECURITY-METHOD: `SA2`
- FW-SIGNATURE (bytecode): `93 27 03 19 46 4C` (6 bytes)

### Key Computation (Flash SA2)
Algorithm: `key = (seed + 0x27031946) & 0xFFFFFFFF` (simple 32-bit addition)

Verified with bri3d/sa2_seed_key library:
| Seed | Key | Status |
|------|-----|--------|
| `C65DDCA1` | `ED60F5E7` | ✅ Math correct |
| `B185965C` | `D888AFA2` | ✅ Math correct |
| `D7B5F230` | `FEB90B76` | ✅ Math correct |

### Live ECU Response
**All computed keys rejected with NRC 0x35 (invalidKey).**

The flash programming SA2 bytecode produces mathematically correct keys per the bri3d library, but the **diagnostic session SA2** (level 0x03 for DID write access) uses a **different key derivation** — likely the vehicle's SKC (Secret Key Code) combined with the seed.

### Architecture Split
```
Flash SA2:       bytecode 93270319464C → key = seed + 0x27031946
Diagnostic SA2:  SKC-derived → key = f(seed, vehicle_SKC)
```

These are separate authentication domains. The flash bytecode is embedded in the FRF/ODX, while the diagnostic SKC is vehicle-specific.

### Next Step: SKC Acquisition
The vehicle's SKC (5-digit code) is needed to compute the diagnostic SA2 key. Sources:
1. VagTacho — reads PIN from instrument cluster EEPROM (requires FTDI hardware)
2. VCDS Login — common codes: 12233, 11463, 00000, 20103
3. Audi dealer — with VIN + proof of ownership
4. Instrument cluster EEPROM dump — SKC at known offset

## Late Session — CP Master Switch Discovery

### VCDS MAS00049 — Component Protection Output Test
Located in: **Address 19 (CAN Gateway) → Output Tests → MAS00049-Component protection**

When activated with "Start parameter: On":
- ALL modules in the car immediately enter CP active state
- MMI displays "features disabled"
- Turning it OFF + reboot → all enrolled modules recover (CP cleared)
- This proves J533 has a **master CP broadcast mechanism**

### Routine 0x0049 on J533
- **Exists**: NRC 0x13 (wrongLength) confirms the routine is registered
- **Format**: Takes exactly **3 bytes of data** (same as routine 0x0226)
- **Previously executed**: RequestResults `71 03 00 49` = positive
- All tested 3-byte values return NRC 0x31 (outOfRange) — correct encoding unknown
- May require SA2/login on J533 before accepting parameters

### Showroom Mode (DID 0x043E) Writable
- Write 0xFF → positive response `6E 04 3E` (write accepted)
- Write 0x00 → no response (J533 hangs — may involve sub-bus notification)
- Currently set to 0xFF

### Implication
Two CP enforcement layers exist:
1. **J533 master broadcast** (routine 0x0049) — affects all modules simultaneously
2. **Module-level IKA verification** — individual challenge-response per module

The master switch may be able to override module-level CP without IKA blobs.
Finding the correct routine 0x0049 parameters could bypass the entire IKA/SA2 chain.
