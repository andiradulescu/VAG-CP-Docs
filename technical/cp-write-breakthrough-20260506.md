# CP Write Breakthrough — 2026-05-06

## Summary

**Component Protection removal via standard UDS WriteDataByIdentifier confirmed working on J533 gateway — no SA2 security access, no SKC, no GEKO online connection required.**

Two commands in extended diagnostic session:
```
10 03                          → Extended session
2E 00 BE [34-byte IKA blob]    → Write IKA key (positive: 6E 00 BE)
2E 04 A3 [10-byte bitmap]      → Write constellation (positive: 6E 04 A3)
```

Both accepted by J533 with positive responses. No security access gate.

The remaining challenge: the IKA blob must match the target module's serial number. A blob from J136 was accepted by J533 but did not clear CP on J255 (different module, different serial).

---

## CP Control Protocol (Decoded from ODIS dprot.htm)

### DID 0x0348 — CP Master Switch (InputOutputControlByIdentifier)

Discovered by cross-referencing VCDS MAS00049 with ODIS diagnostic protocol log.

| Command | UDS Bytes | Response | Effect |
|---------|-----------|----------|--------|
| CP Test ON | `2F 03 48 03 FF FF FF FF` | `6F 03 48 03 FF FF FF FF` | All modules enter CP active |
| CP Test OFF | `2F 03 48 00` | `6F 03 48 00` | Release test mode |

- Service: InputOutputControlByIdentifier (0x2F), NOT RoutineControl (0x31)
- DID: 0x0348 (not 0x0049 as initially assumed from VCDS MAS group number)
- Data: 4 bytes — module mask (`FF FF FF FF` = all modules)
- ODIS used `05 FF FF FF` where `05` = actuation time (5 seconds)
- Confirmed live: CP activated on all modules, MMI showed features disabled
- Temporary — reverts on returnControlToECU or timeout

### DID 0x00BE — IKA Key Write

| Target | UDS Bytes | Response |
|--------|-----------|----------|
| J533 (gateway) | `2E 00 BE [34 bytes]` | `6E 00 BE` (after 0x78 pending) |
| J255 direct (no SA2) | `2E 00 BE [34 bytes]` | `7F 2E 31` (after 0x78 pending) |
| J255 direct (with SA2) | Not tested — needs SKC | — |

- Write through J533 (0x710) works — J533 accepts and stores/forwards the blob
- Write directly to J255 (0x746) is rejected without SA2 unlock
- No security access required on J533 for this write
- The 0x78 pending (~1 second processing) confirms J533 validates/forwards the blob

### DID 0x04A3 — Constellation Bitmap Write

| Command | Response |
|---------|----------|
| `2E 04 A3 FD A1 E8 0C FE 62 60 0D 00 00` | `6E 04 A3` |

- Direct write accepted with no security access
- CP-ACTIVE bitmap: `FD A1 E9 0C FE 62 64 8D 00 00`
- CP-CLEARED bitmap: `FD A1 E8 0C FE 62 60 0D 00 00`
- Bit differences: byte 2 (0xE9→0xE8), byte 6 (0x64→0x60), byte 7 (0x8D→0x0D)

---

## Complete ODIS CP Removal Sequence (from dprot.htm)

Extracted from `NO_ORDER_WAUGGAFC7DN120188_2024-02-27_22-59-28_dprot.htm`.
Session performed by Dealer 17483, ODIS-S Patch 23.0.1, VAS 6154A.

### Phase 1 — Data Collection
1. Read VIN from J518 Kessy → `WAUGGAFC7DN120188`
2. Read J518 FAZIT → `HLH-W41 21.02.13 1003 1126`
3. Read J518 HW number → `4H0907064CR`
4. Read key transponder data from J518 (DID 740, 750, 749)
5. Read J533 FAZIT → `LAK-000 22.02.13 2009 7162`

### Phase 2 — GEKO Online Connection
6. Login to GEKO: `sys_x_x_1_1218_21_login_logon_konzernsystem_00000`
7. Send vehicle data to GEKO for token generation

