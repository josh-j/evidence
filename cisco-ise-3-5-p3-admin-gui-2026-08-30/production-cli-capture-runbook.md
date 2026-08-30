# Production CLI capture runbook

This runbook collects the minimum evidence needed to distinguish an Admin
request surge from a Data Grid failure, a Kong failure, an OOM event, or an
infrastructure trigger on the affected Cisco ISE 3.5 Patch 3 cluster. It uses
supported ISE CLI commands and does not require root access.

Use it on all three nodes: PPAN/PMNT/PSN, SPAN/SMNT/PSN, and the third PSN.
Capture the nodes as close together in time as possible. The third node is an
important independent witness to a PPAN/SPAN topology disagreement.

## Safety and evidence handling

- Prefer three operators collecting concurrently. Record each operator's wall
  clock, timezone, node, and command start and stop times.
- During an incident, collect the quick snapshot before an application restart,
  Data Grid reset, failover, or configuration change when operationally safe.
- If service recovery cannot wait, take the minimum snapshot below, start the
  support bundles, and record the exact recovery action and time.
- Do not enable broad DEBUG logging during the incident without Cisco TAC and
  change approval. High-volume debugging can add CPU, I/O, and logging load and
  change the failure being measured.
- Support bundles, CLI output, and packet captures may contain internal names,
  addresses, usernames, and configuration. Keep them in the private case
  workspace or upload them directly to TAC. Do not commit them to this public
  repository.
- A packet capture can contain credentials or session material. It is not part
  of the default capture and should be scoped with TAC before use.

## Incident worksheet

Fill this out before changing the system:

| Field | Value |
|---|---|
| Incident ID | |
| Timezone | |
| Last known healthy | |
| First known slow | |
| PPAN last application restart | |
| PPAN last Data Grid reset | |
| SPAN last application restart | |
| Recovery action and exact time | |
| Direct PPAN GUI URI/status/latency | |
| Direct SPAN GUI URI/status/latency | |
| VIP/load balancer bypassed? | |
| RADIUS/TACACS latency and success rate | |

Also record whether a backup, report, feed, purge, deployment, VM migration,
snapshot/checkpoint, storage event, vulnerability scan, DNA Center poll, or
other integration began near the onset.

## Minimum snapshot before restart

Run the following on **each node**. The `tech` commands stream repeated samples;
collect three to six samples and stop each one with `Ctrl-C`.

```text
show clock
show timezone
show ntp
show version
show uptime
show application status ise
show ports | include 10800
show ports | include 47100
show ports | include 47500
tech netstat | include :10800
tech netstat | include :47100
tech netstat | include :47500
tech top
tech iostat
tech vmstat
```

The ports have these expected roles:

| Port | Data Grid role | Question answered |
|---|---|---|
| TCP 10800 | Local Ignite thin-client listener used by ISE Java clients | Can this node's Application Server create its local client? |
| TCP 47100 | Ignite inter-node communication | Can members exchange cache/data traffic? |
| TCP 47500 | Ignite discovery | Can members discover and maintain topology? |

A listener proves that a socket is bound, not that the service is healthy. A
missing PPAN `10800` is especially important because Patch 3's
`IgniteClientPool` connects to TLS `localhost:10800`; remote firewall behavior
cannot directly refuse that loopback connection.

At the same time, make one timed request directly to each PAN rather than only
through a VIP. Preserve the URI, source address, HTTP status, total latency,
and whether the request was a login, page navigation, XHR, ERS API, or Open API
request. Do not repeatedly refresh a slow page; that can add queued work.

## Inventory and inspect logs

First discover the exact filenames on the installed build:

```text
show logging application | include ignite
show logging application | include datagrid
show logging application | include console
show logging application | include kong
show logging application | include gateway
show logging application | include ad_agent
show logging application | include dblock
show logging application | include dbalert
show logging application | include monit
show logging application | include ise-psc
show logging system
```

Then inspect the final 80 lines of each file that is actually listed. Candidate
commands are:

```text
show logging application ise-ignite.log | last 80
show logging application datagrid.log | last 80
show logging application console.log | last 80
show logging application ise-psc.log | last 80
show logging application ad_agent.log | last 80
show logging application dblock.log | last 80
show logging application dbalert.log | last 80
show logging application monit.log | last 80
show logging application ise-kong/error.log | last 80
show logging system messages | last 80
```

Use the exact spelling returned by the inventory; Kong/API-gateway paths can
vary by build. The CLI tail is only immediate triage. Preserve full current and
rotated logs in the support bundle, including these additional sources when
present:

- Kong/API-gateway access and error logs.
- `localStore/iseLocalStore.log` for system statistics.
- `replication.log`, `collector.log`, `catalina.log`, and `isebootstrap.log`.
- Protocols Engine/local logs on the PSNs as the authentication-plane control.
- Data Grid diagnostics and container logs supplied in the support bundle.

