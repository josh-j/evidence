# Cisco ISE 3.5 clustered service architecture

Date: 2026-08-31

Status: evidence-qualified architecture map for the three-node incident

This document shows how the major ISE service planes interact. It is not a
claim that every deployment enables every optional service. The affected
deployment has these combined personas:

- Node A: PPAN + PMnT + PSN.
- Node B: SPAN + SMnT + PSN.
- Node C: PSN.

The most important architectural fact is that ISE does **not** have one
universal cluster bus. JGroups, ISE Messaging/LDD, Ignite Data Grid, pxGrid,
and database synchronization are distinct mechanisms with different data,
failure domains, and ports.

## External and per-node request paths

```mermaid
flowchart TB
    subgraph external[External systems]
        direction LR
        admins[Admin browsers]
        integrations[DNA Center / scanners / API clients]
        nads[Switches, WLCs, VPN and other NADs]
        pxclients[pxGrid publishers and subscribers]
        identity[AD / LDAP / MDM / DNS / NTP]
    end

    subgraph node[One ISE node — installed services depend on assigned personas]
        direction TB

        kong[Kong / nginx API gateway<br/>public HTTPS 443]

        subgraph java[ISE Java application services]
            direction LR
            admin[Admin GUI application<br/>Tomcat / jsvc<br/>admin-http-pool on 9443]
            openapi[OpenAPI services<br/>9070 backend]
            ers[ERS / legacy API<br/>9060 backend]
            policy[Protocols Engine<br/>RADIUS / TACACS policy]
            pxprovider[pxGrid controller and providers<br/>REST + WebSocket 8910<br/>when pxGrid service is enabled]
        end

        subgraph localstate[Node-local state and infrastructure]
            direction LR
            oracle[(Local Oracle database<br/>configuration, policy and deployment state)]
            clientpool[IgniteClientPool<br/>main / UI / event clients]
            ignite[(ISE Data Grid<br/>Apache Ignite<br/>cache, events, WAL/persistence)]
            session[Local policy/session cache]
            ldd[(LDD / former LSD<br/>RADIUS Session Directory<br/>Endpoint Owner Directory)]
            messaging[ISE Messaging service<br/>RabbitMQ-based exchange/queues]
            mnt[(MnT operational database<br/>session directory, logs, reports)]
        end
    end

    admins -->|HTTPS 443| kong
    integrations -->|HTTPS 443| kong
    kong -->|/admin and GUI catch-all| admin
    kong -->|/api| openapi
    kong -->|/ers| ers

    nads -->|RADIUS 1812/1813<br/>TACACS 49| policy
    identity <-->|identity, posture and profiling protocols| policy
    identity <-->|administrative integrations| admin

    pxclients <-->|discovery, authentication, authorization,<br/>query and pub/sub on 8910| pxprovider

    admin -->|JDBC / application data| oracle
    openapi -->|configuration and endpoint operations| oracle
    ers -->|configuration and endpoint operations| oracle
    policy -->|policy/config reads and endpoint updates| oracle

    admin --> clientpool
    openapi -. selected cached state .-> clientpool
    policy -. selected endpoint/profile/event state .-> clientpool
    clientpool -->|TLS localhost:10800| ignite
    ignite -. Oracle-backed address discovery<br/>TBL_ADDRS in Patch 3 .-> oracle

    policy -->|authentication/accounting lifecycle| session
    session -->|limited session and owner records| ldd
    ldd <--> messaging
    policy -->|operational events/syslog| messaging
    messaging -->|collection and survivable delivery| mnt
    mnt -->|Session Directory provider<br/>batched topic notifications| pxprovider

    classDef external fill:#eef6ff,stroke:#336699,color:#111;
    classDef edge fill:#fff3d6,stroke:#a06600,color:#111;
    classDef app fill:#eaf7ea,stroke:#3b7d3b,color:#111;
    classDef store fill:#f4ecff,stroke:#7248a3,color:#111;
    class admins,integrations,nads,pxclients,identity external;
    class kong edge;
    class admin,openapi,ers,policy,pxprovider,clientpool,messaging app;
    class oracle,ignite,session,ldd,mnt store;
```

