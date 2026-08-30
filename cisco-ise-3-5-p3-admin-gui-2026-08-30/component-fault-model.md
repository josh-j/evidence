# ISE 3.5 Patch 3 Admin-plane component and fault model

Date: 2026-08-30

Status: evidence-backed working model; production initiator not yet proven

## What this model can and cannot establish

The available evidence is sufficient to explain **how** one unavailable local
Data Grid endpoint can turn into recurring Ignite errors, blocked Admin
requests, thread-pool alerts, Kong pressure, and a very slow GUI while
RADIUS/TACACS continue to work. It is not yet sufficient to prove what makes
the Data Grid endpoint unavailable in production, or whether Ignite is the
first failing component rather than one participant in a feedback loop.

The model below distinguishes:

- **Verified locally:** rooted ISE 3.3 Patch 11 observations or exact ISE 3.5
  Patch 3 files/bytecode from the archived patch.
- **Reported:** observations from the affected production cluster.
- **Inference:** a causal interpretation that needs production timing or a
  controlled experiment.

## Request and service topology

```text
Admin browser / Admin API client
                |
              TCP 443
                v
Kong API gateway container
  nginx/OpenResty master + workers
  hard platform-profile memory limit
                |
       GUI catch-all upstream
                v
Tomcat/jsvc HTTPS connector :9443
  AdminExecutorPool
  thread prefix: admin-http-pool
                |
        application handlers
      /          |           \
 Oracle DB   local Ignite   other ISE services
                 TLS client
              localhost:10800
                    |
           ISE Data Grid container
       Ignite discovery/cluster/WAL/cache
          47500 / 47100 / persistent data

RADIUS/TACACS clients
        |
        v
Protocols Engine / policy-service path
```

The separate policy-service path explains why healthy RADIUS/TACACS does not
contradict severe Admin GUI failure. It is strong evidence against guest-wide
CPU, memory, storage, or kernel failure during the long GUI-only interval.

## Component purposes and verified implementation

