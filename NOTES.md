# Session notes — 2026-09-12

Mic gone again on the same machine (Sony WH-1000XM5 connected, Bose QC45 paired but
off), five weeks after the 2026-08-01 fix. Same symptom: playback fine, no `Headset`
endpoint. State found before touching anything: `Sco Support Type = 2`, both
`Hands-Free AG` devices on `HCIBYPASS`, no active `Headset` capture endpoint, both SST
Bluetooth Audio devices `OK`, Intel BT driver still 24.50.0.4 (`oem194.inf`), fast
startup on.

## What happened, in order

1. 07:5x — set `Sco Support Type = 0` (elevated), adapter untouched.
2. 08:06:26 — user Restarted. Kernel start 08:06:44.
3. **08:06:57 — the adapter's `Device Parameters` key was written** (registry
   last-write timestamp, `RegQueryInfoKey`). Value afterwards: **`2`**.
4. 08:07:06 — Kernel-PnP configured two new audio endpoints; the capture one is
   `Headset (WH-1000XM5)` on the *Intel SST for Bluetooth Audio* interface, **state 1**.
5. Mic works in apps. **No re-pair** — the XM5's `Hands-Free AG` is the same `HCIBYPASS`
   device as before the reboot.

So: the value did not survive the reboot (contradicting the 8/1 note below), the mic
came back anyway, and it came back on the offload path.

## What reverted the value (this is the part the 8/1 notes got wrong)

Not INF ordering, not a driver update:

- **No install or reconfiguration since 8/1.** Kernel-PnP/Configuration has no event
  for the adapter after 2026-08-01 09:22; no UserPnp driver-install events; Windows
  Update history since 7/31 is Defender definitions plus one "ASUSTeK System Driver
  Update 3.1.70.0" on 8/30. `setupapi.dev.log` and its four rotated predecessors: the
  last adapter install is the 8/1 09:22 one. Driver store has no `ibtusb` package newer
  than 8/1.
- **INFs.** All ten Intel `ibtusb` INFs in `C:\Windows\INF` (23.90.0.8, 24.20.0.3,
  24.40.10.3 ×5, 24.50.0.4 ×3) set `Sco Support Type = 0` in `[AudOffload.HW.AddReg]`.
  In-box `bth.inf` (10.0.26100.9444) sets `2` in `[BthPort_SidebandSco.NT.HW.AddReg]`,
  referenced only from `[BthUsb_SidebandSco.NT.HW]`. Intel's `[ibtusb.HW]` does
  `Include=bth.inf` / `Needs=BthUsb.NT.HW` — the *non*-sideband section. INF processing
  cannot produce a `2` here.
- **Who has the string.** UTF-16 `Sco Support Type` occurs in `ibtusb.sys` (every
  flavour in `C:\Program Files (x86)\Intel\Bluetooth\drivers\ibtusb\*` and every driver
  store copy), `ibtpci.sys`, and Microsoft's `bthport.sys`. It does **not** occur in
  `IntcBTAu.sys` (SST BT audio) or in any Intel service binary (`ICPS\*`,
  IntelAudioService, etc.). Adjacent strings in `ibtusb.sys`: `HfpOffloadDisable`,
  `A2dpOffloadDisable`.
- **Disassembly of `ibtusb.sys` 24.50.0.4** (`objdump`, WDF call table resolved by
  index; addresses for image base `0x140000000`):
  - `EvtDriverDeviceAdd = 0x140060BE0` (fed to `WDF_DRIVER_CONFIG_INIT` in DriverEntry).
  - `0x14006B510`: `WdfDeviceOpenRegistryKey(dev, PLUGPLAY_REGKEY_DEVICE=1, KEY_READ)` →
    `WdfRegistryQueryValue(L"HfpOffloadDisable")`, success iff DWORD == 1. That is the
    **Device Parameters** key.
  - `0x140071800(dev, value)`: opens the same key, `WdfRegistryAssignULong(L"Sco Support
    Type", value)`. Three callers, all under DeviceAdd:
    - `0x1400616B3` writes **0** when `HfpOffloadDisable == 1`;
    - `0x14006CD83` / `0x14007B075` write **2** when the UEFI variable `UefiCnvBtAOLD`
      yields 1 or 3 (platform audio offload enabled), else 0.
  - No path through `InstallOrUpgrade` or the `CurrentDriverVersion` /
    `PreviousDriverVersion` values; the write happens on every device add.
  - `A2dpOffloadDisable` (`0x14006A170`) is read the same way and only logged.

