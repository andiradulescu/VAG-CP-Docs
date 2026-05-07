# J533 Constellation Slot Map — Decoded from Live Vehicle

*2013 Audi A6 C7 (4G0), Gateway 4G0907468AC, SW 0037*
*Updated 2026-05-07 — full 80-byte allocation, 160-byte TP-Ident, auth analysis*

---

## Critical Finding: J533 Does Not Enforce CP

Live reads with engine on and all modules powered confirmed:

```
DID 0x04A3  Constellation:   FD A1 E8 0C FE 62 60 0D 00 00
DID 0x2A26  Present bitmap:  FD A1 E8 0C FE 62 60 0D 00 00  ← IDENTICAL
DID 0x0439  Auth incorrect:  00 00 00 00 00 00 00 00 00 00  ← ALL ZEROS
```

**Constellation = Present bitmap.** These are the same value. The constellation
is the module install list, not a CP enrollment table.

**Auth incorrect = all zeros.** No modules are failing CP authentication from
J533's perspective. The gateway is not enforcing CP on any module.

**J255's CP is self-policed.** VCDS reports `U1101 Component Protection Active`
on J255 (Address 08), but J533 sees no auth failure. The CP restriction is
enforced by J255's own internal flash, independently of the gateway.

This means clearing the constellation bitmap will not fix CP. The fix must
target J255 directly — either via UDS write to J255 on the Convenience CAN,
or via chip programmer on J255's V850 data flash.

---

## DID 0x2A2A — Complete Allocation Table (80 Bytes)

Full capture from live vehicle:

```
0102030304050608 090E1011138B3415
1617171819 1B1E20 2228 2E30363C3D3E
4246475253555657 855F622665696C6D
6F723B7F82844858 88898A8D8F90AC44
818EC0A940518CBA 1400000000000000
```

Each byte is the **VCDS module address** for that constellation slot. This was
confirmed by cross-referencing known modules:

- `0x08` in slot 7 = VCDS address 08 = HVAC (J255) ✓
- `0x36` in slot 28 = VCDS address 36 = Driver Seat Memory (J136) ✓
- `0x2E` in slot 26 = VCDS address 2E (46 dec) = Night Vision ✓

### Complete Slot Map (73 modules + 7 empty)

