# Evidence log

## 2026-08-30 — Initial incident handoff

**Source:** affected-environment report  
**Provenance:** Reported

Recorded the recurring Admin GUI degradation, sustained Prometheus user-CPU increase, `jsvc` utilization, administration-thread-pool alerts, Ignite connection refusals, cgroup OOM fragment, and the negative controls: authentication remains healthy, storage metrics are normal, and no major database lock or replication failure was found.

Important correction: a temporary improvement after Data Grid reset/reconfiguration is not a clean intervention because maximum-user-session and time-limit settings changed in the same window.

## 2026-08-30 — Running lab appliance identified

**Source:** local lab discovery  
**Provenance:** Verified locally

`laba-ise-001` is running. The guest identifies as `labb-ise-001`, address `10.200.30.10`, ISE 3.3.0.430 Patch 11, 8 vCPU, and 16 GB RAM.

The running VM answers the source-side questions but cannot reproduce ISE 3.5 Data Grid behavior until a 3.5 target is available.

## 2026-08-30 — Analytics private API and bytecode inspection

**Source:** rooted 3.3 appliance and extracted API schema  
**Provenance:** Verified locally

Found the Analytics miscellaneous-license GET/POST contract and followed the enablement implementation. The handler verifies an RSA256 JWT, passes decoded claims to the licensing utility, updates rows in existing tables, creates entitlement state, and requests license reporting. No DDL was found in that path.

The decoded claim column is limited to 4,000 characters. Claim serialization alone is therefore unlikely to explain sustained two-core Java activity, though a retrying 3.5 consumer of migrated state remains plausible.

Current lab response: `ANALYTICS: DISABLED`. No Analytics rows were present in the three inspected tables.

## 2026-08-30 — Backup transport analysis

**Source:** rooted backup implementation and prior migration experiment  
**Provenance:** Verified locally

The configuration backup exports the CEPM schema broadly, excluding temporary Data Pump tables rather than allow-listing only common configuration. Analytics licensing rows are therefore expected to enter a 3.5 restore unless target restore logic filters or migrates them.

Existing local evidence shows that a 3.3 Patch 11 configuration backup can be accepted by ISE 3.5.0.527 and that target-local identity/CA behavior is preserved.

Relevant source evidence:

- `/srv/nix-config/experiments/ise/006-new-cluster-migration/evidence/n4-cross-version-restore.log`
- `/srv/nix-config/experiments/ise/006-new-cluster-migration/evidence/ca-after-cross-version-restore.md`
- `/srv/nix-config/experiments/ise/docs/api-schemas/openapi/license.json`

Rooted ISE 3.5 Patch 3 experiment material and the associated Nextcloud collection remain inputs for the version-to-version consumer comparison. Their individual artifacts should be indexed here as they are used; raw or sensitive files should remain outside Git.

## 2026-08-30 — Analytics-disabled control created

**Source:** local ISE backup task and repository API  
**Provenance:** Verified locally

Created a configuration-only backup from the Analytics-disabled 3.3 Patch 11 lab:

`ISE_ANALYTICS_RESTORE_CONTROL_20260830_20260830T091538Z-CFG10-260830-0915.tar.gpg`

Task `5e2ca120-a453-11f1-8750-26460f73dc8d` completed. The object name was verified through the repository API. No host-side hash has been obtained because the current user lacks read access to the NFS backing directory.

## 2026-08-30 — Production installation path clarified

**Source:** affected-environment operator  
**Provenance:** Reported

The affected ISE 3.5 nodes were built as fresh appliances from an ISE 3.5 ISO and then brought to Patch 3. They were not in-place upgrades of the 3.3 appliances. Only after the new cluster was at ISE 3.5 Patch 3 was the ISE 3.3 Patch 11 configuration backup restored into it.

This establishes the production sequence as `fresh 3.5 ISO install → Patch 3 → restore 3.3 P11 configuration`. It narrows the migration question to restored configuration/database state and the 3.5 Patch 3 restore/migration code. Inherited operating-system or application-binary residue from an in-place 3.3 upgrade is inapplicable.

The ISE 3.5 base ISO and Patch 3 bundle were subsequently archived outside Git; see the media-archive entry below.