Conclusion: on this platform `ibtusb.sys` writes `2` on every boot and on every adapter
enable (which is exactly what the 8/1 "pending configuration pass" was). The `0` we write
is consumed once, at the next device add, and the observable effect is that the offload
path comes up properly on that boot. Mechanism for *that* not traced.

## Other things checked

- Intel community threads about AX211 (24.20.0.3 killing all BT audio; rollback to
  24.10.0.4) describe a different failure. Downgrading is pointless here: the device-add
  logic is in every version in the store.
- Windows boots since 8/1 (Kernel-General 12 / EventLog 6005): 8/1 ×4, 8/12 ×2,
  9/9 02:51 + 02:52 (update install; `setupapi.dev.log` rotated 02:49), 9/9 19:02,
  9/12 08:06. `BTHUSB` event 18 fires at every start/resume and is a handy marker.
  Which of those boots killed the offload path is unknown — nothing logs it.
- A scheduled-task watchdog to keep the value at `0` was drafted and then dropped:
  the driver rewrites the value on every boot, so it would fight the wrong thing.

## Permanent option (not yet exercised here)

`HfpOffloadDisable = 1` (DWORD, adapter Device Parameters) → driver writes
`Sco Support Type = 0` itself at every device add → HFP over HCI, SST offload out of the
loop. Needs a Restart and a remove + re-pair of each headset (the `Hands-Free AG` device
should then be created without `HCIBYPASS`). Reversible by deleting the value. Left
unapplied on 9/12 because the quick fix had just restored the mic; apply next time it
dies, or sooner.

## Corrections to the 2026-08-01 notes below

- "The value survived the reboot at 0" — it does not; see above. Success should be
  judged by the endpoint state, not the value.
- "Each already-paired headset stays on the old dead path until re-paired" — false for
  the quick fix; the existing `HCIBYPASS` device came back to life. Re-pair is only
  needed after `HfpOffloadDisable`.
- "`bth.inf`'s `2` wins over Intel's `0` during configuration" — false; INFs are not
  involved at runtime. Open item resolved.
- "Driver updates may revert `Sco Support Type` to 2" — every boot does.

---

# Session notes — 2026-08-01

Raw timeline of the debugging session, including dead ends. Machine: ASUS Zenbook Duo
(UX8406), Windows 11 Pro 26200, Intel Wireless Bluetooth `USB\VID_8087&PID_0037`
(driver 24.50.0.4), Intel SST platform audio. Headsets: Sony WH-1000XM5, Bose QC45.

## What broke it (traced after the fix)

Initially the Intel Bluetooth 24.50.0.4 MSI installed at 09:21 the same morning
looked like the culprit — the user then clarified it was **their own fix attempt**
(as was disabling-then-not-re-enabling assorted devices that morning: a VSS shadow
copy of the registry from the previous night showed the SST devices still enabled
and `Sco Support Type` already `2`).

The real history, reconstructed from the Kernel-PnP/Configuration log, VSS shadow
copies of the SYSTEM/SOFTWARE hives, and Windows' ~30-day ghost-device cleanup
semantics:

- **1/31/2026** — Windows 11 feature update to build 26200
  (`Win32_OperatingSystem.InstallDate`).
