# Kali NetHunter on OnePlus 7T Pro (HD1913 / `hotdog`) via LineageOS 23.2

**A complete, reproducible installation guide — from stock OxygenOS to a fully working NetHunter with the `v0lk3n` kernel.**

This guide documents the exact path that produced a working result: internal-Wi-Fi monitor mode, HID/BadUSB, root, Kali chroot, **and** a screen-lock PIN coexisting with the chroot. It is written as a *clean reproduction path* — it deliberately omits the trial-and-error detours (TWRP, the crashdumping `oneplus7-oos` kimocoder kernel) that were dead ends and got wiped anyway.

> **Based on official documentation.** This guide is grounded in and cross-checked against:
> - LineageOS install wiki for `hotdog`: https://wiki.lineageos.org/devices/hotdog/install/
> - NetHunter device/kernel table: https://nethunter.kali.org/kernels.html
> - General NetHunter install method: https://www.kali.org/docs/nethunter/installing-nethunter/
> - Kali mobile downloads: https://www.kali.org/get-kali/#kali-mobile
> - MindTheGapps (Android 16/arm64): https://github.com/MindTheGapps/16.0.0-arm64/releases
> - OS Updater (OxygenOS OTA fetcher): https://github.com/oxygen-updater/os-updater/releases

> **Author's note on honesty:** Every version and hash below is a value that was actually observed on a working device.

---

## 0. Read this first

### 0.1 What you will achieve (and what you will not)

| Capability | Result | Notes |
|---|---|---|
| Boot LineageOS 23.2 (Android 16) | ✅ | Official build |
| Root (Magisk) | ✅ | Survives kernel flash |
| NetHunter kernel `v0lk3n` (`oneplus7-los-23.2`) | ✅ | `4.14.357-v0lk3n-SM8150_OP7Serie-LOS_23.2+` |
| Kali chroot + toolset | ✅ | e.g. `nmap 7.99` |
| HID / DuckHunter / BadUSB | ✅ | Requires enabling HID in **USB Arsenal** (see §5.1) |
| Wi-Fi **monitor** on internal card (`wlan0`) | ✅ | Sniffing, passive handshake capture |
| Wi-Fi **injection** on internal card | ❌ | QCACLD-3.0 = monitor only. Needs an external adapter (RTL8812AU / MT7612U) |
| Screen-lock **PIN** + chroot together | ✅ | Works on LOS FBE (did *not* work on OxygenOS 11) |

### 0.2 Device / environment

- **Device:** OnePlus 7T Pro, model **HD1913 (EU)**, codename **`hotdog`**. A/B partitions, dynamic partitions, FBE.
- **Confirmed model list for LineageOS `hotdog`:** HD1910 / HD1911 / HD1913 / HD1917 (exact match required).
- **Host PC:** Windows (commands below use PowerShell).

**Working-folder convention used throughout:** create one working folder (e.g. `oneplus_7t_pro`) and put the extracted `platform-tools` inside it. Keep all downloaded images in the working folder. Run `adb`/`fastboot` **from inside `platform-tools`**, so images one level up are referenced as `..\<file>`. Adapt to your own layout — no absolute paths are assumed.

> **⚠ Standing note — ADB keeps "resetting" (read this once):**
> - **USB debugging does NOT survive a wipe or an OS change.** After every **Format Data** and after **each first boot** (OxygenOS 12 setup, and again after the first LineageOS boot), Developer options and USB debugging are **off again** — you must re-enable them (Settings → About → tap **Build number ×7** → Developer options → **USB debugging ON**). This is the #1 reason `adb` suddenly "stops working".
> - **First connection needs RSA authorization.** The very first time a freshly-set-up device connects, a dialog pops up on the phone asking to trust this computer's RSA key. **Tap "Always allow", then OK.** Until you do, `adb devices` will list the device as **`unauthorized`** — it's seen but won't accept commands. If you never see the dialog, run `.\adb kill-server; .\adb start-server`, replug the cable, and unlock the screen.



### 0.3 Verified version pins (the "known-good" set)

```
OxygenOS base   : HD1913_11_F.22   (Android 12 / OxygenOS 12.1)  ← "11.F.22" is OnePlus numbering; the Android version is 12
LineageOS       : lineage-23.2-20260717-nightly-hotdog-signed
Kernel (result) : 4.14.357-v0lk3n-SM8150_OP7Serie-LOS_23.2+
Magisk          : v30.7
NetHunter image : kali-nethunter-2026.2-oneplus7-los-23.2-sixteen-full.zip
                  SHA256 471c9a62d0965215cc365077d146807395f710b4c63be1342bef36a841d40731
MindTheGapps    : MindTheGapps-16.0.0-arm64-20260409_073023.zip
chroot sanity   : nmap 7.99
```

