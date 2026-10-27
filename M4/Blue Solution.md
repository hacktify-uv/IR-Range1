# IR-Range1 M4 — Blue Team Incident Response Writeup

**Scenario:** QUIC Protocol ALPN Obfuscation  
**Exercise:** Silent Meridian Range 1, Machine 4  
**Organization:** Vantage Media & Trust  
**Analyst identity:** `isha.menon` (portal response action)  
**VM hostname:** `ir-range1-m3` (VM reused for M4)  
**Gateway under investigation:** `VMT-EGW-01`  
**Case:** `CASE-R1M4-7431`  
**Forensic case path:** `/var/tmp/IR-Range1-M4-CASE-R1M4-7431`  
**Reference:** https://github.com/hacktifytechnologies/IR-Range1/blob/main/M4/Blue-Writeup.md

> This document records evidence actually displayed in the investigation screenshots and conversation. The complete on-VM forensic `.tar.gz` archives are not included in this report-and-screenshots ZIP; their hashes are recorded below. Controlled training-network packet evidence does not establish real Internet exfiltration. External evidence transfer has not been verified.

## 1. Executive summary

On 26 October 2026, the M4 egress gateway's emergency diagnostic workflow allowed a submitted profile to create a 13-packet UDP/443 session from the render-production source `10.44.18.23` (`VMT-RND-03`) to the lab documentation address `198.51.100.42:443`. The session used custom ALPN `vmt-sync/7f3a`, which deviated from the policy-approved `h3`. The network control allowed it using exception `EXC-UDP-1187` despite missing signature, destination, and ALPN enforcement. The proxy-bypass alert, firewall record, QUIC sensor, database and capture agree on this session.

The supplied forensic parser reconstructed the simulated protected content from the packet capture and keylog, yielding a federation certificate matching the Machine 3 thumbprint, cloud federation context, and a command-and-response transcript. After preserving the evidence, analyst `isha.menon` used the portal's **Block direct egress** control. The portal recorded `egress_path_blocked`, disabled diagnostics, and set `EXC-UDP-1187` to **Suspended pending review**. The gateway portal and evidence collector remained operational.

## 2. Preparation and controlled attack validation

The target installation successfully passed `validate-setup.sh`. The portal served TCP `8083` and the local evidence collector listened on UDP `127.0.0.1:9444`. The initial health endpoint showed `diagnostics_enabled:true` and `live_capture_present:false`. The target was Ubuntu 24.04.5 LTS with synchronized time. The official M3 handoff JSON matched expected SHA-256 `b75800cefadbd7468774cebe672c4ddfc80ebc3105b68109359ef6d1f1aed430`, and the approved baseline PCAP matched `e7aaa2d56790121b312043cc7a4479745c3ab68d1f78c8e893dc485e4fa61ce8`.

Nuclei v3.11.1 validated the original attack template (duplicated as `.yaml` to avoid earlier `.yml` loading problems). It returned one critical match at `/api/diagnostics/run`. The official `validate-attack.sh` returned **PASS** for live attack, proxy bypass, custom ALPN session, packet reconstruction and Machine 5 handoff. Session `7f3a4c21190e88b2` and diagnostic execution `EXEC-R1M4-7431` were generated.

**Screenshot evidence:**

![System identity, synchronized time, service and listening ports](images/IR-M4-01-20261026-170652.png)

![Operations brief and M3 handoff with SHA-256](images/IR-M4-02-20261026-170705.png)

![Approved capture and baseline proxy events](images/IR-M4-03-20261026-170727.png)

## 3. Detection and incident declaration

At `2026-10-26T17:05:40.408286+00:00`, the proxy-bypass log recorded source `10.44.18.23`, destination `198.51.100.42:443`, UDP, observed ALPN `vmt-sync/7f3a`, expected ALPN `h3`, exception `EXC-UDP-1187` and high severity. The corresponding firewall event recorded `ALLOW` and 3,115 bytes. Sensor telemetry reported 13 datagrams. The incident was assessed as **High** severity due to unapproved transport and reconstructed identity trust context.

**Screenshot evidence:**

