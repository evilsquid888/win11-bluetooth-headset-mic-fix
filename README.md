# Windows 11: Bluetooth headset mic missing — Intel SST audio-offload fix

Your Bluetooth headset plays music fine, but its **microphone doesn't show up** in
Teams/Zoom/Discord or in Settings → Sound → Input. Re-pairing doesn't help. Rebooting
doesn't help. The built-in laptop mic still works.

This repo documents the root cause and a working fix, found by debugging the issue on an
ASUS Zenbook Duo (2024, Intel Core Ultra) with a Sony WH-1000XM5 and a Bose QC45 on
Windows 11 Pro (26200). It likely applies to other Intel-based laptops that ship with
**Intel® Smart Sound Technology for Bluetooth® Audio**.

> **Update 2026-09-12.** The mic died again five weeks after the first fix. A second
> investigation (registry timestamps, INF tracing, and a disassembly of Intel's
> `ibtusb.sys`) corrected the mechanism: the Intel driver **rewrites `Sco Support Type`
> on every boot**, so the value you set does *not* survive a restart, and re-pairing is
> *not* required. The quick fix still works; there is now also a **permanent switch**
> (`HfpOffloadDisable`). Details below and in [NOTES.md](NOTES.md#session-notes--2026-09-12).

## TL;DR

The headset mic (Hands-Free profile / HFP) is routed through Intel's Bluetooth **audio
offload** path — a hardware side-channel into the Intel audio DSP — and that path dies
silently on some boots (notably after Windows updates). Playback (A2DP) uses a different
route, which is why music keeps working.

**Quick fix (gets the mic back, keeps offload):**

1. In `HKLM\SYSTEM\CurrentControlSet\Enum\USB\VID_8087&PID_XXXX\<instance>\Device Parameters`
   (your Intel Bluetooth adapter's device-parameters key), set the DWORD
   **`Sco Support Type` = `0`**. It will read `2`.
2. **Restart** the PC (Restart, not Shut down — fast startup skips driver re-init).
   Do **not** disable/re-enable the adapter to apply it; see [Gotchas](#gotchas).

That's it. After the restart the value is back at `2` (the driver puts it there), the
`Headset (…)` recording endpoint activates, and apps see the mic. Confirmed twice on the
machine below (2026-08-01 and 2026-09-12). Why writing `0` makes the driver re-provision
the offload path on the next boot is not established; a plain restart without the
write did not do it.

**Permanent fix (turns offload off for good):**

1. On the same key, set the DWORD **`HfpOffloadDisable` = `1`**.
2. **Restart**.
3. **Remove and re-pair** each headset so its Hands-Free device is rebuilt on the
   standard Windows path (its instance ID no longer contains `HCIBYPASS`).

With `HfpOffloadDisable = 1` the Intel driver itself writes `Sco Support Type = 0` at
every device start, so nothing has to be maintained, and the DSP pipeline that keeps
dying is no longer involved. Mic audio goes over the normal Bluetooth HCI path like on
any laptop without Intel offload. Rollback: delete `HfpOffloadDisable`, restart, re-pair.
(As of 2026-09-12 this is verified from the driver's code, not yet exercised on the
machine below — the quick fix had just brought the mic back.)

## Symptoms checklist

- Headset audio (music) works; mic absent everywhere.
- In Device Manager (show hidden devices), under System devices there is a
  `<Headset> Hands-Free AG` device whose instance ID contains **`HCIBYPASS`** — the
  telltale sign the offload path is in use:

  ```
  BTHENUM\{0000111E-...}_HCIBYPASS_VID&...
  ```

- `Sound, video and game controllers` contains **Intel® Smart Sound Technology for
  Bluetooth® Audio** (and a `…for Bluetooth® LE Audio` sibling). If those are disabled
  (error code 22), that's what killed the mic — but merely re-enabling them may not bring
  it back (it didn't here; the offload pipeline never re-activated the endpoint, even
  across reboots and fresh pairings).
- The Core Audio capture endpoint `Headset (<name>)` exists but reports
  **state 4 = DEVICE_STATE_NOTPRESENT** (see [DIAGNOSIS.md](DIAGNOSIS.md) for how to
  query this). Windows Settings may still list it, greyed out — apps like Teams only
  show *active* endpoints, so it's invisible there.

## Root cause

**The offload pipeline.** HFP audio is handed to the Intel SST Bluetooth Audio driver
(`IntcBTAu.sys`) instead of Windows' `bthhfenum`/`bthhfaud` path. When that pipeline
fails to come up at boot, the `Headset (…)` endpoint is registered but never activated
(state 4), and nothing in Windows falls back. On the machine investigated it was dead
from a Windows 11 feature update (build 26200, end of January 2026) until 2026-08-01,
alive after that, and dead again by 2026-09-12 after the September update reboots. Which
boots kill it is not understood.

**The switch, and who owns it.** `Sco Support Type` on the adapter's Device Parameters
key tells Microsoft's `bthport.sys` whether SCO goes over HCI (`0`) or a sideband/offload
link (`2`). It is **not** set by INF ordering, as this repo originally guessed:

- Every Intel `ibtusb` INF in the driver store (23.90 → 24.50) writes `0`
  (`[AudOffload.HW.AddReg]`). Microsoft's in-box `bth.inf` writes `2` only in its
  `[BthUsb_SidebandSco.NT.HW]` section, which Intel's INF never references.
- The value is written at runtime by Intel's **`ibtusb.sys`**, in its `EvtDriverDeviceAdd`
  routine, i.e. on **every boot** and on every adapter enable — 13 seconds into the boot
  on 2026-09-12. The logic (from disassembly of 24.50.0.4):
  - read DWORD `HfpOffloadDisable` from the device hardware key (Device Parameters);
    if it is `1` → write `Sco Support Type = 0`;
  - otherwise write `2` if the UEFI variable `UefiCnvBtAOLD` (the platform's Bluetooth
    audio-offload setting) reports offload enabled, else `0`.
  - There is no install-time or driver-version gating. A sibling `A2dpOffloadDisable`
    is read and only logged.
- No Intel user-mode service touches the value; `bthport.sys` only reads it.

So on an offload-enabled platform the value is `2` after every boot no matter what you
set, and "driver updates revert the fix" was a misreading — the driver reverts it
every time. What the quick fix actually does is make the driver re-provision the offload
path at the next boot.

## Gotchas

- **Don't apply the registry change by disabling/re-enabling the Bluetooth adapter.**
  That runs the driver's device-add path immediately (value back to `2` before your
  reboot), and it can leave the adapter stuck disabled (no Bluetooth at all) if the
  re-enable fails; `pnputil /enable-device <id>` from an elevated prompt recovers it.
  Just set the value and Restart.
- **Expect `Sco Support Type` to read `2` again after the restart.** That is the driver,
  not a failed fix. Judge success by the `Headset (…)` endpoint state, not the value.
- **The mic can die again on a later boot** while offload is in use. Either repeat the
  quick fix or apply the permanent `HfpOffloadDisable` switch.
- **Re-pairing** is only needed after the permanent switch (to rebuild the Hands-Free
  device off the `HCIBYPASS` path). It was not needed for the quick fix.
- Downgrading the Intel Bluetooth driver does not help: every version in the store has
  the same device-add logic.

## Repo contents

- [DIAGNOSIS.md](DIAGNOSIS.md) — the PowerShell commands used to pin this down, with
  what each result means (usable as a step-by-step diagnostic on another machine).
- [NOTES.md](NOTES.md) — raw session notes: timelines, dead ends, exact machine state,
  and the 2026-09-12 re-investigation including the driver disassembly.

## Environment where this was confirmed

| Component | Version |
|---|---|
| Windows 11 Pro | build 26200 |
| Laptop | ASUS Zenbook Duo (Intel Core Ultra, UX8406) |
| Intel Wireless Bluetooth driver (`ibtusb`) | 24.50.0.4 (2026-05-08) |
| Intel SST for Bluetooth Audio | 20.40.12061.0 (2025-04-09) |
| Headsets | Sony WH-1000XM5, Bose QC45 |
