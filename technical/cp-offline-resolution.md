# Offline CP Resolution — Architecture and Research Plan

*A decision-tree approach to resolving Component Protection without VW server access.*

**Status:** Active research — experiments pending  
**Platforms:** Audi A6 C7 (4G), VW Touareg 7P (RNS-850), and all MLB/Lear-gateway vehicles  
**Last updated:** May 2026

---

## Why This Document Exists

GEKO is offline. GRP excludes independent repair by design. Dealers are refusing
used parts or charging $400 for 20 minutes of work — when they understand what
CP is at all. There is no functioning online path for independent CP removal on
these platforms.

This document lays out every offline resolution path we've identified, ordered
by what's testable today, what each result tells us, and what to do next based
on the outcome. It is written to be actionable — not theoretical.

---

## Current Server Status (May 2026)

| System | Status | Impact |
|--------|--------|--------|
| GEKO (legacy) | **Offline** — IMMO/CP endpoints down, no ETA | ODIS 23.x cannot perform CP removal |
| GRP (new) | **Operational for dealers** — IMMO/CP withheld from third-party tools | ODIS 25.x works only with dealer GRP credentials |
| ABRITES CP Online | Operational (paid service, requires AVDI hardware + VN licenses) | ~€2,000 tool investment |
| vw-geko.com (community) | **Degraded/offline** — dependent on GEKO backend | Previously offered €30 sessions |

**Bottom line:** No affordable, accessible online CP removal path exists for
independent owners or shops as of May 2026.

---

## What We Have — The Known Artifacts

### The IKA Blob (34 bytes — one confirmed sample)

From the Feb 27, 2024 ODIS CP removal session on VIN `WAUGGA**********8`,
the IKA key written to J136 (Memory Seat) was logged unmasked:

```
Offset  Hex                                              ASCII
00–0F:  E6 2B 41 D1 1C 44 AF 20 21 77 FB 1F 27 4B 0A C2   .+A..D. !w..'K..
10–1F:  D1 5B D2 62 E4 FD 27 AB 61 D1 23 C2 F1 5A 2C 93   .[.b..'.a.#..Z,.
20–21:  26 00                                              &.
```

All other modules had their blobs masked as `******` in the ODIS log.

### Hypothesized Blob Structure

Based on the MMI3G+ live system analysis confirming AES-128 (`cipher-aes.so`
provides `aes_encrypt_key128` / `aes_decrypt_key128` for CP challenge-response):

```
┌─────────────────────────────────────────────────────┐
│ Bytes 0–15:  AES-128 Key (16 bytes)                 │
│              Used in per-ignition challenge-response │
│              between J533 and each enrolled module   │
│                                                     │
│ Bytes 16–31: MAC / Verification Data (16 bytes)     │
│              Likely HMAC or AES-CMAC over:           │
│              (VIN ‖ FAZIT ‖ serial ‖ ?)             │
│              Verified by J533 MCU on blob acceptance │
│                                                     │
│ Bytes 32:    0x26 — Type/version tag                │
│              Note: 0x0226 is the CP routine ID      │
│              May encode CP generation (Gen3 V12)    │
│                                                     │
│ Bytes 33:    0x00 — Padding or null terminator      │
└─────────────────────────────────────────────────────┘
```

**This structure is hypothesized, not confirmed.** The byte-boundary split and
field assignments require validation (see Experiment 3 below).

### Known GEKO Inputs

Before contacting GEKO, ODIS collected and sent these values:

| Input | Value | Bytes (approx) |
|-------|-------|-----------------|
| VIN | `WAUGGA**********8` | 17 |
| J518 FAZIT | `HLH-W41 21.02.13 1003 1126` | 24 |
| J518 HW number | `4H0907064CR` | 11 |
| J533 FAZIT | `LAK-000 22.02.13 2009 7162` | 24 |
| Target module FAZIT | varies per module | ~24 each |
| Key fob transponder | `******` (masked, unknown) | unknown |

The transponder data is the one unobserved input. Whether it is incorporated
into the IKA derivation or used only as an authorization gate is unknown.

### Constellation Snapshots

DID `0x04A3` on J533 before and after CP removal:

```
Before:  FD A1 E9 0C FE 62 64 8D 00 00
After:   FD A1 E8 0C FE 62 60 0D 00 00
                 ^^           ^^ ^^
                 │            │  └─ Byte 7 changed
                 │            └──── Byte 6 changed
                 └─────────────── Byte 2 changed
```

### Certificates

