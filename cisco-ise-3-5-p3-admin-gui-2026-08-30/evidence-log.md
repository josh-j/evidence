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

## 2026-08-30 — Exact Patch 3 generic-VM profile mapping recovered

**Source:** Patch 3 `bootstrap-3.5.0-527.jar`, `platform.properties`, control
scripts, and rooted 3.3 comparison
**Provenance:** Verified locally by bytecode inspection and read-only rooted
inspection

`PlatformProfileServiceImpl` classifies generic non-cloud VMs using ordered CPU
and `/proc/meminfo` thresholds. A 32-vCPU/64-GiB VM misses the 96-GiB-and-larger
rules and matches `cpu >= 32 && MemTotal >= 31,200,000 kB`, producing profile
`sns3815`. Patch 3's generated values for that profile include Kong 1,536 MiB,
Data Grid 2 GiB, Admin maximum threads 200, and Application Server heap 12,288
MiB. The reported production OOM limit is exactly the predicted Kong limit.

The active properties are local platform state, not restored application
schema state. They are generated into
`/opt/CSCOcpm/config/platform.properties-active` and rebuilt when absent, when
node persona configuration changes, or when RAM changes. The inspected code's
cache invalidation compares RAM but not CPU, creating a separate stale-profile
risk after a CPU-only resize.

This establishes a credible capacity-amplification mechanism but does not yet
prove causality. Production still needs the active profile from all three
nodes, the complete OOM cgroup/victim lines, and runtime container limits. A
Cisco-supported VM shape or TAC-supported correction should be used for any
remediation; generated properties must not be edited manually.

Production is not rooted. Rooted lab inspection shows that the supported
`show tech-support` implementation calls the same platform-property reader and
prints `Profile : <name>` under `Displaying ISE Profile`; a support bundle has a
show-tech area. Therefore the profile can be confirmed without unsupported
access. Generated-property and live container-limit confirmation should be
requested from TAC using the production support bundle.

## 2026-08-30 — One-node Option 45 scope resolved

**Source:** operator clarification and exact Patch 3 `ignite-control.sh` /
Ignite configuration
**Provenance:** Reported for production scope; verified locally for control
behavior

Production ran Data Grid Reset Config only on the PAN, not every deployment
node. Patch 3's reset implementation removes only that appliance's Ignite
container/image and local persistence/work/snapshot/diagnostic data. It neither
drops the `TBL_ADDRS` discovery table nor removes other members' persisted
state. The reported 64-GiB PANs and PSN are all selected as Ignite server-mode
nodes by Patch 3 logic. The rebuilt PAN can therefore rejoin the surviving
cluster and receive replicated cache state after baseline adjustment.

The observed 48-hour improvement should not be described as proof that the
entire Data Grid was cleared. It raises the probability of a PAN-local
container/persistence/listener issue or a transient restart/rebalance benefit.
It remains possible for logical cluster state to be copied back or for the
triggering workload/state to reaccumulate. Simultaneous user-session and
timeout changes prevent assigning the longer healthy interval solely to the
reset.

## 2026-08-30 — Concrete PAN-local 10800 loss mechanisms identified

**Source:** exact Patch 3 `ignite-control.sh`, `ignite-config.xml`, and platform
properties
**Provenance:** Verified locally; production branch not yet selected

For predicted profile `sns3815`, the Ignite container is hard-limited to 2 GiB.
The launcher configures 1,024 MiB initial/max Java heap and about 819 MiB
initial/max persistent data region, leaving about 205 MiB nominal headroom for
all other native/JVM/container memory. It enables `AlwaysPreTouch` and
`ExitOnOutOfMemoryError`. PAN-specific Admin/cache activity can therefore kill
that JVM and remove the local 10800 listener while other nodes remain alive.
Production needs Ignite/container-specific OOM or exit evidence; the reported
1,536-MiB nginx OOM fragment likely describes the separate Kong limit.

The start path also gates on `/tmp/ise-ignite-service.lock`: if the lock exists
while the container is absent, it returns “initializing” without starting the
service. Option 45 removes this lock as well as local persistent/WAL/work state.
This creates two particularly testable explanations for a one-node reset
repair: memory-driven container exit/restart or a stranded initialization/
recovery state. Other focused branches are local WAL/page/checkpoint failure,
Oracle JDBC discovery/baseline/TLS failure, and 10800 bind/firewall state.

## 2026-08-30 — Initial-trigger models separated