## 2026-08-30 — ISE 3.5 media archived

**Source:** operator-provided Cisco downloads  
**Provenance:** Verified locally for filename, byte count, and SHA-256

Moved the reproduction media from `~/Downloads` into the established persistent lab media area:

`/mnt/home/labs/iso/cisco-ise/3.5.0.527/`

The ISO and Patch 3 bundle are marked read-only. Their exact byte counts and SHA-256 values are recorded in [`media-manifest.md`](media-manifest.md). The source files were moved, not duplicated, so there is one canonical local archive copy.

Open authenticity gate: compare against Cisco-published checksums or execute Cisco's supported signature validation before installation. A locally calculated hash identifies the archived files but does not independently prove vendor authenticity.

The canonical archive is currently local-only. A proposed copy into the Nextcloud sync root was not performed because that would upload proprietary Cisco media to another system and requires explicit authorization. The manifest must not describe the media as off-host backed up until remote presence is independently verified.

## 2026-08-30 — Disposable 3.5 Patch 3 restore target started

**Source:** local Nix/Incus lab controller and VM console  
**Provenance:** Verified locally

Declared a separate one-off lab, `iselab35`, without changing the running 3.3 Patch 11 source appliance. Its only guest is `laba-ise-035.ise.lab` (`10.200.30.14`, MAC `52:54:00:1e:00:35`) with 8 vCPU, 16 GiB RAM, and a 300 GiB disk. This is adequate for functional migration-state inspection but is not a production-sizing performance comparison.

Validation completed before creation:

- exact forward and reverse DNS resolve to the target;
- host available memory was 51.1 GiB;
- Incus reported about 695.7 GiB free in the LVM-backed pool;
- the Patch 3 HTTP repository answered and the archived media remained read-only;
- `nix fmt`, repository lint, and `just build maas` passed.

The implementation is isolated on branch `codex/ise35-restore-repro` at commits `7bf57f3b`, `0fb084e2`, `7bad2545`, and `c3d001e0`. Another active host deployment replaced the shared live slab registry, so subsequent experiment commands are pinned to the immutable built generation rather than altering the peer's deployment.

Two controller defects were encountered before installation and did not mutate the 3.3 source:

1. `node-up` attempted an automatic checkpoint of an absent target and refused before creating it.
2. The lifecycle creator masked a failed installer-volume attach because the pack called the ISO-volume helper with the wrong argument shape and continued after the generated create returned nonzero.

The ISO was therefore imported through Incus directly, attached as the target's priority-10 installer, and the VM was started with its private priority-9 ZTP medium. Console evidence shows the Cisco ISE 3.5 boot menu selected `Cisco ISE Installation Through ZTP Configuration (Serial Console)`. The unattended install is now in progress. The ZTP configuration requests Patch 3, so the clean baseline must confirm the installed release and patch before any 3.3 backup is restored.

## 2026-08-30 — Clean ISE 3.5 base installed; Patch 3 download accepted

**Source:** appliance CLI, ADE log, Incus console, and patch-repository journal  
**Provenance:** Verified locally

The ISO installation completed and the appliance reports:

```text
Version      : 3.5.0.527
Install Date : Sun Aug 30 11:14:48 2026
```

During the first application initialization, the clean Data Grid path loaded `ise-ignite:2.16.0`, calculated a 512 MiB heap within a 1 GiB Data Grid allocation for this 16 GiB functional target, and reported a successful Oracle connection to `laba-ise-035.ise.lab:1521:cpm10`. Oracle identifies as 19.26. The Admin login subsequently returned HTTP 200.

The initial ZTP Patch 3 attempt did not fail inside ISE. A concurrent host deployment replaced the shared patch HTTP service at 12:39 local time with a 3.3-only generation. At 13:32, `10.200.30.14` received HTTP 404 for the Patch 3 bundle. The missing read-only symlink was restored in `/run/iselab-patch-repository`, after which a HEAD request returned HTTP 200 with `Content-Length: 1817306968`, matching the archived bundle.

The explicit patch workflow captured the reusable `slab-v1-pre-destructive` checkpoint, but its first invocation checked `/api/v1/patch` only 843 ms after restarting the appliance and falsely failed on unavailable HTTPS. No patch request had been submitted. After `/admin/login.jsp` returned HTTP 200, the workflow was resubmitted and reused the checkpoint without another shutdown.

