# Cisco ISE 3.5 Patch 3 Admin GUI degradation

Session opened: 2026-08-30  
Status: active investigation  
Primary question: can state associated with the Cisco-enabled Analytics license/API in an ISE 3.3 Patch 11 configuration backup survive migration to ISE 3.5, and can that state cause or contribute to recurring Admin GUI saturation?

## Executive summary

The strongest demonstrated symptom is sustained **administration-plane saturation**, not a general ISE or Hyper-V resource failure. During the incident, Admin GUI pages take seconds to minutes, administration HTTP threads time out or exhaust, `jsvc` consumes roughly two CPU cores, and ISE Prometheus shows a sustained increase of about 10% user CPU. RADIUS and TACACS authentication reportedly remain healthy.

### Current working root-cause statement

The most likely initiating fault area is the **local ISE Data Grid/Ignite
service on the affected PAN**, specifically loss or repeated restart/inactive
state of its thin-client listener on `localhost:10800`. The thousands of
`IgniteClientPool` immediate connection refusals are direct evidence that no
listener accepted those local connections during the affected interval. Exact
Patch 3 code then turns that service failure into serialized retry cycles that
block Data Grid-dependent Admin work.

The likely propagation is:

```text
local Data Grid listener stops/restarts or never becomes fully active
    -> localhost:10800 connection refusals
    -> serialized Ignite client retries and blocked Admin requests
    -> admin-http-pool occupancy and threshold alerts
    -> jsvc CPU/log/char[] work and GUI timeouts
    -> Kong retains slow upstream requests, worker pressure, possible OOM
```

The reason the listener becomes unavailable is not yet proven. The leading
branch is a runtime container/JVM exit or restart-suppression problem under
daytime workload. Connector binding and discovery/activation are next.
Permanent on-disk corruption is less consistent with an ordinary application
restart restoring service while retaining the same persisted data; local
recovery/checkpoint state remains possible but is not the leading explanation.
The 32-vCPU/64-GiB platform profile and 09:00 request load are credible
capacity amplifiers. Analytics-enabled migrated state remains a possible input
to the failing path, not the presently demonstrated failing component.

Patch 3 supplies two concrete ways this can remain PAN-local. With predicted
profile `sns3815`, its 2-GiB Ignite container is configured with a 1-GiB Java
heap and an approximately 819-MiB initial/max persistent data region, leaving
about 205 MiB for all remaining JVM native and container memory; the JVM exits
on OOM. Separately, a stranded `/tmp/ise-ignite-service.lock` makes the start
path report “initializing” without starting a missing container, and Option 45
explicitly removes that lock. Local WAL/persistence recovery or discovery/TLS
startup failures are the next focused branches.

These are candidate explanations, not production findings. No Ignite-specific
OOM, exit code, stranded lock, WAL error, or connector bind error has yet been
provided from the affected node. The established lead is the missing/refused
local listener; the first Data Grid transition and container exit reason are
the evidence still required for root cause.

The initial trigger has two testable models. Ignite may fail during startup and
remain mostly latent until 09:00 GUI/API demand exposes and amplifies it; or it
may remain healthy overnight and exit only when work-hour activity crosses a
local runtime threshold. The first refusal timestamp after restart separates
them. Patch 3 stores replicated endpoint, EDDA, licensing, MFC, and OIDC session
context data in Ignite, giving daytime authentication/session/API work concrete
ways to raise local cache/query pressure. The exact session settings changed in
production must be identified before associating them with any specific cache.

The operator has clarified that, once triggered, the GUI can remain slow for
12 hours and recovers only after an ISE application restart. That persistence
substantially weakens a simple work-hour user burst as the complete cause. It
separates the incident into an initiating event and a **latched application
state**. A 09:00 browser/integration burst, scheduled job, or topology event may
initiate the transition, but a failed Data Grid listener/container, stuck
topology or persistence recovery, invalid static Ignite client-pool state,
blocked/serialized Application Server work, or a self-renewing Kong/client
retry loop must keep it present. New traffic may sustain or expose the state;
it need not be the original fault.

