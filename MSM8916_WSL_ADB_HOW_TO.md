# MSM8916 / UZ801 — Attach ADB via WSL2 on Windows 11

Modem: UZ801 (Qualcomm Snapdragon 410 / MSM8916)  
OS: Windows 11 + WSL2 (Ubuntu)  
Tool: ADB inside WSL, USB passthrough via usbipd

---

## Solution

### 1. Install WSL2

Open PowerShell as Administrator and run:
```powershell
wsl --install
```
Restart when prompted. After restart, set WSL2 as default:
```powershell
wsl --set-default-version 2
```

Verify:
```powershell
wsl --list --verbose
```
Expected output shows `*` next to your distro with `VERSION 2`.

### 2. Install usbipd-win

Download from: https://github.com/dorssel/usbipd-win/releases

```powershell
# Download and install (latest release)
Invoke-WebRequest -Uri "https://github.com/dorssel/usbipd-win/releases/download/v5.0.0/usbipd-win_5.0.0_x64.msi" -OutFile "$env:TEMP\usbipd-win.msi"
Start-Process msiexec.exe -ArgumentList "/i $env:TEMP\usbipd-win.msi /quiet" -Wait
```

Or download manually from releases page and install.

### 3. List available USB devices

Plug in the modem first, then list devices:
```powershell
usbipd list
```
> **Important:** Run all `usbipd` commands in **PowerShell as Administrator** or **Command Prompt with administrator privileges**.
>
> Without admin rights you will see:
> ```
> usbipd: error: Access denied; this operation requires administrator privileges.
> ```

Look for your modem entry (VID `05c6:90b6`), e.g.:
```
⎿ 3-1    05c6:90b6  Remote NDIS based Internet Sharing Device #2, Android, AD...  Not shared
```

### 4. Bind the modem to usbipd

Replace `<BUS-ID>` with the actual bus id from the list above:
```powershell
usbipd bind --busid <BUS-ID>
```

Example (BUSID from `usbipd list`):
```powershell
usbipd bind --busid 3-1
```

### 5. Attach to WSL

```powershell
usbipd attach --busid <BUS-ID> --wsl
```

This exports the USB device into WSL.

### 6. Open WSL and verify USB device

```powershell
wsl
```
Inside WSL, check if USB device is visible:
```bash
lsusb
```
Expected output:
```
Bus 001 Device 005: ID 05c6:90b6 Qualcomm, Inc.
```

### 7. Install ADB in WSL

```bash
sudo apt update
sudo apt install -y android-tools-adb
```

Verify:
```bash
adb version
```

### 8. Check ADB device detection

```bash
adb devices
```
If empty, restart adb server:
```bash
adb kill-server
adb start-server
adb devices
```

### 9. Verify device attached

```bash
adb devices -l
```
Expected output:
```
0123456789ABCDEF  device  product:msm8916_32_512  model:UZ801  device:msm8916_32_512
```

### 10. Access shell (root)

```bash
adb shell id
# uid=0(root) gid=0(root) context=u:r:shell:s0

adb shell
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

## USB Configuration (inside WSL)

Modem already exposes ADB function from factory:
```bash
adb shell getprop persist.sys.usb.config
# → rndis,serial_smd,diag,adb

adb shell getprop sys.usb.state
# → rndis,adb
```

---

## Network (RNDIS still works)

Modem provides WiFi AP + USB Ethernet on subnet `192.168.100.0/24`:
```bash
ip addr show usb0 2>/dev/null || ip addr show enp0s20f0u1
```
Modem web interface: http://192.168.100.1/

---

## One-liner (full setup)

```powershell
wsl --set-default-version 2; usbipd bind --busid <BUS-ID>; usbipd attach --busid <BUS-ID> --wsl; wsl -d Ubuntu
```
Then inside WSL:
```bash
sudo apt update && sudo apt install -y android-tools-adb; adb devices
```

---

## Troubleshooting

**usbipd list shows nothing:**
- Unplug and replug the modem
- Run `usbipd list` again

**Device not visible inside WSL (`lsusb` empty):**
```powershell
# Detach and reattach
usbipd detach --busid <BUS-ID>
usbipd attach --busid <BUS-ID> --wsl
```
Then inside WSL:
```bash
lsusb
```

**ADB still shows no devices inside WSL:**
```bash
adb kill-server
adb start-server
adb devices
```

**ADB not showing up — enable via web interface:**
Some units require enabling ADB through the modem's web interface:
1. Open browser, go to http://192.168.100.1/
2. Navigate to http://192.168.100.1/usbdebug.html
3. Enable USB debugging mode
4. Replug the USB cable
5. Run `usbipd attach --busid <BUS-ID> --wsl` again, then inside WSL run `adb devices`

**If device shows as `Error` in Windows Device Manager:**
1. Open Device Manager
2. Right-click device → Update Driver
3. Browse → Let me pick from a list
4. Select **Android Phone** → **Android ADB Interface**

**Verify usbipd is running:**
```powershell
Get-Service usbipd
# Status should be Running
```

---

## Reference

- Tutorial: https://extrowerk.com/2022-07-31/OpenStick.html
- GitHub OpenStick: https://github.com/OpenStick
- usbipd-win: https://github.com/dorssel/usbipd-win
- Kernel: Linux 5.15 LTS (mainline)
- Board: Qualcomm MSM8916 (Snapdragon 410) — 4x ARM Cortex-A53, 512MB RAM
