# Play Integrity / Google Pay fix â€” Poco X7 Pro (rodin) â€” Reproducible Guide

**Device:** Xiaomi POCO X7 Pro â€” codename `rodin`, model `2412DPC0AI`
**OS:** crDroid 13 (Android 17, SDK 37) + NikGapps, build `v13.0-20261031`
**Bootloader:** unlocked at fastboot level (Android props faked to `locked`/`green` by a patched LK â€” pre-existing, do not touch)
**Result achieved:** Play Integrity `MEETS_DEVICE_INTEGRITY` âœ…, Google Pay fully functional âœ…
**Date:** 17 Nov 2026

> **PUBLIC DISTRIBUTION NOTE — READ FIRST.** This repository intentionally **omits** two things:
> 1. The **keybox private key** (keybox_merged_*.xml) — never commit private keys; supply your own hardware keybox (§7.1).
> 2. The two **64 MB boot images** — extract oot.img from your own OTA (§3) and use astboot flash (§4)/recovery (§12).
> 
> Also omitted: the GPay proof screenshot (privacy), and the personal phone number / device SuperKey have been redacted.
---

## 0. What is in this folder

| File | Purpose |
|---|---|
| `GUIDE.md` | This document â€” full step-by-step reproduction |
| `APatch_11282.apk` | APatch Manager 11282 (KernelPatch 0.13.9 bundled) |
| `02_ZygiskNext_1.5.3.zip` | Zygisk Next 1.5.3 â€” Zygisk implementation for APatch |
| `03_PlayIntegrityFork_v19.zip` | Play Integrity Fork v19 â€” build-fingerprint spoofing |
| `04_TEESimulator_v4.0.zip` | TEESimulator v4.0 â€” the key-attestation engine (replaces TrickyStore) |
| `teesim_config.json` | Working TEESimulator config (Pixel identity, patch mode, target packages) |
| `payload-dumper-go.exe` | Tool to extract images from the crDroid OTA `payload.bin` |
| `proof_play_integrity_MEETS_DEVICE.png` | Screenshot: MEETS_DEVICE_INTEGRITY |

**Not included (too big / already on disk):** `crDroidAndroid-17.0-20261031-rodin-v13.0.zip` â€” the ROM OTA, needed to re-extract `boot.img`.

---

## 1. What the problem actually was

Three independent faults, all of which had to be fixed:

1. **TEE attestation keys are not provisioned on this device.**
   `keystore2` logs `ERROR: ATTESTATION_KEYS_NOT_PROVISIONED`; the KeyAttestation app reports *"No attestation keys"*. Hardware cannot emit a valid key-attestation certificate chain, so every Play Integrity verdict is empty. **This is why a software attestation layer is mandatory.**
2. **TrickyStore cannot run on this APatch build.**
   APatch 11282 (Oct 2026) is far newer than TrickyStore 1.4.1 (Nov 2025). Started early, TrickyStore deadlocks boot (`waiting for service package` while PackageManagerService waits on keystore). Started late, it fails self-verification (`verify1 failed / unverified!`) and exits. **Replacement: TEESimulator v4.0**, which runs AOSP's reference KeyMint in-process and supports Android 10â€“17.
3. **Device identity mismatch.**
   crDroid presents different identities to different components: system props say `POCO / rodin_in / 2412DPC0AI`, while GMS's attestation request says `Xiaomi / rodin_global / 2412DPC0AG`. TEESimulator's KeyMint TA rejects this with `attestation ID mismatch for AttestationIdBrand/Model` â†’ `CANNOT_ATTEST_IDS`. The identity in `teesim_config.json` must match **exactly what the requesting process sees**, and PIF must spoof the same identity.

Additionally, the keybox from the original module contained only an **ECDSA** key; TEESimulator requires **both ECDSA and RSA** (`keybox: no <Key algorithm="rsa">`).

---

## 2. Prerequisites & warnings

- Bootloader unlocked (check: `fastboot getvar unlocked` â†’ `unlocked: yes`).
- APatch SuperKey used here: **`YourOwnSuperKey`** (embedded in the patched boot image; APatch 11282 also verifies the manager signature).
- **King config that must not change:** kernel `6.6.89-android15-8`, ROM build `v13.0-20261031`, slot `_b`, `CONFIG_KALLSYMS_ALL=y`, `CONFIG_KPROBES=y`, `CONFIG_MODULES=y`.
- âš ï¸ **Never re-lock the bootloader.** The `locked`/`green` state is faked by a patched LK; re-locking will brick or refuse to boot.
- âš ï¸ **Never flash a stock `vbmeta`** or a different slot boot image.
- Back up data before flashing anything.
- Keep `boot_ORIGINAL_*.img` on the PC â€” it is the one-command recovery.
- Tools: platform-tools (`adb`, `fastboot`) on PATH.

