# Diagnosing a missing Bluetooth headset mic, step by step

All commands are PowerShell (5.1). Run as a normal user unless marked **elevated**.
Each step says what a healthy vs. broken result looks like.

## 1. Is the Hands-Free profile enumerated at all?

```powershell
Get-PnpDevice | Where-Object { $_.InstanceId -match '0000111E' } |
  Format-Table Status, FriendlyName, InstanceId -AutoSize
```

`0000111E` is the Bluetooth Hands-Free Profile service UUID.

- **No results** → the Hands-Free service isn't enabled for the device. Check the
  device's Services (Control Panel → Devices and Printers → device Properties →
  Services → "Handsfree Telephony").
- **Result with `HCIBYPASS` in the instance ID** → mic audio is routed through the
  vendor audio-offload path (Intel SST on Intel platforms), not Windows' standard
  driver. If the mic is missing, this path is the suspect.

## 2. Is the offload audio driver alive?

```powershell
Get-PnpDevice -Class MEDIA | Where-Object FriendlyName -like '*Bluetooth*' |
  ForEach-Object {
    $pc = (Get-PnpDeviceProperty -InstanceId $_.InstanceId -KeyName DEVPKEY_Device_ProblemCode).Data
    "$($_.Status) problem=$pc  $($_.FriendlyName)"
  }
```

- **`Error problem=22`** → the device is *disabled* in Device Manager. This is what
  removes the mic endpoints in the first place. (Re-enable with `Enable-PnpDevice`
  **elevated** — but note that on the affected machine this alone did not resurrect
  the mic.)
- `OK problem=0` with the mic still missing → the offload pipeline is up but not
  activating the endpoint; continue below.

## 3. What does the audio stack actually see? (ground truth)

Registry endpoint cache — quick but can lag reality:

```powershell
Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio\Capture\*' |
  ForEach-Object {
    $props = Get-ItemProperty "$($_.PSPath)\Properties" -ErrorAction SilentlyContinue
    [PSCustomObject]@{
      State = (Get-ItemProperty $_.PSPath).DeviceState
      Name  = $props.'{a45c254e-df1c-4efd-8020-67d146a850e0},2'
      Assoc = $props.'{b3f8fa53-0004-438e-9003-51a46e139bfc},6'
    }
  } | Format-Table -AutoSize
```

Device states: `1` = ACTIVE, `2` = DISABLED (user-disabled in the Sound control
panel), `4` = NOTPRESENT, `8` = UNPLUGGED.

- `Headset` endpoint at **state 4** while everything upstream is `OK` is the signature
  of this bug: registered but never activated. Windows Settings may still display it;
  Teams won't (it only lists active endpoints).
- Old ghost `Headset` entries at state 4 are normal after re-pairs; what matters is
  whether *one* `Headset` entry for the connected headset is at state 1.
- State `2` is a different problem — someone disabled the endpoint in the classic
  Sound control panel (`mmsys.cpl` → Recording tab → right-click → Show disabled).

For live (non-cached) truth, enumerate endpoints via the Core Audio
`IMMDeviceEnumerator` COM API with `stateMask = 15`; see NOTES.md for the Add-Type
snippet used.

## 4. Find the offload switch on the Bluetooth adapter

```powershell
$dev = Get-PnpDevice | Where-Object FriendlyName -eq 'Intel(R) Wireless Bluetooth(R)'
$hw  = "HKLM:\SYSTEM\CurrentControlSet\Enum\$($dev.InstanceId)\Device Parameters"
$p   = Get-ItemProperty $hw
"Sco Support Type=$($p.'Sco Support Type')  HfpOffloadDisable=$($p.HfpOffloadDisable)"
```

- **`Sco Support Type = 2`** → SCO/HFP audio offload is on (the `HCIBYPASS` path). On an
  offload-enabled platform this is what Intel's `ibtusb.sys` writes at **every boot**, so
  seeing `2` after you set `0` is expected, not a failed fix.
- **`0`** → standard path. You will only see this persist if `HfpOffloadDisable = 1`
  (the driver then writes `0` itself) or if the platform's UEFI offload setting is off.
