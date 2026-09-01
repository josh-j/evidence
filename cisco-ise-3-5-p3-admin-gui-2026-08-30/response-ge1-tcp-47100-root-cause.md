The root cause is a Cisco ISE 3.5 multi-interface binding defect in the Data Grid’s Apache Ignite configuration.

ISE correctly pins Data Grid discovery on TCP 47500 to GE0’s management IP, but it does not pin Ignite’s communication service on TCP 47100. Consequently, Ignite advertises both GE0 and GE1 as valid 47100 endpoints.

### Failure mechanism

```text
ISE passes GE0 address as HOST_IP
             │
             ├─ TcpDiscoverySpi / 47500
             │      localAddress = HOST_IP       Correct
             │
             └─ TcpCommunicationSpi / 47100
                    localAddress = unspecified   Defect
                              │
                              ▼
                    Ignite binds 0.0.0.0
                              │
                              ▼
             Enumerates and advertises GE0 + GE1
                              │
                              ▼
       Peer may select GE1_IP:47100 before GE0_IP:47100
                              │
                              ▼
      GE1 does not carry permitted/reachable Data Grid traffic
                              │
                              ▼
                    TCP connection times out
```

Evidence:

- In the FCS configuration, `TcpCommunicationSpi` specifies port 47100 but no `localAddress`: [FCS ignite-config.xml](/tmp/ise35-static/ignite-files/opt/ignite/config/ignite-config.xml:37).
- `TcpDiscoverySpi`, immediately below it, is explicitly bound to `${HOST_IP}`: [FCS ignite-config.xml](/tmp/ise35-static/ignite-files/opt/ignite/config/ignite-config.xml:50).
- ISE derives `HOST_IP` from its configured management interface: [ignite-control.sh](/tmp/ise35-static/ignite-files/opt/CSCOcpm/bin/ignite-control.sh:98).
- The container uses host networking, so Ignite can see GE0 and GE1: [ignite-control.sh](/tmp/ise35-static/ignite-files/opt/CSCOcpm/bin/ignite-control.sh:585).

Apache Ignite 2.16 confirms the resulting behavior:

- An unspecified communication address defaults to any available local address.
- A wildcard address causes Ignite to enumerate all available network interfaces and publish their IPs.
- Peers retrieve this advertised address collection and attempt connections against it.
- GE0 and GE1 are not preferentially ordered; Ignite’s comparator only distinguishes loopback from non-loopback addresses. [Apache Ignite 2.16 communication configuration](https://github.com/apache/ignite/blob/2.16.0/modules/core/src/main/java/org/apache/ignite/spi/communication/tcp/internal/TcpCommunicationConfigInitializer.java), [address-resolution source](https://github.com/apache/ignite/blob/2.16.0/modules/core/src/main/java/org/apache/ignite/internal/util/IgniteUtils.java), [peer-address selection](https://github.com/apache/ignite/blob/2.16.0/modules/core/src/main/java/org/apache/ignite/spi/communication/tcp/internal/CommunicationTcpUtils.java).

Cisco’s own port table says Data Grid ports 47500, 47100, and 10800 apply to GE0/Bond0 and are “not applicable” on the other Ethernet interfaces. Therefore, advertising GE1 contradicts the intended ISE network model. [Cisco ISE 3.5 Installation Guide, page 120](https://www.cisco.com/c/en/us/td/docs/security/ise/3-5/install_guide/b_ise_Installation_Guide_35.pdf).

### Why it says “may fail”

The address ordering between non-loopback interfaces is not guaranteed. Depending on interface enumeration and connection state, a peer can try GE0 first and work, or try GE1 and spend a long time waiting.

ISE also configures unusually long communication retry values:

- Initial connection timeout: 180 seconds.
- Maximum timeout: 300 seconds.
- Reconnect count: 50.

That makes selecting an unusable GE1 address substantially more disruptive than a quick failed connection.

### Correct fix

The code-level correction is to bind communication to the same management address used for discovery:

```xml
<bean class="org.apache.ignite.spi.communication.tcp.TcpCommunicationSpi">
    <property name="localAddress" value="${HOST_IP}"/>
    <property name="localPort" value="47100"/>
</bean>
```

An equivalent correction could set Ignite’s global local-host property to `HOST_IP` or use an address resolver that advertises only GE0.

This should be delivered by Cisco as a patch or hotfix. Manually changing the appliance file would be unsupported and could be overwritten by application restart, Data Grid reset, or patch installation.

### Patch 3 warning

The Patch 3 configuration I inspected still lacks `localAddress` under `TcpCommunicationSpi`: [Patch 3 ignite-config.xml](/tmp/ise35-static/patch-files/ignite-config.xml:37). Its control script also does not set `IGNITE_LOCAL_HOST`.

Therefore, I would not assume Patch 3 fixes this defect merely because Cisco’s published condition lists 3.5 FCS. Ask TAC for:

- The Cisco defect ID corresponding to the exact symptom.
- Its current affected-release list.
- A hotfix that pins 47100 to GE0.
- The first fixed ISE build.

### Relationship to your production GUI incident

If production has GE1 configured and the logged destination is another ISE node’s GE1 address, this is a credible initiator of Data Grid instability. It can cause communication retries, topology changes, rebalance pressure, and stalled operations.

It does not directly explain a local `localhost:10800` refusal or Admin thread-pool exhaustion. Those would be downstream effects only if the 47100 failure destabilizes the local Data Grid service or causes application requests to wait indefinitely.

The decisive production evidence is:

```text
First GE1_IP:47100 timeout
        ↓
Ignite node-left/topology/rebalance errors
        ↓
local Data Grid degradation or 10800 failure
        ↓
admin-http-pool timeouts
        ↓
Administration thread-pool threshold
```

If that ordering appears, the GE1 defect is a strong root-cause candidate rather than incidental log noise.
