# Cisco ISE 3.5 Patch 3 Admin GUI degradation

Session opened: 2026-08-30  
Status: active investigation  
Primary question: can state associated with the Cisco-enabled Analytics license/API in an ISE 3.3 Patch 11 configuration backup survive migration to ISE 3.5, and can that state cause or contribute to recurring Admin GUI saturation?

## Executive summary

The strongest demonstrated symptom is sustained **administration-plane saturation**, not a general ISE or Hyper-V resource failure. During the incident, Admin GUI pages take seconds to minutes, administration HTTP threads time out or exhaust, `jsvc` consumes roughly two CPU cores, and ISE Prometheus shows a sustained increase of about 10% user CPU. RADIUS and TACACS authentication reportedly remain healthy.

An ISE application restart promptly restores GUI performance. A Data Grid reset/reconfiguration coincided with a longer healthy interval, but user-session and timeout settings were changed at the same time. That event therefore does **not** prove that Data Grid is the cause.

The old Analytics enablement is a credible migration variable worth isolating. Rooted 3.3 inspection shows that enabling it writes application data to existing licensing tables and requests entitlement reporting; it does not execute schema DDL in the enablement path. The completed Analytics-disabled control restore proves that ISE 3.5 runs dedicated miscellaneous-license, monitoring-analytics, and Node Exporter migration handlers. The remaining risk is migrated **enabled data or feature state interacting with a 3.5 consumer or migration defect**, rather than the act of enabling Analytics having altered the 3.3 database schema.

Exact Patch 3 bytecode now supplies a concrete saturation mechanism. The
Application Server's Ignite clients connect to `localhost:10800`; client
creation is serialized by one class-wide lock and can make 10 attempts with
3-second pauses. On a primary Admin node, both the state monitor and event
listener repeat every 30 seconds. A persistent local refusal can therefore
produce thousands of errors and hold Data Grid-dependent GUI requests behind
roughly 27-second retry cycles. Production evidence must still establish why
the local listener becomes unavailable and whether that precedes Kong/Admin
pressure.

The reported 1,572,864-kB OOM limit exactly equals 1,536 MiB. Patch 3 applies
`apigateway.memory` as the Kong container's hard limit, and 1,536 MiB is a real
Patch 3 profile value. The full OOM cgroup/victim and production's active
platform properties are required to determine whether Kong actually hit this
limit and whether ISE selected an unexpected profile for the 32-vCPU VM.

## Affected environment — reported

- Cisco ISE 3.5 Patch 3.
- Three nodes:
  - PPAN / PMNT / PSN.
  - SPAN / SMNT / PSN.
  - PSN.
- The nodes were installed fresh from an ISE 3.5 ISO; they were not upgraded in place from the 3.3 appliances.
- The fresh nodes were brought to ISE 3.5 Patch 3 first. The ISE 3.3 Patch 11 configuration then entered the new 3.5 Patch 3 cluster through a configuration restore.
- The 3.3 source cluster had an uncommon Analytics capability enabled by Cisco with a Cisco-signed JWT-style token.
- Hyper-V under clustered SCVMM with S2D/CSV-backed VHDX.
- ISE VM sizing: 32 vCPU and 64 GB RAM.
- Host: about 80 CPU cores, 256 GB RAM, and two physical NUMA nodes.
- The VM fits inside the configured single vNUMA node limits: 32 vCPU is below 64 and 64 GB is below about 250 GB.
- Golden VHDX was captured before ISE `setup`; each node ran setup independently.

## Incident behavior — reported

1. ISE application restart makes the GUI fast.
2. Around 09:00 on a following workday, GUI navigation becomes extremely slow—sometimes minutes per page.
3. User CPU rises by about 10% and stays elevated for the entire degraded interval.
4. `jsvc` is observed near 200% CPU; a profile attributes about 30% CPU to Java `char[]` processing.
5. Admin thread-pool threshold alerts and read timeouts recur while the GUI is slow.
6. RADIUS and TACACS authentications continue to work.
7. Another ISE application restart clears the condition.

The 09:00 onset coincides with the start of work hours. It may represent user/session/API load rather than a scheduled task. This distinction has not been tested.

## Log and metric evidence — reported

### Application and Admin GUI

- `console.log` contains thousands of entries involving:
  - `admin-http-pool`
  - `java.net.SocketTimeoutException: Read timed out`
  - `CharacterEncodingFilter.doFilter(CharacterEncodingFilter.java:123)`
- ISE raises `Administration thread pool reached threshold` or exhaustion alerts.
- Kong worker-pool errors are also reported during the affected production condition. The exact message, component, and timestamp correlation have not yet been captured.
- Threshold alerts have been seen at roughly 15-minute intervals.
- GUI latency has been observed near 22,000 ms and, in later reports, minutes.

