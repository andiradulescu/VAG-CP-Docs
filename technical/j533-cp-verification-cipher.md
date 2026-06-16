# J533 Gateway CP Verification — Cipher and Write‑Path, Reverse‑Engineered

**Status:** Confirmed from firmware (Renesas V850, `gw_fw_main.bin`) and validated live on a
C7 A6 gateway over OBD‑II. June 2026.

This document records what the J533 LEAR D4 gateway *actually* does to verify Component
Protection, established by reverse‑engineering the gateway's own firmware and confirming
each finding against the live vehicle. It is published because the result is directly
relevant to the right‑to‑repair argument: **the layer of Component Protection that blocks
a repair is a fixed‑key check over a public constant — it carries no per‑vehicle secret and
provides no anti‑theft value.**

---

## Summary

CP on this platform is **two independent layers**, and they are routinely conflated:

| Layer | What it is | Security value | Reproducible offline? |
|---|---|---|---|
| **Enrollment** (gateway install‑list) | The gateway's record of which components are "present and enrolled." Verified by a **standard AES‑128 known‑answer test against a fixed, public constant**. | **None** — no per‑vehicle key, no nonce, no challenge. | **Yes.** Fully determined by values printed in the firmware. |
| **Per‑module credential** (IKA key) | The 34‑byte installation key written to each module, historically issued by VW's GEKO server. | The actual cryptographic binding. | No — still GEKO‑derived in our analysis. |

The first layer is the one that produces the "component protection active" diagnostic state
at the gateway. It has **no cryptographic strength.** The second layer is a real credential,
but it is what an authorized tool (or the dealer) writes — it is not a theft deterrent in any
meaningful sense, because the parts in a CP case come from an identical donor owned by the
same person.

---

## The cipher

The gateway's CP verification routine is **textbook AES‑128 in ECB**, implemented in
software (forward S‑box, inverse S‑box, Rcon, and the GF(2⁸) `xtime` table are all the
standard FIPS‑197 constants, byte‑for‑byte). It is **not** a proprietary or per‑vehicle
cipher.

- **Key:** a **fixed ASCII string compiled into the gateway firmware** — identical across
  every gateway of this software build. It carries zero per‑vehicle entropy.
- **Power‑on self‑test:** the firmware decrypts a blank (`0xFF…`) block with that key and
  compares the result to a stored constant — a routine check that the AES engine works. This
  was previously mistaken for a key‑validation step; it is not.

A known‑answer test using standard AES‑128 and the firmware's fixed key reproduces the
stored constant **byte‑for‑byte**, which is what confirms the cipher is standard AES and the
key is the fixed string.

> An earlier analysis pass claimed a "custom cipher" and a per‑vehicle flash‑constant key.
> That was a tooling artifact (a disassembler base‑address offset error that read unrelated
> bytes). Corrected and verified two independent ways: a direct AES known‑answer test, and a
> separate instruction‑level emulation of the firmware routine. Both agree.

---

## The enrollment forge (install‑list)

The gateway stores each enrolled component as an AES‑encrypted record. The verification the
gateway performs on these records reduces to:

```
record_plaintext == AES‑DECRYPT(stored_record, fixed_key)  must equal a FIXED PUBLIC SENTINEL
```

The sentinel is a trivial 16‑byte palindrome constant in the firmware. The learn/re‑pair
operation accepts a candidate record if and only if it matches that same sentinel, then
stores its AES‑encryption. **No per‑vehicle data, no nonce, no module challenge is involved.**

The consequence: the record that marks *any* slot "valid" is fully computable from public
firmware values. This is the gateway's "enrollment view" — and it is forgeable with no key
recovery whatsoever. (This is, in effect, what commercial CP tools do for this layer.)

---

## What is *not* broken by the above

The per‑module **IKA credential** (UDS DID `0x00BE`, 34 bytes) is a separate thing. In the
captured 2024 dealer session, those blobs were **GEKO‑derived** from VIN + module FAZIT codes
+ key‑fob transponder data + VW's signing key. Our firmware RE did **not** recover that
derivation. Writing a stale or forged IKA blob to the gateway is *accepted* by the write
channel but does **not** re‑pair the module — confirmed live (see below).

So a module's *local* function (e.g. the HVAC dropping to defrost‑only, seat memory disabled)
is gated by this credential layer, not the enrollment layer. Restoring it offline requires
either the GEKO‑derived blob or a **per‑module firmware patch** of the module's own CP gate.

---

## Live confirmation (OBD‑II, C7 A6 gateway `710/77A`)

Every finding below was read or written on the live vehicle; nothing was damaged.

- The **entire gateway install‑list is readable with no SecurityAccess**: allocation
  (`0x2A2A`), per‑component status `{0..3}` (`0x2A29`), slot→CAN‑ID map (`0x2A2C`), fault
  bitmap (`0x2A27`), constellation (`0x04A3`).
- **`WriteDataByIdentifier` to CP DIDs (`0x00BE`, `0x04A3`) is accepted in the extended
  session with no SecurityAccess** (`6E…` positive response). The gateway's own SA2 (level
  `0x03`) is not even required for the CP write surface.
- The per‑component **status DID (`0x2A29`) is read‑only** (`7F…31`) — it is the *output* of
  the gateway's self‑scan, not a writable input.
- Writing a real (2024) IKA blob via `0x00BE` was **accepted but inert** — no slot changed,
  and the auth‑failure counter did **not** increment. This confirms the `0x00BE` path is the
  credential layer and is *separate* from the enrollment‑sentinel check.
- The CP‑authentication routine (`RoutineControl 0x31 01 0x0226`) requires an options payload;
  a bare call returns `7F…13` (invalid format).
- The HVAC module occupies install‑list ECU‑Name code `8` (Air Conditioning) and read
  **Valid** at the gateway even while limping — i.e. its limp is enforced **locally**, not by
  the gateway.

---

## Why this matters for advocacy

1. **The enrollment layer has no security.** It is a fixed‑key check against a public
   constant. Anyone with the (freely dumpable) gateway firmware can compute the "valid"
   record. A scheme with no per‑vehicle secret cannot, by definition, be a theft deterrent —
   yet it is sufficient to flag a donor‑sourced part and degrade its function.

2. **The credential layer is gated on a server, not on security.** The IKA blob is whatever
   VW's GEKO server signs. With the server access withheld from third parties by policy, the
   *only* barrier to a legitimate owner repairing their own car with identical OEM parts is an
   authentication gate that has nothing to do with theft.

3. **The only DIY recourse is silicon‑level.** Because the enrollment forge does not restore a
   module's local function, an owner's offline options reduce to patching each module's own
   firmware on the bench — a high‑skill, hardware‑level intervention to undo an
   anti‑repair lock that provides no security benefit.

---

*Released under CC0 (public domain). Findings are from reverse‑engineering hardware and
firmware owned by the researcher, on the researcher's own vehicle.*