**Source:** reported 09:00 timing and exact Patch 3 Ignite cache/configuration
**Provenance:** Mechanisms verified locally; production timing not yet supplied

The visible 09:00 onset can mean either that work-hour demand exposed an Ignite
listener that had already failed during startup, or that a previously healthy
listener disappeared under the new load. The first `IgniteClientPool` refusal
after each restart—not the first user complaint—distinguishes those models.

Patch 3 places replicated endpoint/profile, EDDA, endpoint-license, 30-day MFC,
and OIDC session-context caches in Ignite. `OIDCSessionContextCache` is
SQL-on-heap and stores token strings plus session lifetime/touch data. These are
concrete workload sources, but no affected cache counts or growth series have
been collected. The session-limit/time-limit changes made alongside Option 45
remain confounded until their exact setting names and scope identify whether
they concern Admin sessions, RADIUS accounting/max-session policy, portals, or
external OIDC.

The leading demand-driven initiating class is ordinary Admin/API/session/cache
activity crossing a PAN-local runtime threshold, potentially magnified by a
poller, report, or enabled feature consumer. An abrupt exit can subsequently
leave dirty recovery state or a stranded lock, making persistence/recovery a
secondary reason the listener remains unavailable rather than the original
trigger.

## 2026-08-30 — Midnight and early-morning jobs inventoried

**Source:** exact Patch 3 bundled administrator help and OIDC service bytecode;
rooted 3.3 Patch 11 scheduler inspected as a structural migration control

**Provenance:** Patch 3 behavior verified locally; affected-production schedule
and execution timestamps not yet supplied

Patch 3 does not automatically reset Data Grid at midnight. Its help states
that the `Endpoints Purge Activities` alarm is triggered at midnight, while the
enabled-by-default endpoint purge runs at about 01:00 in the PPAN time zone.
The purge removes endpoints and registered devices older than 30 days and,
above 5,000 endpoints, deletes batches every three minutes. The default
Profiler Feed Service update is also 01:00 when that feature is enabled and can
cause re-profiling and increased load.

Because Patch 3 keeps replicated endpoint/profile objects in Ignite, a large
endpoint purge is the strongest presently identified overnight workload that
could drive replicated cache removal plus WAL/checkpoint work. It could fail or
weaken the local PAN Data Grid hours before 09:00 users expose the condition,
or merely accumulate pressure that work-hour demand later pushes over a
threshold. This is a testable trigger hypothesis, not a production finding.

Exact Patch 3 OIDC bytecode schedules expired-session cleanup every 360 minutes
with the first run 360 minutes after construction. It is tied to Application
Server start time, not midnight. Cisco also documents MnT DBMS statistics at
02:00 and Sunday index maintenance at 01:00. The rooted 3.3 control contains
additional Oracle jobs at 00:00, 00:30, 01:30, 02:00, and 04:00, but their
presence in restored production 3.5 must not be assumed without its scheduler
or support-bundle evidence.

Required discriminator: normalize PPAN, database, and log time zones, then
order endpoint-purge start/end/count, the last successful Data Grid state, the
first refused 10800 connection, and the first Admin thread-pool alert. A first
refusal during 00:00-02:00 makes overnight work a strong trigger lead; a proven
healthy listener until 09:00 points back to work-hour demand.

## 2026-08-30 — jsvc CPU parallelism interpreted

**Source:** production `tech top` and Prometheus observations, exact Patch 3
thread-pool/client-pool configuration, and read-only rooted 3.3 control

**Provenance:** Production utilization reported; control affinity verified;
production affinity/quota not yet verified

`jsvc` at 200–300% represents approximately two to three CPUs in Linux
`top`-style accounting, or 6.25–9.375% of a 32-vCPU VM. It is quantitatively
consistent with the approximately 10% Prometheus increase after other process
and kernel work is included.

Admin pool occupancy is not CPU parallelism. Up to 200 Admin threads on the
predicted Patch 3 `sns3815` profile can be occupied while most wait in socket
reads, Oracle/Ignite calls, retry sleeps, monitors, or queues. The synchronized
Patch 3 Ignite client-construction path is one verified serial point. The few
runnable threads can spend CPU constructing/logging repeated exceptions,
serializing GUI JSON/XML, allocating strings/character arrays, and collecting
the resulting garbage.

The reported `java char[]` profile indicates allocation/string-processing
churn rather than an array being an executable hotspot. It is consistent with
JSON, encoding, stack-trace, and logging work, but does not make
`CharacterEncodingFilter` the cause.

