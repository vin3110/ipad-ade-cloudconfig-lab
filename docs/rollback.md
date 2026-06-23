# Rollback, persistence, and limitations

## Preference backup

During the lab, the original preference file was backed up before editing:

```sh
f=/var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
backup=/var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist.lab-backup-20260623-ade-toggle

[ -f "$backup" ] || cp -p "$f" "$backup"
```

## Restore the backup

If root SSH is available again through palera1n:

```sh
f=/var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
backup=/var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist.lab-backup-20260623-ade-toggle

cp -p "$backup" "$f"
killall cfprefsd 2>/dev/null || true
sync || true
```

Then verify:

```sh
plutil -show "$f"
```

## What survived reboot

After the bypass and full Setup Assistant completion, a normal reboot was tested.

Survived:

- usable iPad state;
- completed Setup Assistant state;
- no return of the broken cloud configuration prompt.

Did not survive:

- palera1n jailbreak state;
- palera1n app;
- root SSH over USB.

That is expected. The jailbreak is semi-tethered; the Setup Assistant bypass is normal user/data state after the local preference change.

## What a full erase/restore may do

A full erase/restore may recreate the original condition if the device is still assigned to an ADE organization server-side.

Expected risk:

```text
erase/restore -> activation -> local cloudconfig indicator returns -> Setup Assistant may block again
```

The long-term administrative fix is still release/removal from Apple School Manager or Apple Business Manager by the organization that owns the ADE assignment.

## Boundaries

This workflow is a bypass PoC for an owned/released lab device. It does not:

- remove the device from ADE server-side;
- defeat Activation Lock;
- remove a real installed MDM profile;
- create an untethered jailbreak;
- guarantee behavior across other iOS/iPadOS versions.

It only changes one local indicator:

```text
LockdownActivationIndicatesCloudConfigurationAvailable = false
```

under:

```text
/var/mobile/Library/Preferences/com.apple.managedconfiguration.notbackedup.plist
```