---

## 3. Phase A â€” Extract `boot.img` from the crDroid OTA

crDroid ships an A/B OTA (`payload.bin`), not flashable images.

```powershell
# list partitions (optional)
.\payload-dumper-go.exe -l "crDroidAndroid-17.0-20261031-rodin-v13.0.zip"

# extract only boot + vbmeta
.\payload-dumper-go.exe -p boot,vbmeta -o out "crDroidAndroid-17.0-20261031-rodin-v13.0.zip"
```

Result: `out\boot.img` (67,108,864 bytes) and `out\vbmeta.img`.

**Verify the device before flashing anything:**

```powershell
adb devices                                            # must show the serial
adb shell getprop ro.product.device                    # rodin
adb shell getprop ro.crdroid.version                   # 17.0
adb shell getprop ro.crdroid.display.version           # v13.0-20261031
adb shell getprop ro.boot.slot_suffix                  # _b
adb shell getprop ro.boot.verifiedbootstate            # green (faked)
```

---

## 4. Phase B â€” Root with APatch

1. Install the manager:
   ```powershell
   adb install -r -g APatch_11282.apk
   adb push out\boot.img /sdcard/Download/boot.img
   ```
2. On the phone: open **APatch** â†’ top-right **Install** icon â†’ **"Select a boot image to patch"** â†’ pick `Download/boot.img` â†’ tap **Custom SuperKey** â†’ type `YourOwnSuperKey` â†’ tap the confirm arrow â†’ **Start**.
   - Output: `/storage/emulated/0/Download/apatch_patched_11282_0.13.9_*.img`
   - Log must show `is_gki: true`, `kpimg 0.13.9`, `patch done: kernel`, `Repack completed`.
3. Pull it and flash to the **current** slot:
   ```powershell
   adb pull /storage/emulated/0/Download/apatch_patched_*.img out\boot_patched.img
   adb reboot bootloader
   fastboot getvar current-slot          # _b
   fastboot flash boot out\boot_patched.img
   fastboot reboot
   ```
4. Verify root:
   ```powershell
   adb shell "su -c 'id'"                # uid=0(root)
   ```

**Why flash `boot` only:** APatch patches the kernel inside `boot.img` (GKI 6.6). It does **not** use `init_boot`/`vendor_boot`. The AVB metadata is preserved by the patcher.

---

## 5. Phase C â€” Install the AndroidPatch (APModule) component

This is easy to miss and **required**: it creates APatch's userspace (`/data/adb/ap/bin/busybox`, `/data/adb/apd`, `kptools`, `magiskpolicy`, `resetprop`) and enables APM modules.

On the phone: **APatch â†’ Home â†’ AndroidPatch card â†’ Install**.
Home must then show **`0.13.9 (11282) - Full`** and *Executable su: `/system/bin/su`*.

Verify from PC:

```powershell
adb shell "su -c 'ls /data/adb/ap/bin'"
# apd  busybox  kptools  magiskpolicy  resetprop
```

---

## 6. Phase D â€” Install the three working modules

Use APatch's own installer (`apd`). Modules are **staged into `/data/adb/modules_update`** and merged into `/data/adb/modules` at the next boot.

```powershell
adb push 02_ZygiskNext_1.5.3.zip       /data/local/tmp/ZygiskNext.zip
adb push 03_PlayIntegrityFork_v19.zip  /data/local/tmp/PlayIntegrityFork.zip
adb push 04_TEESimulator_v4.0.zip      /data/local/tmp/TEESimulator.zip

adb shell "su -c '/data/adb/apd module install /data/local/tmp/ZygiskNext.zip'"
adb shell "su -c '/data/adb/apd module install /data/local/tmp/PlayIntegrityFork.zip'"
adb shell "su -c '/data/adb/apd module install /data/local/tmp/TEESimulator.zip'"

adb shell "su -c '/data/adb/apd module list'"     # all three enabled
```

Notes:
- TEESimulator's installer **auto-disables TrickyStore** if present (correct).
- Do **not** install TrickyStore on APatch 11282 â€” it bootloops (see Â§9).
- After install, reboot once so modules move into place:
  ```powershell
  adb reboot
  ```

Verify after reboot:

```powershell
adb shell "su -c 'ls /data/adb/modules/'"          # playintegrityfix  teesim  zygisksu
adb shell "ps -A | grep -E 'zn-|teesim'"           # zn-daemon + teesim running
```

---

## 7. Phase E â€” TEESimulator keybox + config (the critical part)

### 7.1 Keybox requirements

