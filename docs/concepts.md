# Background Concepts

This document covers the key technologies and concepts behind the ADE cloud configuration bypass documented in this repo. It's meant to give you enough context to understand **why** each step in the walkthrough works — not just **what** to type. If you already know a topic, skip ahead.

---

## checkm8

**checkm8** (pronounced "checkmate") is a bootrom exploit discovered by **axi0mX** in September 2019. It affects Apple's A5 through A11 chips — covering every iPhone from the 4S through the X, and iPads with equivalent SoCs (including the iPad Air 2's A8X).

**Why it matters:**

- The bootrom (SecureROM) is the very first code that runs when the device powers on. It's burned into the silicon at manufacturing time and is **read-only** — Apple cannot patch it with a software update. This makes checkm8 a permanent, unpatchable exploit for affected hardware.
- It allows **arbitrary code execution during the boot chain**, before iOS security mechanisms (like code signing enforcement) are fully active. This is the foundation that makes jailbreaking these devices reliable.

**Limitations:**

- **Tethered via USB.** The exploit must be delivered over USB while the device is in DFU mode. It does not persist across reboots on its own — every cold boot requires a computer to re-exploit.
- **A12 and newer are not affected.** Apple redesigned the bootrom starting with the A12 (iPhone XS, 2018 iPad Pro, etc.), closing the vulnerability.
- It is a **boot-time** exploit only. It does not provide a remote or network-based attack vector.

---

## palera1n

