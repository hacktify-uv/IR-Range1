# IR-Range1 M3 — Blue Team Incident Response Report

**Scenario:** Obfuscated API Resolution  
**Organization:** Vantage Media & Trust  
**Incident node:** `VMT-RND-03`  
**System:** Ubuntu 24.04.5 LTS, glibc 2.39  
**Exercise date:** 26 October 2026  
**Case evidence directory:** `/var/tmp/IR-Range1-M3-20261026T161947Z`  
**Campaign marker:** `MDRN-7F3A`  
**Status:** Technical investigation completed; local containment and hardening verified; final trust remediation outstanding.

> **Basis and limits.** This report follows the repository's official `M3/Blue-Writeup.md` (https://github.com/hacktifytechnologies/IR-Range1/blob/main/M3/Blue-Writeup.md), using the user's terminal outputs and screenshots. Captions describe the observed evidence. It is not a claim that the VM archives themselves were uploaded into this ZIP. The forensic `.tar.gz` archives still need verified off-VM transfer.

## 1. Executive summary

A pre-staged maintenance identity, `svc.render-maint`, authenticated to the Vantage Render Fleet Assurance portal, uploaded `vmt-render-agent-update.bin` to `VMT-RND-03`, and executed it through the maintenance API. The original x86-64 ELF successfully ran after the target environment was upgraded to Ubuntu 24.04.5 LTS, satisfying its glibc 2.38 dependency. Nuclei found one match and the official `validate-attack.sh` reported a pass.

The sample resolved 18 API names using DJB2-style hashing, initiated ptrace-style process modification, ran a memfd-backed helper, read the training federation signing certificate and private key, staged a federation bundle, and emitted a synthetic pseudo-QUIC record for the M4 handoff. These actions were confirmed by application logs, process evidence, recovered memory-backed ELF, JSON evidence, database records, and a training PCAP.

The Blue Team acquired volatile and persistent evidence, isolated `VMT-RND-03`, stopped the suspicious children, removed unauthorized live artifacts, restricted signing-key file permissions and removed the service's unnecessary federation group. The portal stayed healthy with maintenance disabled. Key/certificate replacement, credential rotation and maintenance-workflow hardening are documented as **not completed**.

## 2. Verified identifiers

| Indicator | Value |
|---|---|
| Target | `VMT-RND-03` |
| Service | `vantage-fleet-assurance.service` |
| Portal | TCP `8082` |
| Evidence collector | `127.0.0.1:9443/UDP` |
| Portal analyst | `sana.kapoor` |
| Maintenance identity | `svc.render-maint` |
| Package ID | `PKG-R1M3-8821` |
| Execution ID | `EXEC-R1M3-4421` |
| Uploaded ELF SHA-256 | `ad84ce682067bc52ecf839fbb471264ee8941dda1cf46751f6161ac4e253f8dd` |
| Acquired memfd ELF SHA-256 | `063dc394f7d30adcdc834d372dfcac9146b3e28ef0be1cd1f9734c0967e6701c` |
| Federation certificate SHA-256 thumbprint | `80CF6545F7FC53E3E00D868C9315163D80633B96658BB4356496F1E9760A5EF2` |
| ALPN marker | `vmt-sync/7f3a` |
| Intended destination | `198.51.100.42:443` |
| Handoff token | `R1M4-HANDOFF-4421` |
| Original M2 handoff case | `CASE-R1-M2-4417` |
| M2 handoff ELF SHA-256 | `8c7307d9cfd0c93ff88dc78fed58d46178e2050a88729810459d1784455ee8e3` |

## 3. Environment preparation and initial identification

The M3 application, database, baseline maintenance packages, service identity, certificate prerequisites, APIs and clean-state checks passed `validate-setup.sh`. The original attack initially failed on an Ubuntu 22.04 VM because its ELF required glibc `2.38`, while the VM supplied `2.35`. The exercise was subsequently run successfully on the upgraded Ubuntu 24.04.5 system with glibc `2.39`, without modifying the original payload bytes.

The M2 evidence-intake JSON connected the earlier renderer incident (`VMT-RND-02`) to the M3 response. Investigators inventoried application JSONL telemetry, the SQLite database and WAL files, uploads, runtime artifacts, cache, and the M4 handoff folder. The M2 handoff JSON observed on M3 had SHA-256 `7af7c1f8ecae2276f9c71b46e084fa8d7b81c77d57a5babc2da0128e6f7b8645`. This is the installed handoff file hash, not proof of an off-VM backup of M2.