TEESimulator needs a keybox containing **both** an ECDSA and an RSA key:
`teesim_km_init_ex: keybox: no <Key algorithm="rsa">` means it will refuse to build a profile.

The working file `keybox_merged_EC-hardware_RSA-software.xml` is a merge of:

- **ECDSA block**: genuine hardware attestation chain (issuer chain up to Google Hardware Attestation Root, serial `f92009e853b6b045`). This is what makes the verdict real.
- **RSA block**: software key/cert, present only to satisfy TEESimulator's RSA requirement.

Structure: `<AndroidAttestation><NumberOfKeyboxes>1</NumberOfKeyboxes><Keybox><Key algorithm="ecdsa">â€¦</Key><Key algorithm="rsa">â€¦</Key></Keybox></AndroidAttestation>` (5 certificates total).

### 7.2 Install keybox + config, fix SELinux label

```powershell
adb push keybox_merged_EC-hardware_RSA-software.xml /data/local/tmp/keybox.xml
adb push teesim_config.json                        /data/local/tmp/teesim_config.json

adb shell "su -c 'mkdir -p /data/adb/teesim'"
adb shell "su -c 'cp -f /data/local/tmp/keybox.xml /data/adb/teesim/keybox.xml'"
adb shell "su -c 'cp -f /data/local/tmp/teesim_config.json /data/adb/teesim/config.json'"

# MANDATORY: /data/adb/teesim is created with label adb_data_file, which the
# APatch root domain cannot write. Relabel it, otherwise config/keybox edits fail
# with "Permission denied" even as root.
adb shell "su -c 'chcon -R u:object_r:system_file:s0 /data/adb/teesim'"
```

### 7.3 Config content (already in `teesim_config.json`)

```json
{
  "version": 1,
  "profiles": {
    "default": {
      "keybox": "keybox.xml",
      "mode": "patch",
      "patchLevel": { "system": "2026-11-05", "vendor": "2026-11-05", "boot": "2026-11-05" },
      "osVersion": "16",
      "brand": "google",
      "device": "caiman",
      "product": "caiman_beta",
      "manufacturer": "Google",
      "model": "Pixel 9 Pro",
      "serial": "", "imei": "", "meid": "", "imei2": "",
      "apps": [
        "com.google.android.gms",
        "com.android.vending",
        "com.google.android.apps.nbu.paisa.user",
        "com.phonepe.app",
        "io.github.vvb2060.keyattestation"
      ]
    }
  }
}
```

**Why these values:** they must equal exactly what the requesting process (GMS, after PIF spoofing) sends as `ATTESTATION_ID_BRAND/DEVICE/PRODUCT/MANUFACTURER/MODEL`. If they differ, TEESimulator's reference KeyMint TA returns `CANNOT_ATTEST_IDS` and no verdict is produced. `mode: patch` = re-sign the real TEE attestation; it auto-falls back to full software generation if the hardware level is unavailable.

Config/keybox changes are **hot-reloaded** by TEESimulator (watch its log for `control: ack epoch=â€¦ applied=1 failed=0`).

---

## 8. Phase F â€” PIF fingerprint (must match the attestation identity)

PIF spoofs Google Play services' view of `Build.*` to the Pixel canary profile; TEESimulator attests that same identity. They must agree.

```powershell
# enable PIF if it was ever disabled
adb shell "su -c '/data/adb/apd module enable playintegrityfix'"

# generate a fresh fingerprint profile (needs internet on the phone)
adb shell "su -c 'cd /data/adb/modules/playintegrityfix && sh autopif4.sh -m'"
adb shell "su -c 'cat /data/adb/modules/playintegrityfix/pif.prop'"
adb reboot
```

The working profile (Nov 2026): Pixel canary `google/caiman_beta/caiman:CANARY/ZP11.261016.008/16754936:user/release-keys`, security patch `2026-11-05`, `spoofBuild=1`, `spoofProps=1`, `spoofProvider=0` (0 is correct when TEESimulator provides real attestation).

---

## 9. Phase G â€” Verify

1. Install the checker (sideload, any version of SPIC):
   ```powershell
   adb install -r -g spic.apk        # com.henrikherzig.playintegritychecker
   ```
2. On the phone: open **SPIC** â†’ **Make Play Integrity Request**.
   Expected:
   ```
   Device Integrity â†’ Device recognition verdict â†’ MEETS_DEVICE_INTEGRITY
   ```
3. Open **Google Pay** â†’ should show *Verified* and proceed to login/UPI, instead of "Your device may not be secure".
4. Re-check the attestation engine:
   ```powershell
   adb logcat -d | Select-String "TEESimulator"
   # control: ack epoch=â€¦ applied=1 failed=0
   # resign: patch_attestation -> N cert(s) [patched leaf + keybox EC chain]
   ```