At 13:53 local time, repository logs prove that `10.200.30.14` successfully listed the repository, received HTTP 200 for the exact Patch 3 HEAD request, and completed a GET for:

`ise-patchbundle-3.5.0.527-Patch3-26040703.SPA.x86_64.tar.gz`

The patch workflow is now polling ISE for the installed Patch 3 state. No 3.3 backup has been restored and no Analytics state has been enabled on this target.

## 2026-08-30 — Clean Patch 3 baseline confirmed

**Source:** appliance CLI and bootstrap-authenticated ISE APIs  
**Provenance:** Verified locally

After the patch reboot, the CLI reported:

```text
Version      : 3.5.0.527
Patch Version: 3
Install Date : Sun Aug 30 12:00:12 2026
```

`GET /api/v1/patch`, authenticated with the fresh appliance's bootstrap credential, returned Patch 3 with the same install date. The patch controller's actuator also recorded task `54c453b0-a469-11f1-b247-ca28fd891a46` as accepted and then observed ISE 3.5.0.527 Patch 3.

The controller wrapper nevertheless marked its operation as a disagreement because its final oracle used the separate stored GUI credential and received HTTP 401. That oracle result is a credential-selection defect, not an install failure; both the CLI and bootstrap-authenticated API independently report Patch 3.

The clean, pre-restore Analytics baseline was queried through:

`GET /api/v1/license/system/miscellaneous-license`

and returned only:

```json
{"LicenseType":"ANALYTICS","status":"DISABLED"}
```

No token or claim material was returned or recorded. This is Run A's control state: clean ISO installation, Patch 3 installed, Analytics disabled, and no 3.3 configuration restored.

## 2026-08-30 — Clean snapshot and Analytics-disabled control restore

**Source:** Incus, ISE repository API, backup/restore API, appliance CLI, and ADE log  
**Provenance:** Verified locally

Created an offline Incus snapshot named `clean-p3-pre-restore` after Patch 3 and the clean Analytics-disabled API read-back. This is the reusable rollback point for subsequent restore runs.

The persistent 90-second boot delay is not an ISE Data Grid mount. Console output identifies the missing UUID dependency as `Resume from hibernation`; boot continues after the timeout. Removing the ZTP disk did not remove that installed resume reference.

An attempted root-enablement pass exposed two controller/runtime differences:

- the first pass was correctly denied because device mapper was invoked as non-root;
- the authorized host-root retry mounted and edited the 3.5 root partition, but post-boot root public-key authentication remained denied.

The root incompatibility remains open, but it does not block the supported API restore or Analytics status comparison. The clean snapshot predates the root-instrumentation attempt.

The source NFS export initially allowed only `10.200.30.10`. Added a temporary export scoped exactly to the disposable target `10.200.30.14`, using the same squashed UID/GID and export options. The target then listed the exact control backup.

Submitted the Analytics-disabled 3.3 Patch 11 configuration backup to the clean 3.5 Patch 3 target with `restoreIncludeAdeos=false`:

```text
backup: ISE_ANALYTICS_RESTORE_CONTROL_20260830_20260830T091538Z-CFG10-260830-0915.tar.gpg
repository: ise-nfs
task: d5e12d80-a471-11f1-bf6e-3266243fca15
API response: HTTP 202
trigger: OPEN_API
start: 2026-08-30 12:53:45 UTC
```

Observed progress:

- 35%: `Stopping ISE processes required for restore`
- 40%: `Restoring ISE configuration database`

At 40%, `isecfgrestore.sh` first checked flash-recovery-area capacity and began a safety export of the clean target database at SCN `7680964`. The hourly cron correctly observed the `APP_RESTORE` database lock and aborted its own cleanup. This is expected mutual exclusion during restore, not evidence of an unrelated persistent database-lock incident.

## 2026-08-30 — Control restore entered the 3.3-to-3.5 migration path

**Source:** appliance CLI, `ADE.log`, Cisco database-upgrade logs, and `tech top`  
**Provenance:** Verified locally

The control restore completed Oracle Data Pump import successfully and identified the backup as exactly `3.3.0.430`:

```text
13:10:00 DB Restore using IMPDP successfully completed
13:10:26 ISE DB restore completed without error
13:10:28 BACKUP VERSION NUMBER IN RESTORE 330
13:11:45 Starting ISE data upgrade from 3.3.0.430 to 3.5.0.527
```

The upgrader then ran distinct schema and data phases. The UPS model schema phase completed, Liquibase migration ran, and the workflow entered `UPGRADE STEP 2: Running ISE configuration data upgrade`. The high-level restore indicator remained at 60%, `Adjusting host data for upgrade`, while the detailed logs and process activity continued.

A migration action directly relevant to the Analytics hypothesis was observed:

```text
check if UPSMNTANALYTICSSETTINGS exists...
Removing all rows from UPSMNTANALYTICSSETTINGS .
1 row deleted.
Commit complete.
Removed all records from UPSMNTANALYTICSSETTINGS .
```

This proves that the 3.5 upgrader intentionally purges this legacy monitoring-analytics settings table during the 3.3 restore path. It does not by itself prove that every Analytics entitlement, feature flag, token-related object, or other analytics table is purged. The control backup was Analytics-disabled at source, so the single deleted row must not be treated as evidence that the rare Cisco-enabled entitlement was present.

At 13:28-13:29 UTC, `tech top` showed the migration's Java process (PID `120767`) continuously consuming about 14-17% CPU with about 1.1 GiB resident memory, while an Oracle worker consumed about 11-16% CPU. The appliance had about 8.9 GiB free memory and no OOM condition. This confirms the coarse 60% phase was computationally active rather than stalled.

One schema helper logged `ORA-00904: "MARKFORCOA": invalid identifier` while attempting to drop/recreate `EDDACOA_INDEX`, then continued to report `Done correcting edda indices` and completed the UPS schema upgrade. Final restore status and post-restore service/Analytics verification remain pending at this point in the timeline.

The global data upgrader registered three directly relevant services:

```text
com.cisco.cpm.license.MiscLicenseHandler
com.cisco.mnt.dbms.mntanalytics.util.MnTAnalyticsEnablerService
com.cisco.mnt.dbms.mntanalytics.util.NodeExporterPasswordHandler
```

`MiscLicenseHandler` declared migration version `3.4.0.494` and logged `Successfully applied changes ... in 0 seconds`. `MnTAnalyticsEnablerService` declared versions `3.3.0.417,3.5.0.527`; `NodeExporterPasswordHandler` declared versions `3.3.0.356,3.5.0.527`. This proves that Analytics enablement, miscellaneous-license state, and the monitoring node-exporter credential have explicit versioned migration handlers. At the time sampled, the latter two handlers had been registered/instantiated but had not yet logged a successful-completion line.

The same data-upgrade run recorded a background Hibernate/Oracle error against the unrelated guest-portal table:

```text
ORA-00904: "PORTALSETT0_"."PROVIDER_TYPE": invalid identifier
table: EDF_GUEST_PORTAL_SETTINGS
```

It also contained `InvocationTargetException`/`NullPointerException` markers. These require final-outcome correlation: they may be tolerated per-handler errors, or they may prevent the global migration from completing. They are not, by class or table name, evidence that the Analytics handler itself failed.

At 13:36-13:37 UTC the same Java PID had accumulated more CPU time (about `6:11` through `6:15`) and continued to use roughly 14-19% CPU; the paired Oracle worker remained active at roughly 12-19%. The migration therefore continued doing database work despite the exceptions.

The complete log later proved that the guest-portal exception was tolerated. The target-version GuestAccess handler reported one failed portal object with `could not extract ResultSet`, but the containing handler completed in four seconds and was counted among the successfully completed services.

The two monitoring-analytics handlers then completed explicitly:

```text
NodeExporterPasswordHandler:
  mnt.properties nodeExporter password property value updated
  NodeExporterSettings DB table updated
  successful in 189 seconds

MnTAnalyticsEnablerService:
  Monitoring enabled: false
  Log Analytics enabled: false
  MntAnalytics settings successful
  successful in 20 seconds
```

