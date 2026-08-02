# Windows 11: Bluetooth headset mic missing — Intel SST audio-offload fix

Your Bluetooth headset plays music fine, but its **microphone doesn't show up** in
Teams/Zoom/Discord or in Settings → Sound → Input. Re-pairing doesn't help. Rebooting
doesn't help. The built-in laptop mic still works.

This repo documents the root cause and a working fix, found by debugging the issue on an
ASUS Zenbook Duo (2024, Intel Core Ultra) with a Sony WH-1000XM5 and a Bose QC45 on
Windows 11 Pro (26200). It likely applies to other Intel-based laptops that ship with
**Intel® Smart Sound Technology for Bluetooth® Audio**.

## TL;DR

The headset mic (Hands-Free profile / HFP) was routed through Intel's Bluetooth **audio
offload** path — a hardware side-channel into the Intel audio DSP — and that path was
dead. Playback (A2DP) uses a different route, which is why music kept working. The fix is
to turn off SCO offload on the Bluetooth adapter so mic audio uses Windows' standard
driver:

1. In `HKLM\SYSTEM\CurrentControlSet\Enum\USB\VID_8087&PID_XXXX\<instance>\Device Parameters`
   (your Intel Bluetooth adapter's device-parameters key), set the DWORD
   **`Sco Support Type` = `0`** (it will be `2` when offload is on).
2. **Restart** the PC (Restart, not Shut down — fast startup skips full driver re-init).
   Do **not** disable/re-enable the adapter to apply it; see [Gotchas](#gotchas).
3. **Remove and re-pair** the headset so its Hands-Free device is rebuilt on the new path.

After this, the `Headset (…)` recording endpoint activates and apps can see the mic.

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
- The Core Audio capture endpoint `Headset (<name> Hands-Free)` exists but reports
  **state 4 = DEVICE_STATE_NOTPRESENT** (see [DIAGNOSIS.md](DIAGNOSIS.md) for how to
  query this). Windows Settings may still list it, greyed out — apps like Teams only
  show *active* endpoints, so it's invisible there.

## Root cause

Two layers:

1. **Trigger:** the Intel SST Bluetooth audio devices were disabled in Device Manager
   (problem code 22 — a deliberate disable; commonly recommended online to stop Bluetooth
   audio quality drops during calls, and some OEM utilities do it too). With the adapter
   still configured for offload, Windows has **no fallback** — every pairing rebuilds the
   Hands-Free device wired to the dead DSP path.
2. **Underlying:** even after re-enabling those devices, the offload pipeline never
   activated the mic endpoint again on this driver combination (Intel BT 24.50.0.4 +
   SST 20.40.12061.0). The offload path was broken outright, so the only way back was
   routing around it.

Bonus quirk: the offload switch (`Sco Support Type`) is written by INF files during
device *configuration*. Microsoft's in-box `bth.inf` writes `2` (offload) while Intel's
own `ibtusb` INF writes `0` — and `bth.inf`'s value wins on configuration passes. That's
presumably how machines end up in offload mode, and it's why the registry fix gets
**silently reverted** by driver updates or reinstalls.

On the machine investigated, the breakage was traced (MsiInstaller + Kernel-PnP +
`setupapi.dev.log` events) to an **Intel Wireless Bluetooth 24.50.0.4 MSI driver update
installed minutes before the mic disappeared** — its configuration pass flipped the
routing onto the offload path. Full forensic timeline in [NOTES.md](NOTES.md).

## Gotchas

- **Don't apply the registry change by disabling/re-enabling the Bluetooth adapter.**
  That triggers a device configuration pass, which re-applies `bth.inf`'s
  `Sco Support Type = 2` — undoing your change. It can also leave the adapter stuck
  disabled (no Bluetooth at all) if the re-enable fails; `pnputil /enable-device <id>`
  from an elevated prompt recovers it. Just set the value and reboot.
- **Driver updates can revert the fix.** If the headset mic vanishes again after a
  Bluetooth or Windows update, check `Sco Support Type` first.
- Each **already-paired** headset stays on the old dead path until you remove and
  re-pair it — the profile devices are built at pairing time.
- Rollback: set `Sco Support Type` back to `2` and restart.

## Repo contents

- [DIAGNOSIS.md](DIAGNOSIS.md) — the PowerShell commands used to pin this down, with
  what each result means (usable as a step-by-step diagnostic on another machine).
- [NOTES.md](NOTES.md) — raw session notes: timeline, dead ends, and exact machine state.

## Environment where this was confirmed

| Component | Version |
|---|---|
| Windows 11 Pro | build 26200 |
| Laptop | ASUS Zenbook Duo (Intel Core Ultra, UX8406) |
| Intel Wireless Bluetooth driver (`ibtusb`) | 24.50.0.4 (2026-05-08) |
| Intel SST for Bluetooth Audio | 20.40.12061.0 (2025-04-09) |
| Headsets | Sony WH-1000XM5, Bose QC45 |