| Slot | Addr | VCDS | Module | Enrolled | CAN TX |
|------|------|------|--------|----------|--------|
| 0 | 0x01 | 01 | Engine (J623) | YES | — |
| 1 | 0x02 | 02 | Transmission (J217) | no | — |
| 2 | 0x03 | 03 | ABS/Brakes (J104) | YES | 0x0713 |
| 3 | 0x03 | 03 | ABS/Brakes (2nd) | YES | — |
| 4 | 0x04 | 04 | Steering Angle (G85) | YES | — |
| 5 | 0x05 | 05 | Access/Start (J518) | YES | — |
| 6 | 0x06 | 06 | Seat Passenger (J521) | YES | — |
| **7** | **0x08** | **08** | **HVAC/Climatronic (J255)** | **YES** | **—** |
| 8 | 0x09 | 09 | Central Electronics (J519) | YES | — |
| 9 | 0x0E | 0E | Steering Lock/ELV | no | — |
| 10 | 0x10 | 10 | Parking Aid (J446) | no | — |
| 11 | 0x11 | 11 | Engine 2 | no | — |
| 12 | 0x13 | 13 | Adaptive Cruise (J428) | no | 0x0757 |
| 13 | 0x8B | 8B | Unknown (0x8B) | YES | 0x0756 |
| 14 | 0x34 | 34 | Steering Assist (J500) | no | 0x0755 |
| 15 | 0x15 | 15 | Airbag (J234) | YES | — |
| 16 | 0x16 | 16 | Steering Column Elec. | no | — |
| 17 | 0x17 | 17 | Instrument Cluster (J285) | no | — |
| 18 | 0x17 | 17 | Instrument Cluster (2nd) | no | — |
| 19 | 0x18 | 18 | Aux Heater (J604) | YES | — |
| 20 | 0x19 | 19 | CAN Gateway (J533) | no | — |
| 21 | 0x1B | 1B | Immobilizer 2 | YES | — |
| 22 | 0x1E | 1E | MMI/Info (J794) | YES | — |
| 23 | 0x20 | 20 | Rear Camera (J772) | YES | — |
| 24 | 0x22 | 22 | Sound System (J525) | no | 0x071D |
| 25 | 0x28 | 28 | Driver Seat (J136) | no | — |
| 26 | 0x2E | 2E | Night Vision (J852) | YES | — |
| 27 | 0x30 | 30 | Lane Assist (J769) | YES | — |
| 28 | 0x36 | 36 | Driver Door (J386) | no | — |
| 29 | 0x3C | 3C | Passenger Door (J387) | no | — |
| 30 | 0x3D | 3D | Rear Left Door | no | — |
| 31 | 0x3E | 3E | Rear Right Door | no | — |
| 32 | 0x42 | 42 | Driver Seat Adj | no | — |
| 33 | 0x46 | 46 | Central Conv (J393) | YES | — |
| 34 | 0x47 | 47 | Sound System 2 | YES | — |
| 35 | 0x52 | 52 | Adaptive Headlights L | YES | — |
| 36 | 0x53 | 53 | Adaptive Headlights R | YES | — |
| 37 | 0x55 | 55 | Headlight Range L | YES | — |
| 38 | 0x56 | 56 | Headlight Range R | YES | — |
| 39 | 0x57 | 57 | TV Tuner | YES | — |
| 40 | 0x85 | 85 | Unknown (0x85) | no | — |
| 41 | 0x5F | 5F | Info 2/Rear Display | YES | — |
| 42 | 0x62 | 62 | Rear Lid | no | — |
| 43 | 0x26 | 26 | Electric Steering | no | — |
| 44 | 0x65 | 65 | Tire Pressure (J502) | no | — |
| 45 | 0x69 | 69 | Trailer | YES | — |
| 46 | 0x6C | 6C | Rear Spoiler | YES | — |
| 47 | 0x6D | 6D | Trunk Lid | no | — |
| 48 | 0x6F | 6F | Battery Mgmt | no | — |
| 49 | 0x72 | 72 | Door Electronics RL | no | — |
| 50 | 0x3B | 3B | Rear Seat Climate | no | 0x0721 |
| 51 | 0x7F | 7F | Info Display 3 | no | — |
| 52 | 0x82 | 82 | Headlight L | no | — |
| 53 | 0x84 | 84 | Headlight R | YES | — |
| 54 | 0x48 | 48 | Sound System 3 | YES | — |
| 55 | 0x5B | 5B | Rear Camera 2 | no | — |
| 56 | 0x88 | 88 | Driver Assist | YES | — |
| 57 | 0x89 | 89 | Driver Assist 2 | no | — |
| 58 | 0x8A | 8A | Driver Assist 3 | YES | — |
| 59 | 0x8D | 8D | Pedestrian Protect | YES | — |
| 60 | 0x8F | 8F | Front Sensors | no | — |
| 61 | 0x90 | 90 | Rear Sensors | no | — |
| 62 | 0xAC | AC | Electromech Parking | no | — |
| 63 | 0x44 | 44 | Steering Assist 2 | no | 0x0712 |
| 64 | 0x81 | 81 | Multibeam L | no | — |
| 65 | 0x8E | 8E | Pedestrian Protect 2 | no | 0x0758 |
| 66 | 0xC0 | C0 | Lane Change Assist | no | — |
| 67 | 0xA9 | A9 | Actuator Elect | no | — |
| 68 | 0x40 | 40 | Rear Seat Adj | no | — |
| 69 | 0x51 | 51 | Electric Drive | no | — |
| 70 | 0x8C | 8C | Hybrid Battery | no | — |
| 71 | 0xBA | BA | EPB Actuator | no | — |
| 72 | 0x14 | 14 | Susp. Control (J197) | no | — |
| 73–79 | 0x00 | — | (empty) | no | — |

**Total: 73 module slots used, 31 enrolled, 7 empty.**

Modules with CAN TX = `—` are on sub-buses (Convenience CAN, KWP2000, LIN)
and are reachable only through J533 gateway routing, not directly via the OBD-II
Drive Train CAN pins 6+14.

### J255 Is KWP/Sub-Bus Routed

J255 (slot 7) has no TP-Ident CAN TX ID (0x0000). It communicates via VW TP 2.0
on the Convenience CAN (OBD-II pins 3+11, 100kbps), not ISO-TP UDS.

The MongoosePro ISO2 adapter can only reach Drive Train CAN (pins 6+14, 500kbps).
VCDS and ODIS reach J255 because they support VW TP 2.0 transport protocol.
Standard J2534 ISO-TP cannot reach J255.

**Direct Convenience CAN access is required to talk to J255.** A dual-CAN
ESP32 bridge (MCP2515+TJA1050 on pins 3+11 at 100kbps) is the solution.

---

## DID 0x2A2C — TP-Identifier Table (160 Bytes)

Each slot has 2 bytes for its diagnostic CAN TX ID. Modules with `0x0000`
are not directly addressable via ISO-TP on the Drive Train CAN.

```
Slot  0-7:   0000 0000 0713 0000 0000 0000 0000 0000
Slot  8-15:  0000 0000 0000 0000 0757 0756 0755 0000
Slot 16-23:  0000 0000 0000 0000 0000 0000 0000 0000
Slot 24-31:  071D 0000 0000 0000 0000 0000 0000 0000
Slot 32-39:  0000 0000 0000 0000 0000 0000 0000 0000
Slot 40-47:  0000 0000 0000 0000 0000 0000 0000 0000
Slot 48-55:  0000 0000 0721 0000 0000 0000 0000 0000
Slot 56-63:  0000 0000 0000 0000 0000 0000 0000 0712
Slot 64-71:  0000 0758 0000 0000 0000 0000 0000 0000
Slot 72-79:  0000 0000 0000 0000 0000 0000 0000 0000
```