No Patch 3 three-core jsvc pin was found. On the rooted 3.3 control, the live
Application Server child had more than 1,100 threads but was allowed on every
one of its eight assigned vCPUs according to `taskset`. Exact production 3.5
affinity and quota require TAC/root-level inspection. If CPU remains capped at
exactly about 300% under unrelated parallel workloads, inspect affinity,
cgroup CPU quota, JVM active-processor/GC flags, and Hyper-V per-vCPU state.
Otherwise limited runnable/serial work is the expected explanation.

The next useful capture is repeated per-thread CPU plus JVM thread dumps and GC
logs during degradation. It will separate runnable logging/serialization/GC or
query threads from the larger population of occupied but waiting Admin
threads.

## 2026-08-30 — Admin thread creation path verified

**Source:** exact Patch 3 platform properties and read-only rooted 3.3 Tomcat
configuration

**Provenance:** Connector/executor design verified; affected request rate and
URI mix not yet supplied

Tomcat's local HTTPS connector on 9443 dispatches requests to
`AdminExecutorPool`, whose workers are named `admin-http-pool-*`. Kong proxies
public GUI/API traffic from 443 to this connector. The executor creates
additional reusable workers when requests arrive without enough idle workers,
up to the configured maximum. Patch 3's predicted `sns3815` profile specifies
five spare workers and a 200-thread maximum. The rooted 3.3 control has the
same structure with a generated 450-thread maximum.

GUI page/API calls, AJAX polling, ERS/OpenAPI automation, monitoring, and
qualifying Kong upstream retries create request tasks. Idle authenticated
sessions do not each own a thread, and RADIUS/TACACS traffic uses the separate
Protocols Engine. Slow downstream reads retain rather than create the initial
worker: with 22-second average latency, roughly nine arrivals per second can
occupy 200 workers; with 60-second latency, about 3.3 per second can do so.

The Ignite state-monitor and event-listener threads are separate scheduled
threads, not Admin workers. Their use of the same synchronized Ignite client
pool can nevertheless delay Admin requests. The approximately 15-minute alert
recurrence can therefore be health monitoring repeatedly observing sustained
occupancy, not a new thread-creation event every 15 minutes.

## 2026-08-30 — Admin request timeout layers separated

**Source:** active rooted 3.3 Kong service records and Tomcat connector; exact
Patch 3 Ignite bytecode/configuration

**Provenance:** Control and Patch 3 mechanisms verified; affected-production
Kong service values and timeout stack not yet supplied

No universal Tomcat request-execution deadline was identified. An Admin worker
remains occupied until its application action returns or a dependency-specific
timeout/error unwinds it. The rooted control's GUI service allows 60 seconds to
connect from Kong to Tomcat and one hour (`3,600,000 ms`) for upstream read and
write inactivity, with five eligible retries. Tomcat's five-minute keepalive
controls idle persistent connections, not active request execution.

Exact Patch 3 Ignite configuration uses a five-minute client timeout, while a
missing local listener produces the approximately 27-second Cisco wrapper
retry cycle because the connection refusals are immediate. Neither value can
be assigned to the reported Java read timeout without its complete stack.

The Admin GUI session idle limit is also unrelated: it expires authentication
state and does not release a worker currently blocked inside a request. Thus
many Admin tasks may stay occupied for tens of seconds or minutes, and some
gateway-visible requests can wait much longer, without a single thread-pool
watchdog terminating them.

## 2026-08-30 — Request-source and accidental-flood evidence located

**Source:** active rooted Kong/Tomcat configuration and exact Patch 3 support
bundle collector

**Provenance:** Log fields, bundle path, routes, and executors verified;
affected DNA Center request details not yet supplied

Kong's proxy access log is `/opt/kong/logs/access.log`, also reached through
`/opt/CSCOcpm/logs/ise-kong`. Patch 3 support bundles copy these files to
`logs/apigateway/`. Entries contain client and forwarded IPs, authenticated
user, method/URI, status, User-Agent, upstream status, and upstream
connect/header/response times. `api-gateway.log` records gateway control work
and is not a substitute for the proxy access log.

The active logging filter omits 2xx/3xx requests and retains error statuses.
This should expose a failing DNA Center retry storm but can hide a successful
high-rate poller. Upstream firewall/load-balancer/NetFlow or scoped Hyper-V
capture can still establish source IP and rate; temporary all-status URI
logging on ISE should be arranged through TAC.

