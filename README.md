# High-Availability SD-WAN Simulation

A hub-and-branch WAN lab built in Cisco Modeling Labs (CML) with IOSv routers. Each site connects to two separate providers: a private MPLS provider and a public broadband provider. One GRE tunnel runs over each provider, both tunnels are encrypted with IKEv2/IPsec, and an IP SLA probe decides which tunnel carries the traffic.

The Internet tunnel is the primary path and the MPLS tunnel is the backup. When the Internet provider fails, traffic moves to MPLS automatically, and it moves back when the provider recovers. Every claim in this README is backed by a screenshot.

> This is a manual SD-WAN-style design built with standard Cisco IOS features. It does not use Cisco Catalyst SD-WAN controllers.

## Contents

1. [How it works](#how-it-works)
2. [Topology and addressing](#topology-and-addressing)
3. [Design decisions](#design-decisions)
4. [Key configuration](#key-configuration)
5. [Evidence](#evidence)
6. [Results](#results)
7. [Limits and what was not tested](#limits-and-what-was-not-tested)

## How it works

**SD-WAN idea.** A company site often has more than one WAN link. SD-WAN logic checks the health of each link and sends traffic over the best one. This lab builds that behaviour by hand.

**GRE tunnels.** A GRE tunnel is a virtual point-to-point link between Hub-1 and Branch-1. Tunnel1 runs over the MPLS provider and Tunnel2 runs over the Internet provider. The providers only see the outer packets between the router addresses, never the private LANs.

**IPsec.** GRE alone does not encrypt anything. IKEv2 and IPsec (`tunnel protection`) encrypt both tunnels so the traffic is safe on the public Internet.

**Health check and failover.** Each router sends an ICMP probe through the Internet tunnel every 5 seconds (IP SLA). A track object turns the probe result into Up or Down. The route to the remote LAN through Tunnel2 only exists while the track is Up. A second route through Tunnel1 has a higher administrative distance (50), so it is the floating backup and only appears when the primary route is removed.

## Topology and addressing

![Topology](images/01-topology.png)

| Device | Interface | IP address | Connected to |
|---|---|---|---|
| Hub-1 | Gi0/0 | 10.10.10.10/24 | Prov-MPLS Gi0/0 |
| Hub-1 | Gi0/1 | 172.16.10.10/24 | Prov-INET Gi0/0 |
| Hub-1 | Loopback100 | 192.168.100.1/24 | HQ LAN |
| Hub-1 | Tunnel1 | 192.168.1.1/24 | Branch-1 over MPLS |
| Hub-1 | Tunnel2 | 192.168.2.1/24 | Branch-1 over Internet |
| Branch-1 | Gi0/0 | 172.16.20.10/24 | Prov-INET Gi0/1 |
| Branch-1 | Gi0/1 | 10.20.20.10/24 | Prov-MPLS Gi0/1 |
| Branch-1 | Loopback200 | 192.168.200.1/24 | Branch LAN |
| Branch-1 | Tunnel1 | 192.168.1.2/24 | Hub-1 over MPLS |
| Branch-1 | Tunnel2 | 192.168.2.2/24 | Hub-1 over Internet |
| Prov-MPLS | Gi0/0, Gi0/1 | 10.10.10.1/24, 10.20.20.1/24 | Hub-1, Branch-1 |
| Prov-INET | Gi0/0, Gi0/1 | 172.16.10.1/24, 172.16.20.1/24 | Hub-1, Branch-1 |

Node type: IOSv. The two loopbacks stand in for the HQ and branch LANs.

## Design decisions

- **No default routes anywhere.** Each router has one specific static route per remote transport subnet, so each tunnel destination is reached through its own provider.
- **Providers carry only their own subnets.** They hold connected routes and nothing else, so they cannot reach the private LANs.
- **Each tunnel is tied to one transport.** The tunnel source is a specific interface, so Tunnel1 always uses MPLS and Tunnel2 always uses the Internet.
- **Both tunnels are encrypted.** IKEv2 with AES-256, SHA-256 and DH group 14, using a pre-shared key.
- **Primary and backup by administrative distance.** The tracked Internet route keeps the default distance of 1. The MPLS backup route has a distance of 50.
- **Static routes use a next-hop IP** (the far end of the tunnel) instead of only an exit interface.
- **The probe runs inside the tunnel.** It tests the real overlay path (192.168.2.1 from 192.168.2.2), not just the local interface.
- **Both sites fail over.** Hub-1 and Branch-1 each track their own probe, so traffic in both directions uses the same path.
- **SLA timers:** frequency 5 seconds, timeout 1000 ms, threshold 1000 ms.

## Key configuration

Branch-1 is shown. Hub-1 mirrors it with its own addresses. The pre-shared key is not published.

```text
interface Tunnel2
 description GRE over Internet to Hub-1
 ip address 192.168.2.2 255.255.255.0
 tunnel source GigabitEthernet0/0
 tunnel destination 172.16.10.10
 tunnel mode gre ip
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel protection ipsec profile IPSEC-PROF
```

```text
crypto ikev2 proposal IKE-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
crypto ikev2 policy IKE-POL
 proposal IKE-PROP
crypto ikev2 keyring KEYS
 peer HUB-INET
  address 172.16.10.10
  pre-shared-key <hidden>
crypto ikev2 profile IKE-PROF
 match identity remote address 172.16.10.10 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS
crypto ipsec transform-set TS esp-aes 256 esp-sha256-hmac
 mode transport
crypto ipsec profile IPSEC-PROF
 set transform-set TS
 set ikev2-profile IKE-PROF
```

```text
ip sla 1
 icmp-echo 192.168.2.1 source-ip 192.168.2.2
 threshold 1000
 timeout 1000
 frequency 5
ip sla schedule 1 life forever start-time now
track 10 ip sla 1 reachability

ip route 192.168.100.0 255.255.255.0 192.168.2.1 track 10
ip route 192.168.100.0 255.255.255.0 192.168.1.1 50
```

## Evidence

### 1. Underlay: addressing and routing

Every interface is up with the expected address, and each router has only the static routes it needs.

![Hub-1 interfaces and routes](images/02-hub1-interfaces-routes.png)

*Hub-1 has all interfaces up/up, and static routes to the remote transport subnets and the Branch LAN with no gateway of last resort.*

![Branch-1 interfaces and routes](images/03-branch1-interfaces-routes.png)

*Branch-1 mirrors Hub-1, with its own routes to the Hub transport subnets and the Hub LAN.*

![Prov-MPLS interfaces and routes](images/04-prov-mpls-interfaces-routes.png)

*Prov-MPLS has only connected and local routes for the two MPLS subnets.*

![Prov-INET interfaces and routes](images/05-prov-inet-interfaces-routes.png)

*Prov-INET has only connected and local routes for the two Internet subnets.*

### 2. Underlay: reachability and SSH

![Underlay pings and SSH version](images/06-underlay-ping-ssh-version.png)

*SSH version 2 is enabled on Hub-1, and sourced pings to Branch-1 succeed 100 percent over both the MPLS and Internet subnets.*

![SSH login across the underlay](images/07-underlay-ssh-login.png)

*Hub-1 logs in to Branch-1 over SSH across the MPLS provider and lands on the Branch-1 prompt.*

### 3. GRE tunnels

![Hub-1 GRE tunnels](images/08-hub1-gre-tunnels.png)

*Both Hub-1 tunnels are up/up with the correct source interface and destination, and both tunnel peers answer pings.*

![Branch-1 GRE tunnels](images/09-branch1-gre-tunnels.png)

*Both Branch-1 tunnels are up/up with the mirrored source and destination, and both tunnel peers answer pings.*

### 4. IKEv2 and IPsec

![Hub-1 IKEv2 sessions](images/10-hub1-ikev2-sa.png)

*Hub-1 has two IKEv2 sessions in READY state, one per provider, using AES-CBC-256, SHA-256 and DH group 14.*

![Branch-1 IKEv2 sessions](images/11-branch1-ikev2-sa.png)

*Branch-1 shows the matching two sessions in READY state.*

![IPsec packet counters](images/12-branch1-ipsec-counters.png)

*After 100 pings between the LANs, only the Tunnel2 encrypt and decrypt counters grew, while Tunnel1 stayed at 45, so the traffic used the encrypted Internet tunnel.*

### 5. IP SLA and normal-state routing

![IP SLA configuration and statistics](images/13-branch1-ipsla-config-stats.png)

*The IP SLA probe targets the Hub's tunnel address, returns OK, and shows 298 successes and 0 failures.*

![Branch-1 baseline track and routes](images/14-branch1-baseline-track-routes.png)

*In the normal state the track is Up and the route to the Hub LAN goes through Tunnel2 with distance 1.*

![Branch-1 baseline traceroute](images/15-branch1-baseline-traceroute.png)

*The traceroute from the Branch LAN to the Hub LAN has a single hop, 192.168.2.1, which is the Internet tunnel.*

![Hub-1 baseline track and routes](images/16-hub1-baseline-track-routes.png)

*Hub-1 also has its track Up and its route to the Branch LAN through Tunnel2 with distance 1.*

![Hub-1 baseline traceroute](images/17-hub1-baseline-traceroute.png)

*The traceroute from the Hub LAN to the Branch LAN has a single hop, 192.168.2.2, which is the Internet tunnel.*

### 6. Negative test: providers cannot see the private LANs

![Prov-MPLS cannot reach the LANs](images/18-prov-mpls-lan-ping-fails.png)

*Pings from Prov-MPLS to both private LANs fail 0 out of 5, as expected, because the providers have no route to them.*

![Prov-INET cannot reach the LANs](images/19-prov-inet-lan-ping-fails.png)

*Pings from Prov-INET to both private LANs also fail 0 out of 5, as expected.*

### 7. Failover test: Internet provider goes down

![Prov-INET shutdown](images/20-prov-inet-shutdown.png)

*The Internet provider outage is simulated by shutting Prov-INET Gi0/1 at 15:11:29.*

![Branch-1 track and SLA during the outage](images/21-branch1-failover-track-sla.png)

*At 15:11:34 track 10 goes from Up to Down and the SLA returns Timeout, yet Tunnel2 still shows up/up, so only the IP SLA detects the outage.*

![Branch-1 routes during the outage](images/22-branch1-failover-routes-traceroute.png)

*The Branch route to the Hub LAN becomes the backup route through 192.168.1.1 with distance 50, the ping still succeeds, and the traceroute shows the MPLS tunnel hop.*

![Hub-1 track and routes during the outage](images/23-hub1-failover-track-routes.png)

*Hub-1 also sees its track go Down and switches its route to the Branch LAN to the MPLS tunnel (192.168.1.2, distance 50).*

![Hub-1 traceroute and ping during the outage](images/24-hub1-failover-traceroute-ping.png)

*From the Hub LAN the traceroute shows the MPLS tunnel hop, 192.168.1.2, and the ping to the Branch LAN succeeds.*

### 8. Failback test: provider recovers

![Prov-INET restored](images/25-prov-inet-restore.png)

*The Internet provider interface is brought back up at 15:23:20.*

![Branch-1 failback log and track](images/26-branch1-failback-track-log.png)

*The Branch log shows the full timeline, Up to Down at 15:11:34 and Down to Up at 15:23:19, and the track is Up again.*

![Branch-1 routes after recovery](images/27-branch1-failback-routes-traceroute.png)

*The Branch route returns to the Internet tunnel with distance 1, and the traceroute shows 192.168.2.1 again.*

![Hub-1 track and traceroute after recovery](images/28-hub1-failback-track-traceroute.png)

*Hub-1 logs Down to Up at 15:23:20, its track is Up, and the traceroute shows 192.168.2.2 again.*

![Hub-1 routes after recovery](images/29-hub1-failback-routes.png)

*Hub-1's route to the Branch LAN is back on the Internet tunnel with distance 1.*

## Results

| # | Test | Result | Evidence |
|---|---|---|---|
| 1 | Underlay reachability over both transports | Pass, 100 percent | 06 |
| 2 | SSH version 2 login across the underlay | Pass | 06, 07 |
| 3 | GRE tunnels up on both routers | Pass, 4 of 4 up/up | 08, 09 |
| 4 | IKEv2 sessions established on both tunnels | Pass, READY | 10, 11 |
| 5 | Traffic is encrypted on the active tunnel | Pass, Tunnel2 counters grew | 12 |
| 6 | IP SLA healthy in the normal state | Pass, 0 failures | 13 |
| 7 | Primary path is the Internet tunnel | Pass, both directions | 14 to 17 |
| 8 | Providers cannot reach the private LANs | Pass, pings failed as expected | 18, 19 |
| 9 | Failover to MPLS when the Internet provider fails | Pass, track Down about 5 seconds after the shutdown | 20 to 24 |
| 10 | Tunnel interface alone does not detect the outage | Observed, Tunnel2 stayed up/up | 21 |
| 11 | Failback to the Internet tunnel after recovery | Pass, both directions | 25 to 29 |

The router clocks are not synchronised, so the timings are approximate.

## Limits and what was not tested

- The probe tests reachability only (return code OK or Timeout). It does not measure delay, jitter or packet loss.
- Only a failure of the Internet provider was tested. The case where both providers fail at the same time was not tested.
- The lab uses IOSv in CML with simulated providers and loopback LANs, not real hosts or a real carrier network.