| Certificate | Key | Purpose |
|-------------|-----|---------|
| VW-CA-ROOT-00 | 4096-bit RSA, SHA384 | Root CA — signs all VW artifacts |
| www.vwgeko.net | 2048-bit RSA, self-signed | GEKO server TLS identity |
| dms_client | **Empty** — never provisioned | Would hold dealer auth key |

### Confirmed Protocol Values

| Item | Value | Source |
|------|-------|--------|
| CP routine ID | `0x0226` | `ES_LIBCompoProteGen3V12.sd.db` |
| IKA key DID | `0x00BE` (34 bytes) | AU57X MWB extraction + live ODIS session |
| GKA key DID | `0x00BD` (34 bytes) | AU57X MWB extraction |
| Constellation DID | `0x04A3` (10 bytes) | AU57X MWB extraction + live session |
| Allocation table DID | `0x2A2A` | AU57X MWB extraction |
| J255 ECU Name code | `8` (Air Conditioning) | AU57X TEXTTABLE |
| J136 ECU Name code | `54` (Seat Adjustment Driver) | AU57X TEXTTABLE |
| Security access for writes | `None` — extended session only | AU57X MWB `access_level` field |
| J533 CAN IDs | TX `0x710` / RX `0x77A` | AU57X link layer |
| J255 CAN IDs | TX `0x746` / RX `0x7B0` | AU57X link layer |
| J136 CAN IDs | TX `0x74C` / RX `0x7B6` | AU57X link layer |
| J285 CAN IDs | TX `0x714` / RX `0x77E` | AU57X link layer |

### Gateway Firmware Available

| Platform | Part Number Family | MCU | Status |
|----------|--------------------|-----|--------|
| A6 C7 (4G) | `4G0 907 468x` | Renesas D70F3433(A), V850E | Not yet extracted |
| **A8 D4 (4H)** | `4H0 907 468x` | Renesas D70F3433(A), V850E | **Available locally** |
| Touareg 7P | `7P6 907 530x` | Likely same Lear D70F3433 | Not yet confirmed |

The D4 A8 gateway shares the same Lear architecture and D70F3433 MCU as the C7.
The CP verification routine in the A8 firmware should be structurally identical
or very close to the A6 variant. **This firmware is the most direct path to
understanding the IKA verification algorithm without extracting from a live
gateway.**

---

## The Decision Tree

Everything branches from one experiment.

```
┌─────────────────────────────────────────────────────────┐
│  EXPERIMENT 1: Cross-Module 0x00BE Read                 │
│                                                         │
│  Read DID 0x00BE from J533, J255, J136, J285            │
│  in extended session (0x10 0x03) on the live car        │
│                                                         │
│  Time: ~15 minutes with Mongoose + simos-suite          │
│  Risk: None — read-only                                 │
│  Prerequisites: Car accessible, OBD adapter connected   │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
   ALL BLOBS MATCH            BLOBS DIFFER
   (VIN-bound)                (Module-bound)
          │                         │
          ▼                         ▼
┌─────────────────────┐   ┌──────────────────────────┐
│  REPLAY PATH        │   │  DERIVATION PATH          │
│                     │   │                            │
│  Write known J136   │   │  Must understand how GEKO  │
│  blob to any module │   │  generates per-module keys │
│  via 0x00BE, then   │   │                            │
│  update 0x04A3      │   │  ┌─ Firmware RE (D4 flash) │
│                     │   │  ├─ qconn/GDB on MMI3G+   │
│  Two UDS writes.    │   │  ├─ MAC analysis (if 2+    │
│  No server needed.  │   │  │  blobs available)       │
│  Done.              │   │  └─ GEKO emulator (last    │
│                     │   │     resort)                │
└─────────────────────┘   └──────────────────────────┘
```

---

## Experiment 1: Cross-Module `0x00BE` Read (THE GATE)

**What:** Read DID `0x00BE` from four modules on the live car.  
**Why:** Determines whether the IKA key is shared across all modules (VIN-bound)
or unique per module (module-bound). Every downstream decision depends on this.  
**How:** Extended diagnostic session via Mongoose + python-udsoncan or simos-suite.  
**Time:** 15 minutes.  
**Risk:** Zero — read-only operation.

### Targets