The global upgrader ended with `Successfully completed 50 services` and `ISE GLOBAL DATA UPGRADE COMPLETED`. This control result establishes that an Analytics-disabled 3.3 backup follows dedicated Analytics migration code and remains logically disabled inside that handler. It also shows that the Node Exporter credential/settings are regenerated on the 3.5 target.

At 13:46:04 UTC the parent restore workflow logged `ISE DB UPGRADE SUCCEEDED`. It then:

- started temporary Ignite configuration and verified `Cluster is active` with command exit code 0;
- restored the target's original `ignite-config.xml`;
- deleted secondary certificates and deployment metadata belonging to the 3.3 source node;
- proceeded through indexing/log restoration and service reconstruction.

During service reconstruction, a CA certificate post-commit callback logged `NoSuchRoleException: Role not exist`; final service and task health must determine whether this was tolerated. The newly created Ignite container also raced its first activation command and was not yet running when that command executed. These are startup observations, not yet evidence of a persistent Data Grid failure.

## 2026-08-30 — Control restore final result and post-state

**Source:** restore CLI/history, task API, application CLI, license API, GUI probe, and application logs  
**Provenance:** Verified locally

The restore ran from 12:53:45 through 14:05:47 UTC (about 72 minutes). Database import, schema upgrade, all 50 global data handlers, Analytics migration, indexing restoration, and service reconstruction completed. The final Cisco wrapper nevertheless recorded:

```text
COMPLETED_WITH_FAILURE
Error: Failed to start ISE services. Restore is terminated.
```

The failure occurred while `cpmcontrol.sh` was still launching dependent processes during the second service cycle. Services continued to converge after the wrapper released the restore lock. The final supported service inventory showed every expected enabled process running, including Data Grid, Application Server, Elasticsearch, AD Connector, M&T Session Database and Log Processor, CA/EST, Messaging, API Gateway, pxGrid Direct, Log Analytics Elasticsearch, MFC Profiler, and Protocols Engine.

The final minute also included `Starting system resources check for 3595 profile`. This disposable functional VM has only 8 vCPU and 16 GiB RAM, intentionally below the affected environment's 32-vCPU/64-GiB sizing. The lab's final service-start return code may therefore be influenced by its unsupported functional sizing or a startup race. It must not be treated as proof that the production-sized restore failed the same way.

Post-restore checks:

```text
ISE version:       3.5.0.527
Patch version:     3
Analytics status:  DISABLED
GUI login page:    HTTP 200 in approximately 96 ms
Data Grid:         running
Application:       running
```

The task API reported task `d5e12d80-a471-11f1-bf6e-3266243fca15` as `COMPLETED_WITH_FAILURE`, matching restore history. The configuration/database migration remains present and queryable despite the final wrapper failure.

Post-restore signature checks found no `java.net.SocketTimeoutException` or `admin-http-pool` entry in `console.log`. At the initial check, `ise-ignite.log` contained six `Failed to create Ignite client: Connection refused` retries at 12:11 UTC, before the control restore and during an earlier clean startup. A later planned offline snapshot shutdown produced one additional nine-attempt cycle from 14:18:55 through 14:19:19 UTC as Data Grid stopped before the Application Server. After restart, the state monitor reported the cluster `ACTIVE` and node `RUNNING` every 30 seconds through the end of the captured log, with no further refusal. These bounded lifecycle retries did not produce the sustained GUI incident and should not be equated with the production report of thousands of recurring errors.

The operator reconfirmed that the affected production environment contains `IgniteStateMonitorThread-0` / `IgniteClientPool` `Failed to create Ignite client` errors **thousands of times** while the GUI is persistently slow. That volume and temporal correlation are materially different from this lab's bounded, self-clearing lifecycle retry cycles. A useful discriminator for the affected cluster is therefore the error rate and retry duration, plus the exact refused destination IP/port—not the mere presence of a connection-refused line during startup or shutdown.

The operator also reports Kong worker-pool errors in the affected production environment. The precise message, log source, timestamps, pool occupancy, and relation to the Ignite/Admin failures have not yet been supplied. This is additional evidence of possible API-gateway/control-plane pressure, but it is not yet sufficient to place Kong before or after the initiating failure.

### Control-run conclusion