This also means that an Administration thread-pool alert need not prove the
same 200 requests live for 12 hours. The pool can be occupied by successive
generations of slow requests while the shared dependency remains unavailable.
Patch 3's approximately 27-second refusal cycle and five-minute Ignite timeout,
and the rooted control's one-hour Kong upstream-read timeout, are all shorter
than the reported incident. Twelve hours therefore requires renewed work or a
backend condition that never self-recovers. Application restart simultaneously
reconstructs the Application Server, Kong/API gateway, and local Data Grid
runtime, so restart recovery strongly localizes the problem to that application
stack but does not by itself distinguish which component latched.

The post-Data-Grid-reset recurrence did not begin at 09:00. After approximately
48 healthy hours, degradation returned around 15:00–16:00. This weakens a fixed
clock-time job or start-of-workday event as the universal trigger. It instead
supports a threshold whose runway depends on starting state and accumulated
work: cache/event/session state, a memory or connection leak, persistence/WAL/
checkpoint growth, repeated topology/client recovery, or a request-count-driven
defect. A normal application restart may leave less headroom than the deeper
PAN-local Data Grid rebuild, causing the threshold to be crossed earlier on the
next workday; the reset may provide enough headroom to survive an additional
day and fail only after more daytime demand.

This timing remains confounded because session-related settings changed in the
same maintenance window. It also does not prove that the first Data Grid fault
occurred at 15:00–16:00: the first 10800 refusal or unhealthy state may predate
the first visibly slow page. Analysis should use hours since recovery and
cumulative workload, not only time of day.

A one-to-two-hour Data Grid connectivity break between the PPAN/PMNT and
SPAN/SMNT is now a strong candidate initiator, but only if its chronology is
right. Patch 3 declares a failed member after a configured 120-second failure
detection interval and enables persistent-baseline auto-adjust after only 30
seconds of stable topology. A long break can therefore cause a member-loss
exchange and rebalance, followed by another topology exchange and rebalance
when connectivity returns. All three reported 64-GiB nodes should be Ignite
servers, so the third PSN's view determines whether this is one isolated member
or a different partition shape.

The distinction from the production symptom is important: the Application
Server connects to local `localhost:10800`. A firewall or path failure on
inter-node 47100/47500 cannot directly refuse that loopback connection. It can
indirectly cause the local PAN Ignite member to stop, restart, remain in
recovery, or withdraw its client connector; that produces the observed local
refusal and retry cascade. If 10800 remains listening, the expected evidence is
instead topology/partition-exchange/rebalance delay or cache exceptions. The
network-partition hypothesis becomes strong only if inter-node failures and a
node-left/failed event precede loss of PAN 10800. Recovery of the network also
does not guarantee prompt GUI recovery after Admin and Kong queues have filled
or after the PAN's local Data Grid process has exited.

This working statement is falsifiable without production root access. During
the next onset, supported `show application status ise` and `show ports |
include 10800` should establish whether Data Grid is stopped/restarting or its
listener is absent. First-occurrence timestamps must then order Data Grid,
Admin, and Kong events.

An ISE application restart promptly restores GUI performance. A Data Grid reset/reconfiguration coincided with a longer healthy interval, but user-session and timeout settings were changed at the same time. That event therefore does **not** prove that Data Grid is the cause.

The reported Option 45 reset was performed only on the PAN. Exact Patch 3
control logic makes this a **node-local rebuild**, not a cluster-wide purge: it
stops and removes that node's Ignite container and image and deletes its local
`data`, work database, snapshots, diagnostics, and lock file. It does not drop
the Oracle `TBL_ADDRS` discovery table and does not erase the other nodes'
state. In this deployment, each 64-GiB PAN/PSN is configured by the script as
an Ignite server; after the PAN restarts, baseline auto-adjust and replicated
caches allow it to rejoin and be repopulated from the surviving nodes.

Consequently, the 48-hour improvement is more supportive of a **PAN-local
container/persistence/listener problem or restart/rebalance effect** than of a
cluster-wide bad cache having been permanently removed. A logical state shared
by the surviving nodes could return to the PAN after rejoin or accumulate
again under workload. The simultaneous session-setting changes remain a
confounder.

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
`apigateway.memory` as the Kong container's hard limit. More importantly,
decompiled Patch 3 profile-selection code predicts that a generic non-cloud VM
with the reported 32 vCPU and 64 GiB selects `sns3815`, because it misses the
96-GiB large-profile threshold and then matches the 32-CPU/32-GiB rule.
`sns3815` gives Kong exactly 1,536 MiB, the Admin connector 200 threads, the JVM
12 GiB, and Data Grid 2 GiB. The full OOM cgroup/victim and production's active
properties are still required to confirm this chain, but the numerical match
is now specific rather than coincidental.

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