**Screenshot evidence:**

![Preparation and M2 handoff (capture 1)](images/M3-01-161849.png)

![Preparation and M2 handoff (capture 2)](images/M3-02-161917.png)

## 4. Attack validation and detection

The original template `IR-Range1-M3-Attack-Template.yaml` was validated with Nuclei 3.11.1. The simulated incident used the original embedded package, authenticated, uploaded the ELF, and invoked `/api/maintenance/run`. Nuclei reported **1 match** and `validate-attack.sh` reported:

`[PASS] IR-Range1-M3 live attack, ELF behavior, volatile process evidence, certificate access and Machine 4 network handoff validated`

The package uploaded without completed signature evaluation. Database and audit logs showed an execution status `Completed` and exit code `0`. For incident triage, this is a critical unapproved-execution and federation-trust exposure scenario.

## 5. Volatile evidence acquisition

The case directory `/var/tmp/IR-Range1-M3-20261026T161947Z` was created before containment. The official `collect_live_response.py` collector captured process information, maps, descriptors, sockets, relevant file copies and a collection manifest. Two suspicious children were observed:

- PID `3931`, `vmt-preview`, from the uploaded maintenance executable;
- PID `3932`, `vmt-memfd`, command line `vantage-preview-worker`.

An anonymous `rwxp` executable memory mapping was captured from the first child. The second exposed `/memfd:vmt-preview-cache (deleted)` through `/proc/3932/fd/3`. Investigators extracted the memfd-backed ELF from `/proc/3932/exe` and hashed the acquired copy as `063dc394f7d30adcdc834d372dfcac9146b3e28ef0be1cd1f9734c0967e6701c`. Socket state, interfaces and routes were preserved prior to isolation.

**Screenshot evidence:**

![Volatile process and memory evidence (capture 3)](images/M3-03-162000.png)

![Volatile process and memory evidence (capture 4)](images/M3-04-162015.png)

![Volatile process and memory evidence (capture 5)](images/M3-05-162033.png)

![Volatile process and memory evidence (capture 6)](images/M3-06-162048.png)

## 6. Persistent forensic preservation

The original uploaded ELF and its file identification, application and journal logs, runtime JSON/PID artifacts, restricted federation staging bundle, M4 JSON and PCAP, and an SQLite backup were copied to the forensic case. SQLite `PRAGMA integrity_check` returned `ok`. Auditd records were exported under five M3 rule keys: maintenance upload, runtime, staging, federation material and network handoff.

An initial case checksum manifest covered 59 entries; after investigation and packaging, the verification output reported 100 checked files and no hash mismatches. The final manifest was regenerated after inclusion of the verification output.

**Screenshot evidence:**

![Persistent evidence acquisition and audit records (capture 7)](images/M3-07-162134.png)

![Persistent evidence acquisition and audit records (capture 8)](images/M3-08-162155.png)

![Persistent evidence acquisition and audit records (capture 9)](images/M3-09-162213.png)

![Persistent evidence acquisition and audit records (capture 10)](images/M3-10-162229.png)

![Persistent evidence acquisition and audit records (capture 11)](images/M3-11-162248.png)

![Persistent evidence acquisition and audit records (capture 12)](images/M3-12-162431.png)

## 7. Static ELF analysis

The uploaded sample was identified as a stripped, PIE, dynamically linked 64-bit x86-64 ELF, including a custom `.vmt_hashes` section. The preserved sample was analyzed using `readelf`, `objdump`, and `/opt/vantage-fleet-assurance/tools/analyze_api_hashes.py`.

The analyzer returned **18 resolved API functions** using `djb2-32` hashing:

`open`, `read`, `write`, `close`, `memfd_create`, `fchmod`, `fexecve`, `mmap`, `fork`, `sleep`, `socket`, `sendto`, `ptrace`, `waitpid`, `pipe`, `lseek`, `mkdir`, `chmod`.

The runtime telemetry additionally recorded `algorithm=djb2 xor_key=0x5a`. Static API evidence indicates intended capabilities, while the collected live processes, runtime telemetry and staged bundle confirm the exercised behaviors.

**Screenshot evidence:**

