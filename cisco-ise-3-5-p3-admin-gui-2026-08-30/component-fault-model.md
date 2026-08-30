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

## What creates `admin-http-pool` threads

Tomcat's HTTPS Admin connector creates and reuses these Java worker threads.
The verified connector listens on local port 9443 and references executor
`AdminExecutorPool`, whose thread-name prefix is `admin-http-pool`. Kong
accepts public 443 traffic and proxies applicable GUI/API requests to that
connector. When concurrent requests exceed the currently idle workers,
Tomcat's executor can create more workers up to its configured `maxThreads`;
after completing a request, a worker returns to the pool for reuse.

Patch 3 sets `minSpareThreads=5` and, for the predicted `sns3815` profile,
`maxThreads=200`. The rooted 3.3 control used the same executor/connector shape
but a generated maximum of 450, demonstrating that the maximum is
release/profile state rather than one universal ISE constant.

Sources of Admin requests include browser page loads, the GUI's AJAX/API calls
and polling, ERS/OpenAPI clients, monitoring or automation calling Admin APIs,
and Kong retrying qualifying failed upstream requests. One human page
navigation can therefore create several concurrent tasks. A configured or
idle login session does not own a permanent worker, and increasing maximum
user sessions does not directly create threads. RADIUS/TACACS requests use the
Protocols Engine rather than this executor.

The distinction between creation and retention is central to this incident.
Tomcat creates workers because requests arrive; slow Oracle, Ignite, HTTP, or
inter-node operations keep those workers occupied. At the Patch 3 maximum of
200, an average 22-second service time needs only about nine new requests per
second to occupy the entire pool (`concurrency = rate × latency`). At a
60-second service time, about 3.3 requests per second is enough. Later requests
then queue behind the pool or fail upstream.

`IgniteStateMonitorThread` and `IgniteEventListenerThread` are separately named
scheduled threads; they do not become `admin-http-pool` workers. Their client
creation does share the `IgniteClientPool` class-wide monitor with Admin work,
so their retry/reset activity can delay existing Admin requests and help keep
the pool occupied. A recurring threshold alarm is a periodic observation that
occupancy remains high, not evidence that a new set of threads is created each
time the alarm fires.

## How long an Admin worker can remain occupied

There is no verified single Tomcat "complete every Admin action within N
seconds" deadline. An `admin-http-pool` thread is reusable and can live for the
Application Server lifetime; the relevant duration is how long its current
request task remains inside application code. It returns to the pool only when
the request completes, throws, is cancelled/interrupted, or a dependency's
specific timeout unwinds the call.

Verified rooted 3.3 gateway and connector values illustrate the different
timers:

| Timer | Verified value | What it controls |
|---|---:|---|
| Kong GUI upstream connect timeout | 60,000 ms | Time to establish the Kong-to-Tomcat connection; not request execution time. |
| Kong GUI upstream read timeout | 3,600,000 ms (1 hour) | Maximum idle interval waiting for upstream response data. |
| Kong GUI upstream write timeout | 3,600,000 ms (1 hour) | Maximum idle interval while sending the request upstream. |
| Kong service retries | 5 | Eligible upstream-failure retries; not an unconditional fivefold request duration. |
| Tomcat connector keepalive timeout | 300,000 ms (5 minutes) | Idle time between HTTP requests on a persistent connection; not the execution limit for an active request. |
| Patch 3 Ignite client timeout | 300,000 ms (5 minutes) | Ignite client/network-operation envelope; an immediate refusal instead follows Cisco's wrapper retry path. |
| Missing-listener wrapper retry cycle | about 27 seconds | Ten rapid-refusal attempts separated by nine three-second sleeps. |

The gateway numbers were read from the active rooted 3.3 control's Kong Admin
API; the Patch 3 gateway service record in affected production has not been
read directly. Patch 3's Ignite values come from exact bytecode/configuration.

The reported `java.net.SocketTimeoutException: Read timed out` is the timer of
the particular socket read occurring deeper in that request. It is not
automatically Kong's one-hour read timeout, the five-minute HTTP keepalive, the
Admin login idle timeout, or a thread-pool timeout. The complete intervening
stack frames are required to identify whether the read belongs to Oracle,
Ignite, outbound HTTP, an inbound request body, or another ISE node and thus to
identify its configured deadline.

Consequently, an Admin pool can remain saturated for minutes even without high
CPU: many workers can each wait on a different or serialized operation whose
timeout is tens of seconds, five minutes, or potentially longer. Admin-session
idle timeout controls how long an authenticated session remains valid and does
not bound how long a current request may occupy a worker.

## Identifying who is creating Admin requests

The most direct source is Kong's proxy access log. On the rooted control it is
`/opt/kong/logs/access.log`; `/opt/CSCOcpm/logs/ise-kong` is a symlink to that
directory. Exact Patch 3 `isesupport.sh` copies it into the support bundle under
`logs/apigateway/`, together with `api-gateway*` logs. The custom access format
records:

- timestamp, directly observed source IP, authenticated remote user, method,
  URI, and HTTP version;
- response status/bytes, referrer, User-Agent, and `X-Forwarded-For`;
- upstream status, bytes sent/received, and connect/header/response times;
- client-certificate serial, issuer, and subject when supplied.

`api-gateway.log` is principally gateway control/target activity, not the
complete request ledger. `console.log` identifies the Java worker and backend
stack but normally does not identify the original client as reliably as the
gateway access entry. Correlate access, Kong error, and Java console entries by
normalized timestamp and request URI.

The active Kong configuration has an important blind spot: its `$loggable` map
suppresses 2xx and 3xx proxy access entries and writes other statuses. A DNA
Center retry storm producing 401, 403, 404, 429, or 5xx should therefore be
visible with source, path, User-Agent, and timing. A flood of successful calls
may not be present in the default access log. In that case use upstream
firewall/load-balancer/NetFlow or a scoped Hyper-V virtual-switch capture to
measure source IP, connection rate, bytes, and timing; TLS prevents an
off-appliance capture from revealing the URI. TAC can arrange temporary
complete gateway request logging if URI-level visibility is required in
production.

The path determines whether a DNA/Catalyst Center connection can consume Admin
workers. Verified routing/executor mappings are:

| Incoming function/path | Backend pool/path | Does it directly use `admin-http-pool`? |
|---|---|---|
| GUI and catch-all `/`, including `/admin/...` | Tomcat 9443 / `AdminExecutorPool` | Yes |
| ERS `/ers` | ERS upstream, including connector 9060 / `ers-http-pool` | Normally no |
| OpenAPI `/api` | OpenAPI upstream/9070 / `openapi-http-pool` | Normally no |
| pxGrid integration | pxGrid connector/`pxgrid-http-pool` | No |
| RADIUS/TACACS | Protocols Engine | No |

Therefore “DNA Center failed to connect” is not enough. Preserve its source
IP, destination port, URI, status, User-Agent/client certificate, and matching
thread name. A legacy `/admin/API/...` call, GUI endpoint, or fallback path can
use the Admin pool; an ERS, OpenAPI, or pxGrid failure implicates its own pool
unless it triggers separate internal Admin work.

To distinguish accidental application-layer flooding from a server-side stall,
compare healthy and degraded windows in one-minute bins:

1. requests and TCP connections by source IP;
2. method plus query-stripped URI, status, and User-Agent;
3. Kong upstream connect/header/response time;
4. Admin busy-thread count/threshold events, jsvc CPU, and first Ignite refusal;
5. unique authenticated users and concurrent connections.

A source/URI rate surge that precedes latency and pool occupancy supports a
client retry/polling trigger. A normal request rate followed first by upstream
latency or loss of 10800, with retries growing afterward, supports a backend
failure amplified by clients. This ordering matters because only about nine
22-second requests per second can occupy the 200-thread pool: a low-bandwidth
poller can produce application-layer overload without conspicuous network
throughput.

A controlled discriminator, with the integration owner and change approval,
is to pause or firewall only the suspected DNA Center source before the next
onset. If request rate, pool occupancy, jsvc CPU, and GUI latency fall without
an ISE restart, that is strong causal evidence. If the pool remains saturated
or 10800 remains absent, the client was noise or an amplifier rather than the
underlying fault.

### Could one user or client cause it?

One request is assigned one Admin worker; one ordinary idle login does not
create 200 threads. A single identity or source can nevertheless produce the
incident through two different mechanisms:

1. **Concurrency/retry source:** one browser with several tabs, a broken AJAX
   retry loop, an API script, DNA/Catalyst Center, a monitoring poller, scanner,
   or report/export client submits overlapping requests faster than slow ones
   complete. An expired credential returning errors without backoff, a client
   retrying timeouts while prior requests still run, or a device/laptop waking
   at 09:00 are concrete patterns.
2. **Trigger of shared server contention:** one expensive Context Visibility
   grid, endpoint export, report, broad search/sort, restored-data edge case,
   or integration query enters a global lock, synchronized Ignite-client path,
   pathological database/cache query, or resource failure. Other users' normal
   requests then accumulate behind shared state. Here one user triggers the
   transition, but the persistent deployment-wide failure is a server defect or
   capacity/concurrency weakness rather than normal single-user behavior.

Specific single-source candidates consistent with the 09:00 timing include a
human opening a saved data-heavy page, a workstation resuming with stale tabs,
DNA Center beginning a daily synchronization or retry cycle, an API credential
expiring and creating an authentication loop, a scheduled report/export, a
vulnerability scanner, a reverse-proxy health checker, and OIDC/SSO redirect or
session-refresh loops. A browser can issue many AJAX requests per navigation,
but one normal browser is less likely than automation to sustain the roughly
nine 22-second requests per second needed to fill 200 workers directly.