| Component | Purpose | Verified implementation facts | What failure looks like |
|---|---|---|---|
| Kong API Gateway | Terminates the public Admin/API connection and routes it to ISE services. | On the rooted 3.3 node it is a containerized Kong/nginx/OpenResty service. Public 443 maps to Kong 8443; the GUI catch-all routes to Tomcat 9443. The observed GUI upstream has a 60-second connect timeout, very long read/write timeouts, and five retries. Exact 3.5 Patch 3 control scripts impose `apigateway.memory` as a hard container memory limit. | Worker/connection pressure, upstream timeout errors, 5xx responses, or a container-cgroup OOM. These can originate in Kong or be downstream backpressure from Tomcat. |
| Tomcat/jsvc Application Server | Runs the ISE Admin web application and application APIs. | On the rooted 3.3 node, connector 9443 uses executor `AdminExecutorPool` and threads named `admin-http-pool`. Exact Patch 3 profile settings permit 100–800 Admin threads depending on the active platform profile; `vm_standard3_flex_32` specifies 800. The active production profile and generated value are not known. | Requests remain busy waiting on DB, Ignite, remote services, locks, or serialization. The busy threshold is crossed; later requests queue or time out. More threads do not repair the blocked dependency. |
| `CharacterEncodingFilter` | Applies character-encoding behavior as a servlet filter around Admin requests. | It is part of the common request stack. | Its presence in a blocked request stack identifies the request path, not the operation consuming time. It is not evidence that encoding is the root cause. |
| ISE Data Grid / Apache Ignite | Holds shared, persistent application cache/state and distributes it between eligible ISE nodes. | Patch 3 ships Apache Ignite 2.16.0 as `ise-ignite-container`. It persists Data Grid state/WAL under `/opt/ignite/data`. Its exact configuration includes endpoint/profile state (`EDF2EndPoint`), EDDA integration data (`UPSEddaData`), endpoint license state (`EndPointLicenseInfo`), MFC endpoint cache, and event data. Discovery is Oracle-backed through `TBL_ADDRS`. | Local listener absent, inactive cluster, TLS/config problem, stale discovery/topology, persistence/WAL defect, or resource exhaustion. Application clients fail and Data Grid-dependent application work blocks or errors. |
| `IgniteClientPool` | Gives the Application Server three on-demand thin clients: main, UI, and event-listener clients. | Exact Patch 3 bytecode connects only to TLS `localhost:10800`; client timeout is 300,000 ms, heartbeat 3 seconds, Ignite retry limit 5, and Cisco wrapper maximum retries 10. Failed wrapper retries pause 3 seconds. Client construction is guarded by one class-wide monitor shared by main, UI, and event-listener clients. | `Failed to create Ignite client ... Connection refused`. Ten rapid-refusal attempts consume about 27 seconds of retry sleeps. Other callers that need client construction wait behind the same monitor. |
| `IgniteStateMonitorThread` | Checks cluster membership/state and attempts resiliency/activation work. | Exact Patch 3 bytecode starts after 60 seconds and uses fixed-delay scheduling: every 30 seconds on primary/standalone nodes and 60 seconds on secondary nodes. The primary path calls `getUIClient()`. On errors it can reinitialize the client pool. | A persistent local refusal causes a fresh multi-attempt creation cycle on each run and repeated client-pool resets/reinitialization. |
| `IgniteEventListenerThread` | Polls the Data Grid event cache used by Admin-node notification/application handling. | Exact Patch 3 bytecode runs on PAP nodes after 60 seconds, then every 30 seconds with fixed delay, and calls `getEventListenerClient()`. | It supplies a second repeating client-creation caller which shares the client-pool class lock with the state monitor and GUI callers. |
| Data Grid reset/reconfiguration | Rebuilds the local Data Grid runtime and persisted state. | Patch 3 `ignite-control.sh` stops/removes the container and image and deletes persisted Data Grid data, work database, snapshots, diagnostics, and lock state before rebuilding. | It can clear corruption, stale topology, or incompatible durable state. Improvement after a reset is meaningful but not causal proof when other settings were changed concurrently. |
| AD Agent / PBIS runtime | Joins ISE to AD and supplies DC discovery, Kerberos/NTLM, user/group lookup, and machine-account behavior. It also attempts to emit local audit/event records. | Patch 3 contains BeyondTrust PBIS/AD Runtime 7.1.1. The exact `Failed to write records. Error code [%d]` string is in `libeventlog.so` and `libeventlog_norpc.so` beside `LwEvtWriteRecords`, `LwmEvtWriteRecords`, `localhost`, and local endpoint `/var/lib/pbis/.eventlog`. | This exact message is a failure of the PBIS Event Log client path. It is not by itself a failure to write AD objects, ISE Oracle configuration, or Ignite data. Persistent failure can still create a native retry/logging storm. |
| Protocols Engine | Handles RADIUS/TACACS policy traffic. | It is a distinct ISE process/path from Kong and the Admin connector. | It can remain healthy through Admin-plane saturation, matching production. |

## What the `console.log` timeout stack proves

The reported production combination is:

```text
thread: admin-http-pool-...
java.net.SocketTimeoutException: Read timed out
com.cisco.cpm.admin.infra.utils.CharacterEncodingFilter.doFilter(
    CharacterEncodingFilter.java:123
)
```

The rooted class matching that source line resolves line 123 exactly to:

```java
chain.doFilter(request, response);
```

Patch 3 `web.xml` maps this filter to `/*`, so almost every Admin GUI request
passes through it. The filter sets UTF-8 request/response encoding and delegates
to the rest of the servlet chain. It performs no socket read at line 123. A
deeper handler or filter throws the timeout, and Java unwinds the exception
through this common frame.

`admin-http-pool` identifies the Tomcat executor thread processing the inbound
Admin request. `SocketTimeoutException: Read timed out` means a TCP connection
was established but a read did not return data before its configured timeout.
This differs from Ignite's `Connection refused`, which means no listener
accepted the connection.

The pool alert and timeout are therefore related as load and consequence:

```text
Admin request enters an admin-http-pool thread
                    |
                    v
handler waits on a socket read
                    |
          the thread remains busy
                    |
more requests enter and wait on the same/other dependency
                    |
busy-thread threshold is crossed
                    |
ISE raises Administration thread-pool threshold alert
                    |
individual reads eventually time out and unwind through line 123
```

The usual direction is slow dependency/read first, occupied threads second,
threshold alert third. Pool saturation can then create feedback: requests queue,
Kong retains upstream connections or retries, and new arrivals consume newly
freed threads. The roughly 15-minute alert interval can be the monitoring alarm
evaluation cadence rather than a new failure every 15 minutes.