Specific actions that submit work to this pool are GUI/catch-all requests,
especially `/admin/...`: opening or refreshing data-grid pages, changing their
filter/sort/page, loading endpoint or identity details, running broad searches,
starting or downloading an export/report, GUI dashboard/live-log polling,
login/OIDC/session refresh, legacy integrations calling `/admin/API/...`, and
HTTPS scanners crawling the same catch-all route. Several AJAX calls can be
created by one navigation, and a retrying client can overlap them. ERS `/ers`,
OpenAPI `/api`, pxGrid, SSH, RADIUS, and TACACS normally use other executors.

The strongest Patch 3-specific exhaustion mechanism is a GUI/data-grid/session
request requiring Data Grid state while the local thin-client listener on
`localhost:10800` is unavailable. The request retains its Admin worker while
it waits for the class-wide synchronized `IgniteClientPool` construction path
and/or its approximately 27-second immediate-refusal retry cycle. Other GUI
requests needing the same client accumulate behind that serial point. With the
predicted 200-thread `sns3815` pool, about 7.4 continuously arriving 27-second
requests per second fill every worker; the observed 22-second latency requires
about nine per second. A data-heavy page, several tabs, GUI polling, DNA/legacy
Admin automation, or a scanner can supply that rate without high network
throughput. If the complete timeout stack instead contains Oracle JDBC,
outbound HTTP, Tomcat request parsing, or an ISE inter-node client, then the
specific retaining action is respectively a query/report, remote integration,
slow request body/upload, or deployment call rather than Ignite.

Logged-in user count is not equivalent to pool occupancy: idle sessions own no
worker. At the lab's healthy roughly 0.1-second login-page latency, the
predicted 200-worker pool would require vastly more sustained human traffic
than a normal administrator population. Once backend latency reaches 22–27
seconds, a burst such as 20 users whose page loads each generate ten overlapping
AJAX calls can temporarily occupy 200 workers. Persistence until application
restart, however, favors user demand exposing/amplifying a latched backend,
shared-lock, retry, or automation condition rather than ordinary GUI population
alone. Treat 09:00 as a demand trigger until one-minute request timing proves
whether the source/URI surge preceded the first backend failure.

An HTTPS vulnerability scan is a concrete Admin-pool source only when it hits
443 and Kong forwards paths such as `/admin/...` through the GUI catch-all to
Tomcat 9443. SSH-only ACAS attempts never enter this pool. Fast scanner 401/404
responses are normally cheap; they become material when the catch-all reaches
a slow handler, requests overlap without backoff, or scanning triggers a
shared backend defect. Scanner fingerprints are a dominant source IP/User-Agent,
many paths or methods, 401/403/404/5xx responses, and an onset matching the scan
schedule.

Normal ISE inter-node database replication, messaging/PSC, Ignite 47100/47500,
local Ignite client 10800, pxGrid, RADIUS/TACACS, ERS 9060, and OpenAPI 9070 do
not directly use `admin-http-pool`. Another ISE node consumes the Admin pool
only if it calls the PAN's public GUI/legacy `/admin/...` route, or indirectly
when an already-running Admin request waits on an inter-node call. Group Kong
records by the known PAN/SPAN/PSN addresses to prove this rather than treating
all inter-node traffic as Admin traffic. Disabling ERS/OpenAPI lowers their
priority but leaves the GUI catch-all and `/admin` scanner/legacy paths active.

### Supported mitigation knobs and non-knobs

The strongest ISE-side containment control is **Administration > System >
Admin Access > Settings > Access**. Release 3.5 provides separate access
restriction tabs for Admin GUI/CLI, Admin Services (ERS, OpenAPI, pxGrid, and
Data Connect), and user services. Allowlisting only administrator subnets for
the GUI can prevent ACAS and unintended automation from entering `/admin`
without relying on session counts. Required integrations should be allowlisted
only for the service they actually use and moved away from legacy
`/admin/API/...` calls when supported.

