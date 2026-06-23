# Diagnostics — extended technical evidence

See also: [concepts.md](concepts.md) for background, [palera1n-runbook.md](palera1n-runbook.md) for the palera1n procedure, [rollback.md](rollback.md) for undo steps.

This document collects **all** diagnostic evidence gathered during the lab run.
It is intended as a standalone reference for anyone who wants to verify every
claim made in `walkthrough.md` against raw terminal output.

## Device identity

The target device was identified over USB with `ideviceinfo` (libimobiledevice)
and `palera1n -I`:

```bash
# libimobiledevice — selective key query
ideviceinfo -k ProductType -k ProductVersion -k ActivationState

# palera1n built-in device info
sudo palera1n -I
```

Combined output:

```text
ProductType: iPad5,3
Model: iPad Air 2 Wi-Fi
Board: j81ap
Platform: A8X / t7001
iPadOS: 15.8.8
Build: 19H422
ActivationState: Activated
```

USB bus confirmation:

```text
Bus 003 Device 002: ID 05ac:12ab Apple, Inc. iPad
```

Only non-unique model/build information is shown here.

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

USB lockdown queried via `ideviceinfo`:

```bash
ideviceinfo -q com.apple.mobile.lockdown
```

Relevant fields:

```text
ActivationState = Activated
ActivationStateAcknowledged = true
BrickState = false
PasswordProtected = false
TrustedHostAttached = true
```

Pairing validation:

```bash
idevicepair validate
```

```text
SUCCESS: Validated pairing with device <redacted-pairing-id>
```

Conclusion: activation was complete. The blocker was post-activation
setup/cloud configuration.

## MCInstall observations

Read-only checks performed via the lockdown client (`ideviceinfo`) and
`pymobiledevice3` lockdown service calls:

```text
GetProfileList -> 0 installed profiles
GetCloudConfiguration -> empty / none
```

Interpretation:

- no installed MDM profile was present;
- no local cloud configuration had been saved;
- Setup Assistant was attempting to retrieve cloud configuration during the
  setup flow.

## Failed local SetCloudConfiguration test

A minimal `SetCloudConfiguration` test was attempted and rejected:

```text
MCCloudConfigErrorDomain 33004
This device must be configured via the Device Enrollment Program.
```

No persistent local cloud configuration was written.

## Live syslog findings

Real-time device logs captured with:

```bash
idevicesyslog
```

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

Conclusion: switching Wi-Fi networks was unlikely to solve the problem. The
device was locally committed to an ADE/cloud configuration path whose
server-side configuration was unavailable or broken.

## Mount and filesystem state

Captured on-device after SSH-over-USB (`iproxy 2222 44`, then SSH to
`127.0.0.1:2222`):

```bash
mount | sed -n '1,20p'
```

```text
com.apple.os.update-<redacted>@/dev/disk0s1s1 on / (apfs, sealed, local, nosuid, read-only, journaled, noatime)
/dev/disk0s1s2 on /private/var (apfs, local, nodev, nosuid, journaled, noatime, protect)
```

Key observations:

- **`/` is sealed and read-only** — the root filesystem was not remounted
  writable. This confirms rootless palera1n mode; no system files were touched.
- **`/private/var` is writable** — user-data partition, where
  `com.apple.managedconfiguration.notbackedup.plist` lives. The bypass operated
  entirely within `/var`.

## Pre-bypass preference state

Preference files read via on-device `plutil` (palera1n binpack):

```bash
for f in \
  /var/mobile/Library/Preferences/com.apple.purplebuddy.plist \
  /var/mobile/Library/Preferences/com.apple.purplebuddy.notbackedup.plist \
  /var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
do
  t="/tmp/plist.xml"
  cp "$f" "$t"
  plutil -convert xml1 "$t" >/dev/null 2>&1
  echo "BEGIN_XML:$f"
  cat "$t" | sed -n '1,160p'
  echo "END_XML:$f"
  rm -f "$t"
done
```

### com.apple.purplebuddy.plist

```xml
<dict>
    <key>GuessedCountry</key>
    <array>
        <string>NL</string>
    </array>
    <key>UserChoseLanguage</key>
    <true/>
    <key>setupMigratorVersion</key>
    <integer>11</integer>
</dict>
```