| Module | CAN TX | CAN RX | Expected if VIN-bound |
|--------|--------|--------|-----------------------|
| J533 Gateway | `0x710` | `0x77A` | Same 34-byte blob as J136 |
| J255 Climatronic | `0x746` | `0x7B0` | Same — or 34 zero bytes if CP-active/unenrolled |
| J136 Memory Seat | `0x74C` | `0x7B6` | Should match Feb 2024 captured blob |
| J285 Instrument Cluster | `0x714` | `0x77E` | Same 34-byte blob as J136 |

### UDS Sequence (per module)

```
→ [target_TX] 02 10 03              Enter extended session
← [target_RX] 06 50 03 ...         Session confirmed

→ [target_TX] 03 22 00 BE          ReadDataByIdentifier(0x00BE)
← [target_RX] 22 00 BE [34 bytes]  IKA key response
   OR
← [target_RX] 7F 22 31            requestOutOfRange (DID not supported)
   OR
← [target_RX] 7F 22 33            securityAccessDenied (need higher session)
```

### Interpreting Results

**All four return the same non-zero 34-byte blob:**
→ IKA is VIN-bound. Proceed to Replay Path. The blob you have works for every
module on this car.

**J136 matches Feb 2024 blob, others differ:**
→ IKA is module-bound. Proceed to Derivation Path. Each module gets a unique
key derived from (VIN + module FAZIT + something else).

**J136 matches, J255 returns 34 zero bytes:**
→ J255 has no IKA installed (expected — it was swapped after the Feb 2024
session). J136's blob is still valid. Check J533 and J285 to determine
VIN-bound vs module-bound. The zero response from J255 confirms CP is active
on it, which is what we expect.

**All return NRC `0x33` (securityAccessDenied):**
→ `0x00BE` requires a higher session level than extended. Try programming
session (`0x10 0x02`) or check if SA2 is needed despite the MWB saying
`access_level: None`. This would be a significant finding — document the
NRC and session level.

**All return NRC `0x31` (requestOutOfRange):**
→ DID `0x00BE` is not supported on this module variant. Check that you're
addressing the correct module. This would contradict the AU57X MWB extraction
and should be investigated.

---

## Experiment 2: Constellation Write Test

**What:** Test whether J533 accepts a write to DID `0x04A3` in extended session.  
**Why:** The MWB says `access_level: None` for the write service, but this has
never been empirically verified. If J533 accepts it, the offline replay path
is confirmed end-to-end.  
**How:** Read `0x04A3`, flip one bit in a non-critical unused slot, write it
back, read again, then immediately restore the original value.  
**Time:** 5 minutes.  
**Risk:** Low if done carefully — use an unoccupied slot bit.

### Procedure

```
1. Read 0x04A3 → save as original_constellation
2. Identify an unoccupied slot (a zero bit in the bitmap where no module exists)
3. Flip that one bit to 1
4. Write modified value via WriteDataByIdentifier(0x04A3, modified)
5. Read 0x04A3 again → compare
6. If write took: immediately write original_constellation back
7. Document: did J533 accept the write, or return an NRC?
```

### Possible Outcomes

**Write accepted (positive response `0x6E`):**
→ J533 accepts constellation writes in extended session without GEKO token.
The offline replay path is fully viable — both `0x00BE` and `0x04A3` writes
work without server involvement.

**NRC `0x72` (generalProgrammingFailure):**
→ J533 has an internal consistency check. It may reject constellation changes
unless they match a currently-present module's serial. Try setting a bit that
corresponds to a module actually on the bus.

**NRC `0x31` (requestOutOfRange):**
→ The write DID may differ from the read DID, or require a different payload
format. Check whether `WriteDataByIdentifier` uses `0x04A3` or a separate
write-specific DID.

**NRC `0x33` (securityAccessDenied):**
→ Despite MWB showing `access_level: None`, J533 requires authentication for
writes. Enumerate supported SA2 levels and investigate.

**NRC `0x22` (conditionsNotCorrect):**
→ The write may only be accepted inside the `0x0226` routine context. This
would mean the constellation can only be updated as part of the full CP
authentication flow — significantly harder to do offline.

---

## Experiment 3: IKA Blob Structure Validation

**What:** Test the hypothesis that bytes 0–15 are the AES key by using them
in a simulated challenge-response.  
**Why:** If we can confirm the AES key boundary, we understand what J533
actually checks during the per-ignition verification.  
**How:** Extract the AES key candidate (first 16 bytes of the J136 blob),
then observe a challenge-response between J533 and J136 on the CAN bus
during an ignition cycle using the passive sniffer.  
**Prerequisites:** Passive CAN sniffer (`J2534CANSniffer` or ESP32 bridge).

### What to Capture

