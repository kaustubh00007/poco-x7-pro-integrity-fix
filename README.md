# Poco X7 Pro (rodin) — Play Integrity / Google Pay fix

> Restoring **`MEETS_DEVICE_INTEGRITY`** and working **Google Pay** on a Poco X7 Pro running **crDroid 13 (Android 17)** with an **APatch** kernel root, when the device has **no TEE attestation keys** and **TrickyStore is incompatible** with the installed APatch build.

**Status: verified working** — Play Integrity `MEETS_DEVICE_INTEGRITY`, Google Pay shows *Verified*.
_(Google Pay proof screenshot omitted for privacy; the Play Integrity proof is included.)_

![Play Integrity result](proof_play_integrity_MEETS_DEVICE.png)

---

## The problem

Three independent faults, all required fixing:

| # | Fault | Symptom |
|---|-------|---------|
| 1 | **No TEE attestation keys** (`ATTESTATION_KEYS_NOT_PROVISIONED`) | Hardware cannot emit a key-attestation chain → all Play Integrity verdicts empty |
| 2 | **TrickyStore is incompatible with APatch 11282** | Boot deadlock (`waiting for service package`) or `verify1 failed` |
| 3 | **Identity mismatch** — crDroid exposes `POCO/rodin_in/2412DPC0AI` to the system but `Xiaomi/rodin_global/2412DPC0AG` to GMS | `CANNOT_ATTEST_IDS` / `attestation ID mismatch` |

## The solution (what this repo documents)

```
APatch 11282 (KernelPatch 0.13.9)
  ├── Zygisk Next 1.5.3
  ├── Play Integrity Fork v19          (build-fingerprint spoofing, Pixel canary profile)
  └── TEESimulator v4.0                (software KeyMint + keybox = the actual attestation fix)
        ├── config.json                (identity aligned to what GMS requests)
        └── keybox.xml                 (hardware EC chain + software RSA — NOT included here)
```

---

## Repository contents

| File | Description |
|---|---|
| **`GUIDE.md`** | ⭐ **Start here.** Full step-by-step reproduction: OTA boot extraction → APatch patch/flash → AndroidPatch component → module install → TEESimulator keybox/config → PIF fingerprint → verification → troubleshooting → maintenance → recovery → complete error log |
| `teesim_config.json` | Working TEESimulator config (mode `patch`, Pixel identity, target packages) |
| `pif_profile_reference.txt` | PIF identity values + the rule that they must match TEESimulator exactly |
| `APatch_11282.apk` | APatch Manager used |
| `02_ZygiskNext_1.5.3.zip` | Zygisk implementation for APatch |
| `03_PlayIntegrityFork_v19.zip` | Play Integrity Fork v19 |
| `04_TEESimulator_v4.0.zip` | TEESimulator v4.0 |
| `payload-dumper-go.exe` | Extracts `boot.img` from the crDroid OTA `payload.bin` |
| `proof_play_integrity_MEETS_DEVICE.png` | Verified verdict |

### Intentionally NOT included

- **The keybox** (`keybox_merged_*.xml`) — it contains a **private key**. Never commit private keys to a public repository. Supply your own unrevoked hardware keybox containing **both ECDSA and RSA** chains (see `GUIDE.md` §7).
- **Boot images** (2 × 64 MB) — extract `boot.img` from your own OTA (`GUIDE.md` §3); recovery image instructions are in §12.
- Personal identifiers (phone number, device SuperKey) — redacted.

---

## Quick start

Read **[`GUIDE.md`](GUIDE.md)** end to end before touching the device. Minimum viable path:

1. §3 Extract `boot.img` from your crDroid OTA.
2. §4 Patch with APatch (set your own SuperKey) and `fastboot flash boot`.
3. §5 Install the **AndroidPatch** component (creates `/data/adb/ap/bin`).
4. §6 Install the three modules via `apd module install`.
5. §7 Supply a keybox with **EC + RSA** and a `config.json` whose identity matches what GMS requests.
6. §8 Generate a PIF fingerprint with `autopif4.sh`.
7. §9 Verify with SPIC → expect `MEETS_DEVICE_INTEGRITY`.

---

## Troubleshooting

`GUIDE.md` **§16** contains a 25-entry error/issue log with exact messages, causes and fixes, including:

- `ATTESTATION_KEYS_NOT_PROVISIONED`
- TrickyStore bootloop (`waiting for service package`) and `verify1 failed`
- `keybox: no <Key algorithm="rsa">`
- `CANNOT_ATTEST_IDS` / `attestation ID mismatch for AttestationIdBrand/Model`
- `NO_INTEGRITY` (PIF/fingerprint mismatch)
- `Permission denied` writing `/data/adb/teesim` (SELinux label → `chcon`)
- Modules staged but inactive (`/data/adb/modules_update`)
- PhonePe "declined for security reasons" vs "something went wrong" and what each means

---

## ⚠️ Disclaimers

- **Rooting, custom ROMs and attestation spoofing carry risk**: bootloops, data loss, warranty void, and violation of Google's/banks' terms of service. You are responsible for your device and accounts.
- A **keybox is a private key from a real device**. Using a shared one may violate ToS and can put accounts at risk; `MEETS_STRONG_INTEGRITY` is not guaranteed (it depends on the keybox being unrevoked). `MEETS_DEVICE_INTEGRITY` is what Google Pay needs.
- Never **re-lock** the bootloader or flash stock `vbmeta` on a device whose lock state is faked by a patched LK — you may brick it.
- Provided **as-is, without warranty**. Tested only on the exact device/ROM/build listed.

## Credits

[APatch](https://github.com/bmax121/APatch) · [KernelPatch](https://github.com/bmax121/KernelPatch) · [Zygisk Next](https://github.com/Dr-TSNG/ZygiskNext) · [PlayIntegrityFork](https://github.com/osm0sis/PlayIntegrityFork) · [TEESimulator](https://github.com/JingMatrix/TEESimulator) · [payload-dumper-go](https://github.com/ssut/payload-dumper-go) · [SPIC](https://github.com/herzhenr/spic-android)

## License

MIT — see [LICENSE](LICENSE).