Route mapping prevents treating every DNA Center message as an Admin request:
GUI/catch-all traffic reaches Tomcat 9443 and `admin-http-pool`; `/ers` reaches
the ERS pool, `/api` the OpenAPI pool, pxGrid its own pool, and RADIUS/TACACS the
Protocols Engine. The reported DNA failure needs source IP, destination port,
URI/status, User-Agent or client certificate, and matching thread name before
it can explain Administration pool occupancy.

Accidental application-layer overload is established by ordering rather than
volume alone. Bin source/path/status counts and upstream latency per minute and
align them with the first Ignite refusal, Admin threshold, jsvc CPU rise, and
GUI latency. A client rate surge first supports a poll/retry trigger; backend
latency or 10800 loss first with client retries afterward supports server-side
failure amplification. At 22-second service time, approximately nine requests
per second can occupy the predicted 200-thread Admin pool despite modest
bandwidth.

## 2026-08-30 — Single-user/client mechanisms separated

One request consumes one worker and an idle session consumes none, so an
ordinary single login does not directly create the full Admin pool. One source
can still cause the event either by submitting overlapping requests/retries or
by triggering a shared server-side lock, pathological query, Ignite-client
serialization, or runtime failure behind which other requests accumulate.

Credible 09:00 single-source candidates include a user opening a data-heavy
saved grid/report, a resumed browser with stale polling tabs, DNA Center daily
sync or credential retry, API automation, scheduled export, vulnerability
scanner, health checker, and SSO/session-refresh loop. Automation is more
likely than a normal browser to sustain the approximately nine 22-second
requests per second that can occupy 200 workers. A normal user action that
causes persistent deployment-wide failure still indicates a server defect or
capacity/concurrency weakness, even if that action is the trigger.

Attribution requires source IP/XFF, authenticated user, User-Agent/client
certificate, and query-stripped URI together. A source tuple dominating just
before onset and disappearing when selectively paused supports causality. A
missing 10800 listener before its rate increase, normal traffic distribution,
or continued failure after it stops places the client downstream of the root
fault. A source can also trigger sticky Data Grid state, so lack of immediate
recovery after blocking it does not alone exclude an initiating role.

## 2026-08-30 — ACAS SSH failures scoped

ACAS failing SSH hundreds of times does not directly use Kong, Tomcat 9443,
jsvc, or `admin-http-pool`; it reaches the separate ADE-OS sshd service. The
rooted control uses three authentication attempts, a 60-second login grace, and
`MaxStartups 10:30:100`, which begins dropping unauthenticated connections
probabilistically above ten concurrent sessions. Hundreds spread through a
scan are mainly log/authentication noise; high concurrency can add host
cryptographic, PAM, process, memory, and logging load, which should appear as
sshd/kernel rather than jsvc CPU.

The more important test is whether the same ACAS schedule also probes 443 or
other application ports. Web/TLS plugins can reach Kong and a catch-all GUI
route can consume Admin workers. Kong access/error logs must be searched for
the ACAS IP, including handshake failures that have no HTTP status. ERS,
OpenAPI, pxGrid, and portal destinations use different pools and must not be
misclassified as Admin traffic.

Patch 3 support bundles include `/var/log/*` under `adeos/` when System logs are
selected and gateway logs under `logs/apigateway/`. Align ACAS attempts and
destination ports per minute with the first 10800 refusal, Admin threshold, and
jsvc rise. An approved exclusion window followed by SSH-only versus HTTPS-only
testing can determine causality. The invalid scanner credential should be
fixed even if unrelated, because it risks CLI lockout and obscures genuine
attack evidence.

## 2026-08-30 — ISE safe-mode impact scoped

**Source:** Cisco ISE 3.5 CLI Reference Guide and exact Patch 3 `cpmcontrol.sh`

`application start ise safe` is an Admin lockout-recovery mode, not a minimal
or performance-safe service mode. It sets `ise.startup.safe=true`, relaxes
Admin IP restrictions, bypasses certificate-based Admin authentication in
favor of username/password, and bypasses FIPS/RNG startup integrity checks on
FIPS hosts. `show application status ise` emits an explicit warning while the
mode is active. The changes disappear on the next ordinary application start.

The required prior stop and subsequent safe start restart the application
service stack and clear runtime threads, queues, pools, and retries. They do
not perform Option 45 or erase Ignite persistence. A fast GUI afterward is
therefore restart evidence unless an ordinary-versus-safe controlled comparison
shows different behavior.