This diagram deliberately shows `localhost:10800` as a node-local connection.
The Application Server does not normally cross the network to another node's
thin-client listener. Ignite itself forms the distributed Data Grid over its
separate inter-node ports.

The backend port labels are implementation observations from the rooted
control and Patch 3 files. Public clients normally enter through the supported
service URL rather than addressing those backend ports directly.

## How the three nodes cluster

```mermaid
flowchart TB
    subgraph clients[External consumers and producers]
        direction LR
        gui[GUI and Admin/API clients]
        aaa[RADIUS/TACACS network devices]
        pxc[pxGrid 2.0 clients]
    end

    subgraph deployment[ISE deployment]
        direction LR

        subgraph A[Node A — PPAN + PMnT + PSN]
            direction TB
            ak[Kong + Admin/API]
            ap[Protocols Engine]
            ao[(Local Oracle<br/>authoritative config writes)]
            ar[JGroups replication endpoint]
            am[ISE Messaging]
            al[(LDD/LSD cache)]
            ai[(Ignite member<br/>Data Grid)]
            amt[(Primary MnT<br/>operations/session directory)]
            ax[pxGrid service<br/>if assigned]
            ak --> ao
            ap --> ao
            ao --> ar
            ap --> al
            al <--> am
            am --> amt
            amt --> ax
        end

        subgraph B[Node B — SPAN + SMnT + PSN]
            direction TB
            bk[Kong + Admin/API]
            bp[Protocols Engine]
            bo[(Local Oracle<br/>replicated config copy)]
            br[JGroups replication endpoint]
            bm[ISE Messaging]
            bl[(LDD/LSD cache)]
            bi[(Ignite member<br/>Data Grid)]
            bmt[(Secondary MnT<br/>operations/session directory)]
            bx[pxGrid service<br/>if assigned]
            bk --> bo
            bp --> bo
            br --> bo
            bp --> bl
            bl <--> bm
            bm --> bmt
            bmt --> bx
        end

        subgraph C[Node C — PSN]
            direction TB
            cp[Protocols Engine]
            co[(Local Oracle<br/>replicated config copy)]
            cr[JGroups replication endpoint]
            cm[ISE Messaging]
            cl[(LDD/LSD cache)]
            ci[(Ignite member<br/>Data Grid)]
            cp --> co
            cr --> co
            cp --> cl
            cl <--> cm
        end
    end

    gui --> ak
    gui --> bk
    aaa --> ap
    aaa --> bp
    aaa --> cp
    pxc <-->|8910| ax
    pxc <-->|8910| bx

    ar -->|incremental configuration events<br/>JGroups TCP 12001| br
    ar -->|incremental configuration events<br/>JGroups TCP 12001| cr
    br -. missing-event request / status .-> ar
    cr -. missing-event request / status .-> ar

    am <-->|ISE Messaging TLS 8671<br/>queues/exchanges| bm
    am <-->|ISE Messaging TLS 8671<br/>queues/exchanges| cm
    bm <-->|ISE Messaging TLS 8671<br/>queues/exchanges| cm

    ai <-->|Ignite discovery 47500<br/>communication 47100| bi
    ai <-->|Ignite discovery 47500<br/>communication 47100| ci
    bi <-->|Ignite discovery 47500<br/>communication 47100| ci

    amt <-.->|MnT synchronization / failover| bmt
    ax <-.->|pxGrid inter-node 8910<br/>when both are assigned| bx
    cl -. fallback lookup after PSN connectivity loss .-> amt
    cl -. fallback lookup after PSN connectivity loss .-> bmt

    classDef client fill:#eef6ff,stroke:#336699,color:#111;
    classDef admin fill:#fff3d6,stroke:#a06600,color:#111;
    classDef policy fill:#eaf7ea,stroke:#3b7d3b,color:#111;
    classDef store fill:#f4ecff,stroke:#7248a3,color:#111;
    classDef fabric fill:#ffecec,stroke:#a33b3b,color:#111;
    class gui,aaa,pxc client;
    class ak,bk,amt,bmt,ax,bx admin;
    class ap,bp,cp,am,bm,cm policy;
    class ao,bo,co,al,bl,cl,ai,bi,ci store;
    class ar,br,cr fabric;
```

