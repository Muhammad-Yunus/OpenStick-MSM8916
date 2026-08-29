# MSM8916 / UZ801 — Attach ADB on Windows 11

Modem: UZ801 (Qualcomm Snapdragon 410 / MSM8916)  
OS: Windows 11  
Tool: Android Platform Tools (adb.exe + fastboot.exe)

---

## Solution

### 1. Extract platform-tools

Extract to Downloads folder:
```powershell
Expand-Archive -Path platform-tools-latest-windows.zip -DestinationPath $env:USERPROFILE\Downloads
cd $env:USERPROFILE\Downloads\platform-tools
```

### 2. Restart ADB server

```powershell
.\adb kill-server
.\adb start-server
```

### 3. Unplug and replug USB modem (re-enumeration)

This is the key step. Windows needs to re-enumerate the device so the ADB interface driver installs correctly.

```powershell
# Unplug USB, wait 5 seconds, replug
Start-Sleep -Seconds 5
.\adb devices
```

Or without unplugging:
```powershell
.\adb usb
Start-Sleep -Seconds 5
.\adb devices
```

### 4. Verify

```powershell
.\adb devices -l
```

Expected output:
```
012345689ABCDEF  device  product:msm8916_32_512  model:UZ801  device:msm8916_32_512
```

### 5. Access shell (root)

```powershell
.\adb shell id
# uid=0(root) gid=0(root) context=u:r:shell:s0

.\adb shell
```

---

## USB Device Status (before vs after)

| State | FriendlyName | Status | InstanceId |
|-------|-------------|--------|------------|
| ❌ Failed | Android | Error | `VID_05C6&PID_90B6&MI_02` |
| ❌ Failed | Android | Error | `VID_05C6&PID_90B6&MI_03` |
| ✅ Success | ADB Interface | OK | `VID_05C6&PID_90B6&MI_04` |
| ✅ Success | USB Composite Device | OK | `VID_05C6&PID_90B6` |
| ✅ Success | Remote NDIS | OK | `VID_05C6&PID_90B6&MI_00` |

---

## USB Configuration

Modem already exposes ADB function from factory:

```bash
# In adb shell:
getprop persist.sys.usb.config
# → rndis,serial_smd,diag,adb

getprop sys.usb.state
# → rndis,adb
```

Windows driver: `oem314.inf` (installed automatically by Windows Update).

---

## Network (RNDIS still works)

Modem provides WiFi AP + USB Ethernet on subnet `192.168.100.0/24`:

```powershell
# Check USB Ethernet connection
Get-NetAdapter | Where-Object { $_.Name -match "RNDIS|Ethernet" } | Format-List Name, Status, LinkSpeed

# PC IP in modem subnet
Get-NetIPAddress -InterfaceAlias "RNDIS" -AddressFamily IPv4
```

Modem web interface: http://192.168.100.1/

---

## One-liner

```powershell
cd $env:USERPROFILE\Downloads\platform-tools; .\adb kill-server; Start-Sleep 3; .\adb devices
```

---

## Troubleshooting

**ADB still empty after unplug-replug:**
```powershell
pnputil /enum-devices /class /USB | Select-String "VID_05C6"
```
Ensure there is an entry with `Status: Started` and driver `oem314.inf`.

**If device still shows `Error`:**
1. Open Device Manager
2. Right-click device → Update Driver
3. Browse → Let me pick from a list
4. Select **Android Phone** → **Android ADB Interface**

**Verify RNDIS is active:**
```powershell
Get-NetAdapter | Where-Object { $_.Status -eq "Up" } | Where-Object { $_.InterfaceDescription -match "RNDIS" }
```

**ADB not showing up — enable via web interface:**
Some units require enabling ADB through the modem's web interface:
1. Open browser, go to http://192.168.100.1/
2. Navigate to http://192.168.100.1/usbdebug.html
3. Enable USB debugging mode
4. Replug the USB cable
5. Run `.\adb devices` again

---

## Reference

- Tutorial: https://extrowerk.com/2022-07-31/OpenStick.html
- GitHub OpenStick: https://github.com/OpenStick
- Kernel: Linux 5.15 LTS (mainline)
- Board: Qualcomm MSM8916 (Snapdragon 410) — 4x ARM Cortex-A53, 512MB RAM