![Proxy-bypass alert, firewall allow and QUIC sensor events](images/IR-M4-05-20261026-171009.png)

## 4. Evidence preservation and chain of custody

Forensic evidence was preserved under the restricted case directory with `originals`, `working`, `commands` and `reports`. Copies included the live PCAP, SSL keylog, encrypted stream, session summary, diagnostic profile, policy file, six application log streams, M5 recovery handoff, system/network state, auditd exports, and consistent SQLite backup. The SQLite integrity check returned `ok`. A chain-of-custody record identified the local root forensic session and states that off-VM transfer was not yet performed.

The original-evidence SHA-256 verification passed for **18/18** files. The final case manifest later verified **52/52** files with no reported mismatches. Both full-case and dedicated M5 archives passed SHA-256 validation and gzip integrity checks.

**Screenshot evidence:**

![Restricted case directory and system-state collection](images/IR-M4-06-20261026-171030.png)

![Preserved policy, captures, logs and handoff inventory](images/IR-M4-07-20261026-171048.png)

![Original evidence inventory and checksum manifest](images/IR-M4-08-20261026-171235.png)

![Consistent database backup and integrity check](images/IR-M4-09-20261026-171247.png)

![Auditd evidence exports](images/IR-M4-10-20261026-171301.png)

![All 18 original evidence hashes verified](images/IR-M4-11-20261026-171315.png)

![Local collector and chain-of-custody record](images/IR-M4-12-20261026-171326.png)

![Preserved capture, keylog, reconstruction and hash verification](images/IR-M4-25-20261026-172544.png)

## 5. Network forensic analysis

The incident PCAP was confirmed as classic little-endian Ethernet PCAP. `tcpdump` showed **13 UDP packets** from `10.44.18.23:53144` to `198.51.100.42:443`; 12 packets carried 251 bytes and the last 103 bytes at the UDP payload summary level. The approved baseline used a different source, destination and permitted diagnostic protocol profile. The baseline proxy log explicitly showed UDP/443 ALPN `h3` under exception `EXC-UDP-1187`.

The M4 QUIC sensor identified a single 13-chunk session (`7f3a4c21190e88b2`), ALPN `vmt-sync/7f3a`, and server name `update-checker.net`. The parser `/opt/vantage-egress-assurance/tools/parse_quic_capture.py` successfully reconstructed objects from preserved working copies of the PCAP and keylog, including `saml-signing.crt`, `federation-context.json` and `7f3a4c21190e88b2-decoded.json`.

The certificate's SHA-256 fingerprint was `80CF6545F7FC53E3E00D868C9315163D80633B96658BB4356496F1E9760A5EF2`, matching the M3 handoff. The federation context contained a NimbusSphere tenant `9f3a2c10-6b77-4f2e-a821-0d44a5b87019` and role alias `NS-BridgeAdmin`. Session metadata included a forged token fragment and control messages. Do not disclose full token/secret values outside restricted case files.

**Observed control-message sequence:** server `whoami` request; client `vantage-agent` response; server `collect federation trust context` request; client `certificate and federation context prepared` response. This is evidence of the laboratory's simulated command channel and transfer of identity context, not proof of external Internet delivery.

**Screenshot evidence:**

![Incident PCAP identification and 13 UDP packets](images/IR-M4-13-20261026-171635.png)

![Approved baseline compared with incident packet capture](images/IR-M4-14-20261026-171647.png)

![QUIC sensor sequence and session metadata](images/IR-M4-15-20261026-171706.png)

![Reconstructed session objects and simulated identity context](images/IR-M4-16-20261026-171726.png)

![Certificate fingerprint, federation context and control messages](images/IR-M4-17-20261026-171742.png)

![Cross-log egress correlation report](images/IR-M4-18-20261026-172055.png)

## 6. Application and policy root cause

The diagnostic profile had SHA-256 `23b53aece8435c41252854fbee2db2d0bce6e135d905abb1a0be421f7c9f6444`. It specified `NET-DIAG-R1M4-7431`, `CHG-NET-7431`, domain `update-checker.net`, IP `198.51.100.42`, destination UDP/443, custom ALPN `vmt-sync/7f3a` and session `7f3a4c21190e88b2`.