An Analytics-disabled 3.3 Patch 11 configuration can be imported and migrated into 3.5 Patch 3. The migration deliberately empties `UPSMNTANALYTICSSETTINGS`, regenerates the Node Exporter credential/settings, runs dedicated miscellaneous-license and monitoring-analytics handlers, observes both Monitoring and Log Analytics as disabled, and leaves the public Analytics status `DISABLED`.

This control did not reproduce sustained Ignite failures, admin-threadpool exhaustion, socket read timeouts, or slow GUI behavior. It therefore does not support Analytics as the cause in the disabled case. It also cannot exclude a defect specific to the rare enabled state: the decisive next run requires the affected environment's 3.3 configuration-only backup or a valid Cisco/TAC-issued JWT so the same migration can be compared with Analytics enabled.

## Blockers and required inputs

- Affected environment's **3.3 Patch 11 configuration-only backup** and its backup-encryption key, transferred securely and never pasted into notes or chat.
- Ideally, Cisco/TAC confirmation of the Analytics entitlement state and a supported disable or reissue procedure.
- The old JWT only if Cisco authorizes its use and it remains valid; do not commit or log it.
- Vendor authenticity/applicability verification for the archived ISO and patch remains open. Installation proceeded on the operator's explicit request using the locally recorded SHA-256 values; those hashes establish repeatability, not Cisco provenance.

## 2026-08-30 — Admin-plane architecture and retry amplification resolved

**Source:** rooted ISE 3.3 Patch 11, exact ISE 3.5 Patch 3 filesystem payload and Java bytecode, and live disposable target
**Provenance:** Verified locally, with production causality still inferred

Documented the end-to-end Admin path as public 443 through containerized
Kong/nginx to Tomcat/jsvc 9443 and its `AdminExecutorPool` /
`admin-http-pool`. RADIUS/TACACS use the separate Protocols Engine path, which
explains why authentications can remain healthy during severe Admin-plane
saturation.

Exact Patch 3 `IgniteClientPool` bytecode establishes that the Application
Server connects to TLS `localhost:10800`, not directly to another cluster node.
It maintains main, UI, and event-listener clients; client construction is
guarded by one class-wide monitor. The Cisco wrapper allows 10 attempts, with a
3-second wait between attempts, while the Ignite client configuration has a
300-second timeout, 3-second heartbeat, and retry limit 5.

The exact state monitor starts after 60 seconds and runs with fixed delay every
30 seconds on primary/standalone nodes or every 60 seconds on secondary nodes.
The PAP event listener separately runs every 30 seconds and uses another client
from the same class-wide pool lock. A persistent immediate connection refusal
can therefore create about 27-second serialized retry cycles from two scheduled
callers before any GUI demand is counted. This makes thousands of error lines
and Admin-thread blocking mechanically plausible. It does not prove why the
local 10800 listener is absent or whether that occurs before Kong/Admin
pressure in production.

Exact Patch 3 Data Grid files show Apache Ignite 2.16.0, persistent state/WAL
under `/opt/ignite/data`, Oracle-backed discovery through `TBL_ADDRS`, and
application caches for endpoint/profile, EDDA, endpoint-license, MFC, and event
state. The Data Grid reset script removes the container/image and persisted
Data Grid data, work database, snapshots, diagnostics, and lock data before
rebuilding. The reset result is thus compatible with clearing durable Data Grid
state, but remains confounded by the other production setting changes.

Exact Patch 3 Kong control and platform files also resolve the reported OOM
limit: `1572864 kB` is exactly 1,536 MiB, and `kong-control.sh` applies the
active `apigateway.memory` as a hard container memory limit. Patch 3 assigns
1,536 MiB to several small profiles but 12,288 MiB to
`vm_standard3_flex_32`. The affected VM's 32 vCPU does not prove which profile
ISE selected. The full OOM continuation, cgroup path/victim, and generated
active profile values are now high-priority production captures.

The reported kernel panic remains unverified and separate. An OOM-killer event
is not a panic; a real panic should stop/reboot the whole guest and disrupt
authentication. Exact panic text and uptime/Hyper-V evidence are required
before relating it to the recurring GUI-only interval.

The complete architecture, ranked hypotheses, discriminators, and minimum
production capture are in [`component-fault-model.md`](component-fault-model.md).