Do not infer that a common final Java filter frame is the operation that
blocked. For `SocketTimeoutException: Read timed out`, preserve the complete
stack, its first application-specific caller, the socket peer when shown, the
request URI/correlation ID, and all `Caused by` sections.

Kong access logging may suppress successful 2xx/3xx requests. A failed-request
storm can therefore be visible while a successful polling storm is absent.
Correlate load-balancer, reverse-proxy, firewall/NetFlow, browser, and
integration logs before concluding request volume was normal.

## What each node must contribute

| Node | Highest-priority evidence |
|---|---|
| PPAN | `console.log`, Ignite/Data Grid, Kong/API gateway, `ise-psc`, DB/replication, system/OOM, process samples, direct GUI request |
| SPAN | Same Data Grid/topology/system evidence, direct GUI/Kong behavior, replication state, its view of the PPAN |
| Third PSN | Independent Data Grid topology view, system/process evidence, authentication success and latency |

Use the same time window on every node. Authentication success is valuable as
a negative control, but does not establish that the Admin, Kong, PBIS, Oracle,
or Data Grid paths are healthy.

## Generate support bundles

Take the quick live snapshot first, then collect one bundle per node. From the
GUI, use **Operations > Troubleshoot > Download Logs > Appliance Node List**.
For this stability/performance incident, include debug logs, local logs,
monitoring/reporting logs, system logs, and core files for a date window that
begins while the cluster was still healthy and ends after onset. Because the
visible slowdown may lag the initiator, the prior day through the incident day
is a reasonable minimum.

The supported CLI equivalent is `backup-logs`. For example, enter this as one
line after substituting the node, dates, and configured repository:

```text
backup-logs incident-ppan-YYYYMMDD repository <repository-name> public-key date-from YYYY-MM-DD date-to YYYY-MM-DD debug-logs local-logs mnt-report-logs system-logs core-files
```

`public-key` produces a bundle that Cisco TAC can decrypt. Record bundle start
and completion times because bundle creation itself adds load. Do not include
configuration database, policy configuration, or other sensitive backup data
unless TAC specifically requests it. If TAC needs migrated Analytics state,
transfer that material privately and use TAC encryption; never publish it.

## Questions that must be answered

### Timeline and ordering

1. What were the exact last-healthy, first-slow, and recovery timestamps?
2. Which came first: CPU step, first Ignite refusal, loss of `10800`, an
   inter-node event on `47100/47500`, Admin threshold, Kong error, or OOM?
3. Was onset tied to wall-clock time, elapsed time since application restart,
   elapsed time since Data Grid reset, or demand volume?
4. Did PPAN and SPAN slow simultaneously? Could the SPAN GUI be used directly?
5. Did RADIUS/TACACS merely remain successful, or did their latency and queue
   depth also remain at baseline?

### Data Grid and topology

1. Which nodes had listeners on `10800`, `47100`, and `47500` at the same time?
2. Was the local Data Grid process reported running on a node without `10800`?
3. Did a remote discovery/communication failure precede local `10800` loss, or
   did the local listener/container/persistence path fail first?
4. What did each of the other two nodes report about the first failed member?
5. Were there baseline, topology, rebalance, checkpoint, WAL, persistence-lock,
   TLS, bind, container-exit, or Oracle-discovery errors before saturation?
6. Did `10800` flap, remain absent for the whole 12-hour event, or remain
   present despite the logged loopback refusal?

### Admin requests and threads

1. What are the active production Admin-pool maximum, busy-thread count, queue
   depth, connector timeout, and request timeout? Ask TAC for the generated
   platform-profile values rather than assuming defaults.
2. Are the same threads blocked for hours, or are successive requests consuming
   new generations of threads?
3. What exact downstream operation and destination appears in complete timeout
   stacks? `CharacterEncodingFilter` alone does not answer this.
4. Which source IP, forwarded IP, authenticated user, user-agent, URI, method,
   and status account for the most concurrent and slow requests?
5. Did request rate rise before latency, or did latency rise first at normal
   request rate?
6. Does DNA Center use `/admin`, `/ers`, or `/api`, and at what concurrency,
   retry interval, and timeout? The same questions apply to ACAS, scanners,
   monitoring, scripts, and browser auto-refresh.
7. After suspected client traffic is stopped, does the pool drain and does
   `10800` recover without restarting the application?

### Kong, OOM, and kernel state

1. What process was actually killed? Preserve `Killed process`, `oom_memcg`,
   cgroup path, allocation context, and memory limit lines from the same event.
2. Did the OOM precede Data Grid/Admin degradation or occur after their queues
   and retries grew?
3. Does the active platform profile intentionally give Kong the observed
   1,536-MiB limit, and what are Kong's worker and connection limits?
4. Was there an actual `Kernel panic - not syncing`, reboot/uptime break, and
   matching Hyper-V event? An OOM-killer message alone is not a kernel panic.

### AD agent