Only 8 modules have direct ISO-TP CAN IDs. All other modules are reached
through J533's sub-bus routing (VW TP 2.0, KWP2000, or LIN).

---

## DID Cross-Reference Table

All DIDs read from J533 in a single session:

| DID | Name | Size | Value |
|-----|------|------|-------|
| `0x04A3` | Constellation | 10B | `FD A1 E8 0C FE 62 60 0D 00 00` |
| `0x2A2A` | Allocation table | 80B | (see slot map above) |
| `0x2A26` | Present bitmap | 10B | `FD A1 E8 0C FE 62 60 0D 00 00` |
| `0x2A2C` | TP-Identifiers | 160B | (see TP-Ident table above) |
| `0x0438` | Theft protection | 10B | `05 81 48 08 22 40 00 0C 00 00` |
| `0x0439` | Auth incorrect | 10B | `00 00 00 00 00 00 00 00 00 00` |
| `0x043D` | Successful downloads | 1B | `00` |
| `0x043E` | Showroom mode | 1B | `FF` |

---

## J255 Firmware Analysis

The J255 FRF (`FL_4G0820043HI_0088_S.frf`) was decrypted and analyzed.

### FRF Structure
- Encrypted with VW recursive XOR cipher (standard `frf.key`)
- Contains one ODX file (1.5MB) with two flash data blocks:
  - Block 1: 3,072 bytes (bootloader/header)
  - Block 2: 1,112,064 bytes (main firmware, V850 code)
- **RSA-1024 signed** — both blocks carry SHA1-RSA1024 signatures
- No readable strings in production firmware (standard for V850 automotive)

### Firmware Modification Is Blocked

Both flash data blocks are digitally signed:

```
SECURITY-METHOD: SIG_SHA1-RSA1024_S
FW-SIGNATURE: 526cb08946bd... (DB_1)
FW-SIGNATURE: 5d865afc0eef... (DB_2)
```

Patching the CP check in the firmware would break the signature. The J255
bootloader verifies the signature during flash and rejects modified firmware.
No known bootloader bypass exists for this Continental/Hella V850 platform
(unlike Bosch Simos where branch patches were documented).

### J255 Basic Variant Limitations

The installed J255 runs `EV_AirCondiBasisUDS` (basic 2-zone variant). Live
testing confirmed:

- DID `0x00BE` (IKA Key): **NRC 0x31** (requestOutOfRange) — not supported
- DID `0x00BD` (GKA Key): **NRC 0x31** — not supported
- DID `0xF18C` (Serial): Returns `25021300111130` — works

The basic variant does not implement the IKA/GKA DIDs. The comfort variant
(`EV_AirCondiComfoUDS`) does support them. This means even with direct
Convenience CAN access, writing DID `0x00BE` may not be possible on the
basic variant.

The dual-CAN ESP32 bridge is still the recommended path — it enables probing
J255's full DID and routine space to find alternative CP-related DIDs that
the basic variant does support.

---

## CP Clear Method — Confirmed by Independent Research

Independent reverse engineering of the EEPROM CP Transfer Tool (Nuitka-compiled
Python, 31 recovered profiles) confirms:

**A6 C6 Climate Control — LATE CP:**
```
EEPROM type:    93C86 (2KB dumps)
CP region:      offset 0x0400, length 0x0200 (512 bytes)
Clear method:   fill CP region with 0xFF
```

The CP transfer algorithm is a **byte-level copy at fixed offsets** — no
cryptographic computation. The tool copies CP bytes from a working module's
EEPROM dump (ORI) into the replacement module's dump (DONOR) at the same offset.

CP clearing fills the region with `0xFF` (flash-erased state). This corresponds
to the module's internal V850 data flash at offset 0x0400.

---

## VCDS Gateway Log — Session Confirmation

VCDS autoscan (2026-05-07) with Mongoose writes from previous session:

```
Programming Attempts(data): 2
Successful Attempts(data): 2
Flash Tool Code(data): 00200 785 02391
```

The two successful data programming attempts are our `WriteDataByIdentifier`
commands (IKA blob + constellation) from the previous session. Both were
accepted and logged. The gateway CP fault (`U1101 Component Protection Active`,
fault frequency 2, reset counter 183) is unchanged — same fault that was
present before our writes.

---

## Remaining Path to CP Resolution

1. **Build dual-CAN ESP32 bridge** — MCP2515+TJA1050 on OBD pins 3+11 at
   100kbps for direct Convenience CAN access to J255.

2. **Probe J255's DID and routine space** — the basic variant may support
   CP-related DIDs other than 0x00BE. A full DID scan from the Convenience
   CAN side will reveal what's available.

3. **Attempt CP data erase** — if a writable CP DID is found, write `0xFF`
   fill to clear the internal CP flag. If no UDS path exists, a V850 chip
   programmer targeting data flash offset 0x0400 (512 bytes) is the fallback.

---

*Document version: 2026-05-07 (v2). Based on live vehicle data with full DID
reads, J255 FRF analysis, and independent CP tool reverse engineering. All
testing performed on owner's vehicle for right-to-repair research purposes.*