Safe mode can expose `/admin` to an ACAS/DNA address that the normal ISE Admin
IP allowlist would reject; external firewalls remain effective and SSH is
unaffected. Confirm safe-mode status, the normal allowlist, and Kong requests
from the scanner. If both start modes recover identically, safe is irrelevant.
If only safe recovers, prioritize IP filtering and certificate-authentication
dependencies, while treating the independent 10800 refusals as still requiring
their own explanation. Do not leave production in safe mode after correcting
the access problem.

Safe mode is disabled locally with `application stop ise` followed by ordinary
`application start ise`; it does not expire and a start command cannot replace
an already-running instance. Absence of the safe warning in `show application
status ise` verifies the normal instance. The startup flag does not replicate:
only nodes explicitly started safe relax their own Admin access/authentication.
The restart can still cause local Data Grid leave/rejoin, Admin downtime, and
possibly PAN auto-failover; other PSNs continue independently unless they are
also stopped.

## 2026-08-30 — Recurrence with GUI API services disabled incorporated

**Source:** affected-production operator and exact Patch 3 bundled help for API
Service/API Gateway settings

The operator reports that the same slowness occurred after API services were
disabled in the GUI. If the change was made before a clean restart and remained
in force until recurrence, this is strong evidence that ordinary ERS/OpenAPI
traffic is not required to trigger the incident and lowers DNA Center's
ERS/OpenAPI workload as the primary cause. If made only during degradation, it
does not clear already-stuck Admin, Kong, or Ignite state and is a weaker test.

Patch 3 distinguishes API services from the gateway and GUI. Disabling ERS or
OpenAPI does not stop Kong's public 443 role, the GUI/catch-all route to Tomcat
9443, pxGrid, internal schedules, or Data Grid. OpenAPI is enabled by default
from 3.4, so the exact ERS and OpenAPI toggle states must both be recorded.
Calls to disabled services may continue and time out, potentially causing
client retries visible in Kong logs.

Subject to clean-cycle timing, the updated focus is PAN-local Ignite/
Application Server runtime or internal scheduled/data work, followed by GUI or
scanner catch-all traffic and pxGrid/DNA activity. Ordinary external
ERS/OpenAPI demand is now lower priority but not excluded as gateway error-load
without source/path/status evidence.

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

## 2026-08-30 — `messages` noise and root-to-Oracle rate resolved

**Source:** live rooted ISE 3.3 Patch 11 control, its current and 15 rotated
`messages` logs, audit records, M&T loader scripts, and current process status

**Provenance:** Verified locally on the healthy control; production rate is
operator-reported

The healthy rooted control contains two conspicuous but normally non-fatal
signatures also seen in production:

- `su: (to oracle) root on none`
- `Activated service 'org.fedoraproject.Setroubleshootd' failed: Failed to
  setup environment correctly`

The first line is not merely a generic Oracle health check. A matching audit
record resolves the active command to `sqlldr /@mnt10` with
`ad_operations.ctl`. The persistent parent is the M&T Log Processor's
`ad_operations_sqlldr_driver.sh`. It scans the AD-operation CSV directory once
per second and invokes `ad_operations_sqlldr.sh` whenever a completed buffer is
available. That script changes to the `oracle` account and loads the buffer
into the M&T database. On the control, recent loads report one to three rows
successfully loaded and the queue normally contains only the current buffer.

The healthy-control baseline during the sampled 17:02–17:41 UTC interval was
usually 6–9 root-to-Oracle lines per minute, with periodic peaks of 12–20. A
materially faster production rate therefore means more completed M&T buffers or
loader churn. It can reflect increased AD-operation/event volume, a backlog, or
repeated attempts; `messages` alone cannot distinguish these. Production should
compare the same minute buckets with M&T `collector.log`, the
`ad_operations` SQL*Loader outcome, queue depth/oldest file, and GUI incident
onset. This loader does not execute in `jsvc` or consume `admin-http-pool`
threads directly, so it is a possible shared-load/amplification clue rather
than a complete GUI root cause.

The Setroubleshootd line is an unsuccessful attempt to launch the SELinux
troubleshooting helper after audit events; it is not itself the underlying
denial. On the control, recent AVCs are predominantly permissive container
domain events from PostgreSQL, Elasticsearch, RabbitMQ/Erlang, Grafana,
Prometheus, and ISE status/counter collectors. The records say `permissive=1`,
so the inspected operations were logged but allowed. A faster production rate
means more underlying AVC-producing operations. The decisive evidence is the
associated audit record's `comm`, path, source/target context, operation, and
`permissive` value. If `nginx`/Kong dominates only during degradation, that
supports gateway pressure; PostgreSQL, Ignite/Java, or metric collectors point
elsewhere.