The excerpt does **not** identify which socket was read. Preserve the frames
between the top-level `SocketTimeoutException` and `CharacterEncodingFilter`:

| Adjacent stack family | Likely read direction |
|---|---|
| Tomcat `Http11InputBuffer`, `InputBuffer`, request parameter/body parsing | Reading the inbound request from Kong/client. |
| Apache HTTP client, `HttpURLConnection`, OkHttp, REST client | Application Server making an outbound HTTP(S) call. |
| `oracle.jdbc` | Oracle query/result/socket read. |
| Ignite thin-client classes | Data Grid read, possibly wrapped from Ignite's client exception path. |
| Cisco inter-node/PSC/deployment client classes | Call to another ISE node or local ISE service. |

The full stack, request URI, start/end timestamp, and connected remote endpoint
are needed to name the dependency. Repeated copies of the same full stack are
more useful than the common outer filter frame.

## What `ad_agent: failed to write records error code [1]` is writing

Production reportedly has thousands of these messages. Exact Patch 3 binaries
resolve the call as PBIS Event Log API `LwEvtWriteRecords`. The intended path is:

```text
ISE AD/PBIS operation
       |
       v
PBIS libeventlog client
       |
       +-- local LWMSG endpoint /var/lib/pbis/.eventlog
       `-- localhost RPC compatibility path
                    |
                    v
          PBIS eventlog service
                    |
                    v
          local PBIS event database
```

BeyondTrust's AD Bridge diagnostics likewise describe `eventlog-cli ...
localhost`, `lwsm status eventlog`, and a local event database. On the rooted
ISE 3.3 control, `lwsmd` runs registry, LSASS, I/O, netlogon, redirector, and
NTLM services, but `lwsm list` shows no `eventlog` service and `ss` shows no
`/var/lib/pbis/.eventlog` socket. This suggests ISE's packaged PBIS runtime may
retain an event-emission call while omitting or disabling the general PBIS
Event Log service. The production 3.5 node must be checked directly before
calling that behavior expected or defective.

Code `1` is not sufficiently descriptive to identify the precise lower-level
reason from the log line alone; the library logs only the numeric return from
the eventlog API. It should not be interpreted as an Oracle, Ignite, or AD
server error code.

The error volume matters, but its timing determines its role:

- If the same rate exists while the GUI is healthy, it is probably noisy AD
  audit-path failure and a background load contributor, not the initiator.
- If it starts at the first Ignite/Kong/Admin transition and stops after an
  application restart, it becomes part of the shared retry/backpressure loop.
- If AD joins, DC discovery, group lookup, or AD-authentication latency also
  fail, investigate PBIS/LSASS more broadly. Healthy RADIUS/TACACS alone is not
  enough when those authentications may use non-AD identity sources; identify
  whether production's successful requests actually exercised AD.
- Because this is native PBIS code, it does not directly explain Java
  `char[]` CPU. It can indirectly add CPU, disk logging, and service pressure.

## Exact Ignite retry amplification

For an `IgniteClientPool` stack reporting `Connection refused`, the exact Patch
3 destination is `localhost:10800`. A refusal means the TCP stack found no
accepting listener at that instant. It is different from a discovery failure on
47500 or a cluster-communication failure on 47100, although those server-side
failures can be why the 10800 thin-client listener is unavailable.

The Patch 3 sequence is:

1. A state-monitor, event-listener, or application request asks for a client.
2. If its client is absent/inactive, it enters the one class-wide synchronized
   construction section.
3. Cisco's wrapper makes as many as 10 creation attempts. With immediate TCP
   refusals there are nine 3-second waits, or about 27 seconds, plus overhead.
4. A failed state-monitor run finishes, waits its configured 30 seconds, then
   repeats. The PAP event listener has its own 30-second schedule and repeats
   the same behavior.
5. GUI requests requiring Data Grid work wait for the same construction lock.
   After obtaining it they may perform their own failed retry cycle.
6. Under concurrent 09:00 GUI/API demand, blocked Admin threads accumulate much
   faster than they retire. Threshold alerts and end-to-end timeouts follow.

Two scheduled callers alone can produce roughly 20 error lines per combined
retry round. Over hours, “thousands” is therefore plausible without thousands
of independent root failures. The production error count proves persistence
and amplification; it does not measure the number of listener outages.

The 300-second Ignite client timeout can make non-refusal failure modes much
longer. An immediate `Connection refused` normally returns quickly, leaving the
explicit 3-second retry waits as the dominant delay.

## Working failure propagation model

The best-supported propagation model is:

```text
Trigger or initiating defect
  - local Data Grid container/listener becomes unavailable, or
  - durable cluster/cache state makes it restart/inactivate, or
  - work-hour Admin/API demand reaches a latent failure
                         |
                         v
