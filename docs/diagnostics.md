# Diagnostics notes

This file captures the evidence that led to the final bypass PoC.

## Device identity

The target device was identified over USB as:

```text
ProductType: iPad5,3
Model: iPad Air 2 Wi-Fi
Board: j81ap
Platform: A8X / t7001
iPadOS: 15.8.8
Build: 19H422
```

Unique identifiers are intentionally omitted.

## Setup Assistant symptom

Visible UI sequence:

1. Wi-Fi selected successfully.
2. `Fetching Configuration`.
3. `Configuration could not be retrieved`.

See:

- `media/01-wifi-selected.jpeg`
- `media/02-fetching-configuration.jpeg`
- `media/03-configuration-failed.jpeg`

## Lockdown status

USB lockdown reported:

```text
ActivationState = Activated
ActivationStateAcknowledged = true
BrickState = false
PasswordProtected = false
TrustedHostAttached = true
```

Conclusion: activation was complete. The blocker was post-activation setup/cloud configuration.

## MCInstall observations

Read-only checks:

```text
GetProfileList -> 0 installed profiles
GetCloudConfiguration -> empty / none
```

Interpretation:

- no installed MDM profile was present;
- no local cloud configuration had been saved;
- Setup Assistant was attempting to retrieve cloud configuration during the setup flow.

## Failed local SetCloudConfiguration test

A minimal `SetCloudConfiguration` test was attempted and rejected:

```text
MCCloudConfigErrorDomain 33004
This device must be configured via the Device Enrollment Program.
```

No persistent local cloud configuration was written.

## Live syslog findings

When the failing setup step was triggered, networking itself worked:

- DNS resolution succeeded;
- TCP succeeded;
- TLS 1.3 succeeded;
- server certificate verification succeeded;
- HTTP returned `200`.

The failure occurred in the cloud configuration/enrollment layer:

```text
MCCloudConfigurationErrorDomain Code=34004
The cloud configuration server is unavailable or busy.
CloudConfigurationFatalError
MCCloudConfigErrorDomain code 33001
Could not retrieve cloud configuration
```

Conclusion: switching Wi-Fi networks was unlikely to solve the problem. The device was locally committed to an ADE/cloud configuration path whose server-side configuration was unavailable or broken.

## Pre-bypass preference state

Relevant file:

```text
/var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
```

Before:

```text
HasCheckedForAutoInstalledProfiles = 1
LockdownActivationIndicatesCloudConfigurationAvailable = 1
```

## Post-bypass state

After changing `LockdownActivationIndicatesCloudConfigurationAvailable` to `false` and completing setup:

```text
HasCheckedForAutoInstalledProfiles = 1
LockdownActivationIndicatesCloudConfigurationAvailable = 0
```

PurpleBuddy/Setup Assistant:

```text
SetupDone = 1
SetupFinishedAllSteps = 1
SetupState = SetupUsingAssistant
```

## Privacy notes

The public repo intentionally excludes:

- serial number;
- UDID;
- ECID;
- Wi-Fi MAC;
- pairing records;
- local passwords;
- personal LAN addresses.