During an ignition cycle, J533 sends a challenge to each enrolled module.
The module responds using its AES key. Capture the traffic on J136's
CAN IDs (`0x74C` / `0x7B6`) during startup. The challenge-response frames
will be ISO-TP wrapped UDS or proprietary CAN messages.

If you can capture a challenge and the corresponding response, and then
verify that `AES-128-ECB(key=bytes_0_15, plaintext=challenge) == response`,
the blob structure is confirmed.

---

## Experiment 4: MMI3G+ Runtime Key Extraction (qconn + GDB)

**What:** Extract the live AES key from MMI3GApplication's RAM via the root
shell you already have.  
**Why:** Gives you the key without parsing the blob format — the application
has already parsed it. Also answers VIN-bound vs module-bound from a
different direction (compare key in RAM to J136 blob bytes 0–15).  
**How:** Start qconn on the MMI3G+, attach cross-GDB from your PC, set
breakpoint on CP handler, trigger ignition cycle.  
**Prerequisites:** SH4 cross-compiled GDB, qconn running on the MMI3G+.

### Target

```
Process: MMI3GApplication (PID 204844)
Class:   CSPHComponentProtectionSA
Method:  RQST_AuthString (challenge handler)
         RPST_AuthString (response handler)

Breakpoint on RQST_AuthString entry.
When hit, dump registers and nearby stack/heap for the AES key.
The key is loaded from persistence (DataPST.db or IPC channel)
before the challenge is processed.
```

### SH4 Cross-GDB Setup

```bash
# You need a SuperH (SH4) cross-compiled GDB.
# QNX SDP includes one, or build from GDB source with:
#   --target=sh4-nto-qnx6.3.0
#
# On the MMI3G+:
qconn &    # Starts the QNX remote debug agent

# On your PC:
sh4-gdb
(gdb) target qnx 192.168.0.154:8000
(gdb) attach 204844
(gdb) info functions AuthString
# Find RQST_AuthString address
(gdb) break *0x????????
(gdb) continue
# Turn ignition off and on — breakpoint should hit
(gdb) info registers
(gdb) x/16bx $r4    # or whichever register holds the key pointer
```

---

## Experiment 5: FSC Bypass as CP Bypass Test (MMI3G+ Only)

**What:** Test whether the existing 5-site `EscRsa_DecryptSignature` patch
also disables CP token signature verification on the MMI head unit.  
**Why:** If CP token acceptance goes through the same RSA path as FSC, your
existing firmware patch lets the MMI accept unsigned/replayed IKA blobs.  
**How:** Install an MMI3G+ with the FSC-patched firmware into a car where CP
is active. Check whether CP still restricts audio.  
**Prerequisites:** MMI3G+ running patched K0942 or K0711 firmware.

### Important Caveat

Even if the FSC patch lets the MMI *accept* a replayed blob via
`WriteDataByIdentifier(0x00BE)`, it does NOT affect the AES-128
challenge-response that J533 performs every ignition cycle. The patch
helps with blob *acceptance*, not with ongoing *verification*.

For this to be useful, the blob being written must contain the correct AES
key — which brings us back to whether the IKA is VIN-bound (known key,
replay works) or module-bound (need derivation).

---

## The Replay Path (If Blobs Match)

If Experiment 1 shows all modules carry the same IKA blob, the fix is:

### Step 1 — Write IKA key to target module

```
Target: J255 at 0x746
Session: Extended (0x10 0x03)
Service: WriteDataByIdentifier (0x2E)
DID: 0x00BE
Payload: E6 2B 41 D1 1C 44 AF 20 21 77 FB 1F 27 4B 0A C2
         D1 5B D2 62 E4 FD 27 AB 61 D1 23 C2 F1 5A 2C 93
         26 00
```

### Step 2 — Update constellation bitmap on J533

```
Target: J533 at 0x710
Session: Extended (0x10 0x03)
Service: WriteDataByIdentifier (0x2E)
DID: 0x04A3
Payload: [current value with J255's slot bit set]

To determine J255's slot:
  1. Read DID 0x2A2A from J533
  2. Parse {ECU_ID, ECU_Name} pairs
  3. Find pair where ECU_Name == 8 (Air Conditioning)
  4. That pair's index is J255's slot number
  5. Set bit (slot_number % 8) in byte (slot_number // 8) of 0x04A3
```

### Step 3 — Verify