Exception `EXC-UDP-1187` was intended for renderer health diagnostics with permitted ALPN `h3`, approved diagnostic destinations and change reference `CHG-2025-1187`. Policy `enforcement:metadata-only` and a documented control gap meant the emergency profile runner did not verify profile signatures, destination allowlists or permitted ALPN values. The recorded profile result stated acceptance under the emergency diagnostics exception with signature not evaluated. Identity `svc.edge-diagnostics` uploaded and executed the profile and the execution completed, producing 13 packets.

**Screenshot evidence:**

![Egress policy and metadata-only diagnostic exception](images/IR-M4-04-20261026-170740.png)

![Submitted diagnostic profile and allowed policy exception](images/IR-M4-19-20261026-172117.png)

![Diagnostic execution, transport session and database events](images/IR-M4-20-20261026-172141.png)

## 7. Event timeline (UTC)

| Event | Timestamp |
|---|---|
| `svc.edge-diagnostics` login success | `2026-10-26 17:05:40.392984` |
| Diagnostic profile uploaded | `2026-10-26 17:05:40.400978` |
| Execution started (database) | `2026-10-26 17:05:40.407021` |
| DNS response, firewall allow, proxy bypass | `2026-10-26 17:05:40.408286` |
| First QUIC-like packet observed | `2026-10-26 17:05:40.408469` |
| Last QUIC-like packet observed | `2026-10-26 17:05:41.134330` |
| Transport session completed | `2026-10-26 17:05:41.405053` |
| Profile execution audited | `2026-10-26 17:05:41.407762` |
| Portal analyst blocked egress | `2026-10-27 02:19:48.268709` |

The host's local timezone was Asia/Kolkata, +05:30. The source hostname still carried the M3 lab name because the same VM was reused.

**Screenshot evidence:**

![Chronologically ordered application and network events](images/IR-M4-23-20261026-172430.png)

## 8. Scope and impact

One high-risk session was found by the preserved database query filtering for non-`h3` ALPN or proxy bypass. The source was `VMT-RND-03` at `10.44.18.23`, monitored through gateway `VMT-EGW-01`. Cross-log searches matched `vmt-sync/7f3a`, `198.51.100.42`, and `update-checker.net` in the relevant preserved portal, firewall, proxy, DNS and sensor records. This confirms the observed local scope, not the absence of related incidents outside the collected M4 telemetry.

Reconstructed content showed the M3-linked federation certificate and cloud identity context. The M5 handoff identifies campaign `MDRN-7F3A` and records the tenant and role alias for follow-on investigation.

**Screenshot evidence:**

![Recorded network findings before containment](images/IR-M4-21-20261026-172212.png)

![Network findings identifying the affected gateway and session](images/IR-M4-22-20261026-172249.png)

![Related-session query and indicator searches](images/IR-M4-24-20261026-172500.png)

## 9. Containment and recovery validation

After acquisition and verification, `isha.menon` selected **Response → Block direct egress** in the authenticated M4 portal. A browser confirmation was accepted. The portal then displayed **Direct egress blocked**. The runtime `egress.blocked` marker recorded `blocked_by=isha.menon` and timestamp `2026-10-27T02:19:48.265187+00:00`. The portal audit event `egress_path_blocked` was timestamped `2026-10-27T02:19:48.268709+00:00`, referencing `EXC-UDP-1187`, the custom ALPN and destination.

After containment, the active exception `EXC-UDP-1187` was **Suspended pending review** while unrelated `EXC-NTP-0442` remained Active. The portal returned health `ok`, `diagnostics_enabled:false`, `udp_collector:true`, and `live_capture_present:true`. The systemd service remained active on TCP 8083 and loopback UDP 9444. Containment marker, audit and health results were copied into the case reports.

**Screenshot evidence:**

![Initial check shows containment was not applied](images/IR-M4-26-20261026-172618.png)

![Initial containment evidence collection reports a missing marker](images/IR-M4-27-20261026-172633.png)