`CharacterEncodingFilter` is a stack location, not proof that character encoding is the root cause. Rooted decompilation resolves line 123 exactly to `FilterChain.doFilter(request, response)`, and Patch 3 maps the filter to every Admin path (`/*`). A deeper operation throws the socket-read timeout and the exception unwinds through this common frame. The waiting read holds an `admin-http-pool` thread busy; enough concurrent waits produce the threshold alert. The 15-minute recurrence may be alarm evaluation cadence rather than a 15-minute initiating event.

### Ignite / Data Grid

- `ise-ignite.log` contains thousands of:
  - `Failed to create Ignite client`
  - `Failed to create Ignite client connection`
  - `Connection refused`
- The exact destination IP and port have not been preserved in the handoff.
- Candidate Data Grid ports include TCP 47500, 47100, and 10800.
- Data Grid reset/reconfiguration restored normal operation, but other settings changed in the same maintenance window.

The documented error ordering is a **reported correlation**, not a proven causal chain. Ignite failure could be a cause, a consequence of application saturation, or a separate retry amplifier.

### Memory

- Linux `messages` includes `nginx invoked oom-killer` with a 1,572,864 kB limit, approximately 1.5 GiB.
- The VM has 64 GB, so this is consistent with a cgroup/process limit rather than guest-wide memory exhaustion.
- The actual `Killed process`, `oom_memcg`, and cgroup-path lines have not been captured.

`nginx invoked oom-killer` identifies the allocator that triggered the kernel path, not necessarily the process selected as the victim.

### Other observations

- `ad_agent.log`: thousands of `failed to write records error code [1]` messages. Exact Patch 3 binaries identify this as PBIS Event Log API failure toward the local eventlog service/database, not a direct Oracle, Ignite, or AD-object write. The lower-level meaning of numeric code `1` remains unresolved.
- `dblock.log`: no persistent database-lock failure; queue locks around eight seconds observed.
- `ise-psc.log`: no obvious slow replication, queue-link, major timeout, or resource failure.
- ISE-generated thread dump was blank or whitespace-only.
- `faillog` did not add useful evidence.
- VHDX read latency was mostly below 1 ms, with a peak around 20 ms and sampled average near 0.05 ms.
- CSVFS activity was roughly 400–1,100 reads/s.
- Other general Prometheus metrics were reportedly normal during the event.

## Analytics implementation — verified locally on rooted ISE 3.3 Patch 11

The private API schema defines:

- `GET /api/v1/license/system/miscellaneous-license`
- `POST /api/v1/license/system/miscellaneous-license`
- License type `ANALYTICS`.
- Actions `ENABLE` and `DISABLE`.
- A JWT token described as being provided by ISE product management.

The enablement handler:

1. Verifies an RSA256 Cisco-signed JWT using a public key stored for the miscellaneous license.
2. Decodes the claim payload.
3. Calls `LicenseSettingsUtil.enableAnalyticsLicense`.
4. Creates or updates Analytics status and claim data in existing tables.
5. Creates or updates an Analytics entitlement and asks the licensing path to report it.

Relevant pre-existing tables:

| Table | Relevant state |
|---|---|
| `UPSLicenseMiscSettings` | tag, enabled status, decoded claim data up to 4,000 characters |
| `UPSEntitlementInfo` | entitlement status, count, and enforcement mode |
| `UPSLicenseMiscKeys` | tag and encoded verification public key |

No schema-creation or schema-alteration DDL was found in the enable/disable path. Enabling changes rows and runtime/licensing behavior, not the table layout.

The configuration-backup Data Pump path exports the CEPM schema and excludes only temporary Data Pump tables. Therefore, Analytics settings, claims, key, and entitlement rows are expected to be transportable in a configuration backup unless the 3.5 restore explicitly filters or transforms them.

Confirmed 3.3 consumers include:

- Licensing service state for `ServiceType.ANALYTICS`.
- `LicenseJSONInfo`, which serializes Analytics count, installed state, and decoded claims into telemetry license information.
- Monitoring export behavior that removes `OPERATION_MESSAGE_TEXT` from `Threat-Events` when Analytics is installed.

This proves persisted state and downstream consumers exist. It does not yet prove that Analytics starts a distinct database engine, uses Ignite, or causes the Admin GUI incident.

The local 3.3 lab currently reports Analytics `DISABLED` and has no Analytics rows in the inspected miscellaneous settings, entitlement, or key tables. It cannot reproduce the enabled-state branch without either an affected backup or a valid Cisco-issued token.

## Local lab status — verified