## 2026-08-30 — Disabled control preserved and healthy after restart

**Source:** Incus, appliance CLI, and GUI probe
**Provenance:** Verified locally

Created snapshot `control-disabled-restored-p3` after the completed disabled
control migration. The disposable target is running; Data Grid, Application
Server, API Gateway database/service, M&T components, MFC Profiler, and
Protocols Engine report running. The GUI returned HTTP 302 in approximately
114 ms.

The planned offline snapshot shutdown generated one bounded Ignite refusal
cycle: the state monitor last reported `ACTIVE` at 14:18:25 UTC, then made nine
failed attempts at approximately 3-second spacing from 14:18:55 through
14:19:19 while the client pool/Application Server was closing. After restart,
the state monitor reported cluster `ACTIVE` and the node `RUNNING` every 30
seconds through 14:40:40, with no further refusal. This directly validates the
Patch 3 retry timing and distinguishes a normal service-ordering race from
production's persistent thousands-of-errors interval.

Run C was not fabricated. It remains staged behind the affected
Analytics-enabled 3.3 configuration-only backup and its separately transferred
encryption key.

## 2026-08-30 — AD-agent write target identified

**Source:** affected-environment operator, exact ISE 3.5 Patch 3 AD Runtime RPMs,
rooted ISE 3.3 PBIS runtime, and BeyondTrust AD Bridge diagnostics

**Provenance:** Production volume reported; write-path identity verified locally

The operator clarified that `ad_agent.log` contains thousands of `Failed to
write records error code [1]` messages, rather than an occasional warning.

Patch 3 contains PBIS/AD Runtime 7.1.1. The exact error string resides in
`libeventlog.so` and `libeventlog_norpc.so`, adjacent to the API symbols
`LwEvtWriteRecords`/`LwmEvtWriteRecords`, `localhost`, and local endpoint
`/var/lib/pbis/.eventlog`. This identifies the failed write as PBIS audit/event
emission toward its local Event Log service/database. It is not direct evidence
that the AD agent cannot write AD objects, ISE configuration in Oracle, or Data
Grid state.

The rooted 3.3 control runs PBIS registry, LSASS, I/O, netlogon, redirector, and
NTLM services. It has corresponding local sockets but no `eventlog` service in
`lwsm list` and no `.eventlog` socket. That makes an omitted/disabled local
eventlog service a plausible explanation for this class of error, although the
affected 3.5 node must be checked directly. Numeric code `1` alone does not
resolve the lower-level reason.

The error storm can contribute native CPU and log I/O but has no demonstrated
direct link to Ignite or `admin-http-pool`, and it does not explain Java
`char[]` CPU directly. Its causal weight now depends on per-minute healthy
versus degraded rates and first-occurrence ordering. Healthy authentication is
consistent with failure of this audit path; production must still confirm
whether successful authentications exercised AD rather than another identity
source.

## 2026-08-30 — CharacterEncodingFilter line 123 resolved

**Source:** affected-environment stack excerpt, rooted ISE Admin webapp class,
and exact Patch 3 Admin `web.xml`

**Provenance:** Verified locally for line and filter mapping; timeout destination
not yet captured from production

Decompilation maps `CharacterEncodingFilter.doFilter` source line 123 exactly
to `FilterChain.doFilter(request, response)`. Patch 3 maps the filter to `/*`.
The class sets request/response encoding and delegates; it performs no socket
read at line 123. The frame appears because a deeper request handler threw
`SocketTimeoutException` and the exception unwound through this common filter.

The `admin-http-pool` thread name proves the timeout occurred while a Tomcat
Admin executor thread owned the request. A connected socket then failed to
return data before its read timeout. That waiting interval keeps the thread
busy; enough concurrent waits cross the Administration thread-pool threshold.
The most likely direction is therefore downstream read delay followed by pool
occupancy and alerting, with Kong/request retries potentially forming a
feedback loop. The alert itself does not create the Java timeout.

The excerpt still does not name the socket. Frames above the common filter are
required to distinguish inbound Tomcat request-body reads, outbound HTTP calls,
Oracle JDBC, Ignite, or Cisco inter-node/local-service clients.