---

## 10. Troubleshooting (each of these actually happened here)

| Symptom | Cause | Fix |
|---|---|---|
| Boot loop at crDroid logo, logcat: `TrickyStore: waiting for service package`, system_server stuck at `StartPackageManagerService` | TrickyStore 1.4.x deadlocks with APatch 11282 / this kernel | Boot to recovery, then disable it: `adb shell su -c 'touch /data/adb/modules/tricky_store/disable'`; better: uninstall it â€” use TEESimulator |
| `TrickyStore: verify1 failed / unverified! / write failed with 9` | TrickyStore anti-tamper incompatible with this APatch build | Same as above â€” replace with TEESimulator |
| `TEESimulator: keymint: profile default failed to build (bad keybox?)` + `no <Key algorithm="rsa">` | Keybox has only ECDSA (e.g. Droidwin v4.2b) | Use a keybox with **both** EC and RSA (Â§7.1) |
| `CANNOT_ATTEST_IDS` / `attestation ID mismatch for AttestationIdBrand/Model` | `teesim_config.json` identity â‰  what GMS requests | Read the requested blobs from the `keystore2` error in logcat and set brand/device/product/manufacturer/model (and serial/IMEI if requested) to exactly those values |
| `NO_INTEGRITY` (not even BASIC) | PIF disabled or fingerprint not spoofed; or config `profiles=0/1` | Re-enable PIF + `autopif4.sh`, reboot; check `applied=1` |
| `Permission denied` writing `/data/adb/teesim/config.json` as root | Directory labelled `adb_data_file` | `chcon -R u:object_r:system_file:s0 /data/adb/teesim` |
| Module installed but not active | APatch stages modules in `/data/adb/modules_update` | Reboot once |
| `adb shell` says `Permission denied` on `/system/bin/su` from an app | APatch hides `su` from non-root apps at kernel level (by design) | Nothing to do â€” this is correct |
| KeyAttestation app: "Unable to attest device IDs" | Device-ID (IMEI/serial) attestation is not provisioned; irrelevant for Play Integrity | Ignore â€” plain key attestation is what matters |

---

## 11. Maintenance (expect to repeat this)

1. **PIF fingerprint blocked/expired** â†’ refresh and reboot:
   ```powershell
   adb shell "su -c 'cd /data/adb/modules/playintegrityfix && sh autopif4.sh -m'"
   adb reboot
   ```
2. **Keybox revoked** â†’ Play Integrity starts failing; replace `/data/adb/teesim/keybox.xml` with a fresh, unrevoked **hardware keybox that contains both ECDSA and RSA** chains. Hot-reloads â€” no reboot needed.
3. **Android/KeyMint changes** â†’ update TEESimulator from `github.com/JingMatrix/TEESimulator`.
4. After every change verify with SPIC; if the verdict regresses, read `logcat | grep TEESimulator` first â€” it states exactly why a profile failed.

**Do not:** enable TrickyStore, flash stock `boot`/`vbmeta`, re-lock the bootloader, or change `ro.product.*` spoofing without re-aligning `teesim_config.json`.

---

## 12. Recovery / unroot

**Bootloop, still bootable (adb reachable):**
```powershell
adb shell "su -c 'touch /data/adb/modules/<module-id>/disable'"
adb reboot
```

**Android won't boot at all:**
```powershell
adb reboot bootloader
fastboot flash boot boot_ORIGINAL_crDroid_v13.0-20261031.img
fastboot reboot          # boots unrooted; /data/adb ignored
```

**Restore root afterwards:** `fastboot flash boot boot_APATCH_PATCHED_superkey-YourOwnSuperKey.img`.

**Full unroot:** flash the original boot, then (with root re-flashed) remove `/data/adb/modules/*`, `/data/adb/teesim`, then flash original boot again.

---

## 13. Appendix â€” PhonePe investigation (unresolved, not a device-integrity issue)

PhonePe v26.10.23.0 (Play Store) opens and reaches UPI onboarding, but bank activation fails.

**What the APK inspection + logcat capture established:**

- PhonePe checks: **Developer options** (`development_settings_enabled`, 5 refs), **USB debugging** (`adb_enabled`, `adb_wifi_enabled`), **RootBeer** (`isRooted`, `isRootmanager`, su path list incl. `/system/bin/su`), `test-keys`, **Xposed/LSPosed** (64 refs), and Play Integrity (`playintegrity`, 58 refs). It probes the `adb` system service directly.
- PhonePe has **no `QUERY_ALL_PACKAGES`**, so it cannot scan for the APatch manager package; APatch also hides `su` from apps at kernel level â€” client-side root detection is **not** the current blocker.
- Two earlier blockers were found and removed: **Location was OFF** (`location_mode=0`) and **Developer options/USB debugging were ON**. After fixing both, the error changed from *"declined for security reasons"* to *"Unable to proceed. Something went wrong. Please try again later."*
- In the captured clean-state attempt, PhonePe **never sent an SMS** â€” it fails at the UPI onboarding **API/device-binding** stage, before SMS.
- SIM binding is correct: Vi `[REDACTED]` is both PhonePe's stored number and the default SMS SIM; default SMS app (Google Messages) is set; all PhonePe permissions are granted.