- Incus VM: `laba-ise-001`; guest identity: `labb-ise-001`; address: `10.200.30.10`.
- The VM is running ISE 3.3.0.430 Patch 11 with 8 vCPU and 16 GB RAM.
- ISE 3.3 has no Data Grid service, so it is a source-state/control system rather than an incident reproduction target.
- The lab host has about 24 physical CPUs and 93 GiB RAM; it cannot fairly reproduce the affected 32-vCPU/64-GB performance topology.
- The lab remains useful for schema, migration, row-level, API, and static-code comparison.
- A prior local experiment successfully restored a 3.3 Patch 11 configuration backup to ISE 3.5.0.527. The target identity and CA remained local and a Data Grid certificate was created under the target CA.
- The ISE 3.5 base ISO and Patch 3 bundle are archived read-only under `/mnt/home/labs/iso/cisco-ise/3.5.0.527/`. Exact filenames, sizes, and SHA-256 values are in [`media-manifest.md`](media-manifest.md).
- Disposable target `laba-ise-035` is running ISE 3.5.0.527 Patch 3 at `10.200.30.14`. Its clean pre-restore snapshot is `clean-p3-pre-restore`.

### Analytics-disabled control backup

- Filename: `ISE_ANALYTICS_RESTORE_CONTROL_20260830_20260830T091538Z-CFG10-260830-0915.tar.gpg`
- Task ID: `5e2ca120-a453-11f1-8750-26460f73dc8d`
- Type: `CONFIGURATION_DB` only; no operational data.
- Status: completed.
- Source Analytics state: disabled.
- The repository API verified the exact object name. A host-side hash has not yet been captured because the current user cannot read the NFS repository path.

### Analytics-disabled control restore result

- The configuration/database import and explicit `3.3.0.430` to `3.5.0.527` upgrade completed.
- ISE 3.5 deleted the legacy `UPSMNTANALYTICSSETTINGS` row and successfully ran 50 global-data handlers, including dedicated miscellaneous-license, monitoring-analytics, and Node Exporter handlers.
- The monitoring-analytics handler observed Monitoring and Log Analytics as disabled; the post-restore API still reports Analytics `DISABLED`.
- The final restore wrapper returned `COMPLETED_WITH_FAILURE` while services were still starting, but all expected enabled services subsequently converged and the GUI login returned HTTP 200 in about 96 ms. The 8-vCPU/16-GiB one-off target is below production sizing, so this final readiness result is not evidence of the production defect.
- No post-restore admin HTTP timeout or sustained Ignite failure signature was found. Bounded self-clearing Ignite refusal cycles occurred during startup and a planned shutdown; production instead reports thousands during the persistent slow-GUI interval.

## Current conclusions

### Supported by evidence

- The incident disproportionately affects the Admin/control plane while authentication remains operational.
- Application restart clears accumulated state.
- Analytics enablement on 3.3 persists data in existing licensing tables.
- A configuration backup can carry that data into the restore workflow.
- Enabling Analytics does not appear to alter the 3.3 table schema directly.
- ISE 3.5 intentionally migrates Analytics-related state with versioned handlers; the disabled control remains disabled and does not reproduce the incident.
- Patch 3 `IgniteClientPool` uses TLS `localhost:10800`; repeating, serialized retry cycles can amplify a local refusal into Admin-thread blocking.
- Patch 3 imposes platform-profile-specific hard memory limits on the Kong container; 1,536 MiB exactly matches the reported cgroup limit.
- The AD-agent write error comes from the native PBIS Event Log client path. Its volume may add retry/logging pressure, but it is not evidence that AD authentication state or ISE application data cannot be written.
- Storage, VM-wide memory pressure, duplicate golden-image identity, and vNUMA have not been supported as primary causes by the evidence collected so far.

### Not yet established

- Whether Analytics state actually survived in the affected 3.5 database.
- How those 3.5 migration handlers transform the rare **enabled** Analytics state.
- Whether Ignite connection failures initiate the slowdown or are downstream retries.
- Whether the reported Kong worker-pool errors precede, coincide with, or follow Admin and Ignite saturation.
- Which process is killed by the 1.5-GiB cgroup OOM events.
- Which platform profile and generated Kong/Data Grid/Tomcat limits are active on the production nodes.
- Whether the reported kernel panic is a true panic, and whether it is a separate event; OOM-killer output alone is not a panic.
- Whether the PBIS eventlog write-failure rate changes at GUI onset or is equally present during healthy operation.
- Whether both PANs become slow at the same time.
- Whether the 09:00 onset tracks GUI users, API polling, report generation, session cleanup, or another workday workload.

## Safety boundary

Do not bypass JWT verification, patch the license handler, write Analytics rows directly, or commit JWT/claim/key material. Those approaches would create unsupported state and would make a positive result difficult to interpret. The high-value reproduction paths are an affected 3.3 configuration backup or a valid Cisco/TAC-issued token.

## Files in this session

- [`evidence-log.md`](evidence-log.md) — provenance-aware inventory and dated findings.
- [`experiment-plan.md`](experiment-plan.md) — controlled restore comparison and incident capture plan.
- [`component-fault-model.md`](component-fault-model.md) — rooted/static component architecture, retry amplification, OOM interpretation, and ranked hypotheses.
- [`media-manifest.md`](media-manifest.md) — canonical media locations, byte counts, and SHA-256 values.