Identity, address, and process are not interchangeable. One source IP may hide
many users behind NAT/proxy; one user can appear from several addresses; and one
DNA appliance can run multiple concurrent workers. Attribute with the tuple
`source IP/XFF + authenticated user + User-Agent/client certificate + URI`, not
any single field.

Evidence that favors a single source is a dominant tuple beginning just before
the onset, repeated identical URIs/statuses, overlapping request durations,
and disappearance of new slow requests when only that source is paused.
Evidence against it is a normal source/URI distribution, loss of 10800 before
the source-rate change, or continued pool/GUI failure after the source stops.
If traffic removal stops new requests but does not restore an already-missing
Data Grid listener, the source may have triggered a sticky server failure and a
restart/reset can still be required.

### ACAS repeatedly failing SSH authentication

ACAS/Tenable attempting SSH on TCP 22 reaches ADE-OS `sshd`, not Kong 443 or
Tomcat 9443. Failed SSH authentications therefore do not directly create
`admin-http-pool` threads and do not execute inside jsvc. On the rooted control,
sshd runs in `system.slice/sshd.service` while jsvc is a separate Application
Server process/cgroup. The effective SSH controls are `MaxAuthTries 3`,
`LoginGraceTime 60`, and `MaxStartups 10:30:100`: after ten concurrent
unauthenticated connections sshd begins probabilistic dropping, reaching full
drop at 100.

The total “hundreds” is less important than rate and concurrency. Hundreds
spread over a scan window normally create authentication logs and modest
key-exchange/PAM/process work. Hundreds of simultaneous or rapidly reconnecting
sessions can consume sshd CPU, memory, process slots, cryptography, and log I/O,
but `tech top` should then expose sshd/kernel work rather than attributing the
load primarily to jsvc. SSH failures also do not explain a Kong-container
1,536-MiB cgroup OOM or a local Ignite 10800 refusal directly.

The stronger concern is that the same ACAS job may scan HTTPS 443 and other ISE
application ports in addition to SSH. TLS/web vulnerability plugins, login
tests, crawling, or repeated malformed HTTP requests can reach Kong and, for
GUI/catch-all paths, consume Admin workers. TLS handshakes that fail before an
HTTP status may appear only in Kong `error.log`; HTTP failures should appear in
the error-filtered access log. Also check 9060/ERS, 9070/OpenAPI, 8910/8911
pxGrid, and portal ports so the destination is mapped to the correct process.

Patch 3 support bundles include `/var/log/*` under `adeos/` when System logs are
selected, so SSH events from `secure`, `messages`, `btmp`, and related files can
be aligned with `logs/apigateway/access.log*`, `error.log*`, `console.log`, and
`ise-ignite.log`. For the ACAS source address, calculate per-minute attempts,
concurrent connections, destination ports, and Kong paths/statuses. Search SSH
logs for MaxStartups throttling/drops and separate authentication failure from
KEX/banner/protocol negotiation failure.

The causal test is a scheduled scan window with ISE excluded or with only the
misconfigured SSH credential check disabled, under the scanner owner's change
control. If SSH events vanish but Admin/10800 behavior is unchanged, SSH is
noise. If excluding all ACAS traffic prevents the incident, re-enable or test
SSH-only and HTTPS-only separately; that distinguishes SSH host pressure from
web/API pressure. Fix the invalid credential regardless, because it can lock
the CLI account or conceal real authentication attacks even if it is unrelated
to GUI performance.

### Effect of `application start ise safe`

ISE safe mode is an Admin-access recovery mode, not a reduced-service,
performance, or request-rate-protection mode. Cisco's ISE 3.5 CLI guide and the
exact Patch 3 control script agree that it starts the ISE services while the
Application Server receives `ise.startup.safe=true`. It temporarily:

- relaxes ISE Admin IP access restrictions;
- bypasses certificate-based Admin authentication and permits username/password
  authentication;
- on FIPS hosts, bypasses the startup FIPS integrity and hardware-RNG checks.

