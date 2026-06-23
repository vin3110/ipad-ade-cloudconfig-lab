# Legacy iPad ADE cloudconfig bypass PoC

A proof-of-concept and walkthrough for bypassing a broken Apple Automated Device Enrollment (ADE) / cloud configuration Setup Assistant loop on an owned legacy iPad Air 2.

## Context

This side project started with an old iPad I had used for school. I no longer remembered the passcode, but the device itself was still mine and I wanted to give it a second life instead of letting it sit unused.

The obvious first step was to erase and restore it. That worked: iPadOS installed cleanly and the device reached Setup Assistant. But because this had originally been an education-managed device, setup did not behave like a normal personal iPad. After selecting Wi-Fi, the iPad tried to retrieve an Apple ADE/cloud configuration from an old management path that no longer worked. The result was a hard block during setup:

```text
Fetching Configuration -> Configuration could not be retrieved
```

At that point the iPad was technically alive, activated, and freshly restored, but still unusable. That felt like a waste. So I turned it into a small security research project: understand exactly where the setup flow was failing, prove that it was not a normal network issue, and find the smallest possible local bypass to get the device booted into a usable state.

The final bypass was minimal: a temporary palera1n/checkm8 boot gave root access over USB, which allowed one local ManagedConfiguration preference to be changed under `/var/mobile`. After that, Setup Assistant completed normally and the bypass survived a normal reboot.

## What is ADE?

Apple Automated Device Enrollment (ADE), previously known as DEP, is Apple’s mechanism for automatically enrolling organization-owned devices into management during setup. Schools and companies use it so that an erased iPhone, iPad, or Mac can contact Apple during activation, discover that it belongs to an organization, and then receive a mandatory management configuration.

In a healthy setup, this is useful: the device is restored, connects to Apple, receives the organization’s configuration, and enrolls into MDM.

In this case, the device still appeared to have an ADE/cloud configuration requirement, but the old server-side configuration path was broken. The iPad could activate, but Setup Assistant stayed blocked while trying to retrieve a configuration that could no longer be completed.

<img src="media/03-configuration-failed.jpeg" alt="Setup Assistant: configuration failed" width="420">

## Prerequisites

### Hardware

- A legacy iPad with an A8–A11 chip (e.g. iPad Air 2, iPad mini 4, iPad 5th/6th gen)
- A USB-A to Lightning cable (USB-A preferred; USB-C hubs and adapters can cause timing issues with checkm8)
- A computer with a USB-A port
- A USB stick (8 GB minimum) if booting Kali Live

### Software

- A Linux environment (Kali Live was used in this lab; most Debian-based distros should work)
- Internet access on the host for installing packages

> **Note:** palera1n also provides a macOS binary. Windows is not natively supported; WSL2 can work in some cases but USB passthrough to WSL is unreliable for checkm8 timing. If you don't have a Linux or macOS machine, booting from a Kali Live USB stick is the recommended approach.

### Estimated time

About 1 hour for a first run. DFU mode timing may take a few extra attempts.

## What this repo contains

- [walkthrough.md](walkthrough.md) — the main bypass PoC/walkthrough from symptom to working device.
- [docs/diagnostics.md](docs/diagnostics.md) — USB, activation, MCInstall, and syslog findings.
- [docs/palera1n-runbook.md](docs/palera1n-runbook.md) — Kali Live, palera1n, DFU, and USB SSH procedure.
- [docs/rollback.md](docs/rollback.md) — rollback notes, reboot behavior, and limitations.
- [docs/concepts.md](docs/concepts.md) — background on checkm8, palera1n, DFU, ADE, and other key concepts.
- [media/](media/) — redacted photos of the Setup Assistant failure flow.

**Suggested reading order:**

1. Start with this README for context and scope.
2. Read [docs/concepts.md](docs/concepts.md) if you're unfamiliar with checkm8, palera1n, or DFU mode.
3. Follow [walkthrough.md](walkthrough.md) from start to finish — this is the main guide.
4. Refer to [docs/palera1n-runbook.md](docs/palera1n-runbook.md) when the walkthrough references the palera1n boot procedure.
5. See [docs/rollback.md](docs/rollback.md) for undo steps and persistence notes.

## Device and scope

Target device:

- iPad Air 2 Wi-Fi
- Product type: `iPad5,3`
- SoC/platform: Apple A8X / `t7001`
- OS: iPadOS `15.8.8`
- Build: `19H422`

This was performed on an owned/released device. The workflow is for research on a legacy iPad. Do not use it on devices you do not own or do not have explicit permission to test.

Bypassing ADE on devices you do not own may violate local laws. If you have a device that is legitimately yours but still assigned to an organization in Apple School Manager or Apple Business Manager, the proper administrative path is to contact that organization (or [Apple Support](https://support.apple.com) with proof of purchase) to have the device released.

## Result

After the local-state correction, Setup Assistant completed successfully.

Post-check:

```text
SetupDone = 1
SetupFinishedAllSteps = 1
LockdownActivationIndicatesCloudConfigurationAvailable = 0
```

A normal reboot was then tested:

- the iPad booted back into the usable system;
- Setup Assistant did not return;
- the ADE/cloud configuration error did not return;
- the semi-tethered palera1n jailbreak and root SSH were gone, as expected.

## Redactions

This repo intentionally does not include:

- serial number;
- UDID;
- ECID;
- Wi-Fi MAC address;
- pairing records;
- local SSH passwords;
- personal network details.

The photos in `media/` are copies with GPS EXIF removed.

## License

The repository currently uses the MIT License. See [LICENSE](LICENSE).