![Repeated collection attempt still reports no containment marker](images/IR-M4-28-20261026-172851.png)

![Pre-action status shows diagnostics enabled and exception active](images/IR-M4-29-20261027-021317.png)

![Gateway portal overview before the response action](images/IR-M4-30-20261027-021332.png)

![Verification correctly fails before containment](images/IR-M4-31-20261027-021412.png)

![Unprivileged console cannot read protected response evidence](images/IR-M4-32-20261027-021705.png)

![Privileged verification confirms containment is still pending](images/IR-M4-33-20261027-021805.png)

![Authenticated portal response control before action](images/IR-M4-34-20261027-021937.png)

![Portal confirmation for suspending direct UDP diagnostics](images/IR-M4-35-20261027-021949.png)

![Portal displays direct egress blocked](images/IR-M4-36-20261027-021956.png)

![Audit event records egress blocked and health reports diagnostics disabled](images/IR-M4-37-20261027-022011.png)

![Containment marker and suspended exception confirmed](images/IR-M4-38-20261027-022202.png)

![Post-containment response evidence saved](images/IR-M4-39-20261027-022213.png)

![Recovery health, exception state, preserved capture and database integrity](images/IR-M4-41-20261027-022423.png)

## 10. Eradication review and outstanding remediation

The submitted profile remains present on the VM pending approval for deletion, as required by the lab writeup. Direct egress is contained but long-term remediation is **not complete**. Actions outstanding: cryptographic signing and verification of diagnostic profiles; allowed ALPN/destination enforcement; source/destination/expiration restrictions; two-person approval; rejection of embedded unauthorized data; detection on UDP/443 proxy bypass; diagnostic-service credential rotation and session invalidation; retirement of exposed federation signing material; NimbusSphere cloud identity review; and historical audit of `EXC-UDP-1187`.

Do not reactivate the exception or run `reset.sh` before authorized remediation and off-VM evidence transfer.

**Screenshot evidence:**

![Submitted profile retained while service and collector remain active](images/IR-M4-40-20261027-022230.png)

![Verified controls and outstanding remediation recorded](images/IR-M4-43-20261027-022451.png)

![Remediation-status report prepared](images/IR-M4-44-20261027-022502.png)

## 11. Verified final case and archive integrity

| File on VM | Size (approx.) | SHA-256 | Verification |
|---|---:|---|---|
| `/var/tmp/IR-Range1-M4-Evidence.tar.gz` | 42 KB | `a679c0a84104e702bbe8fa9412eb2e6fffaad5920e8fda4dda19087636054ff0` | `sha256sum -c`: OK; `gzip -t`: passed |
| `/var/tmp/IR-Range1-M5-Handoff.tar.gz` | 7.3 KB | `4958690fddfe2c6a300eb05c8c2eb53bdaf223272b74156a56d6a1cca7772f6f` | `sha256sum -c`: OK; `gzip -t`: passed |

The dedicated M5 handoff contains seven verified files, including the M5 recovery JSON, M4 live PCAP, keylog, session reconstruction report, egress correlation, network findings and remediation status. Its own SHA-256 manifest was verified. **Neither archive is inside this ZIP**; the VM's archives must be copied and verified independently.

**Screenshot evidence:**

![M5 handoff hash and reconstructed session objects](images/IR-M4-42-20261027-022436.png)

![Intermediate case manifest verifies 44 files](images/IR-M4-45-20261027-022745.png)

![Seven-file M5 handoff manifest verified](images/IR-M4-46-20261027-022801.png)

![Final case manifest verifies 52 files](images/IR-M4-47-20261027-022813.png)

![Full case and M5 handoff archives pass integrity checks](images/IR-M4-48-20261027-022827.png)

![Final archive SHA-256 values and verification](images/IR-M4-49-20261027-023308.png)

## 12. Conclusion

M4 was technically solved: the simulated malicious profile executed and validated; investigators proved the proxy-bypass and custom-ALPN root cause, reconstructed the controlled session, preserved hashed evidence, and applied authenticated portal containment. Full permanent eradication and cloud trust recovery remain open; no off-VM backup has been attested.