These changes last only for that application instance. A later ordinary stop
and `application start ise` restores normal behavior. Patch 3's `show
application status ise` explicitly warns when safe mode is active that IP
access restrictions are relaxed and certificate authentication is bypassed.

Safe mode requires the application not already be running, so the usual
sequence `application stop ise` followed by `application start ise safe` is a
full application-service stop/start. It restarts the Application Server,
Kong/API gateway, Data Grid and related runtime services and clears in-memory
threads, queues, connection pools, and retry state. It does **not** perform
Option 45: it does not delete Ignite persistent data/WAL/snapshots or rebuild
the local Data Grid configuration. Immediate GUI recovery after this sequence
therefore has the same restart confound as an ordinary application restart;
it does not prove that relaxed access/authentication caused the improvement.

Safe mode can matter to the ACAS/DNA hypothesis. If an ISE Admin IP allowlist
would normally exclude the scanner, safe mode can let its HTTPS traffic reach
Kong/Tomcat `/admin` paths. Network firewalls still apply, and SSH behavior is
unchanged. Confirm whether safe mode was active and whether the ACAS address was
inside the normal ISE Admin allowlist, then compare Kong traffic by source.

If ordinary and safe restarts both produce the same healthy interval, the
restart is the meaningful operation. If a controlled ordinary restart remains
slow while safe mode becomes fast, investigate Admin IP filtering and
certificate-authentication dependencies (certificate chain, revocation/OCSP,
external identity/AD, and authentication loops). Those access-path defects can
explain login/access behavior but do not by themselves explain thousands of
local Ignite 10800 refusals. FIPS safe-mode bypass mainly changes startup
admission; it is not a likely explanation for a next-day runtime slowdown.

Safe mode should not be left as the steady production state. Correct the
access/authentication configuration and restart normally because the mode
intentionally weakens Admin access controls.

#### Leaving safe mode and cluster scope

Safe mode has no independent disable command or automatic expiry. On the node
where it was started, first correct the Admin IP/certificate-authentication
configuration, then run:

```text
application stop ise
application start ise
```

An ordinary `application start ise` cannot replace a currently running safe
instance; ISE reports that it is already running. After the normal start, `show
application status ise` must no longer print the safe-mode warning. Verify this
from an allowed management address using a known internal administrator so a
bad access rule does not immediately cause another lockout.

The `ise.startup.safe=true` flag is local process state and is not replicated as
a deployment-wide setting. Other nodes do not enter safe mode, relax their IP
restrictions, or bypass certificate-based Admin authentication merely because
the command was issued on the PAN. To leave safe mode on multiple nodes, each
node that was explicitly started safe must be restarted normally.

The stop/start still has ordinary cluster effects. Stopping the PAN removes its
local Application Server, Kong, and Data Grid member temporarily; topology and
baseline/rejoin activity can occur, Admin service is unavailable on that node,
and PAN auto-failover may trigger if configured and the outage exceeds its
window. Other PSNs can continue RADIUS/TACACS, but a stopped PSN cannot serve its
own authentication traffic and NAD failover determines continuity. Do not stop
multiple Data Grid members together merely to clear safe mode.

### Slowness persisted with GUI API services disabled

This observation narrows the client-trigger hypothesis, but its weight depends
on experiment timing. If ERS/OpenAPI was disabled only after the GUI was
already slow, an existing full Admin pool, missing 10800 listener, Ignite
recovery state, or Kong backlog can persist; lack of immediate recovery does
not exonerate or implicate the original request source. If the services were
disabled before a clean application restart and remained disabled through the
next full healthy-to-degraded cycle, then ERS/OpenAPI traffic is not a necessary
trigger and DNA Center use of those APIs moves substantially down the ranking.

Exact Patch 3 bundled help separates three controls:

- **API Service Settings** controls external ERS service availability on the
  PAN/other nodes; OpenAPI is enabled by default from ISE 3.4 and its exact
  toggle state must be confirmed rather than inferred from the ERS switch.
- **API Gateway Settings** selects nodes that run the gateway. The gateway is a
  separate service and remains the public 443 entry point.
- The **Admin GUI** continues to use the Dockerized gateway on 443 and the
  GUI/catch-all route to Tomcat 9443/`admin-http-pool`.

Consequently, “API services off” does not eliminate:

- browser GUI/AJAX traffic or ACAS scans of `/admin` and catch-all paths;
- Kong worker/connection pressure, including calls to disabled endpoints;
- pxGrid/DNA integration traffic and pxGrid Direct/EDDA connectors;
- Admin-node internal scheduled jobs, endpoint purge, reports, or Analytics;
- Oracle/Ignite work initiated inside the Application Server;
- a PAN-local Data Grid runtime/recovery defect.

It does directly weaken ordinary DNA/API automation through `/ers` and `/api`,
especially if disabled throughout a clean recurrence cycle. Calls may continue
to arrive and fail or time out when services are disabled, so Kong logs must
still be checked for the DNA/ACAS source. The route/status shows whether the
toggle removed backend work or merely converted a client poll into an error
retry storm.

The updated ranking, assuming the APIs were disabled before restart and the
incident still recurred, is: local Ignite/Application Server failure or
internal scheduled/data workload first; GUI/catch-all/scanner traffic and
pxGrid/integration traffic next; ordinary ERS/OpenAPI workload lower. It does
not by itself establish whether 10800 loss precedes or follows Admin demand.

## Why jsvc can consume 200–300% CPU without using all 32 vCPUs

Linux `top`-style process percentages count 100% per logical CPU. Thus jsvc at
200–300% means that, averaged over the sampling interval, the Application
Server consumed about two to three CPUs. On a 32-vCPU VM, 300% process CPU is
about 9.4% of total VM capacity. That closely matches the reported roughly 10%
rise in ISE Prometheus CPU after allowing for Kong, kernel, logging, and other
processes.

This is not evidence that the JVM has exhausted the VM's CPU capacity. A Java
process uses another core only when it has another runnable thread doing CPU
work. An Administration pool can contain 200 busy/occupied threads while most
are blocked or sleeping in socket reads, Oracle calls, Ignite retry sleeps,
locks, queues, or downstream-service waits. Such threads consume a pool slot
and can trigger its threshold without consuming a core.

The reported failure has several credible sources of two-to-three-core jsvc
work:

- repeated exception creation, stack-trace formatting, and high-volume log
  output for Ignite and socket failures;
- JSON/XML/string encoding and serialization for slow or repeatedly requested
  GUI data-grid pages;
- garbage collection caused by short-lived exception, `String`, and `char[]`
  allocation;
- a small number of retry/state-monitor/event-listener/request threads cycling
  while many Admin threads wait;
- scheduled endpoint/profile/report/Analytics work that contains serial query
  or cache sections.

The `java char[]` profiling result describes allocation/data churn, not code
that literally executes inside an array. Java strings, JSON, encoding,
stack-trace formatting, and log messages all create character arrays. The
common `CharacterEncodingFilter` frame likewise establishes that the request
was inside the servlet path; it does not prove the filter consumed the CPU.

Patch 3's `IgniteClientPool` gives a concrete parallelism limiter: client
construction for main, UI, and event-listener clients is protected by one
class-wide monitor, and failed attempts sleep between retries. More waiting
Admin requests therefore do not produce proportional CPU parallelism. Similar
serialization can exist in database connections, cache operations, report
queries, and logging appenders.

No three-core jsvc pin was found in the Patch 3 configuration examined. As a
control, the rooted 3.3 Patch 11 Application Server child had more than 1,100
threads and `taskset` allowed all eight assigned lab vCPUs (`0-7`). Production
3.5 affinity/quota remains unverified because it is not rooted. A hard ceiling
near exactly 300% under different workloads would justify having TAC inspect
`Cpus_allowed_list`, cgroup CPU quota, `taskset`, JVM
`ActiveProcessorCount`/GC-thread flags, and Hyper-V per-vCPU scheduling. The
current 200–300% observation alone is more naturally explained by limited
runnable parallel work than by a three-core cap.

The decisive capture is three per-thread CPU/thread dumps spaced 10–30 seconds
apart plus the matching GC log. Repeated runnable stacks in JSON/encoding,
logging, exception construction, GC, Ignite client setup, or a particular
query identify the CPU work. Large populations in `WAITING`, `TIMED_WAITING`,
or socket read identify why the thread pool is full without using more cores.

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

The best-supported current root-cause area is the PAN's local Data Grid/Ignite
service and its TLS thin-client listener on `localhost:10800`. This is stronger
than a generic component correlation: a persistent immediate refusal proves
that nothing accepted the Java client's local connection at those instants.
Kong cannot directly cause the independent Application Server state-monitor's
loopback listener to disappear, whereas a missing Data Grid listener can block
Admin work and create downstream Kong pressure. The best-supported propagation
model is:

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

The remaining root-cause question is therefore not broadly “why is the GUI
slow?” It is: **why is the local Ignite listener unavailable while ISE is
supposedly running?** The focused branches are:

1. the Data Grid container/process exited or is being restarted;
2. Ignite is running but not active enough to bind/retain 10800 because of
   discovery, activation, persistent-state/WAL, or cache initialization state;
3. a local container/network/port-publication failure makes the listener
   intermittently unreachable;
4. work-hour activity or migrated data drives one of the preceding conditions.

The platform profile reduces Kong/JVM/Admin headroom and gives Data Grid a
bounded 2-GiB envelope; the exact Data Grid memory arithmetic below makes that
a plausible local failure contributor, although the profile alone is not proof
that the process exited. The rare Analytics state is likewise a candidate
workload/data trigger, not yet evidence of the failing subsystem.

### How the PAN alone can lose 10800

The local listener exists only while that node's Ignite JVM successfully
starts and remains alive. Patch 3 provides several concrete PAN-local failure
paths:

1. **Container/JVM memory exit.** Under predicted profile `sns3815`, Docker
   caps Ignite at 2 GiB. The launcher sets Java heap initial/max to 50% (1,024
   MiB) and the persistent data region initial/max to 40% (about 819 MiB),
   leaving only about 205 MiB of nominal cgroup headroom for metaspace, code
   cache, thread stacks, direct buffers, networking, GC/native structures, and
   other container memory. `AlwaysPreTouch` is enabled for the heap and Ignite
   is launched with `ExitOnOutOfMemoryError`. PAN-local Admin/cache activity can
   therefore terminate its JVM without requiring the SPAN/PSN to fail. This is
   a concrete risk, not proof of the production exit.
2. **Stale initialization lock.** The launcher uses
   `/tmp/ise-ignite-service.lock`. If it exists while the container is not
   running, `start_ignite()` reports “initializing” and returns without
   creating/starting the container. Normal shell exit removes the lock, but an
   abnormal setup interruption can strand it. Option 45 explicitly removes the
   lock, making this a good fit for “reset repairs one node.”
3. **Local persistence/WAL/checkpoint failure.** The node keeps persistent
   pages and WAL under `/opt/ignite/data`. Corrupt pages, incomplete recovery,
   filesystem/permission errors, or a fatal Ignite failure handler can stop
   startup before the `0.0.0.0:10800` connector binds. Option 45 deletes that
   local state, while the surviving replicated members can repopulate the PAN.
4. **Discovery/activation failure local to the PAN.** Ignite uses Oracle-backed
   `TcpDiscoveryJdbcIpFinder`, inter-node discovery/communication, TLS material,
   and baseline topology. A bad local database connection, hostname/IP,
   certificate/keystore, discovery response, or baseline recovery can prevent
   that JVM from completing startup even though the other members remain
   active.
5. **Local connector bind/network state.** Port 10800 is configured at
   `0.0.0.0`, TLS-enabled, with no fallback port range. A bind conflict or local
   firewall/container host-network publication failure leaves no usable 10800.
   This is less likely than a stopped JVM when the error is a loopback refusal,
   but it is distinguishable in the logs.

Why the PAN can be first even with identical VMs: it handles PPAN Admin/UI and
configuration activity, primary event listeners, and cluster activation work.
At 09:00 it can experience local cache reads/writes, serialization, scheduled
work, and connection activity not mirrored by the other nodes. Replicated cache
contents do not imply identical transient/native memory or thread load.

The status at first onset selects the branch:

| Supported observation | Most likely branch |
|---|---|
| Data Grid `not running`; 10800 absent | JVM/container exit; inspect OOM and exit/restart evidence. |
| Data Grid remains `initializing`; 10800 absent | Stale lock or persistence/discovery startup loop. |
| Data Grid `running`; 10800 intermittently absent | JVM/container restart loop or connector bind failure. |
| 10800 present but cluster inactive/unhealthy | Discovery, activation, baseline, or persistent-state problem. |
| PAN alone affected | Local memory/persistence/lock/bind state. |
| All members affected together | Shared discovery, cluster state, or common workload/configuration. |

Preserve `datagrid.log`, `ise-ignite.log`, `ADE.log`, container logs, GC logs,
the complete kernel OOM window, and the first state transition. High-value
signatures include `OutOfMemoryError`, exit code 137, `Killed process`, heap
dump/error-file creation, repeated “initializing,” WAL/checkpoint/page errors,
Oracle JDBC discovery errors, TLS/keystore errors, and `Address already in use`.

Evidence weighting matters. Runtime memory/exit and restart-state defects rank
above hard persistence corruption because a normal ISE application restart
restores the GUI without deleting `/opt/ignite/data`. Truly unreadable durable
pages would ordinarily fail again during immediate recovery. Persistence still
matters if the defect is transient checkpoint/recovery state, topology-related,
or only becomes fatal after new writes. At present there is no production
Ignite OOM, exit code, lock-state, WAL, or bind signature, so none of these
mechanisms should be stated as the established cause.

### What could initiate the local failure

There are two different timing models, and “the GUI became slow at 09:00” does
not distinguish them:

1. **Ignite failed during or shortly after startup but remained latent.** The
   first Admin pages used after work began exposed an already-missing listener,
   and request volume made retry CPU/logging visible. In this model the initial
   cause is initialization/recovery: stale service lock, Oracle JDBC discovery,
   TLS/keystore, cluster activation/baseline, or WAL/checkpoint recovery.
2. **Ignite was healthy overnight and failed at work-hour load.** Admin/UI/API,
   endpoint/profile, session, or scheduled activity raised local transient
   heap/native/direct-buffer/thread/queue demand until the PAN's container/JVM
   exited or the connector stopped accepting connections. The subsequent
   retries and logging produce the observed sustained `jsvc` CPU increase.

Patch 3's replicated Ignite caches include endpoint/profile data
(`EDF2EndPoint`), EDDA data, endpoint licensing, a 30-day MFC endpoint cache,
and `OIDCSessionContextCache`. The OIDC cache is SQL-on-heap and stores ID/token
strings and session lifetime/touch fields. These are concrete sources of cache
and query work, but no production cache count or growth measurement exists.
The operator changed maximum-session and time-limit settings at the same time
as Option 45; those changes are relevant only after identifying their exact
scope. Administrative-session, RADIUS max-session, portal, and external-OIDC
settings are not interchangeable, so the current evidence does not prove that
the changed values control `OIDCSessionContextCache`.

The most plausible demand-driven initiators, in order, are:

- ordinary 09:00 Admin/API activity crossing a tight local runtime threshold;
- retained/session/cache entries or expensive Data Grid SQL/query work growing
  between restarts;
- an external poller, scheduled report, or enabled feature consumer beginning
  a burst of Data Grid-dependent work;
- a topology/discovery/checkpoint event coinciding with that load.

An abrupt runtime exit can then create secondary recovery trouble: dirty WAL or
checkpoint state, a restart loop, or a stranded initialization lock. Thus
“persistence/recovery failure” may be the reason the listener stays down rather
than the original initiating event.

Use the first `IgniteClientPool` refusal after restart to distinguish the two
models. Refusals within minutes of startup point to initialization/recovery.
A verified active 10800 listener followed by its disappearance at 09:00 points
to demand-driven runtime failure. Refusals hours before GUI degradation weaken
the claim that the listener loss itself initiated the visible incident unless
the later request burst demonstrably amplified it.

### What happens at midnight and early morning

There is no single ISE "midnight job" and there is no automatic Data Grid reset
at midnight. The exact Patch 3 bundled help identifies these relevant events:

| Time/cadence | Activity | Relevance |
|---|---|---|
| 00:00 | The `Endpoints Purge Activities` alarm is triggered for the preceding 24 hours. | This is reporting/alarm timing; it is not by itself evidence that deletion or Ignite failure starts at 00:00. |
| About 01:00, PPAN time zone | The enabled-by-default endpoint purge deletes endpoints and registered devices older than 30 days. Above 5,000 endpoints, deletion proceeds in batches every three minutes. | Strongest known overnight workload related to endpoint state and therefore a credible cache/replication/persistence-pressure trigger. |
| 01:00 local time, if enabled | Profiler Feed Service update. | Can download profiles and re-profile endpoints; Cisco's own help warns that an update can increase system load. |
| Every six hours after cache construction/Application Server start | OIDC expired-session cleanup. | Exact Patch 3 bytecode uses a 360-minute initial delay and fixed 360-minute period. It is restart-relative, not midnight-relative. |
| 02:00 on MnT | DBMS statistics. | Cisco documents expected temporary MnT CPU load; it is less direct for a PAN-local 10800 refusal. |
| 01:00 Sunday | Weekly large-index maintenance. | Relevant only if the recurrence is weekly/Sunday, which the reported daily behavior is not. |

The Patch 3 endpoint caches are replicated in Ignite. A sufficiently large
purge can consequently entail repeated cache removes, replication, persistent
WAL/checkpoint activity, and endpoint/profiler processing. This makes a causal
sequence possible: the 01:00 purge creates or accumulates pressure, local Data
Grid exits or becomes unhealthy overnight, and 09:00 Admin/API demand exposes
and amplifies the already-present fault. It is equally possible that the purge
finishes normally and unrelated 09:00 demand is the initiating load; no
production timestamps yet select between these models.

Structural inspection of the rooted 3.3 Patch 11 control also found Oracle
scheduler work at 00:00 (audit archive timestamp maintenance), 00:30 (marking
expired endpoint certificates), 01:30 (MnT database-size job), 02:00 (MnT
statistics), and 04:00 (expired endpoint-certificate deletion). Those timings
are useful migration clues, not proof that the restored 3.5 production schema
retains identical jobs. Scheduled backups and reports can add site-specific
overnight work.

The decisive timeline is therefore not the first 09:00 complaint. For each
restart cycle, obtain the last successful Data Grid `ACTIVE`/`RUNNING` state,
the first `IgniteClientPool` refusal, endpoint-purge start/end and deletion
count, and the first Admin thread-pool alert in one normalized time zone. If
10800 disappears during 00:00-02:00, overnight maintenance becomes a strong
trigger lead. If 10800 is verified healthy until the workday surge, it does
not.

## What a one-to-two-hour PAN-to-SPAN Data Grid break would do

This is long enough to be a real topology event, not merely a delayed packet.
Exact Patch 3 configuration establishes:

- TCP 47500 is the fixed discovery port and TCP 47100 is the fixed inter-node
  communication port; neither has a fallback port range;
- server and client failure detection are configured for 120 seconds;
- discovery socket/acknowledgement timeouts are 180 seconds, the maximum
  acknowledgement and connection-recovery timeouts are 300 seconds, and the
  communication/discovery paths request up to 50 reconnects;
- the primary enables persistent-baseline auto-adjust with a 30-second stable
  topology timeout;
- the five declared application caches are replicated; `EDF2EndPoint` and
  `UPSEddaData` additionally specify `READ_WRITE_SAFE` partition-loss policy.

All three reported appliances have 64 GiB and should therefore be server-mode
members. There is not enough information to call the result a simple
"PPAN-versus-SPAN" split: the third PSN's reachability to each PAN determines
the actual topology. A failure of only the direct PPAN-to-SPAN path can also
affect TCP communication even if each still discovers the third node.

If Ignite declares a server failed or left, the cluster performs a partition
map exchange. After 30 seconds without another topology event, ISE's configured
auto-adjust can change the persistent baseline to the surviving topology and
trigger rebalancing. When the missing member returns, another join, baseline
change, and rebalance can occur. Replication makes loss of one member less
likely to lose these cache contents, but it does not make topology exchange,
transactions, queries, checkpoints, or rebalancing free. New transactions can
wait behind partition map exchange, and the exact ISE XML sets a 15-second
transaction timeout during that exchange.

There are two materially different PAN outcomes:

1. **PAN Ignite remains running and 10800 remains open.** Local Java thin
   clients can still connect. Expected evidence is node-left/failed messages,
   topology-version changes, partition-map exchange, rebalance, cache invalid
   state, or operation timeouts. This can slow GUI requests, but it does not by
   itself explain an immediate TCP refusal to `localhost:10800`.
2. **PAN Ignite exits, restarts, remains inactive/recovering, or withdraws its
   thin-client connector.** The local 10800 listener disappears. This directly
   produces the reported `IgniteClientPool` connection refusals and enables the
   verified serialized-retry-to-Admin-pool cascade. The exact XML does not
   define a custom segmentation resolver/policy, so the surviving side and the
   PAN process outcome cannot be inferred from configuration alone.

An hour is ample time to generate the reported log volume. Two built-in
Application Server callers use the same ten-attempt client-construction path
and roughly 27-second immediate-refusal cycle. Because creation is serialized,
continuous internal attempts alone are of order 1,000 failures per hour; GUI
traffic adds more. Thus "thousands" is compatible with a one-to-two-hour local
10800 outage without requiring thousands of human page loads.

Restoring inter-node connectivity does not guarantee immediate recovery. A
running member may rejoin and rebalance normally, but a dead local container,
stranded initialization lock, failed WAL/persistence recovery, or saturated
Admin/Kong request queues can persist after the network heals. An application
restart clearing the condition is compatible with a network event that was the
initiator but no longer exists by the time users report the slow GUI.

This hypothesis has a strict evidence test. On all three nodes, align the first
47100/47500 transport/discovery error, `NODE_LEFT`/`NODE_FAILED` and coordinator
or topology-version change, baseline auto-adjust, partition exchange/rebalance,
PAN Data Grid process transition, first missing PAN 10800 listener, first local
`IgniteClientPool` refusal, and first Admin threshold alert. Hyper-V virtual
switch, firewall, and physical-network telemetry should be aligned to the same
window. The third PSN's log is the best independent view of which node became
isolated. Inter-node failure preceding PAN 10800 loss strongly supports this
trigger; PAN 10800 loss preceding any inter-node evidence argues for a local
Data Grid failure instead.

References: [Cisco's ISE 3.5 release notes](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/release_notes/cisco-ise-release-notes-35.html)
document the Data Grid port purposes;
Apache Ignite documents [baseline auto-adjustment](https://ignite.apache.org/docs/ignite2/latest/clustering/baseline-topology),
[partition-map exchange](https://ignite.apache.org/docs/latest/data-modeling/data-partitioning),
and [TCP discovery/network behavior](https://ignite.apache.org/docs/ignite2/latest/clustering/network-configuration).

## What a one-node Option 45 reset means

The production Option 45 operation was run on the PAN only. The exact Patch 3
Data Grid `reset-config` implementation is local to the selected appliance. It:

1. stops the local Ignite container if it is running;
2. removes the local container and image;
3. deletes local `/opt/ignite/data/*`, work database, snapshots, diagnostics,
   and the local initialization lock;
4. allows the service to be rebuilt and started locally.

The reset path does not call the available `drop_ignite_address_table()`
function and does not erase another node's Ignite storage. PAN and SPAN are
always configured as Ignite servers. Other nodes with more than 32 GiB are
also server-mode nodes; therefore all three reported 64-GiB appliances should
be server-mode members. On return, the primary activates the cluster if needed
and enables 30-second baseline auto-adjust. Patch 3 defines the observed ISE
caches as replicated, allowing surviving cluster members to repopulate the
rebuilt PAN.

Operationally, the PAN temporarily loses local 10800 service and then rejoins;
the surviving members retain cluster state. Rejoin can cause topology change,
baseline adjustment, cache transfer/repopulation, and temporary CPU/network
work. Resetting multiple members together would remove that safety and should
not be inferred as the next experiment without Cisco guidance.

Diagnostic interpretation:

- improvement after only the PAN reset supports PAN-local persistence,
  container, port/listener, or client-state cleanup;
- it does **not** prove that cluster-wide logical cache contents were purged;
- bad logical state retained by the other members can be copied back;
- recurrence after 48 hours fits either reaccumulation under load or a local
  service/container failure recurring after the reset;
- because session limits/timeouts changed at the same time, the duration is
  not a clean causal measurement.

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