[palera1n](https://palera.in) is a jailbreak tool built on top of checkm8. It supports **A8 through A11** devices running iOS 15 and later.

**Semi-tethered model:**

palera1n is *semi-tethered*, which is an important distinction:

- After jailbreaking, you can **reboot the device normally** and it will boot into stock iOS — fully usable, but without jailbreak features (no SSH, no root access, no tweaks).
- To restore jailbreak functionality, you connect to a computer and **re-run palera1n** via USB. The device re-enters DFU, gets re-exploited, and boots with jailbreak active again.
- This means you're never "stuck" — a reboot always gets you back to a working (non-jailbroken) device.

**Rootless vs. rootful:**

palera1n offers two modes:

| | **Rootless** | **Rootful (fakeFS)** |
|---|---|---|
| System partition | Untouched, sealed | Copied to a writable fake filesystem |
| Writes go to | `/var` (user data partition) only | Both system and user partitions |
| SSV intact? | Yes | No (bypassed) |
| Risk level | Lower — system integrity preserved | Higher — modifies more of the device |
| Sufficient for this bypass? | **Yes** | Also works, but unnecessary |

For this ADE bypass, **rootless is sufficient and recommended**. The plist we need to modify lives on the user data partition (`/var`), so there's no reason to touch the sealed system volume.

**Typical invocation:**

```bash
# Rootless jailbreak (default, recommended)
sudo palera1n

# Rootful — only if you have a specific reason
sudo palera1n --force-revert   # clean up first if switching modes
sudo palera1n -f -c            # create fakeFS
sudo palera1n -f               # boot rootful
```

---

## DFU Mode

**Device Firmware Upgrade (DFU) mode** is the lowest-level state an iOS device can enter. Understanding what it is — and what it isn't — is critical for checkm8 exploitation.

**DFU vs. Recovery Mode:**

| | **DFU Mode** | **Recovery Mode** |
|---|---|---|
| What loads | SecureROM (bootrom) only | SecureROM → **iBoot** |
| Screen | **Completely black** — no logo, no graphics | Shows the "connect to computer" icon |
| Responds to | Low-level USB commands | iTunes/Finder restore commands |
| Purpose | Firmware-level recovery, development | Normal iOS restore/update |

**Why checkm8 needs DFU:**

The exploit targets a vulnerability in the **USB handling code within the bootrom itself** — before iBoot or any other software loads. In recovery mode, iBoot is already running and the vulnerable bootrom USB code is no longer active. DFU is the only state where the device is sitting in the bootrom's USB stack, waiting for input.

**How to recognize DFU:**

The device appears to be completely off — **black screen, no Apple logo, no "connect to computer" graphic**. The only indication it's in DFU is that your computer detects it as a USB device (e.g., `lsusb` shows an Apple device in DFU mode, or palera1n reports "device found").

**Entering DFU on iPad Air 2 (Home button device):**

1. Connect to USB.
2. Hold Power + Home for 8 seconds.
3. Release Power, keep holding Home for ~5 more seconds.
4. If the screen stays black and the computer detects the device, you're in DFU. If the Apple logo appears, you went into recovery — start over.

---

## ADE / DEP (Automated Device Enrollment)

ADE — **Automated Device Enrollment** — is Apple's system for organizations to automatically configure devices they've purchased. (You may also see the older name **DEP**, Device Enrollment Program — it's the same thing, rebranded.)

**The full lifecycle:**

1. An organization purchases devices through Apple or an authorized reseller.
2. The devices are **registered in Apple School Manager (ASM) or Apple Business Manager (ABM)**, tied to the organization's account.
3. An MDM server URL is assigned to those serial numbers in ASM/ABM.
4. When the device is activated (after a factory reset or first power-on), it contacts Apple's servers during Setup Assistant.
5. Apple looks up the serial number → finds the ADE assignment → tells the device to fetch its MDM configuration from the organization's server.
6. The device enrolls in MDM, which can push profiles, restrictions, apps, etc.

**Why this breaks on old devices:**

The problem occurs when the **server-side assignment still exists at Apple, but the path it points to is broken**:

- The organization may no longer exist.
- The organization's MDM server may have been decommissioned.
- The device was sold/donated but **never removed from ASM/ABM**.
- The ABM account may be locked, inaccessible, or the admin has moved on.

The device dutifully contacts Apple, receives instructions to fetch a cloud configuration from a server that doesn't respond (or rejects the request), and **gets stuck in a loop** — Setup Assistant keeps trying and failing, never completing.

**The two layers of ADE state:**

This distinction is central to the bypass:

1. **Server-side (at Apple):** The serial number → MDM server mapping in ASM/ABM. You cannot change this without access to the organization's ASM/ABM account or an Apple support escalation.
2. **On-device (local):** A plist file that records whether the device *thinks* a cloud configuration is available. This is what Setup Assistant checks. **This is what the bypass modifies.**

The bypass works because the device checks its *local* indicator during Setup Assistant. By flipping that indicator, Setup Assistant no longer attempts the (broken) cloud configuration fetch and proceeds normally.

---

## MDM (Mobile Device Management)

MDM is the protocol and infrastructure that lets an organization remotely manage iOS devices — push configuration profiles, enforce restrictions, install apps, wipe devices, etc.

**How MDM relates to ADE:**

ADE is the *enrollment mechanism* — it gets the device to the MDM server. MDM is what happens *after* enrollment. In a working setup:

```
Device activates → ADE directs it to MDM server → MDM enrolls device → MDM pushes profiles
```

**Why this device was blocked without an installed MDM profile:**

In our case, the device had **no MDM profile installed** — there was no active management, no restrictions from an MDM server, no enrollment had actually completed. The problem was entirely at the *enrollment attempt* stage:

- The on-device ADE indicator said: "a cloud configuration is available, go fetch it."
- Setup Assistant tried to fetch it.
- The server-side MDM endpoint was broken/unreachable.
- Setup Assistant couldn't proceed past this step.

The device was not *managed* — it was stuck trying (and failing) to *become* managed.

---

## libimobiledevice

[libimobiledevice](https://libimobiledevice.org/) is an open-source library that implements Apple's proprietary **lockdown protocol** (usbmuxd) on Linux (and other platforms). It's how you communicate with iOS devices without iTunes or Finder.

**Key tools used in this bypass:**

| Tool | Purpose |
|---|---|
| `ideviceinfo` | Query device properties — serial, UDID, iOS version, device class, etc. |
| `idevicesyslog` | Stream the device's live system log to your terminal. Essential for diagnosing what Setup Assistant is doing. |
| `idevicepair` | Manage the trust pairing between your computer and the device. Needed before most other tools work. |
| `iproxy` | Forward a local TCP port to a port on the device over USB. Used to access Dropbear SSH (port 44) as `localhost:2222` on your machine. |

**Typical usage pattern for this bypass:**

```bash
# Check that the device is connected and get basic info
ideviceinfo -s

# Pair with the device (may require trust prompt on device)
idevicepair pair

# Forward local port 2222 to device port 44 (Dropbear SSH)
iproxy 2222 44 &

# SSH into the jailbroken device
ssh -p 2222 root@localhost

# Stream device logs (useful during Setup Assistant debugging)
idevicesyslog
```

---

## Key iOS Internals for This Bypass

### plist Files

Property list (plist) files are Apple's standard configuration/settings format — analogous to JSON or INI files. They come in two encodings: XML and binary. On-device, most plists are binary and are read/written with the `plutil` or `defaults` commands.

**The file that matters:**

```
/var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
```

This plist contains (among other keys) the local indicator that tells Setup Assistant whether a cloud configuration is available. Specifically, the `LockdownActivationIndicatesCloudConfigurationAvailable` key controls whether Setup Assistant attempts the ADE enrollment flow.

### cfprefsd

`cfprefsd` is the preferences caching daemon on iOS. It reads plist files into memory and serves them to processes that request preference values. This means:

- If you edit a plist file directly on disk (via SSH), **cfprefsd may still serve the old cached version** to processes that ask for it.
- After modifying a plist, you need to **restart cfprefsd** to force it to re-read from disk:

```bash
killall cfprefsd
```

This is a common gotcha — if you edit the plist but skip this step, the change may appear to have no effect.

### Dropbear (SSH on Port 44)

palera1n starts a **Dropbear** SSH server on the jailbroken device, listening on **port 44**. Dropbear is a lightweight SSH implementation commonly used in embedded/resource-constrained environments.

- Default credentials: `root` / `alpine` (the classic iOS root password).
- Accessible over USB via `iproxy` (not over Wi-Fi by default).
- Only active when the device has been booted with the jailbreak — after a normal reboot, Dropbear is not running.

### Sealed System Volume (SSV)

Since iOS 15, the system partition is protected by a **Sealed System Volume** — a cryptographic hash tree (similar to dm-verity on Android) that covers every byte of the system filesystem. Any modification to a system file breaks the seal and the device refuses to boot.

**Why this matters for the bypass:**

- System files (under `/`) are **read-only and tamper-evident**. You cannot modify them in rootless mode.
- User data files (under `/var`) are **not covered by the seal** and are writable.
- The managed configuration plist lives under `/var/mobile/`, which is on the user data partition — so rootless jailbreaking is sufficient to modify it. No need to break the SSV.

### PurpleBuddy (Setup Assistant)

**PurpleBuddy** is the internal (code) name for the iOS **Setup Assistant** — the sequence of screens you see after a factory reset (language selection, Wi-Fi, Apple ID, etc.). It's the process that checks the ADE/cloud configuration indicator and initiates the MDM enrollment flow.

When you see references to `PurpleBuddy` in system logs (`idevicesyslog`), that's Setup Assistant. Log entries from this process are useful for diagnosing exactly where and why the activation flow is getting stuck.

---

## Further Reading

- **[Walkthrough](../walkthrough.md)** — Step-by-step bypass procedure.
- **[Diagnostics](diagnostics.md)** — How to verify device state and troubleshoot.
- **[palera1n Runbook](palera1n-runbook.md)** — Detailed palera1n usage notes.
- **[Rollback](rollback.md)** — How to undo the bypass.