---

## 1. Prerequisites & files to gather

### 1.1 Hardware / drivers (Windows)

- Android **platform-tools** (adb/fastboot).
- **Qualcomm 9008 (EDL) driver** installed — only needed for the MSM rescue (§7.2). If Device Manager doesn't show the port, install the Qualcomm driver `.cab` via Windows Update / vendor package.
- **OnePlus USB driver** (for reliable ADB/fastboot detection).

### 1.2 Bootloader must be unlocked (prerequisite)

On the device: **Settings → About → tap Build number ×7** → back → **Developer options** → enable **OEM unlocking** and **USB debugging**. Then:

```powershell
cd platform-tools
.\adb reboot bootloader
.\fastboot devices          # must list your device
.\fastboot oem unlock       # THIS WIPES THE DEVICE — device replies OKAY on success
```

Confirm on-device with volume/power. *(Some newer setups use `.\fastboot flashing unlock` instead; on this device `oem unlock` worked and returned OKAY.)* The bootloader stays unlocked through everything below — OTA/local-install does **not** relock it.

### 1.3 Files to download (with hash verification)

| File | Source | Notes |
|---|---|---|
| `HD1913_11_F.22` full OTA (~3.97 GB) | **OS Updater** app → see §1.4 | Android 12 base — mandatory for LOS 23.2 |
| `lineage-23.2-<date>-nightly-hotdog-signed.zip` | `download.lineageos.org/hotdog` | latest build |
| `recovery.img` (Lineage Recovery) | same build, same page | **not** TWRP |
| `dtbo.img` | same build, same page | |
| `vbmeta.img` | same build, same page | |
| `kali-nethunter-2026.2-oneplus7-los-23.2-sixteen-full.zip` | `kali.org/get-kali` → **#kali-mobile** → OnePlus 7 … LineageOS 23.2 | carries the `v0lk3n` kernel |
| `Magisk-vXX.apk` (newest stable; v30.7 used here) | `github.com/topjohnwu/Magisk/releases` | |
| `MindTheGapps-16.0.0-arm64-20260409_073023.zip` | `github.com/MindTheGapps/16.0.0-arm64/releases` | Google apps (optional) |

> **Rule:** the 4 LineageOS files (`ROM`, `recovery`, `dtbo`, `vbmeta`) **must all come from the same build date**.

**Verify every file** against the hash shown next to it on the source page:

```powershell
certutil -hashfile ..\<file> SHA256
```