- **2/20–21** — Intel SST / Bluetooth audio driver refresh right after (PnP log
  begins 2/20; SST BT devices configured+started 2/21 with "settings not migrated
  from previous OS installation"). One pre-existing Headset endpoint from before
  this date never logged another event — dead relic.
- **3/3** — WH-1000XM5 paired; its Hands-Free endpoint devnode created. No
  evidence it was ever active afterwards.
- **4/13** — Windows' periodic stale-device cleanup deleted that endpoint. The
  cleanup only reaps devices **not present ≥ ~30 days** → the mic endpoint had
  been continuously dead since **mid-March at the latest**, likely since the 3/3
  pairing.
- **4/1, 4/30, 7/11** — user re-pairs (QC45, XM5×2); each endpoint recreated dead
  (state 4). **5/18, 7/8** — further ghost-cleanups reap them again (the 7/8 pass
  also deleted a phone, USB sticks, and a portable display in the same second —
  clearly bulk phantom cleanup, not a targeted event).
- **7/31 (VSS snapshot)** — all three Headset endpoints state 4; SST devices
  enabled; `Sco Support Type = 2`. Broken-at-rest state, pre any fix attempts.
- **8/1** — user's fix attempts (driver MSI install 09:21, device disabling),
  then the actual fix (`Sco Support Type = 0` + restart + re-pair).

**Conclusion:** the offload path never worked again after the late-January feature
update / late-February Intel audio-driver refresh. Everything in between was fix
attempts and Windows housekeeping on top of a path that had been dead since
roughly February–March. Exact triggering event (OS update vs. the SST driver
refresh that followed it) is not distinguishable from surviving logs — the PnP log
only reaches back to 2/20 and Windows Update history had rolled over.

## Timeline

1. **Symptom**: headset mic gone from all apps; playback fine. Both paired headsets
   affected.
2. Found both `Intel® Smart Sound Technology for Bluetooth® Audio` and
   `…for Bluetooth® LE Audio` MEDIA devices at **Error, problem code 22**
   (= `CM_PROB_DISABLED`, i.e. disabled in Device Manager, cause unknown — user tweak
   or OEM utility). All three `Headset` capture endpoints at state 4 (NOTPRESENT).
   Hands-Free AG devices present and OK, instance IDs contain `HCIBYPASS` (Intel
   offload / sideband path).
3. **Re-enabled** both SST BT devices (elevated `Enable-PnpDevice`) → drivers OK,
   problem 0. Endpoints stayed state 4. ❌
4. Restarted the Hands-Free AG devices, cycled the XM5's Bluetooth connection,
   restarted `AudioEndpointBuilder`/`Audiosrv` → still state 4. ❌
5. Full reboot (later realized the machine had already rebooted with drivers healthy) →
   still state 4. ❌ Conclusion: offload pipeline itself broken, not stale state.
6. **Removed + re-paired the XM5** → Hands-Free AG rebuilt *again with `HCIBYPASS`* →
   still state 4. ❌ Conclusion: adapter-level setting forces the offload path at pair
   time.
7. Found `Sco Support Type = 2` under the adapter's `Device Parameters`, while Intel's
   own INF (`oem194.inf`, `[AudOffload.HW.AddReg]`) defaults it to **0**. In-box
   `bth.inf` line ~535 sets **2**.
8. Set it to 0, tried to restart the adapter to apply:
   - `Disable-PnpDevice` on the adapter → "Not supported", left the adapter **stuck
     disabled** → total Bluetooth outage (user noticed). Recovered with
     `pnputil /enable-device`.
   - The enable ran a pending configuration pass which **reverted the value to 2**.
9. Set `Sco Support Type = 0` again, this time touched nothing else, user did a
   normal **Restart** + **re-pair** → `Headset (WH-1000XM5)` endpoint went **state 1**.
   ✅ Mic visible in Teams.

## Outcome

Fixed and verified the same day (2026-08-01):

- `Sco Support Type = 0` written with the adapter left untouched, full Restart,
  XM5 removed and re-paired.
- `Headset (WH-1000XM5)` capture endpoint went **state 1 (active)**; mic visible
  and working in Teams. The value survived the reboot at 0. *(2026-09-12: this
  was wrong or a lucky read — the driver rewrites it at every boot; see top of file.)*
- The registry flip only sticks if nothing triggers a device reconfiguration
  before the reboot — the first attempt was reverted (`bth.inf` re-applied 2)
  because a pending configuration completed during an adapter enable. Second
  attempt (write value, touch nothing, reboot) held.
- Collateral incident during the first attempt: `Disable-PnpDevice` on the
  adapter failed mid-cycle and left it stuck disabled → total Bluetooth outage,
  recovered with elevated `pnputil /enable-device`. Lesson folded into the
  "don't restart the adapter to apply this" guidance.
- Old `Headset (WH-1000XM5 Hands-Free)` ghost endpoint remains, harmless. QC45
  left un-re-paired (playback-only) by choice.

## Dead ends (don't bother)

- Restarting audio services / endpoint builder — endpoint stays NOTPRESENT.
- Power-cycling or re-pairing the headset while `Sco Support Type = 2` — the
  Hands-Free device is rebuilt on the same dead offload path every time.
- Waiting for the offload pipeline to "come back" after re-enabling the SST devices —
  it survives reboots in the registered-but-dead state indefinitely.

## Open items

- **Bose QC45** still has its old `HCIBYPASS` Hands-Free device; needs remove +
  re-pair if its mic is ever needed. Playback unaffected.
- The old dead endpoint `Headset (WH-1000XM5 Hands-Free)` lingers at state 4 as a
  harmless ghost entry; the live one is `Headset (WH-1000XM5)`.
- ~~Unverified hypothesis: which INF ordering rule makes `bth.inf`'s `2` win over
  Intel's `0` during configuration.~~ Resolved 2026-09-12: no INF rule; `ibtusb.sys`
  writes it on every device add.
- ~~Watch item: next Intel BT driver update may revert `Sco Support Type` to 2.~~ Every
  boot does. Watch the endpoint state instead.

## Live endpoint enumeration snippet

The registry MMDevices view can lag; this queries the Core Audio API directly
(PowerShell 5.1 `Add-Type`):

```powershell
Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;
[ComImport, Guid("BCDE0395-E52F-467C-8E3D-C4579291692E")] class MMDeviceEnumeratorComObject { }
[Guid("A95664D2-9614-4F35-A746-DE8DB63617E6"), InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
interface IMMDeviceEnumerator { int EnumAudioEndpoints(int dataFlow, int stateMask, out IMMDeviceCollection devices); int GetDefaultAudioEndpoint(int dataFlow, int role, out IMMDevice device); }
[Guid("0BD7A1BE-7A1A-44DB-8397-CC5392387B5E"), InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
interface IMMDeviceCollection { int GetCount(out int count); int Item(int index, out IMMDevice device); }
[Guid("D666063F-1587-4E43-81F1-B948E807363F"), InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
interface IMMDevice { int Activate(); int OpenPropertyStore(int access, out IPropertyStore props); int GetId([MarshalAs(UnmanagedType.LPWStr)] out string id); int GetState(out int state); }
[Guid("886d8eeb-8cf2-4446-8d02-cdba1dbdcf99"), InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
interface IPropertyStore { int GetCount(out int count); int GetAt(int index, out PropertyKey key); int GetValue(ref PropertyKey key, out PropVariant value); }
[StructLayout(LayoutKind.Sequential)] struct PropertyKey { public Guid fmtid; public int pid; public PropertyKey(Guid f, int p){fmtid=f;pid=p;} }
[StructLayout(LayoutKind.Explicit)] struct PropVariant { [FieldOffset(0)] public short vt; [FieldOffset(8)] public IntPtr pointerValue; }
public class AudioDevices {
    public static void ListCapture() {
        var en = (IMMDeviceEnumerator)(object)new MMDeviceEnumeratorComObject();
        IMMDeviceCollection col; en.EnumAudioEndpoints(1, 15, out col); // 1 = capture, 15 = all states
        int n; col.GetCount(out n);
        var nameKey = new PropertyKey(new Guid("a45c254e-df1c-4efd-8020-67d146a850e0"), 14);
        for (int i = 0; i < n; i++) {
            IMMDevice d; col.Item(i, out d);
            int state; d.GetState(out state);
            IPropertyStore ps; d.OpenPropertyStore(0, out ps);
            PropVariant v; ps.GetValue(ref nameKey, out v);
            string name = v.vt == 31 ? Marshal.PtrToStringUni(v.pointerValue) : "?";
            Console.WriteLine("state={0}  {1}", state, name);
        }
    }
}
'@
[AudioDevices]::ListCapture()
```