The Admin GUI concurrent-session limit (supported range 1–20) and idle timeout
(6–100 minutes) limit authenticated sessions, not in-flight requests per
session. They cannot stop a scanner before login or a retrying authenticated
client, and an idle session consumes no Admin worker. The separate **Maximum
Sessions per User/Group** and **Counter Time Limit** settings govern RADIUS
network-access session counters on PSNs; changing them is unrelated to the
Tomcat Admin executor.

Ensure diagnostic components have reverted to their default logging levels.
Excessive debug logging can amplify exception formatting, `char[]` allocation,
GC, and disk activity, but suppressing an ERROR storm does not repair its
dependency. Disabling unused ERS/OpenAPI services and restricting their source
addresses reduces API demand, but does not stop Kong, the GUI catch-all, or
legacy `/admin` paths.

The 32-vCPU/64-GiB VM is a supported SNS 3815-equivalent small profile, not an
invalid shape. It nevertheless generates the locally verified small-profile
limits: 200 Admin workers, 1,536 MiB for Kong, 2 GiB for Data Grid, and a
12-GiB Application Server heap. Cisco recommends the SNS 3895 class for PAN and
MnT personas. If production load exceeds the small profile, the supported
capacity lever is an exact larger Cisco profile and/or dedicated PAN, MnT, and
PSN persona placement—not hand-editing individual limits. CPU and memory must
be fully reserved with no hypervisor limit or oversubscription. A move should
be validated with TAC and use a supported CPU/RAM pair.

No supported GUI/CLI knob was identified for `AdminExecutorPool.maxThreads`,
Kong worker count/cgroup memory, Ignite heap, `IgniteClientPool` retries, or the
local 10800 listener. These are generated internal profile/service settings;
changing files or cgroups manually would create unsupported, restart-fragile
state. Ensure Data Grid ports 47500, 47100, and 10800 meet Cisco's network
requirements. If local 10800 disappears, an application restart or Data Grid
reset is recovery, not a durable tuning fix; persistent recurrence requires a
TAC defect investigation/hotfix using the first-failure logs and active profile.

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
- A healthy rooted control also emits `su: (to oracle) root on none`, normally about 6–9 times/minute with sampled peaks of 12–20. Audit evidence maps these invocations to the M&T Log Processor loading small `ad_operations` buffers through Oracle SQL*Loader. A much higher production rate is therefore an M&T AD-operation volume/backlog/retry discriminator, not direct evidence of Admin-thread creation.
- The healthy control emits frequent `Setroubleshootd ... Failed to setup environment correctly` messages after permissive SELinux container AVCs. The helper failure is noise; the associated audit record's process/path/context identifies the real source. A production rate increase should be broken down by `comm`—especially nginx/Kong versus PostgreSQL, Java/Ignite, or monitoring collectors.
- Daily midnight `ise-kong/error.log` truncation notices are routine log rotation. The rooted control had no OOM/panic signature in 15 days of rotated `messages`; production's `nginx invoked oom-killer` remains a real event pending the victim and cgroup lines.
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

### Is 3.5 less robust than 3.3?

Not as a general product conclusion. The evidence does support saying that ISE
3.5 has **weaker fault containment for this particular Admin-plane failure
mode**. The rooted 3.3 Patch 11 system has Kong and the same general
Tomcat/Admin executor architecture, but no ISE Data Grid service. Data Grid is
new in 3.5. It adds a persistent distributed Ignite member, inter-node
47100/47500 dependencies, a local TLS 10800 connector, topology/baseline/
rebalance behavior, and an Application Server client-construction path shared
by Admin work and recurring monitors. A failure in that new dependency can be
amplified into Admin-thread saturation while RADIUS/TACACS remains healthy.

This makes 3.5 **architecturally more complex and potentially more brittle in
the GUI/control plane when Data Grid is unhealthy**. It does not prove that
3.5 is less reliable under normal operation, or that its policy-service plane
is less robust. The observed authentication continuity is evidence of useful
fault isolation in that plane.

