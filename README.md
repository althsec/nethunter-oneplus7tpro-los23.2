# Kali NetHunter on OnePlus 7T Pro (`hotdog`) via LineageOS 23.2

A complete, reproducible guide to running **Kali NetHunter with the `v0lk3n` kernel** on the **OnePlus 7T Pro (HD1913, codename `hotdog`)**, on top of **LineageOS 23.2 (Android 16)**.

I couldn't find any existing walkthrough for this specific kernel + ROM combination on this device — the official device page was years out of date, the kernel's source column was empty, and there were no public confirmations that it would even boot on `hotdog`. So I tested it end-to-end on my own device and wrote the missing documentation, including the fixes for the things that break along the way.

> Built and verified on a personal device, on a personal network, for learning. Every version and hash in the guide is a value observed on a working phone.

---

## Status — what works (and what doesn't)

| Capability | Result |
|---|---|
| Boot LineageOS 23.2 (Android 16) | ✅ |
| Root (Magisk, survives kernel flash) | ✅ |
| NetHunter kernel `v0lk3n` (`oneplus7-los-23.2`) | ✅ |
| Kali chroot + toolset | ✅ |
| HID / DuckHunter / BadUSB | ✅ (after enabling HID in USB Arsenal) |
| Wi-Fi **monitor** on the internal card (`wlan0`) | ✅ (sniffing, passive handshake capture) |
| Wi-Fi **injection** on the internal card | ❌ (QCACLD-3.0 = monitor only; needs an external adapter) |
| Screen-lock **PIN** + chroot together | ✅ (works on LOS FBE; did *not* on OxygenOS 11) |

**Honest limitations:** packet **injection on the internal Wi-Fi chip does not work** on this SoC — monitor mode only. For deauth / PMKID / Evil-AP you need a supported external adapter (RTL8812AU / MT7612U) over a USB 2.0 OTG cable. This is documented, not hidden.

---

## Overview (4 phases)

1. **Phase 1 — Android 12 base.** LOS 23.2 requires an Android-12 stock base; get the device onto OxygenOS 12 (`HD1913_11_F.22`) first.
2. **Phase 2 — LineageOS 23.2.** Flash `dtbo` / `vbmeta` / Lineage Recovery, format data, sideload the ROM (+ optional GApps).
3. **Phase 3 — Root.** Patch the LOS `boot.img` with Magisk and flash it.
4. **Phase 4 — NetHunter.** Install the `oneplus7-los-23.2` image as a Magisk module (it carries the `v0lk3n` kernel), then verify HID / Wi-Fi / PIN.

➡ **Full step-by-step with every command: [`GUIDE.md`](GUIDE.md)**

---

## Fixes documented along the way

- **Crashdump / bootloop** from the wrong kernel → the correct path for `hotdog` (and which kernels to avoid).
- **HID "not enabled / check your kernel"** → it's a USB-gadget config step (USB Arsenal), not a kernel defect.
- **`apt full-upgrade` jammed behind 452 packages** by a misleading `Cannot open '/etc/machine-id': Protocol driver not attached` — the file was fine; the tool returns an internal sentinel in a chroot. Full root-cause + a reversible fix in [`GUIDE.md` §6](GUIDE.md#6-appendix-a--kali-full-upgrade-the-systemd--machine-id-blocker-and-its-fix).

---

## Verified version pins

```
OxygenOS base   : HD1913_11_F.22   (Android 12 / OxygenOS 12.1)
LineageOS       : lineage-23.2-20260717-nightly-hotdog-signed
Kernel (result) : 4.14.357-v0lk3n-SM8150_OP7Serie-LOS_23.2+
Magisk          : v30.7
NetHunter image : kali-nethunter-2026.2-oneplus7-los-23.2-sixteen-full.zip
MindTheGapps    : MindTheGapps-16.0.0-arm64-20260409_073023.zip
```

Sources it's built on: [LineageOS install wiki](https://wiki.lineageos.org/devices/hotdog/install/) · [NetHunter kernel table](https://nethunter.kali.org/kernels.html) · [NetHunter install docs](https://www.kali.org/docs/nethunter/installing-nethunter/) · [Kali mobile downloads](https://www.kali.org/get-kali/#kali-mobile).

---

## Recovery net

Bricked / bootloop? The guide keeps you covered: roll back to the clean LOS boot with `fastboot flash boot boot.img`, or go nuclear with MSM back to clean OxygenOS. See [`GUIDE.md` §7](GUIDE.md#7-appendix-b--recovery-net-get-out-of-any-brick).

---

## ⚠ Disclaimer

This is educational documentation. Flashing an unlocked bootloader, custom firmware, and a custom kernel **can wipe or permanently damage your device** — you do it at your own risk. NetHunter is an offensive-security toolkit: **only use it on devices and networks you own or are explicitly authorized to test.** You are solely responsible for complying with the laws that apply to you. The author accepts no liability for misuse or damage.

---

## Contributing

Found a newer build, a better fix, or got internal injection working on this SoC? Open an issue or a PR — corrections and confirmations from other `hotdog` owners are welcome.

## License

[MIT](LICENSE)