```
Read DID 0x00BE from J255 → should return the blob just written
Read DID 0x04A3 from J533 → should show J255's slot bit set
Clear DTCs on J255 (service 0x14)
Cycle ignition
Check: does J255 still report U110100?
```

If U110100 clears and stays cleared across ignition cycles, CP is resolved.
No server was involved.

---

## The Derivation Path (If Blobs Differ)

If Experiment 1 shows different blobs per module, we need to understand the
key derivation function. Multiple approaches, in order of feasibility:

### Approach A: D4 A8 Gateway Firmware Analysis

**The most promising near-term path.**

The D4 A8 (`4H0 907 468x`) gateway uses the same Lear D70F3433 MCU as the
C7 A6. The firmware is available locally. The CP verification routine is
compiled V850 assembly in the MCU flash image.

#### What to Look For

The verification function takes the 34-byte IKA blob and checks it. At
minimum it:

1. Extracts the AES key (bytes 0–15)
2. Computes an expected MAC over known inputs (VIN, FAZIT, serial, ...)
3. Compares computed MAC against bytes 16–31 of the blob
4. Returns pass/fail

The MAC computation uses a **master key** or **HMAC key** that is either
hardcoded in the firmware or stored in a protected flash region. This key
is the single secret that enables offline blob generation.

#### Analysis Steps

```
1. Load D4 gateway flash dump into Ghidra/IDA with V850 processor module
2. Search for AES-related constants:
   - AES S-box: 63 7C 77 7B F2 6B 6F C5 30 01 67 2B FE D7 AB 76
   - AES round constants: 01 02 04 08 10 20 40 80 1B 36
3. Find functions that reference AES constants → these are the crypto routines
4. Trace callers of the AES routines → find the CP verification wrapper
5. In the wrapper: identify where it reads the 34-byte blob, where it splits
   at byte 16, what inputs it feeds to the MAC computation
6. Extract the master/HMAC key from the firmware image
```

#### Cross-Platform Validation

If the D4 firmware yields a master key, test it against the known J136 blob:

```python
import hmac, hashlib

master_key = bytes.fromhex("...")  # extracted from firmware
ika_blob = bytes.fromhex("E62B41D11C44AF202177FB1F274B0AC2"
                          "D15BD262E4FD27AB61D123C2F15A2C93"
                          "2600")

aes_key = ika_blob[0:16]
mac_from_blob = ika_blob[16:32]

# Try HMAC-SHA256 truncated to 16 bytes
inputs = vin + j533_fazit + j136_fazit  # various orderings
computed = hmac.new(master_key, inputs, hashlib.sha256).digest()[:16]

if computed == mac_from_blob:
    print("DERIVATION FOUND")
```

Try multiple MAC algorithms (HMAC-SHA256, HMAC-SHA1, AES-CMAC) and input
orderings. The firmware analysis narrows the search space dramatically —
the code tells you which algorithm and which input order.

### Approach B: qconn Runtime Extraction (MMI3G+ Side)

See Experiment 4 above. If the AES key extracted from J136's RAM matches
bytes 0–15 of the known blob, the structure hypothesis is confirmed. If
keys from multiple modules differ, capture all of them and look for
patterns against the module FAZITs.

### Approach C: Constrained MAC Search (Two-Blob Comparison)

If we obtain a second unmasked IKA blob (from a different module on the
same car, or from the Touareg), we can diff the two:

```
Blob A (J136): E6 2B 41 D1 ... D1 5B D2 62 ... 26 00
Blob B (J285): ?? ?? ?? ?? ... ?? ?? ?? ?? ... ?? ??
                ^^^^^^^^^^^     ^^^^^^^^^^^
                AES key         MAC

If AES keys match → VIN-bound (no derivation needed)
If AES keys differ → module-bound derivation

If MACs differ but AES keys match → MAC includes module serial
If everything differs → fully module-specific derivation
```

Two blobs from the same VIN is the minimum needed for differential analysis.
Two blobs from *different* VINs (your A6 vs the Touareg) would additionally
reveal which parts of the blob are VIN-dependent.

### Approach D: GEKO Protocol Emulation (Last Resort)

Only pursue this if all other paths dead-end. A full GEKO emulator requires:

1. Understanding the ODIS → GEKO HTTPS request format
2. Understanding the GEKO → ODIS response format
3. Generating valid signed IKA blobs (requires the signing key)
4. Spoofing the `www.vwgeko.net` TLS certificate (MITM ODIS)
5. Handling mTLS client certificate verification