All three reported production nodes have 64 GiB RAM and are therefore expected
to be eligible Data Grid server members under the inspected Patch 3 logic. That
must still be confirmed from production service status and listeners.

## The cluster fabrics are not interchangeable

| Mechanism | Purpose | Scope and direction | Storage or state | Relevant ports |
|---|---|---|---|---|
| Local Oracle | Durable configuration, policy, deployment, identity-source and other application state on each node | PPAN accepts authoritative configuration changes; replicated copies are applied to other nodes | Local Oracle files/database on every node; it is not one shared VHDX/database | 1521 is an observed local/database listener; inter-node replication uses the mechanisms below |
| JGroups replication | Reliable transport for configuration and database change messages | Primarily PPAN to secondary nodes for incremental synchronization; receivers can request missed events | JGroups transports messages; it does not store the database | TCP 12001; HTTPS/SOAP 443 also participates in deployment operations |
| Full database sync | Rebuild a subscriber's configuration database after registration or loss of incremental consistency | PPAN exports/transfers a complete configuration snapshot to the target node | Database dump and rebuilt target-local Oracle state | Separate from ordinary JGroups incremental messages; exact transfer workflow is Cisco-controlled |
| ISE Messaging | Inter-node message queues/exchanges, including LDD and operational-log delivery | Full-mesh service relationships across ISE nodes | RabbitMQ-style queues with finite retained operational data | TLS TCP 8671; ISE internal communication TCP 15672 is also documented |
| LSD / LDD | Limited active-session and endpoint-owner lookup data used by PSNs, especially for CoA and ownership | Every PSN publishes/consumes limited records; not a full session or configuration database replica | Local RADIUS Session Directory and Endpoint Owner Directory caches; implementation logs identify `LSDRedisClient` | Uses ISE Messaging; profiler endpoint ownership synchronization TCP 6379 is separately documented |
| Ignite / Data Grid | Distributed, persistent cache and event/state service used by ISE applications | Eligible Data Grid members form a cluster; local Java clients normally enter through their own node | Replicated caches plus local Ignite persistence/WAL under `/opt/ignite/data` | 10800 client, 47500 discovery, 47100 inter-node communication |
| pxGrid 2.0 | Secure discovery, authentication, authorization, query, and publish/subscribe integration with external security products | External clients connect to configured pxGrid service nodes; ISE components such as MnT act as providers | Topic sequence/recovery state and data supplied by providers; it is not ISE configuration replication | TCP 8910 client and inter-node communication |
| MnT | Collection, storage, reporting, and Session Directory for operational data | PSNs/nodes send events toward PMnT/SMnT; MnT can provide session data to pxGrid and fallback data to LDD | Monitoring operational database, live/session data, reports | ISE Messaging/UDP paths depend on configuration; Data Connect is a separate reporting interface |
| Kong/API gateway | External HTTPS ingress and routing for GUI and API surfaces | Client-to-node, then node-local upstream to the appropriate Java service | Connection/worker state and logs; not a cluster database | Public 443; rooted/Patch 3 evidence maps local upstreams such as 9443, 9060, and 9070 |

## Two similarly named session systems

`LSD/LDD RADIUS Session Directory` and `pxGrid Session Directory` are related
to session context but are not the same service:

```mermaid
flowchart LR
    auth[Successful authentication on a PSN] --> local[PSN local session cache]
    local --> limited[LDD/LSD limited record<br/>owner, MAC/IP, PSN, state]
    limited <-->|ISE Messaging full mesh| peers[Other PSNs' local LDD caches]
    peers --> coa[Fast local owner lookup for CoA]

    auth --> event[Accounting and operational events]
    event --> mnt[MnT Session Directory<br/>richer deployment session view]
    mnt --> provider[ISE pxGrid Session Directory provider]
    provider -->|batched topic<br/>/topic/com.cisco.ise.session| subscriber[External pxGrid subscribers]

    peers -. connectivity-loss fallback .-> mnt
```

