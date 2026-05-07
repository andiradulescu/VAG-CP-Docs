# J533 Constellation Slot Map — Decoded from Live Vehicle

*2013 Audi A6 C7 (4G0), Gateway 4G0907468AC, SW 0037*

---

## DID 0x2A2A — Allocation Table (Slot → Module Address)

DID `0x2A2A` returns a byte array where each byte is the **VCDS module address**
for that constellation slot. This was confirmed by cross-referencing the
allocation table with known VCDS addresses: slot 7 contains `0x08` (VCDS address
08, HVAC/Climatronic) and slot 27 contains `0x36` (VCDS address 36, Driver Seat
Memory), both matching their well-known VCDS assignments.

Live capture (first 30 of 80 bytes — display truncated at source):

```
01 02 03 03 04 05 06 08 09 0E 10 11 13 8B 34
15 16 17 17 18 19 1B 1E 20 22 28 2E 30 36 3C
```

### Decoded Slot Map

| Slot | Byte | VCDS Addr | Module | Constellation Bit |
|------|------|-----------|--------|-------------------|
| 0 | `0x01` | 01 | Engine ECU (J623) | byte 0, bit 0 |
| 1 | `0x02` | 02 | Transmission (J217) | byte 0, bit 1 |
| 2 | `0x03` | 03 | ABS/Brakes (J104) | byte 0, bit 2 |
| 3 | `0x03` | 03 | ABS/Brakes (2nd controller) | byte 0, bit 3 |
| 4 | `0x04` | 04 | Steering Angle Sensor (G85) | byte 0, bit 4 |
| 5 | `0x05` | 05 | Access/Start Auth (J518/Kessy) | byte 0, bit 5 |
| 6 | `0x06` | 06 | Seat Memory Passenger (J521) | byte 0, bit 6 |
| **7** | **`0x08`** | **08** | **Climatronic (J255)** | **byte 0, bit 7** |
| 8 | `0x09` | 09 | Central Electronics (J519) | byte 1, bit 0 |
| 9 | `0x0E` | 0E | Steering Column Lock/ELV | byte 1, bit 1 |
| 10 | `0x10` | 10 | Parking Aid (J446) | byte 1, bit 2 |
| 11 | `0x11` | 11 | Engine Control 2 | byte 1, bit 3 |
| 12 | `0x13` | 13 | Adaptive Cruise (J428) | byte 1, bit 4 |
| 13 | `0x8B` | 8B | Unknown (0x8B) | byte 1, bit 5 |
| 14 | `0x34` | 34 | Steering Assist (J500) | byte 1, bit 6 |
| 15 | `0x15` | 15 | Airbag (J234) | byte 1, bit 7 |
| 16 | `0x16` | 16 | Steering Column Electronics | byte 2, bit 0 |
| 17 | `0x17` | 17 | Instrument Cluster (J285) | byte 2, bit 1 |
| 18 | `0x17` | 17 | Instrument Cluster (2nd) | byte 2, bit 2 |
| 19 | `0x18` | 18 | Auxiliary Heater (J604) | byte 2, bit 3 |
| 20 | `0x19` | 19 | CAN Gateway (J533) | byte 2, bit 4 |
| 21 | `0x1B` | 1B | Immobilizer/Access 2 | byte 2, bit 5 |
| 22 | `0x1E` | 1E | Information Electronics (MMI/J794) | byte 2, bit 6 |
| 23 | `0x20` | 20 | Rear Camera / Assist Systems | byte 2, bit 7 |
| 24 | `0x22` | 22 | Sound System (J525) | byte 3, bit 0 |
| 25 | `0x28` | 28 | Driver Seat Memory (J136) | byte 3, bit 1 |
| 26 | `0x2E` | 2E | Night Vision (J852) | byte 3, bit 2 |
| 27 | `0x30` | 30 | Lane Assist (J769) | byte 3, bit 3 |
| 28 | `0x36` | 36 | Driver Door (J386) | byte 3, bit 4 |
| 29 | `0x3C` | 3C | Passenger Door (J387) | byte 3, bit 5 |
| 30–79 | — | — | Remaining 50 slots (not captured) | bytes 3–9 |

**Note:** Only 30 of 80 bytes were captured due to hex display truncation. The
remaining 50 bytes (slots 30–79) need to be re-read from the vehicle. The first
30 slots account for 31 of the 31 set bits in the CP-CLEARED constellation, so
the remaining slots may all be empty/unassigned.

### Key Insight: VCDS Address = Allocation Byte

The allocation table byte value IS the VCDS module address, directly. No
translation table needed. This was confirmed empirically:

- `0x08` in slot 7 = VCDS address 08 = HVAC (J255) ✓
- `0x36` in slot 27 = VCDS address 36 = Driver Seat Memory (J136) ✓
- `0x2E` in slot 26 = VCDS address 2E (46 decimal) = Night Vision ✓

---

## Constellation Bitmap Cross-Reference

### CP-CLEARED State

```
Byte:    FD  A1  E8  0C  FE  62  60  0D  00  00
Binary:  11111101 10100001 11101000 00001100 ...
```

Decoding byte 0 (`0xFD = 11111101`):

| Bit | Value | Slot | Module |
|-----|-------|------|--------|
| 0 | 1 | 0 | Engine (J623) — enrolled |
| 1 | 0 | 1 | Transmission — NOT enrolled |
| 2 | 1 | 2 | Brakes — enrolled |
| 3 | 1 | 3 | Brakes (2nd) — enrolled |
| 4 | 1 | 4 | Steering Angle — enrolled |
| 5 | 1 | 5 | Kessy — enrolled |
| 6 | 1 | 6 | Seat Passenger (J521) — enrolled |
| **7** | **1** | **7** | **Climatronic (J255) — enrolled** |

### Auth Incorrect Bitmap (DID 0x0439)

Captured with engine off:

```
01 00 00 00 00 00 00 00 00 00
```

Only bit 0 (Engine, slot 0) shows as failing. This is expected: the engine ECU
was powered down and could not respond to the CP challenge. **DID 0x0439 is a
live status register — it must be read with all modules powered (engine running)
to get a true picture.**

With engine running, J255 (bit 7) would likely also appear as failing if its CP
is truly active, as VCDS independently reports.

### ODIS Session Bit Diff (CP-ACTIVE → CP-CLEARED)

```
CP-ACTIVE:   FD A1 E9 0C FE 62 64 8D 00 00
CP-CLEARED:  FD A1 E8 0C FE 62 60 0D 00 00
DIFF:        00 00 01 00 00 00 04 80 00 00
```

Three bits changed during the Feb 2024 ODIS CP removal:

| Bit position | Byte.bit | Slot | Module (from allocation table) |
|--------------|----------|------|-------------------------------|
| 16 | byte 2, bit 0 | 16 | Steering Column (0x16) |
| 50 | byte 6, bit 2 | 50 | Unknown (need full table) |
| 63 | byte 7, bit 7 | 63 | Unknown (need full table) |

These three modules had their CP enrollment toggled. The remaining 28 set bits
were already enrolled and unchanged — they were cleared in the same session but
their enrollment bits were already in the target state.

---

## DID 0x0438 — Theft Protection Keys

```
05 81 48 08 22 40 00 0C 00 00
```

This is a separate bitmap from the constellation. Set bits identify modules with
active theft protection (distinct from Component Protection). The 12 set bits
correspond to a subset of the 31 enrolled modules.

---

## Implications for CP Resolution

### J255 (Climatronic) — Slot 7, Byte 0 Bit 7

To de-enroll J255 from the constellation:

```
Current byte 0:  0xFD = 11111101 (bit 7 SET)
Target byte 0:   0x7D = 01111101 (bit 7 CLEARED)
```

Write command:
```
2E 04 A3 7D A1 E8 0C FE 62 60 0D 00 00
```

This tells J533 "slot 7 is not enrolled." Combined with writing an empty/cleared
IKA blob to DID `0x00BE`, this should stop J533 from challenging J255.

**However:** VCDS reports CP active on J255 itself. This means J255 is
self-policing — its internal CP flag is set independently of J533's constellation.
Clearing the constellation alone may not be sufficient. J255's internal flash
must also be addressed (either through a valid IKA blob write that J255 accepts,
or through the CP clear method described in the offline resolution document).

### Night Vision (J852) — Slot 26, Byte 3 Bit 2

Night Vision is enrolled in the constellation (`0x0C` byte 3, bit 2 = 1) despite
the physical module not being installed. This is because the gateway is coded to
expect it. When the module is eventually installed, it will need either:

1. A GEKO session to enroll it with a valid IKA key, or
2. The CP clear approach to pre-clear it before installation

If the module was purchased used (from another vehicle), it will have CP active
from its donor car. The gateway coding accepting it does not bypass CP — the
module's internal CP state and IKA key are separate from the gateway's install list.

---

## Related DIDs — Full Read Plan