![Static ELF and API-hash analysis (capture 13)](images/M3-13-162443.png)

![Static ELF and API-hash analysis (capture 14)](images/M3-14-162459.png)

![Static ELF and API-hash analysis (capture 15)](images/M3-15-162512.png)

![Static ELF and API-hash analysis (capture 16)](images/M3-16-162519.png)

## 8. Runtime, memory and federation exposure

The implant events recorded `api_hash_resolution`, `credential_material_access`, `ptrace_memory_write`, `memfd_execution`, `network_handoff` and `implant_completed`, attributed to campaign `MDRN-7F3A`. The helper-stage JSON identified PID `3932` and name `vmt-preview-cache`.

The certificate access JSON recorded event `federation_certificate_accessed`, matching the training certificate thumbprint, with 1,391 certificate bytes and 1,704 key bytes. The `.federation.bundle` contained both certificate and private-key markers. Sensitive key contents were not printed to the chat or report.

Before remediation, the service identity `vantage-agent` belonged to `federation-audit`, and `/etc/vantage-federation/private/saml-signing.key` was owned by `root:federation-audit` with mode `640`. This granted the application service unnecessary read access to federation signing material.

**Screenshot evidence:**

![Runtime and federation exposure evidence (capture 17)](images/M3-17-162654.png)

![Runtime and federation exposure evidence (capture 18)](images/M3-18-162705.png)

![Runtime and federation exposure evidence (capture 19)](images/M3-19-162716.png)

![Runtime and federation exposure evidence (capture 20)](images/M3-20-162731.png)

## 9. Network and Machine 4 handoff

The network collector recorded a controlled `pseudo_quic_datagram` at `127.0.0.1`, carrying `ALPN=vmt-sync/7f3a`, `DST=198.51.100.42:443`, campaign `MDRN-7F3A`, the known certificate fingerprint and token `R1M4-HANDOFF-4421`.

The handoff inspector validated the M4 JSON and the 265-byte synthetic PCAP, reporting at least one packet record. **The PCAP is simulated handoff evidence, not proof of public-network exfiltration.**

**Screenshot evidence:**

![Controlled network and M4 handoff evidence (capture 21)](images/M3-21-162746.png)

![Controlled network and M4 handoff evidence (capture 22)](images/M3-22-162759.png)

## 10. Timeline and scoping (UTC, 26 October 2026)

| Timestamp | Event |
|---|---|
| `16:17:05.522` | `svc.render-maint` login success |
| `16:17:05.535` | Package upload recorded; `PKG-R1M3-8821` |
| `16:17:05.539` | Execution started on `VMT-RND-03` |
| `16:17:05` | API resolution, key access, ptrace and memfd activity |
| `16:17:05.546` | Synthetic pseudo-QUIC handoff logged |
| `16:17:05.549` | Execution complete, exit code `0` |
| `16:19:47` | Case directory and evidence acquisition initiated |
| `16:33:23.003764` | Isolation marker recorded by `sana.kapoor` |

Filesystem and indicator searches identified the live upload, runtime artifacts, staged bundle, known repository payload and the investigator's case copies. The original process search found the two suspect children. Findings support the affected M3 node; other fleet nodes were not independently inspected.

**Screenshot evidence:**

![Timeline and affected-scope investigation (capture 23)](images/M3-23-162844.png)

![Timeline and affected-scope investigation (capture 24)](images/M3-24-162900.png)

![Timeline and affected-scope investigation (capture 25)](images/M3-25-162914.png)

![Timeline and affected-scope investigation (capture 26)](images/M3-26-162933.png)

![Timeline and affected-scope investigation (capture 27)](images/M3-27-162945.png)

![Timeline and affected-scope investigation (capture 28)](images/M3-28-163000.png)

## 11. Containment

Using the Fleet Assurance portal's Response Controls, analyst `sana.kapoor` selected **Isolate VMT-RND-03** after acquiring volatile evidence. An earlier verification correctly revealed maintenance still enabled, prompting completion of the portal action. A subsequent verification showed `maintenance_enabled: false`, the `maintenance.disabled` marker, `node_isolated` audit event and no running suspicious children. The isolation time was `2026-10-26T16:33:23Z` UTC.

**Screenshot evidence:**

![Initial verification before portal containment (capture 29)](images/M3-29-163113.png)