- LDD exists so a PSN can find limited session/owner information locally
  without repeatedly calling MnT.
- MnT holds the richer operational session view and publishes Session Directory
  data to external pxGrid subscribers.
- A failure in one path does not automatically prove a failure in the other,
  although ISE Messaging, node connectivity, and shared application load can
  create common-cause failures.

## Incident path overlaid on the architecture

The verified Patch 3 retry path implicated by the production logs is narrower
than the complete architecture:

```mermaid
flowchart LR
    request[GUI request] --> kong[Kong worker]
    kong --> thread[admin-http-pool worker]
    thread --> handler[Admin handler]
    handler --> pool[IgniteClientPool<br/>class-wide synchronized construction]
    pool -->|TLS localhost:10800| grid[Local Ignite/Data Grid listener]
    grid -->|normal| cache[distributed cached state]
    grid -. absent/refused .-> retry[10 wrapper attempts<br/>about 27 seconds for immediate refusals]
    retry --> pool
    pool -. retains request work .-> thread
    thread -. many concurrent waits .-> alert[Admin thread-pool threshold]
    alert -. upstream backlog/retry .-> kong
    kong -. worker/memory pressure .-> oom[possible Kong cgroup OOM]

    classDef normal fill:#eaf7ea,stroke:#3b7d3b,color:#111;
    classDef fault fill:#ffecec,stroke:#a33b3b,color:#111;
    class grid,cache normal;
    class retry,alert,oom fault;
```

This explains a plausible amplification mechanism; it does not yet establish
whether local Data Grid failure, inter-node topology loss, traffic, OOM, or
another shared dependency is the first production fault.

## What is verified and what remains qualified

### Verified or documented

- Cisco documents JGroups on TCP 12001 for configuration/data synchronization
  and explicitly states that JGroups transports replication messages rather
  than storing the replicated data.
- Cisco documents LDD as the newer name for LSD. It contains the RADIUS Session
  Directory and Endpoint Owner Directory and uses ISE Messaging for inter-node
  communication.
- Cisco documents Data Grid ports 10800, 47100, and 47500 as separate from
  JGroups and ISE Messaging.
- Patch 3 files identify Apache Ignite 2.16, local TLS client connections to
  `localhost:10800`, Oracle-backed address discovery, replicated caches, and
  local persistence/WAL.
- Cisco documents pxGrid 2.0 REST/WebSocket service on 8910. MnT batches Session
  Directory notifications for external subscribers.
- Rooted control and Patch 3 evidence map Kong to the node-local Java GUI/API
  services and identify `admin-http-pool` as the Admin connector executor.

### Deployment-dependent or still unproven

- Which production nodes have pxGrid service assigned and which is active.
- The exact production profile-derived Java, Kong, and Data Grid limits.
- Which specific caches and application operations in the affected production
  workload touch Data Grid at incident onset.
- Whether the production failure begins in Ignite, ISE Messaging/LDD, Oracle,
  Kong, external demand, or an infrastructure transient.
- The exact mechanism used for every MnT replication/failover flow. It is shown
  at the service level rather than mislabeled as JGroups, Ignite, or pxGrid.

## Sources

- [Cisco: Understand and Troubleshoot ISE Replications](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/226120-understand-and-troubleshoot-ise.html)
- [Cisco ISE 3.5 deployment guide: Light Data Distribution](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/admin_guide/b_ise_admin_3_5/b_ISE_admin_deployment.html)
- [Cisco ISE 3.5 ports used by all nodes](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/install_guide/b_ise_Installation_Guide_35/g_cisco_sns-3400_series_appliance_ports_reference-1/r_cisco_ise_all_nodes_ports.html)
- [Cisco ISE 3.5 pxGrid guide](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/admin_guide/b_ise_admin_3_5/b_ISE_admin_pxgrid.html)
- [Cisco pxGrid technical overview](https://developer.cisco.com/docs/pxgrid/technical-overview/)
- [Cisco: Troubleshoot ISE Session Management and Posture](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/215419-ise-session-management-and-posture.html)
- [`component-fault-model.md`](component-fault-model.md) for rooted/Patch 3 implementation evidence and the incident-specific fault model.