Release maturity is also a relevant but non-causal difference. ISE 3.3 was
released in July 2023 and Patch 11 in April 2026; ISE 3.5 was released in
September 2025 and Patch 3 in April 2026. Patch count is not a reliability
metric, but 3.3 Patch 11 represents a longer-lived branch. Older 3.3 patches
also had serious GUI/MnT memory defects: Cisco field notice FN74403 affected
3.3 through Patch 4 and was fixed in Patch 5. Therefore the comparison is
between a mature 3.3 Patch 11 system and a newer 3.5 architecture, not proof
that 3.3 is intrinsically superior.

For this incident, a precise regression statement becomes justified if a
supported capture proves: Data Grid/inter-node or local 10800 failure precedes
Admin saturation, the same workload is healthy on 3.3, and a clean 3.5 control
does not fail unless the Data Grid condition or migrated production state is
introduced. The Analytics-disabled 3.3-to-3.5 restore control remained healthy,
so the current evidence does not yet distinguish a general 3.5 defect from a
production-specific topology/runtime trigger or migrated enabled state.

Sources: [Cisco ISE 3.3 release notes](https://www.cisco.com/c/en/us/td/docs/security/ise/3-3/release_notes/b_ise_33_RN.html),
[Cisco ISE 3.5 release notes](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/release_notes/cisco-ise-release-notes-35.html),
and [Cisco field notice FN74403](https://www.cisco.com/c/en/us/support/docs/field-notices/744/fn74403.html).

### Environment-specific trigger versus product defect

The severity, apparent rarity, and absence of a matching public 3.5 Patch 3
caveat make an environment-specific precondition the strongest prior. This does
not require the full failure mechanism to be unique. A production-specific
condition can make local Data Grid unavailable, while Patch 3's serialized
client retries and weak Admin-plane fault containment turn that condition into
a general product-level failure cascade.

The leading environment-specific inputs are, in order of current evidence:

1. PAN-local Data Grid runtime, persistence, listener, or inter-node topology/
   connectivity state;
2. rare Analytics-enabled configuration/database state migrated from 3.3;
3. production-only Admin/legacy API, Catalyst Center/DNA, scanner, report, or
   browser workload that triggers or sustains the state;
4. co-located PAN/MnT/PSN roles operating within the generated `sns3815`
   Data Grid and Kong limits;
5. another restored-data edge case or unusually large endpoint/profile/session
   population.

Hyper-V NUMA and storage remain lower because no correlated resource anomaly
has been shown. The healthy Analytics-disabled 3.3-to-3.5 restore control also
argues against base 3.5, every restore, or the golden-image procedure being
sufficient by itself. Public release-note silence is only supporting prior
evidence: TAC-only defects, unreported cases, and a new defect with a rare
precondition remain possible.

### Can Hyper-V, S2D/CSV, and VHDX be disregarded?

They can be strongly deprioritized as the mechanism that **sustains** the
12-hour GUI outage, but not entirely eliminated as an **initiating event**. The
measured VHDX read latency was normally below 1 ms, the sampled average was
about 0.05 ms, the observed maximum was about 20 ms, and ISE storage/memory/
network metrics showed no onset-correlated anomaly. RADIUS/TACACS remained
healthy. Most importantly, an ISE application restart restores the GUI without
moving or rebooting the VM, changing the VHDX, changing CSV ownership, or
repairing the storage cluster. Recovery lasting another 24–48 hours while the
same infrastructure remains in place strongly favors application runtime state.

Infrastructure can still supply a short trigger that no longer exists during
the long degraded interval. A Hyper-V virtual-switch or physical-network flap
can interrupt Ignite 47100/47500; a live migration, backup/checkpoint, CSV path
transition, or brief tail-latency/fsync stall can disrupt Ignite persistence or
checkpointing. Average latency counters can hide a short outlier. Once local
Data Grid exits or wedges, the application-level retry and Admin/Kong cascade
can persist after the infrastructure recovers.

The infrastructure branch can be operationally closed only with synchronized
evidence around the **first** Ignite/Data Grid transition—not the later GUI
complaint: per-VHDX read/write tail latency and errors, CSV/S2D path and
ownership events, backup/checkpoint/live-migration activity, Hyper-V host CPU
scheduling, and virtual-switch/NIC drops or resets. Repeated incidents showing
none of those before the first 10800/47100/47500 failure would make Hyper-V and
storage non-actionable residual risks. Until then they remain low-priority
trigger candidates, not plausible explanations for the latched 12-hour state.

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