**Conclusion:** the remaining failure is server-side/account-state (device-binding conflict with another phone, rate-limit after repeated failures, or a risk flag on this modified device) â€” not something on the device can fix. Recommended: wait 24 h, log out of PhonePe on any other device / use the device-switch flow, reinstall from Play Store, retry on mobile data with the second SIM disabled. If it still fails, use PhonePe on a stock device; **Google Pay works on this phone**.

**Do not add LSPosed/Hide My Applist for PhonePe** â€” PhonePe explicitly detects Xposed/LSPosed; it would make things worse.

---

## 14. Worked-state reference

| Item | Value |
|---|---|
| Root | APatch 11282, KernelPatch 0.13.9, `boot_b` |
| SuperKey | `YourOwnSuperKey` |
| Modules | `zygisksu` 1.5.3, `playintegrityfix` v19, `teesim` v4.0 (enabled) |
| Removed modules | `tricky_store` (incompatible), `droidwin_keybox` (keybox incomplete; content merged into working keybox) |
| TEESimulator config | mode `patch`, identity `google/caiman/caiman_beta/Google/Pixel 9 Pro`, osVersion 16, patch 2026-11-05 |
| Keybox | merged: hardware EC chain + software RSA, 5 certs |
| PIF | v19, Pixel canary `ZP11.261016.008`, patch 2026-11-05 |
| Bootloader | faked `locked`/`green` â€” leave as-is |
| Verified | `MEETS_DEVICE_INTEGRITY`; GPay "Verified" |
| Not guaranteed | `MEETS_STRONG_INTEGRITY` (depends on the keybox staying unrevoked) |

> âš ï¸ **Legal/ToS note:** the ECDSA half of the keybox is a shared/public hardware keybox. Using one violates Google's/bank's terms and can put accounts at risk. The RSA half is software, present only to satisfy TEESimulator's requirement.

---

## 15. Device cleanup performed (and how to repeat it)

After everything was verified working, the unused modules, diagnostic apps and test files were removed. The phone was then re-verified (`MEETS_DEVICE_INTEGRITY` still passing).

```powershell
# 1. Uninstall the two modules that did NOT work (TrickyStore deadlocks on APatch 11282,
#    Droidwin keybox lacked the required RSA key). Removal finalises on reboot.
adb shell "su -c '/data/adb/apd module uninstall tricky_store'"
adb shell "su -c '/data/adb/apd module uninstall droidwin_keybox'"
adb reboot

# 2. Remove diagnostic apps, keep SPIC (needed to re-verify after updates)
adb shell "pm uninstall icu.nullptr.applistdetector"        # ApplistDetector
adb shell "pm uninstall io.github.vvb2060.keyattestation"   # KeyAttestation

# 3. Remove test files from the phone
adb shell 'su -c "rm -rf /data/local/tmp/*.zip /data/local/tmp/*.log /data/local/tmp/*.sh /data/local/tmp/*.apk /data/local/tmp/*.xml /data/local/tmp/*.json /data/local/tmp/*.txt"'
adb shell 'su -c "rm -f /sdcard/Download/pp3.log /sdcard/Download/boot.img /sdcard/Download/apatch_patched_*.img"'

# 4. Revert diagnostics settings
adb shell "settings put global stay_on_while_plugged_in 0"
adb shell "settings put system screen_off_timeout 60000"
```

**Final device state â€” only these remain:**

| Kept | Why |
|---|---|
| `zygisksu` (Zygisk Next 1.5.3) | required by PIF |
| `playintegrityfix` (PIF v19) | build-fingerprint spoofing |
| `teesim` (TEESimulator v4.0) | key-attestation engine |
| `/data/adb/teesim/{config.json,keybox.xml}` | working config + merged keybox |
| SPIC app (`com.henrikherzig.playintegritychecker`) | re-verification tool |
| `/data/adb/ap/ori.img` | APatch's own backup of the original boot image |

