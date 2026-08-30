# Controlled experiment and capture plan

## Questions to answer

1. Which Analytics rows and claim fields exist in the affected 3.3 backup?
2. Which of those rows survive restore or are transformed by ISE 3.5?
3. Does an Analytics-enabled restore produce behavior absent from an Analytics-disabled control?
4. Can the behavior be reversed through a Cisco-supported Analytics disable operation?
5. Are Ignite failures causal, downstream, or incidental?

## Restore matrix

Use identical target builds, VM sizing where practical, restore sequence, traffic, GUI/API workload, observation window, and capture intervals.

| Run | Source state | Target | Purpose |
|---|---|---|---|
| A | No restore | Clean ISE 3.5 Patch 3 | Establish clean target baseline |
| B | Local 3.3 P11 config backup; Analytics disabled | Clean ISE 3.5 Patch 3 | Control for the 3.3-to-3.5 restore path |
| C | Affected 3.3 P11 config backup; Analytics enabled | Clean ISE 3.5 Patch 3 | Test the unique migrated state |
| D | Run C followed by Cisco-supported Analytics disable | Same supported target state | Test reversibility and causality |

Run C is the decisive comparison. Merely forcing rows or bypassing the JWT would not reproduce a supported state and could create a false positive.

## Stage gates

### 1. Acquire and verify media

- Media acquisition is complete; see [`media-manifest.md`](media-manifest.md).
- Exact ISO and patch filenames, versions, sizes, and locally calculated SHA-256 hashes are recorded.
- Keep binaries outside Git.
- Compare the files against Cisco-published checksums or use Cisco's supported signature validation.
- Confirm that Patch 3 is cumulative and applicable to the base image using Cisco release documentation.

The reproduction must start from the clean ISE 3.5 ISO, matching production. Do not perform an in-place upgrade of the 3.3 lab VM.

Required production-faithful order: install the base ISO, apply Patch 3, capture the clean Patch 3 baseline, and only then restore the 3.3 Patch 11 configuration backup.

### 2. Preserve the source controls

- Record backup filename, task ID, source version, patch, persona, Analytics GET state, and SHA-256 when filesystem access permits.
- Store the encryption key separately from the backup and this repository.
- Obtain the affected backup before changing or disabling the source feature, if that source still exists.

### 3. Establish each target baseline

Before restore, collect:

- ISE version and patch.
- Node identity and personas.
- VM CPU and memory.
- Application status.
- Miscellaneous-license GET output with claim/token values redacted.
- Relevant table row presence and metadata, if rooted inspection is authorized.
- Admin GUI latency, user CPU, `jsvc` CPU, and admin-pool state.
- Data Grid/Ignite health and endpoints.

### 4. Restore and compare state

For Runs B and C:

- Restore configuration only, never operational data.
- Record restore task timestamps and every migration warning.
- Diff table schema definitions separately from row contents.
- Compare Analytics row presence, enabled status, entitlement state, key-row presence, and redacted claim keys.
- Do not publish claim values, tokens, verification keys, credentials, or backup passwords.
- Confirm whether 3.5 exposes the Analytics GET state after restore.

### 5. Apply repeatable load and observe

Observe for at least 72 hours because the reported recurrence varied between about 24 and 48 hours. Include the 09:00 workday boundary.

Use the same workload for every run:

- Timed GUI navigation to a fixed set of pages.
- Controlled Admin API polling at a recorded rate.
- Recorded concurrent-admin and session counts.
- Representative RADIUS/TACACS traffic as a negative control.
- No unrelated session-limit or timeout changes during a run.

### 6. Capture the onset, not only the saturated state

At baseline and at short intervals around onset, capture:

- Wall-clock timestamp and timezone.
- GUI request URI, status, and end-to-end latency.
- Admin HTTP active/busy/max threads and queue depth, if exposed.
- Exact Kong worker-pool error, log source, active/busy/max workers, queue depth, and first-occurrence timestamp.
- `jsvc` process CPU and per-thread CPU.
- JVM heap, non-heap, GC, allocation rate, and thread count.
- Prometheus user/system CPU separately.
- Exact first Ignite exception including destination IP, port, thread, and retry interval.
- `datagrid.log`, `ise-ignite.log`, and the matching `console.log` window.
- Full OOM event from invocation through `Killed process`, `oom_memcg`, and cgroup path.
- Active GUI sessions, source addresses, API clients, reports, and scheduled jobs.
- Authentication latency and success rate.

Capture a working thread dump with an alternative supported method before saturation if the ISE-generated dump remains blank.

## High-value checks in the affected cluster

### Compare PPAN and SPAN during the same incident

- PPAN slow while direct SPAN GUI is normal: favors PPAN-local state, process, or workload.
- Both PAN GUIs slow: favors shared migrated configuration, shared dependency, or cluster-wide Admin behavior.

Record whether the test uses a direct node address or a load balancer/VIP.

### Resolve every `Connection refused`

For representative first and repeated Ignite failures, preserve:

- Source node/process.
- Destination IP and port.
- Whether the destination is local, another ISE node, or a stale address.
- Listener state on the destination at that instant.
- Connection-attempt rate.

This distinguishes missing listener/configuration from a retry storm caused by an already-saturated service.

### Correlate 09:00 with demand

Compare the onset against:

- Admin login and active-session count.
- Browser/API request rate and slow endpoints.
- External monitoring or inventory pollers.
- Scheduled reports, feeds, backups, purges, and synchronization jobs.
- Authentication volume.

## Decision criteria

Evidence for the Analytics-migration hypothesis becomes strong only if Run C differs reproducibly from Runs A and B, the relevant state is shown to survive restore, and supported disablement in Run D removes or materially changes the behavior.

If Runs B and C fail alike, focus on the general 3.3-to-3.5 restore/migration path. If all runs fail, focus on 3.5 Patch 3, the test workload, and target infrastructure. If only the affected production cluster fails, prioritize production-specific request load, integrations, topology, and data volume.