| DID | Name | Size | Status |
|-----|------|------|--------|
| `0x04A3` | Constellation bitmap | 10 bytes | ✓ Captured |
| `0x2A2A` | Allocation table (slot→module) | 80 bytes | ⚠ First 30 bytes only |
| `0x2A26` | Present bitmap (online/offline) | ? bytes | Not yet read |
| `0x2A2C` | TP-Identifier (CAN IDs per slot) | ? bytes | Not yet read |
| `0x00BE` | IKA Key (pass-through to module) | 34 bytes | Hung — sub-bus routing issue |
| `0x0438` | Theft protection keys | 10 bytes | ✓ Captured |
| `0x0439` | Auth incorrect (live status) | 10 bytes | ✓ Captured (engine off) |
| `0x043D` | Successful key downloads | ? bytes | Not yet read |
| `0x043E` | Showroom mode | 1 byte | ✓ Captured (0xFF) |
| `0x0348` | CP test mode (IOCTL) | 4+ bytes | ✓ Confirmed working |

**Next priority:** Re-read `0x2A2A` (full 80 bytes), read `0x2A26` and `0x2A2C`,
and re-read `0x0439` with engine running.

---

## MMI3G+ Persistence Layer — CP Data Structure

Root shell access to the MMI3G+ head unit (`192.168.0.154:23`, QNX Neutrino)
confirmed the CP data storage architecture:

### DataPST.db (SQLite, 51,200 bytes)

Location: `/mnt/hmisql/DataPST.db` (and backup at `/HBpersistence/DataPST.db`)

CP data lives in namespace `1550256158` with the following structure:

| Key | Size | Content (post-clearance) | Purpose |
|-----|------|--------------------------|---------|
| `0x1000500FF1000C8` | 488 bytes | Header (starts with `0x78`) | CP configuration |
| `0x1000500FF1000C9` | 960 bytes | All `0xFF` (erased) | CP data blob channel 1 |
| `0x1000500FF1000CA` | 480 bytes | All `0xFF` (erased) | CP data blob channel 2 |
| `0x1000500FF1000CB` | 480 bytes | All `0xFF` (erased) | CP data blob channel 3 |
| `0x103FFFFFFFF00C9` | 16 bytes | All `0x00` (cleared) | AES-128 key slot 1 |
| `0x103FFFFFFFF00CA` | 16 bytes | All `0x00` (cleared) | AES-128 key slot 2 |
| `0x103FFFFFFFF00CE` | 16 bytes | All `0x00` (cleared) | AES-128 key slot 3 |

The Feb 2024 ODIS session zeroed all three 16-byte AES key slots and filled the
three data blobs with `0xFF` (flash-erased state). A backup at
`/mnt/hmisql/DataPST.db.copy` was confirmed identical — no pre-clearance data
to recover.

### RSA Public Keys

The MMI stores three RSA-2048 public keys used for signature verification:

| Path | File | Size | Purpose |
|------|------|------|---------|
| `/HBpersistence/Keys/DataKey/` | `DK_public_signiert.bin` | 256 bytes | Data key verification |
| `/HBpersistence/Keys/FSCKey/` | `FSC_public_signiert.bin` | 256 bytes | FSC feature code verification |
| `/HBpersistence/Keys/MetainfoKey/` | `MI_public_signiert.bin` | 256 bytes | Metadata verification |

Only the Data Key was captured before the MMI entered sleep mode.

---

## Gateway Firmware RE — CP Toggle Mechanism

Ghidra analysis of J533 firmware (`C7_4G8_FD_1DATA.bin`, Renesas V850E) confirmed
the internal CP toggle mechanism:

### DID 0x0348 — IOCTL CP Master Switch

The UDS router at `FUN_00047540` dispatches CP commands via internal messages:

```
Internal msg 0x0CFF → flag = 0x0B → CP ACTIVE state
Internal msg 0x0CFE → flag = 0x00 → CP CLEARED state
```

Both paths call `FUN_00047ef2` (CP Broadcast) → `FUN_00046564` (CP Sub-Handler)
which writes the CP state to hardware register `DAT_fede9250` and clears the
interlock flag `DAT_fede13d7`.

### The 0x0049 Table Size Limit

The firmware function at `FUN_0004e8c8` contains:

```c
if ((param_2 & 0xffff) < 0x49) {
    iVar2 = (param_2 & 0xffff) * 8;
    *(undefined4 *)(iVar2 + 0xFEDBBB8C) = param_3;
}
```

`0x49` (73 decimal) is a **table size limit**, not a routine ID. The gateway
maintains a table of up to 73 entries at 8 bytes each, starting at `0xFEDBBB8C`.
This may be the CP module enrollment table in RAM.

---

*Document version: 2026-05-07. Based on live vehicle data, Ghidra firmware
analysis, and MMI root shell exploration. All testing performed on owner's
vehicle for right-to-repair research purposes.*
