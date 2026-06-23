# Legacy iPad ADE cloudconfig bypass PoC

A proof-of-concept and walkthrough for bypassing a broken Apple Automated Device Enrollment (ADE) / cloud configuration Setup Assistant loop on an owned legacy iPad Air 2.

The short version: the iPad was already Apple-activated, but Setup Assistant still believed a cloud configuration was available and mandatory. The cloud configuration retrieval path failed server-side, leaving the device stuck at `Fetching Configuration`. A temporary palera1n/checkm8 boot gave root access over USB, which allowed a minimal local-state bypass under `/var/mobile`. After that, Setup Assistant completed normally and the bypass survived a normal reboot.

![Setup Assistant: Wi-Fi selected](media/01-wifi-selected.jpeg)

## What this repo contains

- [walkthrough.md](walkthrough.md) — the main bypass PoC walkthrough from symptom to working device.
- [docs/diagnostics.md](docs/diagnostics.md) — USB, activation, MCInstall, and syslog findings.
- [docs/palera1n-runbook.md](docs/palera1n-runbook.md) — Kali Live, palera1n, DFU, and USB SSH procedure.
- [docs/rollback.md](docs/rollback.md) — rollback notes, reboot behavior, and limitations.
- [media/](media/) — redacted photos of the Setup Assistant failure flow.

## Device and scope

Target device:

- iPad Air 2 Wi-Fi
- Product type: `iPad5,3`
- SoC/platform: Apple A8X / `t7001`
- OS: iPadOS `15.8.8`
- Build: `19H422`

This was performed on an owned/released lab device. The workflow is documented as a legacy iPad bypass PoC and research walkthrough. Do not use it on devices you do not own or do not have explicit permission to test.

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

## Privacy and redaction

This repo intentionally does not include:

- serial number;
- UDID;
- ECID;
- Wi-Fi MAC address;
- pairing records;
- local SSH passwords;
- personal network details.

The photos in `media/` are lab copies with GPS EXIF removed.

## License

The repository currently uses the MIT License selected during GitHub repo creation. See [LICENSE](LICENSE).

If this repository later grows into a mix of prose and scripts, a split license may be useful:

- docs/research notes: Creative Commons;
- code/scripts: MIT.
