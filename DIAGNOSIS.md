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
- State `2` is a different problem — someone disabled the endpoint in the classic
  Sound control panel (`mmsys.cpl` → Recording tab → right-click → Show disabled).

For live (non-cached) truth, enumerate endpoints via the Core Audio
`IMMDeviceEnumerator` COM API with `stateMask = 15`; see NOTES.md for the Add-Type
snippet used.

## 4. Find the offload switch on the Bluetooth adapter

```powershell
$dev = Get-PnpDevice | Where-Object FriendlyName -eq 'Intel(R) Wireless Bluetooth(R)'
$hw  = "HKLM:\SYSTEM\CurrentControlSet\Enum\$($dev.InstanceId)\Device Parameters"
(Get-ItemProperty $hw).'Sco Support Type'
```

- **`2`** → SCO/HFP audio offload is on (the `HCIBYPASS` path).
- **`0`** → standard path (what Intel's own INF sets as default: check
  `C:\Windows\INF\oem*.inf` for `[AudOffload.HW.AddReg]` → `"Sco Support Type",0x00010001,0`,
  versus in-box `bth.inf` which sets `2`).

## 5. The fix (elevated)

```powershell
Set-ItemProperty -Path $hw -Name 'Sco Support Type' -Value 0 -Type DWord
```

Then **Restart** Windows (not Shut down; fast startup would skip driver re-init), and
**remove + re-pair** the headset. Do *not* disable/re-enable the adapter to apply the
change — a configuration pass re-applies `bth.inf`'s value `2` over your `0`, and a
failed re-enable can leave you with no Bluetooth (recover with elevated
`pnputil /enable-device '<adapter instance id>'`).

## 6. Verify

Re-run step 3: `Headset (<name>)` should now be **state 1**. Communication apps will
list the mic immediately.