### Phase 3 — Module Enrollment (IKA + GVA Writes)
8. J136 (Memory Seat): TrainICA `******` + TrainGVA `******`
9. J136 IKA blob written via GatewUDS: `WriteDataByIdentifier(0x00BE)` = `E62B41D1...2600` ← only unmasked blob
10. J518 (Kessy): TrainICA + TrainGVA (masked)
11. J519 (Central Electrics): TrainICA + TrainGVA (masked)
12. J525 (Sound System): TrainICA + TrainGVA (masked)
13. J854/J855 (Seatbelt Tensioners): TrainICA + TrainGVA (masked)
14. Additional modules: TrainICA + TrainGVA (masked)

### Phase 4 — CP Test
15. CP test mode ON: `2F 03 48 03` → response `05 FF FF FF`
16. Status check: "actuator test running"
17. Wait 5 seconds → "actuator test finished - timeout detected"
18. CP test mode OFF: `2F 03 48 00`

### Phase 5 — Gateway Constellation Write
19. Read constellation from J533: `FD A1 E9 0C FE 62 64 8D 00 00` (CP-ACTIVE)
20. **Write CP-CLEARED constellation**: `2E 04 A3 FD A1 E8 0C FE 62 60 0D 00 00`
21. Verify: read back → `FD A1 E8 0C FE 62 60 0D 00 00` (confirmed CLEARED)

### Phase 6 — Completion
22. Clear DTCs on all modules
23. "The basic programming of the Gateway was performed. Next the remaining components are enabled."

---

## Key Architectural Findings

### 1. No Security Access Required on J533
Both DID 0x00BE (IKA blob) and DID 0x04A3 (constellation) are writable in extended diagnostic session without any SecurityAccess (0x27) handshake. This contradicts the assumption that SA2/SKC gates the CP write path on the gateway.

### 2. Constellation is Written, Not Computed
ODIS directly wrote the CP-CLEARED bitmap to J533. The gateway does not compute this value — it accepts whatever is written. The constellation is purely a bookkeeping record that ODIS manages.

### 3. IKA Blob is Module-Serial-Specific
The J136 blob (`E62B41D1...2600`) was accepted by J533 but did not clear CP on J255 basic (serial `25021300111130`). The blob was generated by GEKO for J136 (serial `00001002393673`). Each module needs its own blob derived from its specific serial number.

### 4. CP Enforcement Model (Final)
```
J533 Gateway:
  - Stores IKA blobs per module slot (writable, no SA2)
  - Stores constellation bitmap (writable, no SA2)
  - Broadcasts CP state to modules during boot
  - Does NOT actively enforce — just stores and relays

J255 Module:
  - Receives challenge from J533 on boot
  - Compares against internal CP credentials
  - Self-restricts if credentials don't match
  - CP fault U1101 is set internally by the module
```

### 5. TrainICA vs WriteDataByIdentifier
- **UDS modules** (J255, J533, J234, J285, J794): CP data written via `WriteDataByIdentifier(0x00BE)`
- **KWP2000 modules** (J136, J518, J519, J525, J854, J855): CP data written via `TrainICA` service
- The J136 blob appeared in plaintext because it went through `WriteDataByIdentifier` on the gateway rather than the masked `TrainICA` path

---

## Remaining Path to CP Removal

### What Works
- [x] Write IKA blob to J533 (DID 0x00BE) — no SA2 needed
- [x] Write constellation to J533 (DID 0x04A3) — no SA2 needed
- [x] CP test mode activation/deactivation (DID 0x0348)
- [x] SA2 seed request from J255 (level 0x03, 4-byte seed)

### What's Missing
- [ ] **Correct IKA blob for J255 basic** (serial `25021300111130`)
  - Options: find in ODIS logs, dump from 4-zone EEPROM, derive from known algorithm
- [ ] **Understanding blob derivation** — is it VIN + serial + GEKO secret, or simpler?
  - Two known blobs from same VIN would answer this (compare J136 vs 4-zone)

### Fastest Paths
1. **Find J255 ODIS session** — Andrew searching another ODIS VM
2. **Dump 4-zone EEPROM** — read the enrolled blob from the physical chip
3. **Derive the algorithm** — compare multiple blobs from same VIN to isolate the derivation
