# palera1n / Kali Live runbook

This runbook documents the access path used for the lab.

palera1n is an open-source jailbreak tool for A8–A11 devices that uses the checkm8 bootrom exploit. It provides a semi-tethered jailbreak: after a normal reboot the device works fine but jailbreak features are gone until palera1n is run again via USB. For more background, see [concepts.md](concepts.md).

## Goal

Boot the iPad Air 2 temporarily with palera1n/checkm8 in rootless mode, then use USB SSH to inspect and modify local Setup/ManagedConfiguration state for the bypass PoC.

This does not make the jailbreak persistent. After a normal reboot, root SSH and jailbreak functionality disappear until palera1n is run again.

## Host environment

Used successfully:

- Kali Linux Live `2026.1`, amd64;
- USB stick;
- network access;
- official palera1n Linux x86_64 binary.

## Kali setup

Enable SSH:

```bash
sudo systemctl enable --now ssh
ip -brief address
```

Install USB/iOS tooling:

```bash
sudo apt-get update
sudo apt-get install -y libimobiledevice-utils libusbmuxd-tools
```

Check tools:

```bash
command -v ideviceinfo
command -v iproxy
lsusb | grep -i apple
```

## Install palera1n

Official install script:

```bash
sudo /bin/sh -c "$(curl -fsSL https://static.palera.in/scripts/install.sh)"
palera1n --version
```

Observed during the run:

```text
Build tag: v2.3
USB backend: libusb
```

## Read-only device check

```bash
sudo palera1n -I
```

Expected for this lab device:

```text
Mode: normal
ProductType: iPad5,3
Architecture: arm64
Version: 15.8.8
DisplayName: iPad Air 2 (WiFi)
```

## Rootless palera1n boot

```bash
sudo palera1n -l
```

The device will usually be placed into recovery first, then palera1n will ask for DFU mode.

## DFU timing for iPad Air 2

DFU mode is a low-level USB state used by checkm8. The screen stays completely black — there is no on-screen indicator.

For the iPad Air 2 (Home button device):

1. Connect the iPad to the host via USB.
2. When palera1n says `Press Enter when ready for DFU mode`, press Enter.
3. Hold **Home + Power/Top** simultaneously for about **8–10 seconds**.
4. Release only **Power/Top**, keep holding **Home** for another **10+ seconds**.
5. Release Home when palera1n confirms DFU.

### How to know it worked

- The screen stays **completely black** (no Apple logo, no graphics).
- The palera1n terminal prints `Device entered DFU mode successfully`.

### How to know it didn't work

- **Apple logo appears** → the device is booting normally. Hold Power/Top to force-restart and try again.
- **Cable/computer graphic appears** → you are in **recovery mode**, not DFU. palera1n may offer to kick you out of recovery; otherwise hold Power/Top to force-restart and retry.

> **Tip:** timing takes practice. It is normal to need 2–3 attempts before the timing clicks.

Success markers:

```text
Device entered DFU mode successfully
Checkmate!
Found PongoOS USB Device
Booting Kernel...
```

## USB SSH after boot

After the palera1n boot, Dropbear was available on iDevice port `44`.

Start USB forwarding:

```bash
iproxy 2222 44
```

Check for a banner:

```bash
nc -vz 127.0.0.1 2222
```

Expected banner:

```text
SSH-2.0-dropbear_2022.83
```

Login:

```bash
ssh -p 2222 root@127.0.0.1
```

The default root password is `alpine`.

### SSH host key warning

On first connection (or after a new palera1n boot) you will see a host key verification prompt. Type `yes` to continue.

If you get a `REMOTE HOST IDENTIFICATION HAS CHANGED` warning on a subsequent boot, remove the old key first:

```bash
ssh-keygen -R '[127.0.0.1]:2222'
```

## Useful read-only checks

```sh
id
uname -a
mount
plutil -show /var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
plutil -show /var/mobile/Library/Preferences/com.apple.purplebuddy.plist
```

The root filesystem should be sealed/read-only. The target preference file lives under writable `/private/var`.

## Expected reboot behavior

After a normal reboot:

- iPad can boot normally;
- the bypassed Setup state can persist;
- palera1n/jailbreak is gone;
- palera1n app can disappear;
- root SSH over USB is unavailable until another palera1n/checkm8 boot.

That is normal for palera1n’s semi-tethered model.

## Troubleshooting

### palera1n doesn't detect the device

Check the USB connection and try a different port. USB 2.0 / USB-A ports are preferred over USB 3.0 / USB-C hubs because checkm8 timing is sensitive to USB controller behavior. Verify the host sees the device:

```bash
lsusb | grep -i apple
```

### DFU mode fails repeatedly

Timing is the most common issue. Try on a flat surface so you can press both buttons cleanly. Make sure you are releasing **Power/Top first**, not Home. Some USB-C to Lightning cables have issues with checkm8; try a USB-A to Lightning cable instead.

### iproxy connection fails

Make sure the palera1n boot completed successfully (look for `Booting Kernel...` in the terminal output). Verify the device is still connected with `lsusb`. Dropbear may take a few seconds to start after the kernel boots.

### SSH connection refused

Wait 10–15 seconds after palera1n boot completes. Check that `iproxy` is still running. Test the port directly:

```bash
nc -vz 127.0.0.1 2222
```

### Device reboots during checkm8

This can happen, especially with timing or cable issues. Re-enter DFU mode and try again. Ensure the USB cable is firmly seated at both ends.

## Do not do during the minimal bypass PoC

- Do not erase/restore unless intentionally retesting from scratch.
- Do not use rootful/fakeFS for the first pass.
- Do not delete Setup.app.
- Do not patch the sealed system volume.
- Do not run random third-party bypass scripts.