localhost:10800 refuses IgniteClientPool connections
                         |
                         v
30-second state monitor + 30-second PAP event listener
perform 10-attempt serialized client creation cycles
                         |
                         v
Data Grid-dependent GUI requests queue behind the same lock
and/or repeat the same retry work
                         |
                         v
admin-http-pool busy threads accumulate
                         |
                         +--> threshold alert evaluation repeats
                         +--> console read timeouts / slow pages
                         +--> jsvc CPU and allocation/logging increase
                         |
                         v
Kong workers retain slow upstream requests and gateway memory grows
                         |
                         v
Kong worker errors and possibly its cgroup OOM
```

This ordering is an inference. Kong pressure could instead be the initiating
fault, or both components could be driven by the same 09:00 request source.
Only first-occurrence timestamps and live listener/cgroup state can order them.

`java char[]` CPU is compatible with JSON/HTML serialization, exception and
stack-trace formatting, logging, or repeated request handling. It does not
select one of those causes by itself.

## Kong/nginx OOM interpretation

The reported values are exact:

```text
mem   1572864 kB
limit 1572864 kB
```

1,572,864 KiB is 1,536 MiB. Exact Patch 3 scripts read
`apigateway.memory` from the active platform properties and pass it to the
container runtime as a hard memory limit. Exact Patch 3 profiles that assign
1,536 MiB include several small physical/cloud/VM profiles. Other profiles
assign more; for example:

| Patch 3 profile | Kong limit |
|---|---:|
| `vm_standard3_flex_4` | 1,536 MiB |
| `vm_standard3_flex_8` | 3,072 MiB |
| `vm_standard3_flex_16` | 6,144 MiB |
| `vm_standard3_flex_32` | 12,288 MiB |
| `sns3815` | 1,536 MiB |
| `sns3595` | 3,072 MiB |

Patch 3's decompiled `PlatformProfileServiceImpl.mapVmToProfile()` makes the
reported 32-vCPU/64-GiB combination especially significant. For a generic,
non-cloud `ISE-VM`, it tests CPU and RAM thresholds in order. The VM misses the
96-GiB-and-larger rules and then matches `cpu >= 32 && MemTotal >= 31,200,000
kB`, selecting `sns3815`. The generated `sns3815` properties assign:

| Property | `sns3815` value | `vm_standard3_flex_8` (64-GiB shape) |
|---|---:|---:|
| Kong `apigateway.memory` | 1,536 MiB | 3,072 MiB |
| Data Grid `ise-data-grid-service.ram` | 2 GiB | 1 GiB |
| Admin `tomcat.adminThreadPool.maxThreads` | 200 | 500 |
| Application Server `java.maxHeap` | 12,288 MiB | 16,384 MiB |

This is not a restore-time database decision. The local appliance service
computes the profile from platform identity, CPU count, and `/proc/meminfo`,
then writes `/opt/CSCOcpm/config/platform.properties-active`. It rebuilds that
file when absent, when the node persona configuration changes, or when RAM
changes. The demonstrated invalidation check does not compare the saved CPU
count, so a CPU-only resize can leave stale generated properties until another
rebuild trigger.

This makes a Kong/nginx cgroup OOM technically credible and elevates the active
profile from a generic lead to a concrete prediction: absent cloud detection
or other platform classification, the reported VM shape should select
`sns3815`. Production does not require root to confirm the profile: the
supported `show tech-support` path invokes ISE's platform-property reader and
prints `Profile : <name>` under `Displaying ISE Profile`. A GUI-generated
support bundle also includes show-tech material. Production must still prove
that the 1,536-MiB cgroup is Kong; `nginx invoked oom-killer` alone is
insufficient, and TAC/root assistance may be needed for runtime container
inspection.
Do not manually edit Cisco's generated properties or container limit. Correct
the VM to a Cisco-supported CPU/RAM shape or obtain a Cisco-supported profile
correction from TAC.

However, `nginx invoked oom-killer` names the allocating task that entered the
kernel OOM path, not necessarily the victim. Proof requires the continuation:

```text
oom_memcg=...
Memory cgroup out of memory ...
Killed process <pid> (<name>) ...
```

The cgroup/container path and `memory.events` counters should agree. The VM's
64 GiB total memory is not the relevant limit for this event.

## The reported kernel panic is a separate branch until proven otherwise

An OOM-killer event is **not** a kernel panic. A real panic normally stops or
reboots the entire guest, which should interrupt Admin, RADIUS, and TACACS and
produce an uptime discontinuity. That is incompatible with a long GUI-only
degradation while authentication continues, unless the panic was a later or
separate event.

Record the kernel issue as reported but unclassified until the following are
available:

- the exact `Kernel panic - not syncing: ...` signature and timestamp;
- guest reboot and uptime evidence;
- Hyper-V VM/host event timestamps;
- kdump/vmcore presence or console capture;
- RADIUS/TACACS continuity across the alleged panic time.

If there is no reboot/uptime discontinuity and authentication continued, do
not use “kernel panic” as the explanation for the Admin incident.

## Ranked hypotheses and discriminators

| Rank | Hypothesis | Present assessment | Decisive discriminator |
|---:|---|---|---|
| 1 | Local Ignite thin-client service becomes unavailable; synchronized retries amplify it into Admin saturation. | Best match for the exact repeated stack, restart recovery, and possible Data Grid-reset benefit. The initiator is still unknown. | At first onset, show whether `localhost:10800` loses its listener or the Data Grid container restarts/inactivates **before** Admin busy threads rise. |
| 2 | 09:00 GUI/API demand triggers or magnifies a latent Admin/Data Grid failure. | Strong temporal fit; no request-source inventory yet. | Correlate first minute with Kong access logs, Admin sessions, API clients, scheduled reports, and page/URI latency. Temporarily remove one identified poller in a controlled window. |
| 3 | Kong memory/worker pressure is an initiating or feedback-loop fault. | A 1.5-GiB cgroup OOM and worker errors are material, but the victim, cgroup, and ordering are missing. | Full OOM record, Kong cgroup counters, worker/connection counts, exact error, and timestamp preceding or following Ignite/Admin onset. |
| 4 | The 32-vCPU/64-GiB mixed VM shape selects the undersized `sns3815` profile and makes Kong/Admin capacity a failure amplifier. | Exact Patch 3 bytecode predicts `sns3815`: 1,536-MiB Kong, 2-GiB Data Grid, 200 Admin threads, and 12-GiB JVM heap. The OOM limit exactly matches, but its cgroup/victim and the production active profile remain unproven. | Run supported `show inventory` and `show tech-support` on all nodes; record the latter's `Profile :` line. Ask TAC to confirm active limits and container cgroups. If confirmed, test a Cisco-supported CPU/RAM shape with TAC guidance and compare the same work-hour load. |
| 5 | PBIS Event Log client retries are an initiator or resource amplifier. | Thousands of errors make this material, but the write target is the local PBIS eventlog path rather than ISE/AD application data. It has no demonstrated dependency on Ignite or the Admin executor. | Compare its per-minute rate in healthy and degraded windows; capture PBIS `eventlog` service/socket state and the immediately preceding AD-agent messages. |
| 6 | Rare Analytics-enabled 3.3 state survives restore and activates a defective 3.5 consumer or unusual migrated data path. | Plausible unique variable. Enablement uses existing tables rather than schema DDL; 3.5 has explicit Analytics migration handlers. Disabled restore control is healthy. | Restore the affected enabled backup as Run C and compare rows, handlers, Data Grid behavior, and 72-hour load with Runs A/B. Then use Cisco-supported disablement for Run D. |
| 7 | A general 3.3-to-3.5 schema migration defect causes the incident. | Weakened by the successful Analytics-disabled 3.3→3.5 control, but data-volume/object-specific defects remain possible. | Compare affected versus control migration warnings, object counts, and state. |
| 8 | Hyper-V storage, vNUMA, or guest-wide resource exhaustion. | Low support: latency and general metrics are normal, VM fits vNUMA limits, and authentications remain healthy. | A time-aligned host/guest anomaly beginning before the Admin event. |
| 9 | Kernel panic causes the recurring GUI-only slowdown. | Currently inconsistent with continued authentication and persistent GUI-only degradation. | Exact panic signature plus reboot/uptime break at the same onset. |

## Minimum decisive production capture

Capture the first healthy-to-degraded transition, not only the saturated tail.
Use supported ISE collection where possible and have Cisco TAC obtain root-only
container/cgroup details.

1. Preserve the first `IgniteClientPool` exception and confirm its source
   thread and `localhost:10800` destination.
2. At the same second, record Data Grid process/container state, restart count,
   cluster active state, listener state on 10800, and relevant 47100/47500
   errors.
3. Record Admin connector active/busy/max threads and queue depth. Capture a
   working JVM thread dump before total saturation; group stacks by wait/lock
   target rather than by the outer servlet filter.
   For every `SocketTimeoutException`, retain the complete frames above
   `CharacterEncodingFilter.java:123`, request URI, and socket destination.
4. Preserve exact Kong worker errors and access-log request rate/URI/source.
   Obtain Kong container `memory.current`, `memory.max`, `memory.events`, OOM
   counters, worker count, connections, and restart count.
5. Preserve the complete kernel OOM record, including cgroup path and killed
   process.
6. Capture `show inventory` and the `Profile :` line from supported
   `show tech-support` on every node. Ask TAC to capture generated values for
   Kong memory, Data Grid RAM, and Admin max threads, plus runtime container
   limits.
7. Probe the PPAN and SPAN directly during the same minute. Record whether only
   the PPAN, both PANs, or the shared VIP is slow.
8. Record active Admin sessions, external API clients, scheduled jobs/reports,
   and authentication rate around 08:45–09:15.
9. Count `ad_agent.log` write failures per minute in both healthy and degraded
   windows. Have TAC capture PBIS `lwsm` service state, `.eventlog` socket
   presence, local eventlog diagnostics, and whether successful policy
   authentications actually used AD.

The alert recurring every 15 minutes should initially be treated as alarm
evaluation/reporting cadence. Do not infer a 15-minute initiating job without
a distinct first-error sequence at that cadence.

## Safe remediation logic

- An ISE application restart is a demonstrated containment measure, not a root
  fix. Preserve the onset evidence before restarting when operationally safe.
- Do not simply raise the Admin thread count. It can admit more blocked work,
  increase heap/allocation pressure, and move the failure into Kong or the JVM.
- Do not manually override container memory or edit generated platform files;
  first prove the selected profile and ask Cisco to correct an erroneous
  profile/limit through a supported path.
- Treat Data Grid reset as a destructive rebuild of persistent Data Grid state,
  not a harmless cache flush. Use it with Cisco guidance and collect the old
  Data Grid diagnostics first.
- Do not bypass the Analytics JWT check or write licensing rows directly. It
  would create an unsupported state and invalidate the experiment. Use the
  affected configuration backup or a Cisco-issued token.
- If Run C uniquely reproduces the problem and Cisco-supported disablement in
  Run D clears it, the durable fix belongs with Cisco: supported entitlement
  disable/cleanup, a migration hotfix, or a corrected release. Retain before
  and after row metadata and logs for TAC without exposing JWT/claim material.

## Local control status

The disposable target `laba-ise-035` is currently running ISE 3.5.0.527 Patch
3 with the Analytics-disabled 3.3 configuration restored. Data Grid,
Application Server, API Gateway database/service, M&T, and Protocols Engine are
running. On 2026-08-30 at the check recorded for this document, the GUI returned
HTTP 302 in approximately 114 ms. Snapshot
`control-disabled-restored-p3` preserves this state; snapshot
`clean-p3-pre-restore` preserves the clean Patch 3 baseline.

The planned offline shutdown used to create the control snapshot supplies a
useful bounded comparison. The state monitor saw its last `ACTIVE` result at
14:18:25 UTC, then logged nine connection-refused attempts at 3-second spacing
from 14:18:55 through 14:19:19 as services stopped. The client pool began
closing at 14:19:01. After the VM restarted, the monitor reported cluster
`ACTIVE` and node `RUNNING` every 30 seconds through 14:40:40 with no further
refusal. This is a lifecycle race, not the production signature of repeating
cycles throughout a slow-GUI interval.

Run C remains blocked on the affected Analytics-enabled 3.3 configuration-only
backup and its encryption key. No local backup or JWT was found, and no
unsupported Analytics state was fabricated.