![Initial verification before portal containment (capture 30)](images/M3-30-163129.png)

![Authenticated portal containment and verification (capture 31)](images/M3-31-163241.png)

![Authenticated portal containment and verification (capture 32)](images/M3-32-163316.png)

![Authenticated portal containment and verification (capture 33)](images/M3-33-163357.png)

![Authenticated portal containment and verification (capture 34)](images/M3-34-163411.png)

![Authenticated portal containment and verification (capture 35)](images/M3-35-163424.png)

## 12. Eradication, access hardening and recovery

With copies confirmed inside the case directory, investigators removed the live unauthorized uploaded ELF, `.federation.bundle` and incident runtime JSON/PID artifacts. They removed `vantage-agent` from `federation-audit`, changed the original signing-key ownership to `root:root`, set mode `600`, removed the `SupplementaryGroups=federation-audit` service directive and reloaded systemd.

After restarting the portal, the service was active as `vantage-agent`, the running process showed only its normal primary group, no suspicious processes remained, the unauthorized live artifacts were absent, and database integrity returned `ok`. The health endpoint stayed `ok` with `maintenance_enabled: false`.

**Important limitation:** Restricting the still-existing exposed training key is not cryptographic key rotation. The exposed signing key was not reported as replaced/revoked, and the maintenance identity and insecure workflow were not reported as fully remediated.

**Screenshot evidence:**

![Eradication and service-account access controls (capture 36)](images/M3-36-163546.png)

![Eradication and service-account access controls (capture 37)](images/M3-37-163558.png)

![Eradication and service-account access controls (capture 38)](images/M3-38-163609.png)

![Eradication and service-account access controls (capture 39)](images/M3-39-163700.png)

![Eradication and service-account access controls (capture 40)](images/M3-40-163748.png)

![Recovery verification (capture 41)](images/M3-41-163942.png)

![Recovery verification (capture 42)](images/M3-42-163950.png)

![Recovery verification (capture 43)](images/M3-43-164004.png)

![Recovery verification (capture 44)](images/M3-44-164019.png)

## 13. Post-incident activity and outstanding actions

Recommended follow-up actions: replace and retire the exposed federation signing certificate/key, rotate `svc.render-maint` credentials and invalidate sessions, enforce signed and approved maintenance packages, isolate execution into a sandbox, restrict service-account permissions, and alert on memfd execution, anonymous RWX mappings, ptrace behavior, and relevant IOCs.

The recorded post-incident summary, chain-of-custody text, case integrity manifest, M4 handoff package and verified archives were prepared. The chain-of-custody record explicitly states **external transfer not yet verified**.

**Screenshot evidence:**

![Post-incident records (capture 45)](images/M3-45-164107.png)

![Post-incident records (capture 46)](images/M3-46-164117.png)

## 14. Final on-VM archives and hashes

| Archive | Approximate size | SHA-256 | Verified on VM |
|---|---:|---|---|
| `IR-Range1-M3-Evidence.tar.gz` | 599 KB | `878d8db28ab77a4503cfb15f801feacc653f97da115f99646a24a3b37718b897` | Yes |
| `IR-Range1-M4-Handoff.tar.gz` | 1.7 KB | `1b53dd48ada890ca83a80066b97457b6869d15752cae1a941bff8c05e1d0fbe0` | Yes |

Both accompanying `.sha256` files and `gzip -t` checks passed. **These `.tar.gz` archives are on the user's VMware system and are not included in this documentation ZIP.** Verify copies outside the VM before resetting M3.

**Screenshot evidence:**

![Final evidence and handoff packaging (capture 47)](images/M3-47-164133.png)

![Final evidence and handoff packaging (capture 48)](images/M3-48-164145.png)

![Final evidence and handoff packaging (capture 49)](images/M3-49-164157.png)

![Final evidence and handoff packaging (capture 50)](images/M3-50-164210.png)

![Final evidence and handoff packaging (capture 51)](images/M3-51-164246.png)

## 15. Source and report qualification

Official scenario procedure: [M3 Blue-Writeup.md](https://github.com/hacktifytechnologies/IR-Range1/blob/main/M3/Blue-Writeup.md). The account, tool outputs, timestamps, hashes and successful local validation results above are taken from the user's M3 lab transcript and screenshots. No unobserved production remediation or external archive transfer is asserted.