1. What is the per-minute rate of `failed to write records error code [1]`
   before, during, and after GUI degradation?
2. Does PBIS `eventlog` service/socket health change at onset? TAC should inspect
   this because the supported CLI does not expose the complete rooted state.
3. Did the successful authentications use AD as their identity source?
4. Is the error constant background noise, a shared-resource symptom, or a
   retry/logging amplifier that begins with the incident?

### Infrastructure and migrated state

1. Does every VM have a unique writable VHDX leaf, even if the files share one
   CSV or a read-only parent?
2. Did a live migration, checkpoint, backup, CSV pause, storage-path event,
   virtual-switch event, or host pressure precede the **first** application
   fault—not merely occur during the 12-hour aftermath?
3. Does the affected Analytics-enabled 3.3 backup reproduce the fault when
   restored to a clean 3.5 Patch 3 target while the disabled restore control
   stays healthy?
4. Which Analytics rows/feature state survive migration, and does a
   Cisco-supported disable operation change reproducibility?

## Interpretation matrix

| Observation | Best-supported direction |
|---|---|
| PPAN `10800` absent while its `47100/47500` remain present | PAN-local thin-client connector, Ignite service/container, or persistence/recovery failure |
| Inter-node `47100/47500` failure is first and multiple nodes agree | Network/topology event may be the initiator |
| `10800` is present while the same-time log says connection refused | Listener flapping, wrong timestamp correlation, or service namespace detail; tighten collection |
| `10800` and Ignite stay healthy while GUI slows | Investigate other request, DB, report, replication, or Kong backends |
| Normal request rate followed by backend latency and pool growth | Backend failure is the likely initiator; normal clients amplify it |
| Source/URI rate surge clearly precedes backend latency | User, scanner, integration, or retry traffic is a plausible trigger |
| OOM event precedes listener loss | Local memory/container failure is a plausible initiator |
| OOM occurs after queues, retries, and thread growth | OOM is more likely part of the cascade |
| External traffic is removed but alerts and missing `10800` persist | Latched internal state; traffic may amplify but does not sustain it |
| Pool drains after traffic removal but Data Grid remains failed | Request source contributed to saturation; Data Grid still needs separate cause |
| Repeatable infrastructure event precedes the first Data Grid fault | Reopen Hyper-V/storage/network as an initiating cause |
| No correlated infrastructure event across recurrences | Continue deprioritizing infrastructure as the sustained cause |
| Only Analytics-enabled restored Run C fails | Migrated feature state becomes a strong causal lead |
| Disabled and enabled restores both fail | General restore/migration state or common workload becomes more likely |
| Production alone fails | Prioritize production-specific topology, integrations, request volume, and data scale |

No single row proves root cause. The goal is a repeatable ordering observed on
multiple recurrences.

## Immediately after recovery

Repeat this subset on all nodes as soon as the application restart, failover,
or Data Grid reset completes:

```text
show clock
show uptime
show application status ise
show ports | include 10800
show ports | include 47100
show ports | include 47500
tech netstat | include :10800
tech netstat | include :47100
tech netstat | include :47500
tech top
```

Record the exact times that each listener returns, the GUI becomes fast, CPU
falls, Admin alerts stop, and Ignite/Kong errors stop. Preserve the pre-restart
bundle; a healthy post-restart bundle cannot reconstruct the failed state.

## Questions for Cisco TAC

- What generated Admin connector/pool limits, timeouts, Kong limits, Ignite
  heap/data-region limits, and container limits are active for each production
  platform profile?
- What were the Data Grid container restart count, exit reason, OOM state,
  listener bind state, persistence lock, WAL/checkpoint state, topology, and
  baseline membership at the incident timestamps?
- Why did the supported thread-dump collection return blank, and what supported
  alternate capture can obtain Admin thread states during the next incident?
- Which process/cgroup was the OOM victim, not merely the allocating process?
- Are there ISE 3.5 Patch 3 defects or fixes involving `IgniteClientPool`, local
  `10800`, Data Grid recovery, Admin-pool exhaustion, Kong workers/OOM, 3.3
  configuration migration, or the Cisco-enabled Analytics feature?
- What is the exact supported name, persisted state, migration handler, and
  disable procedure for the JWT-enabled 3.3 Analytics capability?
- Is a later patch, engineering special, or supported configuration change
  indicated by the captured signature?

## Cisco references

- [Cisco ISE 3.5 CLI Reference Guide](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/cli_guide/b_ise_CLI_Reference_Guide_35.pdf)
- [Cisco ISE 3.5 Administration Guide: Troubleshooting](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/admin_guide/b_ise_admin_3_5/b_ISE_admin_troubleshooting.html)
- [Collect a Support Bundle on Cisco ISE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/214153-collect-support-bundle-on-cisco-identity.html)
- [Use the Debugging System to Troubleshoot ISE](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/222247-use-debugging-system-to-troubleshoot-ise.html)
