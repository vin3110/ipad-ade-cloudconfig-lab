# palera1n / Kali Live runbook

This runbook documents the access path used for the lab.

## Goal

Boot the iPad Air 2 temporarily with palera1n/checkm8 in rootless mode, then use USB SSH to inspect and modify local Setup/ManagedConfiguration state for the bypass PoC.

This does not make the jailbreak persistent. After a normal reboot, root SSH and jailbreak functionality disappear until palera1n is run again.

## Host environment

Used successfully:

- Kali Linux Live `2026.1`, amd64;
- persistence-enabled USB stick;
- network access;
- SSH into the Kali live environment;
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

DFU has a black screen. If the cable/computer graphic appears, the device is in recovery mode, not DFU.

For the iPad Air 2 with a Home button:

1. Hold `Home` + `Power/Top`.
2. Release only `Power/Top` when prompted.
3. Keep holding `Home`.
4. Release when palera1n confirms DFU.

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

## Do not do during the minimal bypass PoC

- Do not erase/restore unless intentionally retesting from scratch.
- Do not use rootful/fakeFS for the first pass.
- Do not delete Setup.app.
- Do not patch the sealed system volume.
- Do not run random third-party bypass scripts.
- Do not publish device identifiers.