The signing key is the hard part. Without it, you can intercept ODIS's
request and see the plaintext inputs, but you can't generate a valid
response. The ODIS-under-Java-debugger approach (launch ODIS with
`-Xdebug`, watch the pre-TLS request assembly) gives you the request
format even without a working server — but the response generation
still requires the private key.

**The firmware RE path (Approach A) is strictly better** — it gives you
the verification algorithm and master key directly, from which you can
build a standalone key generator without emulating any network protocol.

---

## The Touareg 7P — What Transfers

Your buddy's bricked RNS-850 situation is architecturally parallel:

| Component | A6 C7 (4G) | Touareg 7P |
|-----------|------------|------------|
| Gateway | `4G0 907 468x` (Lear) | `7P6 907 530x` (Lear — confirm) |
| MCU | D70F3433 | D70F3433 (likely) |
| CP library | `ES_LIBCompoProteGen3V12` | Same (shared across MLB) |
| IKA DID | `0x00BE` (34 bytes) | `0x00BE` (34 bytes — confirm) |
| Constellation DID | `0x04A3` | `0x04A3` (confirm) |
| Head unit | MMI3G+ (K0942) | RNS-850 (K0711) |
| FSC patch | `0x001B11F6` | `0x001B151A` |
| CP routine ID | `0x0226` | `0x0226` (confirm) |

**Action items for the Touareg:**

1. Confirm gateway part number prefix (`7P6 907 530x` vs other)
2. Read DID `0x00BE` from the gateway and head unit — compare structure
3. If the blob structure matches the C7 format (34 bytes, same layout),
   all C7 research applies directly
4. The D4 A8 firmware analysis covers both platforms if they share the
   same D70F3433 verification routine

**Does your buddy still have the old (bricked) RNS-850?** If so, its EEPROM
still carries the enrolled CP data for his VIN — it's a data source even if
the unit doesn't boot.

---

## What's NOT Worth Trying

For the record, and to save future contributors time:

| Approach | Why it fails |
|----------|-------------|
| Brute-force AES-128 | 2^128 keyspace — physically impossible |
| Brute-force RSA-2048/4096 | Factoring problem — beyond current capability |
| Clearing DTCs to remove CP | Module re-reads its own EEPROM flag on every power cycle |
| EEPROM clone of J533 95320 | Only suppresses CP — MCU flash holds the real constellation |
| VCDS adaptation writes | Cannot write CP-protected channels without GEKO token |
| Reflashing module firmware | CP data persists in EEPROM through firmware flash |
| Resetting module to "virgin" with AVDI/FVDI | May permanently brick ODIS adaptability |
| Waiting for GEKO to come back | VW has moved to GRP and excluded third parties by design |

---

## Research Priority Order

| # | Experiment | Time | Depends On | Unlocks |
|---|-----------|------|------------|---------|
| 1 | Cross-module `0x00BE` read | 15 min | Car + Mongoose | Everything — determines replay vs derivation |
| 2 | Constellation `0x04A3` write test | 5 min | Exp 1 done | Confirms offline write path |
| 3 | D4 A8 gateway firmware analysis | Days | Ghidra + V850 plugin | Verification algorithm + master key |
| 4 | qconn + GDB on MMI3G+ | Hours | SH4 cross-GDB | Live AES key from RAM |
| 5 | FSC-patch-as-CP-bypass test | 30 min | Patched MMI3G+ in car | Whether RSA bypass helps blob acceptance |
| 6 | Touareg 7P parallel confirmation | 30 min | Buddy's car accessible | Second VIN for differential analysis |
| 7 | IKA blob structure validation | Hours | Passive CAN capture | Confirms AES key boundary |
| 8 | Constrained MAC analysis | Days | Two known blobs | Derivation function |
| 9 | GEKO protocol capture (via ODIS debugger) | Days | ODIS-S installed | Request format (response still needs key) |
| 10 | Full GEKO emulator | Months | Everything above | Complete offline server — last resort |

**Start with 1. Everything else waits.**

---

## Contributing

If you have:
- A VAG vehicle with active CP and the ability to read DIDs via UDS
- Gateway firmware dumps from any Lear-platform vehicle
- ODIS session logs (even partial/masked)
- Access to ABRITES VN017/VN020 and willingness to document the wire protocol

Your data advances this research. See `CONTRIBUTING.md` for submission guidelines.
Cross-VIN data is especially valuable — two IKA blobs from different vehicles
is the minimum needed for differential cryptanalysis of the derivation.

---

*Released under CC0 (public domain). Use freely for repair, research, and advocacy.*