For the NetHunter image, the expected value is:
```
471c9a62d0965215cc365077d146807395f710b4c63be1342bef36a841d40731
```
(matches the SHA256 published at https://www.kali.org/get-kali/#kali-mobile)

### 1.4 Get the OS Updater app (no Google account needed)

The Android-12 base is fetched with **OS Updater** (formerly "Oxygen Updater"), open source:

```powershell
# Download the APK from: github.com/oxygen-updater/os-updater/releases
.\adb install ..\os-updater-<version>.apk
```

In the app: select **OnePlus 7T Pro → stable channel → OxygenOS 12 → Full → Download**. It pulls the official OnePlus ZIP and verifies MD5 automatically. (Root is *not* required just to download.)

---

## 2. Phase 1 — Get the device onto OxygenOS 12 (Android 12)

**Why:** LineageOS 23.2 for `hotdog` **requires an Android-12 stock base** (per the LineageOS install wiki). Being on Android 11 firmware will cause install failure / post-install crashes. In this build, `HD1913_11_F.22` is the final A12 firmware.

> This was done here **without MSM** — a direct "local install" from the existing OxygenOS 11 worked. MSM is kept only as a rescue (§7.2).

### 2.1 Back up the downloaded ZIP to PC first

The A12 install / any wipe can remove the ZIP from the phone. OS Updater stores it under a random UUID name, so find it by size:

```powershell
.\adb shell su -c "find /storage/emulated/0 -iname '*.zip' -size +1G 2>/dev/null"
# identify the ~3.8 GiB file, then:
.\adb pull "/storage/emulated/0/<uuid>.zip" ..\OnePlus7TPro_HD1913_F.22_EU_full.zip
certutil -hashfile ..\OnePlus7TPro_HD1913_F.22_EU_full.zip SHA256
```
*(If `adb pull` hits a permission error because the file is `rw-rw----`, copy it via root to `/sdcard/Download/` and `chmod 644` first, then pull.)*

### 2.2 Install A12 via OS Updater "Option 1: local install"

In OS Updater: **Installation guide → Option 1 (local install, built into the OS — works for all regions except NA)**. Run it; the OS writes the full package to the inactive slot and reboots. **Options 2/3 are not for you** (Option 3 is non-A/B only).

- First A12 boot is long (several minutes).
- Magisk/root from the old OS 11 will be gone after this — that's expected; root is re-done in Phase 3.

### 2.3 Verify the base

- **Settings → About → Android version = 12** (ignore if the vendor string is `11.F.22`).
- **Test radio:** make a call, send an SMS, confirm **LTE data** (toggle Wi-Fi off) and Wi-Fi. All must work before continuing.
- Remove any Google account you added during A12 setup (FRP hygiene).
- VoLTE: if your SIM/region doesn't expose the toggle, skip it — not required.
- **Re-enable Developer options + USB debugging** (they were reset by the A12 setup — see the Standing note in §0.2), and **tap "Always allow"** on the RSA prompt when you replug for the next fastboot step.

✅ **Checkpoint:** clean Android 12, radio OK.

---

## 3. Phase 2 — Install LineageOS 23.2

> Uses **Lineage Recovery + fastboot + adb sideload** (NOT TWRP). Follow the official wiki in parallel: https://wiki.lineageos.org/devices/hotdog/install/

### 3.1 Flash dtbo, vbmeta, and Lineage Recovery

```powershell
.\adb reboot bootloader
.\fastboot devices                       # must list device
.\fastboot flash dtbo    ..\dtbo.img
.\fastboot flash vbmeta  ..\vbmeta.img
.\fastboot reboot bootloader
.\fastboot flash recovery ..\recovery.img
```

### 3.2 Boot into Lineage Recovery

From the bootloader screen, use **Volume** to select **"Recovery mode"**, **Power** to confirm.

> ✅ You must see the **LineageOS logo**. If you see any other recovery (stock/TWRP), stop and re-do §3.1 (nothing is wiped yet, this is safe).

### 3.3 Format data (⚠ POINT OF NO RETURN — wipes the device)

In Lineage Recovery: **Factory reset → Format data / factory reset** → confirm. This removes encryption and clears `/data`. Return to the main menu.

### 3.4 Sideload the ROM

On the device: **Apply update → Apply from ADB**. On PC:

```powershell
.\adb -d sideload ..\lineage-23.2-20260717-nightly-hotdog-signed.zip
```

> **Normal quirks — do not panic:** progress may stall at ~47% and PC may print `adb: failed to read command: Success`. The install still completed. **Do NOT reboot to system yet.**

### 3.5 Install GApps — MUST be before first boot

After the ROM installs, recovery asks whether to reboot to recovery to install add-ons → choose **Yes** (it reboots back into Lineage Recovery). Then **Apply update → Apply from ADB**:

```powershell
.\adb -d sideload ..\MindTheGapps-16.0.0-arm64-20260409_073023.zip
```

> A **"Signature verification failed"** prompt is expected for add-ons → choose **Yes**. Skip this whole step if you want a Google-free install.

### 3.6 First boot

**Reboot system now.** First LineageOS boot is long (5–10 min).

- During setup, **do NOT set a screen lock/PIN yet** (PIN is tested last, in §5.4).

✅ **Checkpoint:** LineageOS 23.2 boots and is usable.

---

## 4. Phase 3 — Root with Magisk

LineageOS doesn't provide root; the standard method is patching the LOS `boot.img` with Magisk. Use the `boot.img` **from the exact same build you installed**.

```powershell
# 4.1 Developer options + USB debugging were RESET by the LOS wipe/first boot (see §0.2 Standing note).
#     Re-enable them: Settings → About → tap Build number ×7 → Developer options → USB debugging ON.
#     On the first replug, tap "Always allow" on the RSA prompt, or adb shows the device as "unauthorized".

# 4.2 Install Magisk app
cd platform-tools
.\adb install ..\Magisk-v30.7.apk

# 4.3 Push the matching boot.img
.\adb push ..\boot.img /sdcard/Download/boot.img
```

On the phone: **Magisk app → Install → "Select and Patch a File" → `/Download/boot.img` → Let's Go.**
This creates `magisk_patched-<...>.img` in `Download`.

```powershell
# 4.4 Pull the patched image back
.\adb shell ls /sdcard/Download/magisk_patched*.img
.\adb pull "/sdcard/Download/magisk_patched-<exact-name>.img" ..\magisk_patched_LOS.img

# 4.5 Flash it
.\adb reboot bootloader
.\fastboot flash boot ..\magisk_patched_LOS.img
.\fastboot reboot

# 4.6 Verify root (grant the Magisk prompt on the phone)
.\adb shell su -c id            # expect: uid=0(root) ... context=u:r:magisk:s0
```

✅ **Checkpoint:** `uid=0`, Magisk shows "Installed".

> **Keep `boot.img` and `magisk_patched_LOS.img` on your PC — they are your recovery net (§7).**

---

## 5. Phase 4 — Install NetHunter (los-23.2 image)

The official *device* page for the OnePlus 7 is outdated (Android-10 flow). Follow the **general** NetHunter method (https://www.kali.org/docs/nethunter/installing-nethunter/): install the image **as a Magisk module** (the current recommended method). The los-23.2 image **carries its own kernel** — no separate kernel flashing, no DM-Verity/ForceEncrypt disabler (that step is only for Android 9/10/11; this is Android 16).

```powershell
# Push the image
.\adb push ..\kali-nethunter-2026.2-oneplus7-los-23.2-sixteen-full.zip /sdcard/Download/
.\adb shell su -c "sha256sum /sdcard/Download/kali-nethunter-2026.2-oneplus7-los-23.2-sixteen-full.zip"
# expect: 471c9a62d0965215cc365077d146807395f710b4c63be1342bef36a841d40731
```

On the phone: **Magisk app → Modules → Install from storage → select the los-23.2 zip.**

> **Keep the screen ON and unlocked for the whole install** (Android may kill the app on a locked screen). The install runs a Boot-Patcher (AnyKernel3, A/B slot detected, `Magisk detected! …reflashing Magisk` = root survives) and extracts the Kali rootfs (**up to ~25 min** — do not touch, do not unplug, do not reboot). Wait for **Done**, then reboot.

The line `[!] Magisk doesn't have permission to access boot scripts` during install is a **harmless** warning.

### 5.0 Post-boot verification

```powershell
.\adb shell su -c id                 # still uid=0
.\adb shell su -c "uname -a"         # expect: ...4.14.357-v0lk3n-SM8150_OP7Serie-LOS_23.2+...
```
- Open the **NetHunter app** → grant root → let it finish setup.
- **NetHunter Store:** update the NetHunter app (it's an F-Droid fork, no Google account). If the store is already current, nothing to do.
- **Kali Chroot Manager** should read **Running**. Sanity check in NetHunter Terminal (Kali session): `nmap --version`.

The `v0lk3n` kernel's feature set is listed in the NetHunter kernel table: https://nethunter.kali.org/kernels.html

✅ **Checkpoint:** kernel = `v0lk3n`, root OK, chroot Running.

---

## 5.1 HID / DuckHunter (BadUSB)

HID does **not** work until the USB gadget is reconfigured to expose the HID function. This is a config step, not a kernel defect.

```powershell
.\adb shell su -c "ls -l /dev/hidg*"     # "No such file or directory" before enabling — normal
# optional kernel confirmation:
.\adb shell su -c "zcat /proc/config.gz | grep -iE 'CONFIGFS_F_HID|USB_G_HID'"   # expect =y
```

Fix:
1. **NetHunter → USB Arsenal →** enable a config that includes **HID** (ideally **HID + ADB** so ADB survives). Apply/Set. *(If ADB drops, that's expected — continue from the on-device NetHunter Terminal.)*
2. **Force-close the NetHunter app and reopen it.**
3. Verify the nodes now exist; fix permissions if needed:
   ```
   ls -l /dev/hidg*
   chmod 666 /dev/hidg0 /dev/hidg1     # only if not already 666
   ```
4. Test in **NetHunter → HID Attacks → DuckHunter** with a harmless script (e.g. `GUI r` → `STRING notepad` → `ENTER`). Use the **USB** tab, not **BT Ducky**.

> Note: DuckHunter defaults to a US keymap; on a PL host, some characters may be off — the point of the test is that keystrokes are injected at all.

✅ **Result:** HID working.

---

## 5.2 Wi-Fi — internal card (monitor)

```
airmon-ng check kill        # stop wpa_supplicant etc. that fight monitor mode
airmon-ng start wlan0
iw dev                      # wlan0 shows: type monitor
```

> **Important driver quirk:** the `icnss`/QCACLD driver puts **`wlan0` itself** into monitor mode; it does **not** create a `wlan0mon` interface. So use `wlan0`, not `wlan0mon`:

```
airodump-ng wlan0           # you should see APs + associated STATIONs
```

✅ **Result:** internal monitor mode + passive capture works.

## 5.3 Wi-Fi — injection (honest limits)

```
aireplay-ng --test wlan0
```
- Observed here: **`No answer… found 0 AP` → internal injection does NOT work.** This is normal/expected for QCACLD-3.0 on this SoC (monitor only).
- **What you can do now without an adapter:** passive recon and **passive WPA handshake capture** (wait for a client to (re)connect while sniffing on `wlan0`), then crack offline (e.g. hashcat on a desktop GPU).
- **What needs an external adapter:** deauth, PMKID (active), Evil AP — anything requiring TX/injection. Use a supported adapter (RTL8812AU / MT7612U) over USB 2.0 OTG.

## 5.4 Screen-lock PIN + chroot (final test)

This is low-risk and reversible (worst case: remove the PIN and the chroot returns).

1. Confirm chroot works (baseline): `nmap --version` in a Kali session.
2. **Settings → Security → Screen lock → PIN** (choose a PIN you'll remember).
3. **Reboot.** Unlock with the PIN.
4. Confirm the chroot still starts: **NetHunter → Kali Chroot Manager → Start**, or `nmap --version`.

✅ **Result observed:** PIN + chroot coexist on LineageOS FBE (this did **not** work on OxygenOS 11).

---

## 6. Appendix A — Kali `full-upgrade`: the systemd / `machine-id` blocker and its fix

#### Symptom

`apt install <pkg>` throws a cascade of `but it is not going to be installed` for a dozen+ dependencies. Alongside it:

```
systemd-sysv : Depends: systemd (= 260.1-1) but 261.1-2 is installed
452 not upgrading, 17 not fully installed or removed
```

`apt --fix-broken install` and `full-upgrade` die on:

```
Setting up systemd (261.1-2)...
Cannot open '/etc/machine-id': Protocol driver not attached
dpkg: error processing package systemd (--configure)
```

#### Cause

One toppled domino. `udev`, `libpam-systemd`, `xserver-xorg-core` and the whole 452-package queue are waiting behind an **unconfigurable `systemd`**.

The systemd postinst (line 72 of `/var/lib/dpkg/info/systemd.postinst`) calls `systemd-machine-id-setup`, and that tool in a chroot returns EUNATCH (`Protocol driver not attached`). EUNATCH is systemd's internal sentinel, returned when systemd is not PID 1 and cannot obtain a machine-id from the host / D-Bus — **not** an I/O error. The `/etc/machine-id` file itself is valid the whole time, so regenerating it does **not** help.

#### Confirm it's the tool, not the file

```bash
exec 3<>/etc/machine-id && echo "O_RDWR OK" && exec 3>&-   # -> O_RDWR OK  (kernel opens it fine)
systemd-machine-id-setup; echo "exit=$?"                   # -> Cannot open '/etc/machine-id'... exit=1
grep -n "machine-id" /var/lib/dpkg/info/systemd.postinst   # shows how postinst invokes the tool
```

Same file, same open mode: the kernel opens it without a blink, `systemd-machine-id-setup` insists it can't. The fault is in the **tool**, not the filesystem — so the fix is to stop the postinst from calling it, not to touch the file.

#### Fix — TL;DR

```bash
printf '#!/bin/sh\nexit 0\n' > /usr/local/bin/systemd-machine-id-setup
chmod +x /usr/local/bin/systemd-machine-id-setup
hash -r
dpkg --configure systemd
rm -f /usr/local/bin/systemd-machine-id-setup && hash -r
dpkg --configure -a
apt --fix-broken install
apt full-upgrade
```

The shim intercepts the call because the postinst invokes the tool **without an absolute path**, and `/usr/local/bin` is ahead of `/usr/bin` in `PATH` (check with `echo $PATH`).

#### Precondition

The shim skips machine-id generation, so the file must already exist and be valid — exactly 32 hex chars:

```bash
cat /etc/machine-id
```

If it's missing:

```bash
printf '%s\n' "$(tr -d '-' < /proc/sys/kernel/random/uuid)" > /etc/machine-id
ln -sf /etc/machine-id /var/lib/dbus/machine-id
```

#### Fallback — patch the postinst (untested here)

> I did not need this in my run — the PATH shim above was enough. Provided for the case where a future version calls the tool by an absolute path (the shim wouldn't be picked up):

```bash
cp -a /var/lib/dpkg/info/systemd.postinst /root/systemd.postinst.bak
sed -i 's|systemd-machine-id-setup|true|g' /var/lib/dpkg/info/systemd.postinst
dpkg --configure systemd
cp -a /root/systemd.postinst.bak /var/lib/dpkg/info/systemd.postinst
```

#### Debconf prompts during the upgrade

Questions about modified config files (e.g. `sshd_config` from `openssh-server`):

| File | Choice |
|---|---|
| touched by you or NetHunter (ssh, sudoers, network) | **keep the local version** |
| never touched | **install the package maintainer's version** |

`sshd_config` → **keep local**. The maintainer's version restores `PermitRootLogin prohibit-password` and cuts SSH to the chroot.

After the upgrade, check that the old config has no directives removed in newer OpenSSH:

```bash
sshd -t          # silence = OK
```

On errors, a fresh template is at `/etc/ssh/sshd_config.dpkg-dist`.

#### Verification

```bash
dpkg --configure -a          # returns immediately, nothing to do
apt --fix-broken install     # 0 upgraded, 0 newly installed
sshd -t
apt install <pkg> && <pkg> --help
```

#### Warnings to ignore (chroot, not errors)

| Message | Reason |
|---|---|
| `dpkg-realpath: symbolic link '/bin' size has changed from 18 to 7` | different symlink path length host vs chroot — cosmetic |
| `Failed to chase and open directory '.../hwdb.d', ignoring: Protocol driver not attached` | udev without host `/sys` access; note the `ignoring` / `skipping` |
| `invoke-rc.d: could not determine current runlevel` | systemd isn't PID 1, services don't start — and that's fine |
| `Failed to chase ... '/usr/lib/systemd/catalog', ignoring` | same; postinst still returns 0 |

What matters is the **absence** of `dpkg: error processing` and `Errors were encountered while processing`.

#### Recurrence

**It comes back on every `systemd` upgrade** — the postinst always calls `systemd-machine-id-setup`, which always fails in a chroot. Just drop the shim in → `dpkg --configure -a` → remove it. 30 seconds. The NetHunter startup script needs no changes — machine-id is persistent and valid; the problem was purely in the tool.

#### Takeaways

- On a cascade of `not going to be installed`, hunt for the **one** package blocking the queue, don't fix a dozen dependencies separately.
- `Protocol driver not attached` in a chroot ≠ an I/O error. Confirm with `exec 3<>file`: if the kernel opens it, the file is fine and regenerating it won't help — the tool is the problem.
- A rolling release held 452 packages behind will keep throwing this conflict on every install. Run `full-upgrade` regularly.

---

## 7. Appendix B — Recovery net (get out of any brick)

### 7.1 Kernel/boot rollback (after LOS is installed)
If a kernel/module flash causes a crashdump/bootloop, restore the clean LOS boot (keeps LineageOS, loses root/NetHunter):
```powershell
# enter fastboot: power off → Power + Vol Up + Vol Down
.\fastboot flash boot ..\boot.img
.\fastboot reboot
```

### 7.2 MSM (nuclear — back to clean OxygenOS 11)
`MsmDownloadTool V4.0.exe` (the repack, **not** `_factory`), Target **EU**, mode Other/no password. Phone in **EDL**: power off → **Vol Up + Vol Down** + USB. Requires the Qualcomm 9008 driver. This fully wipes back to stock OOS 11 — from there you can climb to A12 again (§2) and restart.

---

## 8. Key lessons (why this path, not the others)

1. **OxygenOS 11 is a dead end for "everything"** — broken FBE (no PIN + chroot), and the `oneplus7-oos` (kimocoder) kernel **crashdumps** on `hotdog`.
2. **LineageOS 23.2 + `v0lk3n` `oneplus7-los-23.2` kernel is the winning path** — and its "Internal Monitor" feature is real (monitor works; injection still needs an adapter). See the kernel table: https://nethunter.kali.org/kernels.html
3. **The kernel must match the Android version** — a los-23.2 kernel for A16; don't mix "eleven" kernels onto A16.
4. **Android-12 firmware is mandatory** before installing LOS 23.2 — don't stay on A11.
5. **Always keep `boot.img` + a patched boot on the PC**, and MSM as the ultimate rescue.

---