### com.apple.purplebuddy.notbackedup.plist

```xml
<dict>
    <key>setupAnalytics</key>
    <dict>
        <key>hasCompletedInitialSetup</key>
        <false/>
        <key>numberOfPanesPresented</key>
        <integer>2</integer>
    </dict>
</dict>
```

### com.apple.managedconfiguration.notbackedup.plist

```xml
<dict>
    <key>HasCheckedForAutoInstalledProfiles</key>
    <true/>
    <key>LockdownActivationIndicatesCloudConfigurationAvailable</key>
    <true/>
</dict>
```

Note: `LockdownActivationIndicatesCloudConfigurationAvailable = true` is the
flag that causes Setup Assistant to loop on the cloud configuration fetch.

## Post-bypass state

After changing `LockdownActivationIndicatesCloudConfigurationAvailable` to
`false`, killing `cfprefsd`, and completing Setup Assistant on-device, full
readback:

```bash
plutil -show /var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
plutil -show /var/mobile/Library/Preferences/com.apple.purplebuddy.notbackedup.plist
plutil -show /var/mobile/Library/Preferences/com.apple.purplebuddy.plist
```

### com.apple.managedconfiguration.notbackedup.plist

```text
{
    HasCheckedForAutoInstalledProfiles = 1;
    LockdownActivationIndicatesCloudConfigurationAvailable = 0;
}
```

### com.apple.purplebuddy.notbackedup.plist

```text
{
    CloudConfigPresented = 1;
    LocationServices5Presented = 1;
    acceptedSLAVersion = 1776;
    setupAnalytics =     {
        hasCompletedInitialSetup = 0;
        numberOfPanesPresented = 2;
        softwareUpdatePerformed = 0;
        usedProximitySetup = 0;
    };
}
```

### com.apple.purplebuddy.plist

```text
{
    AssistantPresented = 1;
    AutoUpdatePresented = 1;
    Language = "nl-NL";
    Locale = "nl_NL";
    PrivacyPresented = 1;
    RestoreChoice = 1;
    ScreenTimePresented = 1;
    SetupDone = 1;
    SetupFinishedAllSteps = 1;
    SetupLastExit = "2026-06-23 16:44:19 +0000";
    SetupState = SetupUsingAssistant;
    SetupVersion = 11;
    SiriOnBoardingPresented = 1;
    UserChoseLanguage = 1;
}
```

Configuration profile directories post-bypass:

```text
/var/containers/Shared/SystemGroup/systemgroup.com.apple.configurationprofiles/Library/ConfigurationProfiles/CloudConfigurationDetails.plist
/var/containers/Shared/SystemGroup/systemgroup.com.apple.configurationprofiles/Library/ConfigurationProfiles/PayloadManifest.plist
/var/containers/Shared/SystemGroup/systemgroup.com.apple.configurationprofiles/Library/ConfigurationProfiles/ProfileTruth.plist

/var/mobile/Library/UserConfigurationProfiles/ClientTruth.plist
/var/mobile/Library/UserConfigurationProfiles/PayloadManifest.plist
/var/mobile/Library/UserConfigurationProfiles/ProfileTruth.plist
```

## Process state

Relevant MDM/Setup processes observed post-bypass:

```bash
ps -A -o pid,comm,args 2>/dev/null \
  | grep -Ei 'mdm|managed|purplebuddy|budd' \
  | grep -v grep
```

```text
ManagedSettingsAgent
budd
```

Pre-bypass launchd service list (from `launchctl print system`):

```text
com.apple.devicemanagementclient.mdmd
UIKitApplication:com.apple.purplebuddy[...][rb-legacy]
com.apple.devicemanagementclient.mdmuserd
com.apple.purplebuddy.budd
com.apple.managedconfiguration.mdmdservice
```

Observations:

- `budd` (PurpleBuddy daemon) was still running — it continues after setup
  completes; this is normal.
- `ManagedSettingsAgent` was present — the managed-configuration subsystem
  remains loaded regardless of whether an MDM profile is installed.
- The full set of `mdmd`/`mdmuserd`/`mdmdservice` launchd jobs were registered
  at system level. Their presence does not imply an active MDM enrollment; these
  are stock system services.
