# IR-Range1 M5 — Blue Team Incident Response Report

**Scenario:** Recovery Media Persistence  
**Organization:** Vantage Media & Trust  
**Recovery coordinator:** `VMT-REC-01`  
**Exercise date:** 27 October 2026 (UTC)  
**Case:** `CASE-R1M5-2719`  
**Forensic case directory:** `/var/tmp/IR-Range1-M5-CASE-R1M5-2719`  
**Official procedure:** [M5/Blue-Writeup.md](https://github.com/hacktifytechnologies/IR-Range1/blob/main/M5/Blue-Writeup.md)  
**Disposition:** Lab resolution validator passed; intake and automated recovery remain suspended pending approved reopening.

> **Evidence basis and limitation.** This report documents the terminal and portal screenshots shared during the exercise. The screenshot ZIP is an investigation submission, not a substitute for the VM's original `.tar.gz` forensic archives. Off-VM transfer and receipt of those original forensic archives were not evidenced. Early asset-verification screenshots may not be available in the submitted set. Screenshots are preserved in chronological order and may include terminal errors later resolved.

## 1. Executive summary

The M5 incident involved an attacker-submitted compressed FAT16 recovery image processed by the vulnerable `REC-EMERGENCY-LEGACY` validation path. Nuclei returned one critical match, and the official `validate-attack.sh` passed. The validator accepted the media because its visible FAT root files matched an approved image and its basic format checks passed, while VBR slack, reserved-sector comparison and embedded-stage analysis were deferred. An isolated rebuild then reported an unexpected startup unit, `vantage-render-resume.service`, with campaign marker `MDRN-7F3A`.

Blue Team preserved the compressed submission, decompressed candidate, approved image, logs, validation record, rebuild workspace and SQLite database. The candidate and approved images differed in sectors 0, 2 and 3 (928 altered bytes), despite matching filesystem fingerprints. An `MDRNVBR1` bootstrap marker and encrypted stage-2 object were recovered from the candidate's VBR and reserved sectors. The recovered object passed CRC32 and SHA-256 verification and identified loader `STG2-R1M5-2719`, a federation-related cross-range context, and the `IR-Range2-M1` handoff.

Through the authenticated Recovery Assurance portal, analyst `neha.raman` applied action `RESP-R1M5-2719` at `2026-10-27T03:05:54.632873+00:00`. The candidate was quarantined, intake and automated recovery were disabled, the approved baseline was staged as trusted rebuild media, and the lab's certificate-chain validation flag was set. The trusted rebuild had zero changed sectors, matched the approved filesystem fingerprint and had no malicious bootstrap. A signed cross-range handoff was created and verified, and the official `validate-resolution.sh` passed.

## 2. Key identifiers and hashes

| Item | Verified value |
|---|---|
| Recovery node | `VMT-REC-01` |
| Media ID | `MED-R1M5-2719` |
| Validation job | `VAL-R1M5-2719` |
| Isolated rebuild job | `RBLD-R1M5-2719` |
| Emergency validation profile | `REC-EMERGENCY-LEGACY` |
| Submission account | `svc.recovery-intake` |
| Approved media SHA-256 | `8f42a6bc85ac38a8ac6f6f8569b24fee13fe681f122b4cab8a5454011a119e5a` |
| Candidate raw image SHA-256 | `bf42f34d576416da5d46018b994deb63cd1380ccc18316455db93794e7991472` |
| Submitted gzip SHA-256 | `9bfb46ea7891a163f15ee7d4cecb4b41ba4044fb2069118bd73c00a4d732b51b` |
| Unexpected systemd unit SHA-256 | `512be87eb05b0ae6d42153d4bff740579520379009227294a30ae8b293712890` |
| Recovered stage-2 SHA-256 | `642f12219b41361d0cbb3d5ea789de94d5ac764238165c82f1f255e074bc192a` |
| Bootstrap marker | `MDRNVBR1` |
| Stage-2 loader | `STG2-R1M5-2719` |
| Persistence unit | `vantage-render-resume.service` |
| Campaign | `MDRN-7F3A` |
| Containment action | `RESP-R1M5-2719` |
| Range 2 handoff ID | `HDO-R1-R2-2026-0627` |
| Range 2 destination | `IR-Range2-M1`, NimbusSphere Cloud Services |
| Cloud tenant | `9f3a2c10-6b77-4f2e-a821-0d44a5b87019` |
| Cloud role alias | `NS-BridgeAdmin` |
| Scenario domain | `update-checker.net` |

## 3. Setup and controlled attack

`validate-setup.sh` passed all prerequisite checks, service `vantage-recovery-assurance.service` was active, and the portal listened on TCP `8084`. Prior to attack, `/health` reported `attack_completed: false`, `incoming_media_present: false`, and `recovery_suspended: false`. The approved baseline's SHA-256 matched the official reference.

Nuclei v3.11.1 ran `IR-Range1-M5-Attack-Template.yaml` against `http://127.0.0.1:8084`, yielding one critical match at `/api/recovery/simulate`, with `MED-R1M5-2719`, `RBLD-R1M5-2719`, and `vantage-render-resume.service`. `validate-attack.sh` reported PASS for original media, raw candidate, validation gap, simulated rebuild variation, VBR/reserved-sector artifacts, encrypted stage-2 and telemetry. The environment explicitly identifies itself as an isolated simulator without external networking; the generated service is restricted to `/bin/true`.

## 4. Evidence acquisition and integrity

The forensic case was created at `2026-10-27 02:45:36 UTC` with restricted permissions. Acquired items included compressed and raw candidate media, approved baseline image, M4 handoff, recovery policy, media-validation and boot-observation JSON, rebuild workspace (including unexpected unit), application logs, and consistent SQLite backup. Database integrity returned `ok`. The gzip decompressed to the exact candidate SHA-256. Working media copies were analyzed without modifying the originals.

The application timeline export held 40 lines; correlation found the candidate hash in media-intake and validation logs and `MDRN-7F3A`, `STG2-R1M5-2719`, and the startup-unit indicators in the preserved rebuild and extraction artifacts. The chain of custody identifies the local root forensic collector, incident case and containment actor. External evidence transfer remains unverified.

**Screenshot evidence:**

![Restricted forensic case and UTC collection time](images/IR-M5-01-20261027-024553.png)

![Preserved media, logs, handoff and rebuild evidence inventory](images/IR-M5-02-20261027-024644.png)

![Approved, candidate and compressed-media hashes verified](images/IR-M5-03-20261027-024703.png)

![Working media copies prepared for analysis](images/IR-M5-04-20261027-024718.png)

![Required forensic artifacts confirmed present](images/IR-M5-21-20261027-025638.png)

![Chain-of-custody record identifies collector and unverified external transfer](images/IR-M5-41-20261027-031344.png)

## 5. Validation gap and FAT16 analysis

Emergency validation reported `accepted: true`, profile `REC-EMERGENCY-LEGACY`, successful gzip integrity, size, root-file fingerprint, boot signature and volume label; checks for `vbr_slack_comparison`, `reserved_sector_comparison` and `embedded_stage_analysis` were `deferred`. The isolated rebuild identified an unexpected unit and set `variance_detected: true`; the unit was not executed and had `ExecStart=/bin/true`.

Both disk images were 4,194,304 bytes, FAT16, volume label `VMTRECOVERY`, serial `7F3A2719`, and eight reserved sectors. `fsstat` displayed no differences. The official VBR inspector initially failed JSON serialization for candidate bootstrap bytes; the analyst patched **only a working copy** to hex-encode bytes and successfully reran it, preserving original images and installed tool. The approved image showed `vbr_bootstrap_present: false` and the candidate `true`. Candidate reserved-region entropy was approximately `2.6021` versus zero in approved media.

| Changed sector | Byte offset | Changed bytes | Interpretation |
|---|---:|---:|---|
| 0 | 0 | 69 | VBR slack/bootstrap metadata containing `MDRNVBR1` |
| 2 | 1,024 | 506 | Hidden reserved-sector content |
| 3 | 1,536 | 353 | Additional hidden reserved-sector content |
| **Total** | | **928** | All altered bytes lie within reserved/VBR space |

Filesystem fingerprints matched, demonstrating that visible-file validation could not establish raw-media trust.

**Screenshot evidence:**

![Emergency validation shows deferred deep media checks](images/IR-M5-05-20261027-024959.png)

![Unexpected startup unit identified in the simulated rebuild](images/IR-M5-06-20261027-025013.png)

![Approved and candidate filesystem comparison](images/IR-M5-07-20261027-025026.png)

![Working-copy VBR inspector adjusted for byte serialization](images/IR-M5-08-20261027-025207.png)

![Candidate VBR bootstrap and FAT16 metadata](images/IR-M5-09-20261027-025223.png)

![Candidate root-file metadata and bootstrap presence](images/IR-M5-10-20261027-025231.png)

![Matching filesystem fingerprints with changed sectors 0, 2 and 3](images/IR-M5-11-20261027-025243.png)

![Approved and candidate media comparison](images/IR-M5-12-20261027-025253.png)

![Hex comparison locates VBR bootstrap differences](images/IR-M5-13-20261027-025305.png)

## 6. Hidden payload recovery and analysis

The analyst independently extracted the first eight sectors (4,096 bytes) from approved and candidate working images, hashed them and recorded 928 byte differences. The VBR bootstrap indicated stage-2 beginning at sector 2, spanning two sectors with encrypted length 867 bytes. The official `extract_stage2.py` recovered and validated the embedded object. Expected and actual CRC32 were `2cadb4a8`; expected and actual SHA-256 were `642f12219b41361d0cbb3d5ea789de94d5ac764238165c82f1f255e074bc192a`.

The extracted configuration identified `STG2-R1M5-2719`, campaign `MDRN-7F3A`, `vantage-render-resume.service`, destination `IR-Range2-M1`, NimbusSphere tenant and cloud role `NS-BridgeAdmin`, and the federation certificate thumbprint `80CF6545F7FC53E3E00D868C9315163D80633B96658BB4356496F1E9760A5EF2`. The recovered service was inspected as text only, not enabled or run.

**Screenshot evidence:**

![Reserved-sector extraction and hashes](images/IR-M5-14-20261027-025416.png)

![Byte comparison records 928 changed bytes](images/IR-M5-15-20261027-025431.png)

![Stage-2 extraction identifies loader and campaign](images/IR-M5-16-20261027-025443.png)

![Recovered object hash and bootstrap validation](images/IR-M5-17-20261027-025458.png)

![Recovered configuration and persistence-unit text](images/IR-M5-18-20261027-025507.png)

## 7. Incident timeline

| Time (UTC) | Observed action |
|---|---|
| 2026-10-27 02:43:30.125836 | Media upload recorded in intake telemetry |
| 2026-10-27 02:43:30.138994 | Emergency validation recorded; malicious media accepted |
| 2026-10-27 02:43:30.147706 | Isolated rebuild started |
| 2026-10-27 02:43:30.147721 | Rebuild observation completed; unexpected startup unit found |
| 2026-10-27 02:45:36 | Forensic case collection time recorded |
| 2026-10-27 03:05:54.632873 | Portal containment applied by `neha.raman` |
| 2026-10-27 03:08:58.627132 | Signed Range 2 handoff created |

The recorded timestamps are from lab artifacts and provided screenshots; the listed sequence is not a claim of additional independently observed host activity.

**Screenshot evidence:**

![Database backup and 40-line application timeline](images/IR-M5-20-20261027-025624.png)

## 8. Scope, root cause and impact

**Confirmed scope:** One submitted candidate image with media ID `MED-R1M5-2719`, the `svc.recovery-intake` validation path, the isolated `RBLD-R1M5-2719` rebuild workspace, and the corresponding recovery portal case. No evidence shown establishes execution of this unit on a real production renderer. Relevant connected risk involves federation trust and NimbusSphere cloud identity context referenced by M4 and recovered stage-2 configuration.

**Immediate technical cause:** Additional content in FAT16 VBR slack and reserved sectors was omitted from root-file fingerprint validation. **Enabling control failure:** The legacy emergency intake route explicitly deferred deep VBR/reserved-sector and embedded-stage checks. Potential business effects include persistence reintroduced into recovery images, invalid eradication claims, longer recovery timelines and delayed federation certificate restoration.

**Screenshot evidence:**

![Candidate hash, campaign and persistence indicators correlated](images/IR-M5-19-20261027-025611.png)

## 9. Containment, recovery and verification

At `2026-10-27T03:05:54.632873+00:00`, the authorized portal recorded containment action `RESP-R1M5-2719` by `neha.raman` and reported quarantining candidate media, suspending automated recovery and intake, staging the approved baseline as a trusted rebuild, and validating certificate-chain readiness in the simulator.

| Setting | Post-containment value |
|---|---|
| `automated_recovery_suspended` | `1` |
| `intake_enabled` | `0` |
| `recovery_execution_enabled` | `0` |
| `media_quarantined` | `1` |
| `trusted_rebuild_completed` | `1` |
| `certificate_chain_validated` | `1` |
| `/health` `recovery_suspended` | `true` |

The quarantined media retained the candidate SHA-256. `compare_media.py` confirmed the trusted rebuild had matching filesystem fingerprint, an empty changed-sector array and no bootstrap record. The trusted rebuild's reported SHA-256 matched the approved media baseline. The official `validate-resolution.sh` returned:

```text
[PASS] containment, trusted rebuild, certificate-chain validation and Range 2 evidence handoff verified
```

This certifies the **training scenario resolution**, not real-world credential rotation or production trust replacement.

**Screenshot evidence:**

![Initial pre-containment check finds no response record or quarantined media](images/IR-M5-22-20261027-025652.png)

![Pre-containment settings show recovery execution still enabled](images/IR-M5-23-20261027-025811.png)

![Recovery Assurance portal sign-in](images/IR-M5-24-20261027-030013.png)

![Recovery portal overview before containment](images/IR-M5-25-20261027-030039.png)

![Repeated pre-action health and settings check](images/IR-M5-26-20261027-030048.png)

![Candidate quarantine remains absent before the response action](images/IR-M5-27-20261027-030414.png)

![Authenticated recovery response controls](images/IR-M5-28-20261027-030541.png)

![Confirmation to quarantine media and suspend automated recovery](images/IR-M5-29-20261027-030553.png)

![Portal reports containment applied](images/IR-M5-30-20261027-030616.png)

![Containment record and recovery settings verified](images/IR-M5-31-20261027-030649.png)

![Quarantined-media hash and suspended recovery health](images/IR-M5-32-20261027-030701.png)

![Trusted rebuild comparison has no changed sectors or bootstrap](images/IR-M5-33-20261027-030823.png)

![Trusted rebuild matches the approved filesystem fingerprint](images/IR-M5-34-20261027-030831.png)

![Post-containment evidence preserved](images/IR-M5-35-20261027-030844.png)

![Official scenario resolution validator passes](images/IR-M5-39-20261027-030931.png)

## 10. Signed Range 2 handoff

A restricted stage-2 configuration copy was placed under `/var/lib/vantage-recovery/extracted/CASE-R1M5-2719`. The official `build_range2_handoff.py` created `/var/lib/vantage-recovery/handoff/IR-Range2-M1-Federation-Handoff.json`; `verify_handoff.py` returned `status: verified` for ID `HDO-R1-R2-2026-0627`. The handoff bound the M4 intake, quarantined source media hash, recovered stage-2 configuration, target NimbusSphere and destination `IR-Range2-M1`.

The handoff's **internal** verification hash (as recorded in the signed JSON) was `bf2fcbc5b9a87e7ccd8929559ffd6498cd76ee8301173454ead77a44f545558a`. The separate filesystem SHA-256 of the completed JSON was `5fc2bb0e14fcf3650b5cb669d20e253d6265cfc36fff49d75881b4c30d52fc9b`. These represent different hash scopes and should not be conflated.

**Screenshot evidence:**

![Verified stage-2 configuration staged for signed handoff](images/IR-M5-36-20261027-030855.png)

![Range 2 handoff created](images/IR-M5-37-20261027-030906.png)

![Handoff signature, identifiers and filesystem hash verified](images/IR-M5-38-20261027-030921.png)

## 11. Final evidence packaging

The Range 2 case subfolder held six evidence JSON files with validated SHA-256 checks, plus its manifest. The full M5 case manifest verified **57 files with no reported hash failures**. Both compressed archives passed `.sha256` checks and gzip integrity tests:

| Archive on VM | Size | SHA-256 |
|---|---:|---|
| `/var/tmp/IR-Range1-M5-Evidence.tar.gz` | 59 KB | `0c623f86342608e8c4f610255654d4357555166d91afb0d9b24a62b8eca7834` |
| `/var/tmp/IR-Range2-M1-Handoff.tar.gz` | 2.4 KB | `0e82eed4c78647d712075f171f7a66ab53a9d428a7aea750a8fd4ec2ca86aa24` |

These original `.tar.gz` archives are **not included in the screenshot submission ZIP**. Copy them, along with their `.sha256` files, to controlled external evidence storage and verify the transferred bytes before any VM reset.

**Screenshot evidence:**

![Range 2 handoff evidence manifest verified](images/IR-M5-40-20261027-031328.png)

![Final case manifest verifies 57 files](images/IR-M5-42-20261027-031355.png)

![Evidence and handoff archives pass SHA-256 and gzip checks](images/IR-M5-43-20261027-031408.png)

## 12. Corrective actions and lessons learned

1. Retire or disable the incomplete `REC-EMERGENCY-LEGACY` intake profile until a reviewed replacement is active.
2. Require approved whole-image SHA-256, not just matching visible root-file fingerprints.
3. Inspect all VBR slack, reserved sectors and other non-filesystem image regions.
4. Preserve both the original compressed submission and its decompressed raw image.
5. Alert on identical filesystem fingerprints with differing whole-media hashes.
6. Require two-person authorization for emergency recovery media acceptance and bind media serials to approved hashes.
7. Rotate or disable the affected intake service identity and invalidate sessions following formal incident review.
8. Coordinate federation trust material replacement or revocation and NimbusSphere identity/session review with responsible organizations; these are **outstanding**, not proven complete by the lab validator.
9. Keep automated recovery and intake suspended pending remediation approval.
10. Retain chain-of-custody and hash-verified cross-range handoffs, including their source evidence and signature verification.