- Intel's own INF (`C:\Windows\INF\oem*.inf`, `[AudOffload.HW.AddReg]`) writes `0`;
  in-box `bth.inf` writes `2` only in a section Intel's INF doesn't use. Neither is the
  runtime writer — the driver is.

## 5. Quick fix (elevated)

```powershell
Set-ItemProperty -Path $hw -Name 'Sco Support Type' -Value 0 -Type DWord
```

Then **Restart** Windows (not Shut down; fast startup would skip driver re-init). Do
*not* disable/re-enable the adapter to apply the change — the driver's device-add path
runs immediately and writes `2` back, and a failed re-enable can leave you with no
Bluetooth (recover with elevated `pnputil /enable-device '<adapter instance id>'`).

No re-pair needed. After the restart the value reads `2` again and the
`Headset (<name>)` endpoint should be at state 1 (step 3).

## 6. Permanent fix (elevated)

```powershell
Set-ItemProperty -Path $hw -Name 'HfpOffloadDisable' -Value 1 -Type DWord
```

**Restart**, then **remove and re-pair** each headset. Verify:

- step 4 now reads `Sco Support Type=0  HfpOffloadDisable=1` after the boot;
- step 1 shows the `Hands-Free AG` device **without** `HCIBYPASS`;
- step 3 shows a `Headset (<name>)` endpoint at state 1 whose `Assoc` is the Microsoft
  Bluetooth hands-free interface, not "Intel Smart Sound Technology for Bluetooth Audio".

Undo: `Remove-ItemProperty -Path $hw -Name HfpOffloadDisable`, restart, re-pair.

## 7. Who touched the key, and when? (forensics)

Registry keys carry a last-write timestamp. Compare it with boot time and PnP events to
tell "driver rewrote it at boot" from "something reinstalled the driver":

```powershell
Add-Type -TypeDefinition @'
using System; using System.Runtime.InteropServices; using System.Text;
public class RegTime {
  [DllImport("advapi32.dll", CharSet=CharSet.Unicode)] static extern int RegOpenKeyEx(IntPtr h, string sub, int opt, int sam, out IntPtr res);
  [DllImport("advapi32.dll")] static extern int RegQueryInfoKey(IntPtr h, StringBuilder c, IntPtr cl, IntPtr r, out uint a, out uint b, out uint d, out uint e, out uint f, out uint g, out uint s, out long lastWrite);
  [DllImport("advapi32.dll")] static extern int RegCloseKey(IntPtr h);
  public static DateTime Get(string sub) { IntPtr h; if (RegOpenKeyEx((IntPtr)unchecked((int)0x80000002), sub, 0, 0x20019, out h) != 0) throw new Exception("open failed");
    uint a,b,c,d,e,f,g; long t; RegQueryInfoKey(h, null, IntPtr.Zero, IntPtr.Zero, out a, out b, out c, out d, out e, out f, out g, out t); RegCloseKey(h); return DateTime.FromFileTime(t); }
}
'@
[RegTime]::Get("SYSTEM\CurrentControlSet\Enum\$($dev.InstanceId)\Device Parameters")

# Kernel start times and driver (re)configuration events for the adapter
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-Kernel-General'; Id=12} -MaxEvents 5 | Select-Object TimeCreated
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'} -ErrorAction SilentlyContinue |
  Where-Object { $_.Message -match [regex]::Escape($dev.InstanceId) } |
  Select-Object TimeCreated, Id | Select-Object -First 10
```

- Key written seconds after a Kernel-General 12 (OS start) with no Kernel-PnP 400
  (configured) event for the adapter → the driver's device-add path, i.e. normal boot.
- A Kernel-PnP 400 for the adapter → a driver install/reconfiguration happened.

To confirm which binary knows the value name, search for the UTF-16 string
`Sco Support Type` in `ibtusb.sys`, `bthport.sys` and any vendor services; only the
Intel Bluetooth drivers (`ibtusb.sys`, `ibtpci.sys`) and `bthport.sys` contain it.