**Known harmless leftover:** `/data/adb/tricky_store/` (an old config directory from TEESimulator's install-time keybox adoption). It is **not** a module, is never mounted or executed, and the APatch root domain cannot delete it because of its `adb_data_file` SELinux label (`rm` â†’ `Permission denied`). It has no effect on integrity; ignore it.

**Do NOT delete** `/data/adb/modules/{zygisksu,playintegrityfix,teesim}` or `/data/adb/teesim` â€” that breaks the fix.

**After any cleanup, re-verify:**

```powershell
# SPIC â†’ Make Play Integrity Request â†’ expect MEETS_DEVICE_INTEGRITY
adb logcat -d | Select-String "TEESimulator"   # expect: control: ack epoch=â€¦ applied=1 failed=0
```

---

## 16. Complete error & issue log â€” every problem encountered and how it was solved

### 16.1 Quick reference

| # | Error / issue (exact text) | Where it appeared | Cause | Solution |
|---|---|---|---|---|
| 1 | `keystore2: Error::Km(r#ATTESTATION_KEYS_NOT_PROVISIONED)` / KeyAttestation: *"No attestation keys"* | logcat / KeyAttestation app | Device has no factory-provisioned TEE attestation keys | Install **TEESimulator** (software KeyMint + keybox) |
| 2 | `TrickyStore: waiting for service package` + boot stuck at crDroid logo | logcat, first boot with modules | TrickyStore deadlocks with APatch 11282 (system_server waits on keystore) | Disable/replace TrickyStore; boot recovery via disable marker |
| 3 | `TrickyStore: verify1 failed` / `unverified!` / `write failed with 9: Bad file descriptor` | logcat | TrickyStore anti-tamper incompatible with this APatch build | Same â€” replace with TEESimulator |
| 4 | `teesim_km_init_ex: keybox: no <Key algorithm="rsa">` / `profile default failed to build (bad keybox?)` | logcat (`TEESimulator`) | Keybox has only ECDSA (Droidwin 4.2b) | Merge EC hardware chain + RSA key (Â§7.1) |
| 5 | `Hal(CannotAttestIds, "attestation ID mismatch for AttestationIdBrand")` â†’ `keystore2: CANNOT_ATTEST_IDS` | logcat | TEESimulator identity â‰  GMS-requested identity | Set config identity to GMS's exact values (Â§7.3) |
| 6 | `attestation ID mismatch for AttestationIdModel` (requested `2412DPC0AG`, configured `â€¦AI`) | logcat | Model differs between ROM and GMS view | Set `model` to the requested blob (`2412DPC0AG`) |
| 7 | Play Integrity `NO_INTEGRITY` (not even BASIC) | SPIC | PIF disabled / fingerprint not applied (`profiles=0/1`) | Re-enable PIF, run `autopif4.sh -m`, reboot |
| 8 | `cp: /data/adb/teesim/config.json: Permission denied` **as root** | adb shell `su` | Directory labelled `adb_data_file`, APatch root domain can't write | `chcon -R u:object_r:system_file:s0 /data/adb/teesim` |
| 9 | Module installed but nothing runs | `apd module list` | APatch stages in `/data/adb/modules_update` | Reboot once |
| 10 | `mkdir: '/data/adb/ap': Permission denied` (during in-app patching, before root) | APatch patch log | Harmless â€” APatch has no root yet at patch time | Ignore; fixed after root + AndroidPatch |
| 11 | APatch Home: `KernelPatch not installed`, `Root is unavailable` | APatch app | Boot not patched/flashed yet | Patch boot + `fastboot flash boot` (Â§4) |
| 12 | APatch Home: `AndroidPatch â€” Not installed` | APatch app | APModule/userspace component missing | Home â†’ AndroidPatch â†’ **Install** (Â§5) |
| 13 | Android props say `locked`/`green`, fastboot says `unlocked: yes` | `getprop` vs `fastboot getvar` | Patched LK fakes lock state (pre-existing) | **Leave it** â€” do not re-lock, do not flash stock `vbmeta` |
| 14 | `adb: no devices/emulators found` right after a command | adb | `settings put global adb_enabled 0` kills the session (intended) | Re-enable USB debugging in Settings |
| 15 | `/system/bin/su` visible to `adb shell` | shell | APatch exposes su to privileged shell | By design; apps get ENOENT (kernel-level hiding) â€” don't "fix" |
| 16 | `sh: service.sh: No such file or directory` / `timeout: exec ./daemon: No such file` | adb quoting | Nested quotes broke `cd` in a one-liner | Put commands in a script file and run it by absolute path |
| 17 | `ls /data/adb: Permission denied` (transient, as root) | adb shell | SELinux label of a freshly created dir (`adb_data_file`) | Use absolute paths; relabel with `chcon` when writing is needed |
| 18 | Google Pay: *"Your device may not be secure"* | GPay | `NO_INTEGRITY` from Play Integrity | Fixed by the full stack (PIF + TEESimulator + aligned identity) |
| 19 | Google Pay: *"Could not send SMS. Verifying with OTPâ€¦"* | GPay UI | Normal auto-SMS fallback on a fresh install | Not an error â€” proceed with OTP |
| 20 | PhonePe: *"Unable to proceed. Your payment is declined for security reasons."* | PhonePe bank activation | Developer options + USB debugging ON, location OFF | Turn both debugging toggles OFF, enable Location (Â§13) |
| 21 | PhonePe: *"Unable to proceed. Something went wrong. Please try again later."* | PhonePe UPI onboarding | Server-side/device-binding/rate-limit; **no SMS is even sent** | See Â§13 â€” account/device-binding side, not fixable on-device |
| 22 | `adb pull /data/local/tmp/pp3.log: Permission denied` | adb pull | Log file is `root`-only (640) | `su -c "cp â€¦ /sdcard/Download/â€¦; chmod 644 â€¦"` then pull |
| 23 | `am start -n â€¦MainActivity` â†’ `Error type 3 â€¦ does not exist` | adb | Wrong activity name | Get it with `cmd package resolve-activity --brief <pkg>` |
| 24 | `monkey -p me.bmax.apatch` â†’ *"No activities found"* | adb | APatch hides its launcher entry | Launch explicitly: `am start -n me.bmax.apatch/.ui.MainActivity` |
| 25 | Play Integrity `MEETS_DEVICE_INTEGRITY` but no STRONG | SPIC | Keybox is shared/public â†’ may be revoked; STRONG needs a live hardware keybox | Accept DEVICE (enough for GPay); replace keybox for STRONG |

---

### 16.2 Detailed explanations of the important ones

#### â‘  `ATTESTATION_KEYS_NOT_PROVISIONED` â€” the root cause
```
E keystore2: While generating without a provided attestation key and params: [â€¦]
E keystore2: 1: Error::Km(r#ATTESTATION_KEYS_NOT_PROVISIONED) for AppUid(10233)
W KeyAttestationHandler: Device supports DeviceIdAttestation, but KeyStoreException â€¦ error code 10
```
The TEE cannot produce an attestation certificate chain, so Play Integrity has nothing to evaluate â€” every verdict is empty. KeyAttestation app shows **"No attestation keys"**.
**Fix:** a software attestation layer. TrickyStore was the plan but is incompatible (below), so **TEESimulator v4.0** runs AOSP's reference KeyMint in-process and signs with the keybox.
**Verify:** KeyAttestation app changes from *"No attestation keys"* to a working chain (it may still say *"Unable to attest device IDs"* â€” that's IMEI/serial provisioning, irrelevant to Play Integrity).

#### â‘¡ TrickyStore bootloop (`waiting for service package`)
```
I TrickyStore: Trying to register keystore interceptor ...
I TrickyStore: waiting for service package
D SystemServerTiming: StartPackageManagerService     â† hangs here forever
```
Circular deadlock: TrickyStore waits for the `package` service, while `system_server`'s PackageManagerService blocks on the broken-TEE keystore.
**Fix (device still reachable via adb â€” it is, adbd starts early):**
```powershell
adb shell "su -c 'touch /data/adb/modules/tricky_store/disable'"
adb reboot
```
**Fix (Android won't boot):** `fastboot flash boot boot_ORIGINAL_â€¦img` â†’ boots unrooted â†’ remove the module dir â†’ re-flash the patched boot.
**Permanent fix:** don't use TrickyStore on APatch 11282 â€” use TEESimulator (its installer auto-disables TrickyStore).

#### â‘¢ `verify1 failed / unverified!`
Same incompatibility, seen when TrickyStore is started late instead of early. There is no user-side fix; the version is simply incompatible with this APatch/kernel. Replacing it with TEESimulator resolves it.

#### â‘£ Bad keybox (`no <Key algorithm="rsa">`)
TEESimulator needs **both** an ECDSA and an RSA key in the keybox. The provided Droidwin keybox had only ECDSA (3 certs).
**Fix:** merged keybox = ECDSA block from the hardware keybox (chain to Google Hardware Attestation Root, serial `f92009e853b6b045`) + RSA block from a software keybox â†’ 5 certs, both algorithms. Verified in logs:
```
TrickyStoreâ†’TEESimulator: keybox EC chain: 3 cert(s) â€¦
TEESimulator: keymint: boot_info { device_boot_locked: true, verified_boot_state: Verified }
```

#### â‘¤/â‘¥ Identity mismatch (`CANNOT_ATTEST_IDS`)
TEESimulator's KeyMint TA compares the `ATTESTATION_ID_*` values requested by the caller with its configured identity. crDroid makes different components report different identities:

| Field | System prop | What GMS requested |
|---|---|---|
| brand | `POCO` | `Xiaomi` |
| product | `rodin_in` | `rodin_global` |
| model | `2412DPC0AI` | `2412DPC0AG` |

**How to read the exact requested values** â€” the `keystore2` error dumps the parameters as byte arrays:
```powershell
adb logcat -d | Select-String "ATTESTATION_ID_"
# e.g. ATTESTATION_ID_MODEL value: Blob([50,52,49,50,68,80,67,48,65,71])
# 50,52,49,50,68,80,67,48,65,71 = "2412DPC0AG"
```
**Fix:** put those exact strings into `teesim_config.json` (`brand/device/product/manufacturer/model`, and serial/IMEI if requested). Then TEESimulator logs:
```
control: ack epoch=â€¦ applied=1 failed=0
generate: emitted 4-cert chain (whole key minted in the TA)
```

#### â‘¦ `NO_INTEGRITY` after the keybox was fixed
Cause: PIF was disabled (I had disabled it while testing identity alignment) â†’ Play Integrity's software/DroidGuard side fails, so the verdict is `NO_INTEGRITY` rather than `MEETS_DEVICE_INTEGRITY`.
**Fix:**
```powershell
adb shell "su -c '/data/adb/apd module enable playintegrityfix'"
adb shell "su -c 'cd /data/adb/modules/playintegrityfix && sh autopif4.sh -m'"
adb reboot
```
Rule: **BASIC/DEVICE needs PIF (fingerprint) + TEESimulator (attestation); both must agree on identity.**

#### â‘§ `Permission denied` writing TEESimulator config as root
```
cp: /data/adb/teesim/config.json: Permission denied
```
The directory is created with SELinux label `adb_data_file`, which APatch's `magisk` root domain cannot write (unix perms are fine â€” it's SELinux).
**Fix:** `chcon -R u:object_r:system_file:s0 /data/adb/teesim` (then writes succeed). Same class of problem caused the transient `ls /data/adb: Permission denied`.

#### â‘¨ Modules staged, not active
`apd module install` writes to `/data/adb/modules_update`; APatch merges it into `/data/adb/modules` at boot. No reboot â†’ module present in the list but doing nothing. **Fix: reboot once.**

#### â‘© PhonePe â€” *"declined for security reasons"*
Captured logcat during the failure showed PhonePe probing `adb` and the developer-settings flag, and `getScanResults` failing with `SecurityException: Location mode is disabled`.
**Fix applied:** `settings put global development_settings_enabled 0`, `adb_enabled 0`, location ON (`location_mode 3`).
Result: the message changed â†’ security block cleared.

#### â‘ª PhonePe â€” *"Something went wrong. Please try again later."* (current state)
Log capture proved PhonePe **never sends an SMS** in this flow: no `SmsManager`/`SmsController` call from `com.phonepe.app`; it fails in the UPI-onboarding API/device-binding stage. SIM binding is correct (Vi `[REDACTED]` = stored number = default SMS SIM), default SMS app set, permissions granted, and PhonePe has no `QUERY_ALL_PACKAGES` so it can't scan for the root manager.
**Conclusion:** server-side / account-state (device still bound to another phone, rate-limit after repeated attempts, or risk flag on a modified device). Documented steps to try: wait 24 h, log out on the old device / device-switch flow, reinstall from Play Store, retry on mobile data with the second SIM disabled. If it still fails â†’ PhonePe on a stock device. **Do not add LSPosed/HMA** (PhonePe detects Xposed/LSPosed: 64 references). GPay works.

#### â‘« Diagnostic techniques that produced these answers (reuse them)
```powershell
# which packages/modules are really enabled
adb shell "su -c '/data/adb/apd module list'"

# what the attestation engine is doing (most useful log)
adb logcat -d | Select-String "TEESimulator"

# what identity the system is asking for
adb logcat -d | Select-String "ATTESTATION_ID_|CANNOT_ATTEST_IDS|keystore2"

# Play Integrity verdicts from a normal app
# install SPIC â†’ Make Play Integrity Request â†’ read MEETS_DEVICE_INTEGRITY

# attestation sanity check
# install KeyAttestation (io.github.vvb2060.keyattestation) â†’ "No attestation keys" = broken TEE

# reproduce a UI failure while capturing, surviving adb being turned off
# (root logcat detached from adbd):
# echo script to /data/local/tmp/cap.sh:
#   exec /system/bin/logcat -b all -v threadtime -f /data/local/tmp/cap.log -r 32768 -n 2
# adb shell "su -c '/data/adb/ap/bin/busybox setsid /data/local/tmp/cap.sh >/dev/null 2>&1 </dev/null &'"
# then disable adb, reproduce, re-enable, copy the log out via su to /sdcard
```