Other routine signatures found on the control include transient rsyslog
refusals to `127.0.0.1:5514` during service startup, DNF make-cache warnings,
container `mqueue` SELinux mount warnings, and the daily midnight rsyslog
`ise-kong/error.log` truncation notice caused by log rotation. Across the 15-day
window there was no `invoked oom-killer`, `Memory cgroup out of memory`,
`Killed process`, kernel-panic, lockup, or hung-task signature. The production
`nginx invoked oom-killer` fragment must therefore remain classified as a real
OOM event pending its victim/cgroup lines, not as this routine appliance noise.

## 2026-08-30 — One-to-two-hour inter-PAN Data Grid partition modeled

**Source:** operator hypothesis, exact Patch 3 `ignite-config.xml` and
`ignite-control.sh`, and Apache Ignite 2 documentation

**Provenance:** Patch behavior verified locally; production occurrence not yet
established

Exact Patch 3 fixes discovery and communication to TCP 47500 and 47100 with no
port range, configures 120-second failure detection, and uses discovery
connection-recovery and maximum-acknowledgement windows up to 300 seconds. The
primary enables persistent-baseline auto-adjust after 30 seconds of stable
topology. Therefore a one-to-two-hour inability of PPAN/PMNT and SPAN/SMNT to
communicate is long enough to create member-failure/left processing, a baseline
change and rebalance, then a second join/baseline/rebalance cycle after healing.
The third 64-GiB PSN is also expected to be a server-mode member and its
reachability determines the real partition shape.

This is a plausible initiator for the production cascade, not yet proof of it.
The Patch 3 Application Server connects to `localhost:10800`; remote 47100 or
47500 filtering cannot directly issue that loopback refusal. It must first
cause the PAN's local Ignite connector/process to disappear or remain unusable.
If local 10800 stays present, topology exchange, rebalance, cache exceptions,
or timeouts—not immediate connection refusal—are expected. If it disappears,
the two internal ten-attempt/approximately-27-second callers can themselves
produce roughly a thousand failure lines per hour, before GUI traffic. Network
healing need not clear a stopped Data Grid container, persistence/recovery
failure, or already-saturated Admin/Kong queues, which is compatible with GUI
recovery only after an ISE application restart.

The next production bundle must establish ordering on all three nodes:
47100/47500 failure, node-left/failed event, coordinator/topology version,
baseline adjustment, partition exchange/rebalance, PAN 10800 loss, first
`IgniteClientPool` refusal, and first Admin threshold. The PSN view is the
independent discriminator. If inter-node failure precedes local listener loss,
this becomes a strong root-cause branch; if the listener disappears first, the
trigger is PAN-local.

## 2026-08-30 — 3.3 versus 3.5 robustness claim bounded

**Source:** rooted 3.3 Patch 11 control, exact 3.5 Patch 3 artifacts, local
cross-version restore experiment, and Cisco release/field-notice documentation

**Provenance:** Architectural difference verified locally; general reliability
comparison not established

ISE 3.5 cannot be declared generally less robust from this incident. A narrower
statement is supported: its Admin plane has a new failure and amplification
path that the rooted 3.3 Patch 11 node does not have. ISE 3.5 adds persistent
Data Grid/Ignite, 47100/47500 cluster dependencies, the local TLS 10800 client
connector, and serialized client construction shared by Admin-dependent work
and recurring monitors. Failure there can saturate the Admin plane while the
separate Protocols Engine continues RADIUS/TACACS service. This is weaker fault
containment for the demonstrated failure mode, not proof of worse normal-case
or policy-service reliability.

The comparison also pits a longer-lived 3.3 branch against a newer 3.5
architecture. Cisco dates 3.3 GA to July 2023 and Patch 11 to April 2026, versus
3.5 GA in September 2025 and Patch 3 in April 2026. Patch count alone does not
measure quality, and 3.3 had serious earlier defects: field notice FN74403
documents shared-memory allocation failures causing extreme GUI slowness and
MnT/Application Server disruption through 3.3 Patch 4, fixed in Patch 5. The
healthy Analytics-disabled restore control further prevents attributing the
production incident to every 3.3-to-3.5 migration. Production chronology and
the Analytics-enabled backup remain the discriminating evidence.
